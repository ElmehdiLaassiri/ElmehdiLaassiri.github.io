---
title: " Home Lab 1 : Minimal Active Directory Setup on Debian "
date: 2026-07-10 00:00:00 +0000
categories: [Home Lab ]
tags: [Active Directory, Homelab]
---



## Overview : 

This is a minimal Active Directory setup , it will contain 1 Domain Controller , 1 Certificate Authority (CA) , 2 Windows Client machines , and our attacker machine of course , the idea is pretty simple , set up our own environment to test multiple Active Directory attacks later on , but most importantly understand why they exist in the first place , and how we can mitigate them .

For the attacks , i will divide them to multiple blogs , but to give you an idea of the attacks that we will be checking in depth : 

- NTLM Relay Attacks
- Unconstrained , Constrained , RBCD Attacks
- ACL attacks
- Kerberos Attacks
- ADCS attacks and much more

This Blog will be divided into multiple sections , first i will start by explaining the setup , then i will explain briefly how virtualization works on Linux , since we will be using a Debian machine as our Host and not Windows this time , then i will walk you through installing each VM, then once every machine is set we will start configuring our domain .  


## Quick Explanation :

Before we start, let's define few things first, you've probably used Vmware workstation on Windows to manage your virtual machines, well on Debian there is no such single app. Instead, Linux splits the same job across a small stack of tools : QEMU, KVM, libvirt, and virt-manager which we'll walk through now.

### Windows Equivalent : 

If you're coming from Windows, VirtualBox or VMware Workstation bundle all four of these jobs into one installer. 
 
The Linux stack splits them out: 

 - QEMU is the hidden emulation engine
 - KVM is the equivalent of the vboxdrv / vmmon kernel driver
 - libvirt is the VM registry + VBoxManage CLI combined
 - Virt-manager is the Manager window. 

Same work, split into four replaceable pieces which is why on Linux you can swap the GUI for virsh, or swap QEMU for Xen under the same libvirt.

### Kernel space vs userspace: 

The kernel is the privileged core of the OS (drivers, memory management, scheduling). Userspace is everything else, your programs, QEMU included. A program in userspace can only touch the hardware by asking the kernel to do it for them.

### Quemu : 

This is our emulator , specifically a userspace machine emulator, When a VM is running on your Linux, there's a qemu-system-x86_64 process on the host, that process is the VM. It emulates the virtual hardware: chipset, PCI bus, firmware, disks, NICs, display. 

On its own it emulates the CPU in software too which makes it slow, it's slow because QEMU emulates every piece of the virtual PC in software, including the CPU. Faking a disk or a NIC in software is lowkey cheap, but faking a CPU is brutal, since every guest instruction has to be translated on the fly. KVM fixes this by handing CPU execution over to the real host CPU through hardware virtualization extensions, so guest code just runs natively instead of being emulated.

### KVM : 

A Linux kernel module (kvm.ko + kvm-intel.ko / kvm-amd.ko) that exposes CPU virtualization extensions (Intel VT-x, AMD-V) to userspace via /dev/kvm. (it's okay if you don't remember these modules name, i don't either but it's good to have heard of them)

QEMU opens that device and uses it to run guest instructions directly on the real CPU instead of translating them. KVM only covers CPU and memory virtualization, devices are not its problem.

This is also exactly why QEMU is still needed, KVM makes the CPU and memory fast, but it doesn't know what a disk, a NIC, or a BIOS is. Those stay QEMU's job. That's the division of labor: QEMU fakes the hardware, KVM makes the CPU part fast.

### Libvirt : 

A management daemon (libvirtd) that acts as a wrapper around QEMU it doesn't run the VMs itself, it tells QEMU how to run them. It stays running in the background all the time, waiting for you to tell it what to do.

The way it works: when you create a new VM (through virt-manager, or by hand with virsh), libvirt writes a configuration file describing it in XML things like how many vCPUs, how much RAM, which disk, which network, etc.

That file lives at /etc/libvirt/qemu/<vm-name>.xml, and it stays there permanently. Think of it as the equivalent of VMware's .vmx file or VirtualBox's .vbox file the main difference is libvirt separates the config (in /etc/libvirt/qemu/) from the actual disk image (in /var/lib/libvirt/images/), while VMware and VirtualBox bundle them into one folder per VM.

### Virt-Manager :

This is our GUI client that talks to libvirtd over its API. Everything it does maps to a virsh command or an XML edit, no exclusive magic. If it crashes mid-session, our VMs will keep running, because the VM is a QEMU process libvirtd started, independent of the UI.

This is what it would look like : 

<img width="906" height="470" alt="image" src="https://github.com/user-attachments/assets/8fc39846-4d62-4063-8da6-21442976c52f" />

Virt-manager (the window) --> QEMU/KVM (what we're using to run VMs) --> the VMs themselves (here, linux2024, currently running).

### Summary :

QEMU emulates the hardware, KVM accelerates the CPU, libvirt manages the lifecycle, virt-manager is the GUI.



## Lab Setup : 

### Setting up the Virtualization : 

#### Quick checks : 

Before we start we need to check if our CPU actually supports virtualization : 

```bash
grep -E -o 'vmx|svm' /proc/cpuinfo | head -1
```

<img width="544" height="75" alt="image" src="https://github.com/user-attachments/assets/308a7651-813e-4df8-acd9-7663b5d2fce0" />

It should either return svm (AMD-V) or vmx (Intel VT-x)  


If the result is empty it means it's not enabled so you probably need to enable it from the BIOS . 

Reboot into BIOS/UEFI --> enable "Intel Virtualization Technology" Or "SVM Mode" --> save --> boot back up --> re-check

#### Install the Stack :

```bash
sudo apt update
sudo apt install -y \
  qemu-system-x86 \
  qemu-utils \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  virt-manager \
  ovmf \
  swtpm
``` 

Quick exp :
- qemu-system-x86 + qemu-utils --> QEMU itself + helper tools (like qemu-img)
- libvirt-daemon-system + libvirt-clients --> the libvirtd daemon + virsh CLI
- bridge-utils --> Linux bridge tooling (needed for the virtual networks)
- virt-manager --> the GUI
- ovmf --> UEFI firmware images for guests (needed for modern Windows installs)
- swtpm --> software TPM 2.0 emulation (needed for Windows 11)


#### Verifying : 

```bash
systemctl status libvirtd
``` 

<img width="1114" height="406" alt="image" src="https://github.com/user-attachments/assets/81928137-e5c0-47cf-8a38-466c14bc643a" />

On Debian it auto-starts and auto-enables after install, so no manual systemctl enable needed. If it's not running you can enable it :

```bash
sudo systemctl enable --now libvirtd.
```

#### Groups : 

Now we need to add ourselves to the libvrt and kvm group, otherwise we can't really run virsh commands without sudo .

```bash
sudo usermod -aG libvirt,kvm $USER
```

These 2 groups are created by default when you set up libvirt and KVM .

<img width="1000" height="226" alt="image" src="https://github.com/user-attachments/assets/7daf5443-f638-4315-8af5-63eed3338c77" />

Once you add yourself to these groups , either log out then login back to apply the changes , or just run this : 

```bash
newgrp libvirt 
```

which will apply the changes to your current session without needing to log out . 


#### Checking the entire stack : 

Before touching anything else, let's verify each layer of the stack is actually working. We'll go bottom-up: kernel --> QEMU --> libvirt --> GUI.

**1/ KVM kernel module + device**

```bash
ls -l /dev/kvm
```

This checks that the KVM kernel module is loaded and exposed as a device.

**2/ QEMU is installed**

```bash
qemu-system-x86_64 --version
```

This confirms the QEMU binary exists and runs. You should get the version string back. 


**3/ libvirtd is running AND we can talk to it**

```bash
virsh list --all
```

This will check both , whether Libvirtd is running and whether our user has enough permissions to talk to it . 

**4/ libvirt's network subsystem is alive**

```bash
virsh net-list --all
```

Lists virtual networks libvirt knows about. On your install it may be empty, or it may show a default network both are fine. We'll create our own network shortly

**5/ virt-manager (the GUI) is installed**

```bash
virt-manager --help | head -1
```

Simple presence check. Should print the usage line.

<img width="748" height="338" alt="image" src="https://github.com/user-attachments/assets/9c14fdc8-d295-4f7b-accb-1aa0e4e945ba" />

If all five checks come back clean, the stack is installed, running, and reachable from your user without sudo. Then we can move to building our first Network . 


### Setting Up the Network : 

**1/ What we're building and why**

Our lab needs its own private network a bridge where the DC, client VMs, and attacker all live, completely separated from your real LAN and the internet. The reasons:
- **Safety:**  We're going to deliberately break things on the DC (weak passwords, missing LDAP signing, etc.). We don't want the DC itself to send ANY traffic outward not to your home router, not to Microsoft Update, not to the internet. If any of our test attacks plants a C2 beacon, we want it stuck inside the lab, not phoning home. If a service leaks DNS, we don't want it leaking lab metadata to our ISP. Isolated means the DC has zero route to the outside. Zero accidents.
- **Realism:** A real AD environment is a subnet of its own. Isolating ours mirrors reality.
- **Reproducibility:** Everyone following along uses the same subnet and names, so IPs in later attack commands always match.

Now for the Specs : 

- Name: lab-net
- Bridge interface on the host: virbr-lab
- Subnet: 10.10.10.0/24
- Gateway (the host's IP on the bridge): 10.10.10.1
- DHCP range: 10.10.10.100 – 10.10.10.200 (so we can hand out IPs to anything, but still statically assign 10.10.10.10 to the DC, .20, .30 etc. to other VMs)


**2/ Isolated vs NAT**

**NAT:** VMs can reach the outside world (internet, your host's LAN) via NAT on your Debian.
- Useful if you need Windows Update or want to download tools inside the VMs.
- Important: this does not mean your VMs are reachable from the internet. External machines can't connect to 10.10.10.x libvirt doesn't set up any port forwarding by default.
- The actual downside: the VMs can reach OUT to anywhere your host can reach. For a deliberately broken DC, that means malware or weird services on the DC could phone home.

**Isolated:** VMs can ONLY talk to each other and the host. No internet, no outbound.
- Downside: no Windows Update, so bring any installers in via shared folders or snapshots.
- Upside: nothing leaks outward, and (same as NAT) nothing from outside can reach in either.

One honest note worth adding

In both modes, the host itself always has direct access to the VMs through the bridge because the host sits on virbr-lab as 10.10.10.1. That's not a NAT thing, it's a bridge thing. So stuff running on my host (like Exegol in --network host mode) can always reach the VMs regardless of which mode you pick. The isolated/NAT choice only controls what the VMs can reach outbound.


**3/ Create the network via CLI (XML) path**

Now with libvirt, we can either describe the network in an XML file and hand it over, or click through virt-manager's GUI. Both end up in the same place, libvirt reads whatever you give it, then stores its own canonical copy at **/etc/libvirt/qemu/networks/lab-net.xml**. From that point on, that's the file libvirt actually uses . 

So now let's start by creating the file , we'll name it **lab-net.xml**

```XML
<network>
  <name>lab-net</name>
  <bridge name='virbr-lab' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
```

The main thing here is that we have no <forward/> tag , this is what makes this isolated , if you wanted NAT you could add **<forward mode='nat'/>** just under the <name> line and libvirt would set up NAT rules automatically.

<img width="739" height="306" alt="image" src="https://github.com/user-attachments/assets/f0238cac-ae27-47e9-9a30-abf6cc4e5833" />

Then we just tell lib-virt to use it first we register the network with lib-virt :

```bash
virsh net-define ~/Lab/lab-net.xml
```

Then to make life easier, we'll make it so that it comes up at every boot : 

```bash
virsh net-autostart lab-net
```

And finally we can start it : 

```bash
virsh net-start lab-net
```
<img width="862" height="216" alt="image" src="https://github.com/user-attachments/assets/4bd4f6dc-932d-48ff-af06-cf2c5b8b8dff" />

If it gives you a permission error like the one i got, this is normal since even if we are a member of the libvirt group , when we run vish commands it defaults to **quemu:///session** a per user libvirt instance that runs under YOUR user, not root.

So our user can : 
- Define networks 
- Set autostart 
- NOT create actual Linux bridge interfaces (since it will need root on the host networking stack)

Now to fix this, we can switch the default to System mode and make it persistent : 

```bash
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.bashrc
source ~/.bashrc
```

Now to verify :

```bash
virsh uri
```

It should now say qemu:///system 

<img width="847" height="233" alt="image" src="https://github.com/user-attachments/assets/85056a04-da8b-4934-b1aa-82d89c1b974a" />

Perfect, now we can rerun the commands to start our new Network Interface. 

If you want you can destroy the old one just to avoid any issues : 

```bash
virsh net-destroy lab-net 2>/dev/null
virsh net-undefine lab-net
```

Then redo everything : 

```bash
virsh net-define ~/Lab/lab-net.xml
virsh net-autostart lab-net
virsh net-start lab-net
```

<img width="876" height="380" alt="image" src="https://github.com/user-attachments/assets/d9d90a4f-090c-4ccc-9ab3-43914bf180e6" />

Now our libvirt virtual network is set . 

**4/ Create the network via GUI path (virt-manager)**

If you want to add it from the GUI , just open Virt-Manager and go to :  

Edit --> Connection Details --> Virtual Networks tab --> + button.

```text
name lab-net
IPv4 10.10.10.0/24
DHCP range 10.10.10.100 10.10.10.200
```


<img width="1220" height="1092" alt="image" src="https://github.com/user-attachments/assets/5d77bd19-a5ba-4024-baa4-3cdd8fd77416" />


**5/ Verifying :**

Now if we check Virt Manager : 

<img width="1199" height="748" alt="image" src="https://github.com/user-attachments/assets/c2ccf09c-1b41-4cda-b8d9-600672acb402" />

We can see our New virtual network created , we can also verify using virsh : 

```bash
virsh net-list --all
``` 

You should see the lab-net network , by default our host will also get an IP address on the bridge (we alread set it up in the XML file , the 10.10.10.1) 

```bash
ip addr show virbr-lab
```

<img width="1124" height="315" alt="image" src="https://github.com/user-attachments/assets/71b80fa0-b9cb-40df-be19-f9fd00e198e4" />

We can also try pinging our IP on the bridge : 

<img width="698" height="234" alt="image" src="https://github.com/user-attachments/assets/fd8f4fc2-9226-4c3e-b0ba-32b435cae95e" />

Perfect now the Networking section is pretty much done. 


### ISO Installation : 

Now for this section , we will need these :  

┌──────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────┬─────────────────────────────┬──────────────┐
│                     What                     │                                 Why                                 │           Source            │ Trial length │
├──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼─────────────────────────────┼──────────────┤
│ Windows Server 2022 (or 2019)                │ DC + CA role                                                        │ Microsoft Evaluation Center │ 180 days     │
├──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼─────────────────────────────┼──────────────┤
│ Windows 11 Enterprise                        |  The 2 client VMs                                                    │ Microsoft Evaluation Center │ 90 days      │
├──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼─────────────────────────────┼──────────────┤
│ virtio-win ISO                               │ Windows drivers for virtio disks/NICs — makes VMs noticeably faster │ Fedora People repo          │ —            │
└──────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────┴─────────────────────────────┴──────────────┘

We will have 1 Windows 2022 server , 1 Windows 2019 server , 1 Windows 11 and 1 Windowd 10 : 

**Windows Server:**

We can get a windows server 2022 directly from microsoft eval center : 

```bash
https://www.microsoft.com/fr-fr/evalcenter/evaluate-windows-server-2019
https://www.microsoft.com/fr-fr/evalcenter/evaluate-windows-server-2022
```

<img width="1705" height="620" alt="image" src="https://github.com/user-attachments/assets/8c45f81d-fdcc-4122-a01e-bb87b70cc82c" />

Make sure u get the ISO file not the VHS : 

They will ask you to fill in a form : 

<img width="1412" height="1002" alt="image" src="https://github.com/user-attachments/assets/1330b681-f6e5-463e-ad0a-40b1ada3b2c6" />

Once done you just choose the version you need  i chose the french one : 

<img width="1774" height="849" alt="image" src="https://github.com/user-attachments/assets/3fe28312-d4ce-43d4-b656-c2ef4d15bb96" />

**Windows Enterprise :**

Same thing as the Windows server , we can get it directly from Eval center : 

```bash
https://www.microsoft.com/fr-fr/evalcenter/download-windows-11-enterprise
``` 

**Grab the virtio-win ISO**

Virtio is used by QEMU to make things waaay faster. Instead of emulating real hardware (like a SATA disk or an Intel NIC which works, but is slow because QEMU has to simulate every tiny thing that real hardware does), QEMU can expose virtio devices a fake kind of device that only exists inside VMs, designed from scratch to be fast. 

The catch: since virtio doesn't exist in real PCs, Windows doesn't ship with drivers for it. That's why we need to load them manually during install.

I don't fully get it tbf , but what you should get is Windows doesn't ship with this driver , and for that we need to download it separately and we need this so that it is faster :)

```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

Now we should have everything that we need : 

<img width="1535" height="380" alt="image" src="https://github.com/user-attachments/assets/4a327532-38c5-4624-aea0-9fa18eed5e89" />


### VM Configuration

#### DC01 : 

This will be the Windows 2022 Server.

These are the specs :
- Hostname: DC01
- OS: Windows Server 2022 Standard (Desktop Experience)
- RAM: 4 GB
- vCPU: 2
- Disk: 60 GB (qcow2, thin-provisioned so it only uses what's written)
- Firmware: UEFI (via OVMF)
- Network: lab-net, static IP 10.10.10.10

These will be the steps for all the other machines as well : 

First create a New virtual Machine : 

<img width="777" height="634" alt="image" src="https://github.com/user-attachments/assets/8576902c-0c20-49aa-a53e-df917f8a4f5b" />

Click on Browse then Browse Local : 

<img width="865" height="424" alt="image" src="https://github.com/user-attachments/assets/d81343fa-a1c8-4e44-b37c-cea9461539ca" />

Choose our 2022 ISO image , change the OS to Unknown and hit forward :

<img width="688" height="389" alt="image" src="https://github.com/user-attachments/assets/b3205326-d70f-4e59-9879-25ba7b564a6a" />

Give it the RAM and the CPU specified in the specs .

<img width="607" height="411" alt="image" src="https://github.com/user-attachments/assets/e0b23b1e-3f8d-46ec-ab2b-aa96aef34b40" />

For the Storage 60 Gb is enough .

The name will be DC01 , Network Selection Select Lab-net and make sure you check the Customize configuration before install and finally hit Finish. 

<img width="819" height="522" alt="image" src="https://github.com/user-attachments/assets/7a71ae9d-4f2c-4cb5-90c1-b00fed165b15" />

Now once you hit Finish , we should still configure some few things : 

**Overview tab**
- Change Firmware from "BIOS" to UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.fd. (Required for Windows 11 later, also good practice for Server.)
- Chipset: leave as Q35.

<img width="946" height="564" alt="image" src="https://github.com/user-attachments/assets/8669130a-6773-4f2a-a01d-b3af9ed341bf" />

**CPU tab**

Expand Topology --> check Manually set CPU topology --> 1 socket, 2 cores, 1 thread. Not strictly necessary but makes the VM look cleaner to Windows.

<img width="700" height="522" alt="image" src="https://github.com/user-attachments/assets/271c2f1d-8271-4d09-a8e8-0ab5294095a5" />

**SATA Disk 1 tab:**

- If you want virtio for speed: change Disk bus from SATA to VirtIO. (You'll need to load the driver during install , remember the section where we installed the virtio-win.iso)
- If you want zero hassle: leave it as SATA. Slower but works out of the box.

<img width="829" height="582" alt="image" src="https://github.com/user-attachments/assets/3bdab428-0427-460f-bb44-75637c2023f7" />

**NIC tab:**

- Change Device model from e1000e (or rtl8139) to virtio. Faster (remember Virtio means faster hh) 
- MAC address: i will let virt-manager auto-generate it. 

<img width="773" height="580" alt="image" src="https://github.com/user-attachments/assets/24d97cee-a5d0-4e8b-95f7-f2d1b073a2f7" />

**SATA CDROM:**

- Need to change the Disk Bus from IDE to SATA otherwise we will get an error .

<img width="1279" height="624" alt="image" src="https://github.com/user-attachments/assets/afbce25c-161f-420e-8d11-eb379c56a734" />

**Add Hardware --> Storage (for the virtio-win CD):**

- Click Add Hardware --> Storage.
- Select or create custom storage --> Browse --> pick ~/Lab/ISOs/virtio-win.iso.
- Device type: CDROM device. Bus: SATA.
- Click Finish. We now have two CD-ROMs attached to the VM: the Windows installer + the virtio drivers.

<img width="1330" height="849" alt="image" src="https://github.com/user-attachments/assets/15b269b5-d64a-4ca7-96a7-e0390494cb8f" />

Finally we just Click **Begin installation**

<img width="1173" height="551" alt="image" src="https://github.com/user-attachments/assets/b19fe823-e228-4c56-9242-344be7c366c4" />

Firs we're dropped into this shell cause it couldn't find anything bootable to auto boot from . 

Just hit exit and we should access the UEFI setup menu , from there : 

- 1/ Go to Boot Manager (or Boot Maintenance Manager --> Boot Options)

<img width="1089" height="759" alt="image" src="https://github.com/user-attachments/assets/afc5f422-09ff-40aa-b17e-be412c95b2c3" />

- 2/You should see entries for your CD-ROMs (likely labeled something like UEFI QEMU DVD-ROM or similar)

<img width="1078" height="718" alt="image" src="https://github.com/user-attachments/assets/c38a2e13-7914-4906-914f-5ce953edbecc" />

I pressed 0001 but didn't boot up so i changed it to 0003 either way one of them is the windows installer , the other one is the virtio disk

You might not even get this error , if not just move on to the next section . 

**Installation Guide :**

<img width="965" height="577" alt="image" src="https://github.com/user-attachments/assets/36d9e886-dde0-49db-8e6e-25d0c93572cc" />

From here the installation is pretty straight forward , just press Install Now : 

<img width="1147" height="741" alt="image" src="https://github.com/user-attachments/assets/2507410d-ca7c-4205-a3ec-6cc34038c8bd" />

Select the Standard Evaluation with Desktop experience , accept few things : 

<img width="1018" height="548" alt="image" src="https://github.com/user-attachments/assets/30bda89c-993b-4c6b-8bce-61f4d613ee30" />

I didn't choose the custom installation . 

<img width="913" height="531" alt="image" src="https://github.com/user-attachments/assets/fb79db2d-8c26-4960-9770-470fdf0a08ce" />

Now this is the part where we can't find the disk , this is where we specify the virtio driver we installed earlier . 

<img width="813" height="544" alt="image" src="https://github.com/user-attachments/assets/7251ea12-b426-4ebc-9833-0b8e4db9a736" />

You will see a bunch of virtio drivers , each compiled for a specific version , i did check the 2K22 one , although some say that any of these versions will work . 

<img width="880" height="546" alt="image" src="https://github.com/user-attachments/assets/74e99f05-e069-4c81-825b-6945c39a93b6" />

Now finally we get this disk space that we can use , hit next and wait for the Installation . 

Once done , it will Reboot and ask you for a password : 

<img width="1066" height="629" alt="image" src="https://github.com/user-attachments/assets/c7b24f34-14d7-4b4a-8370-4e20825c0ef7" />

Now once we get the Login screen, we'll stop .

<img width="1246" height="764" alt="image" src="https://github.com/user-attachments/assets/440fbe26-385d-4fdf-9b9f-9f223fcca3f8" />


#### CA01 : 

For the CA01 machine , we'll use the Windows Server 2019 ISO that we downloaded , same steps as the DC01 , just different ISO , so you should be good :) 

These are the specs :
- Hostname: CA01
- OS: Windows Server 2022 Standard (Desktop Experience)
- RAM: 4 GB
- vCPU: 2
- Disk: 60 GB (qcow2, thin-provisioned so it only uses what's written)
- Firmware: UEFI (via OVMF)
- Network: lab-net, static IP 10.10.10.10

**Add Hardware --> Storage (for the virtio-win CD):**

You should get a Warning during this section 

<img width="1246" height="855" alt="image" src="https://github.com/user-attachments/assets/649d7118-6163-4050-a08d-4f04310bbb74" />

This is completely normal. By default it assumes a disk should have one writer at a time (to prevent two VMs from corrupting the same disk image). But ISOs are read only so multiple VMs can safely mount the same ISO at the same time, nothing to corrupt.

Once evertyhing is set, we should start the installation for our 2019 server .

<img width="1192" height="697" alt="image" src="https://github.com/user-attachments/assets/d8e142a6-2614-444d-b7b6-5b10b9971587" />

Same thing , we choose the Desktop experience , 

<img width="1054" height="577" alt="image" src="https://github.com/user-attachments/assets/bac3e3de-3605-4a18-a4db-35d30ac126fc" />

Here we choose Custom , from there we just add the Driver just like we did with the 2022 Server : 

<img width="942" height="521" alt="image" src="https://github.com/user-attachments/assets/578288ed-caaa-45cb-8b3b-77d03b8f7e5b" />

For this one i chose the 2019 version 

<img width="921" height="531" alt="image" src="https://github.com/user-attachments/assets/e150f31c-a328-4da8-8867-85e007280912" />

Once we select the Driver , we should be good to continue the installation :

<img width="830" height="532" alt="image" src="https://github.com/user-attachments/assets/b1490bb7-74ef-4101-93cf-6efb9b6c9703" />

Now we just wait again :

<img width="820" height="525" alt="image" src="https://github.com/user-attachments/assets/fa973635-3421-459e-a4cb-4b7e55e2b657" />

Finally, we just setup the password for the Administrator and it will login automatically : 

<img width="1212" height="718" alt="image" src="https://github.com/user-attachments/assets/50bec800-4816-419c-93c9-0e575efd413c" />

For now we will leave it here , and move on to the 2 Windows machines .


**Workstation01 / Workstation02**

This one is pretty straight forward as well . It's pretty similar to what we did for the Servers , only this time we will need Secure Boot, TPM2.0 , let's start : 

Here are the specs : 

Hostnames: WKS01 and WKS02
- OS: Windows 11 Enterprise (90-day eval)
- RAM: 4 GB (6 GB if you can spare it — Win 11 is heavier than Server)
- vCPU: 2
- Disk: 60 GB qcow2
- Firmware: UEFI with Secure Boot (OVMF_CODE_4M.secboot.fd)
- TPM: 2.0 (required by Win 11)
- Network: lab-net, static IPs 10.10.10.30 and 10.10.10.40

<img width="594" height="411" alt="image" src="https://github.com/user-attachments/assets/056d9df4-f884-4a68-91cb-63623b30a0c0" />

ALl of these steps apply for both machines of course :

<img width="539" height="500" alt="image" src="https://github.com/user-attachments/assets/74a990e0-99e5-4b43-a6de-2b6c6a67880e" />

**Overview :**

Here we MUST specify the secure boot, otherwise it just won't boot up later on . 

<img width="1054" height="621" alt="image" src="https://github.com/user-attachments/assets/7754ffdb-789b-480c-b7a9-9f9ccd070360" />

**TPM 2.0:**

TPM 2.0 is required here, Add Hardware --> TPM --> TIS / Emulated / 2.0. Non-negotiable, Win 11 won't install without it.

<img width="1274" height="876" alt="image" src="https://github.com/user-attachments/assets/92f83940-fbb6-4023-8478-7ea97af6c7a4" />

**The Rest**

Just like what we did for the 2019 and 2022 server :

- Overview --> UEFI secboot firmware
- SATA Disk --> VirtIO bus (optional for speed)
- NIC --> virtio model
- Add second CD-ROM --> virtio-win.iso on SATA bus (not IDE, Q35 trap again)
- Add TPM 2.0
- Begin Installation
- Boot Manager --> pick the Win 11 ISO entry

**Account creation:**

The Account creation forces a Microsoft account  Win 11 Home/Pro will force you into signing in with a Microsoft account when trying to setup the machie.

Win 11 Enterprise lets you create a local account without the fuss, which is why we picked Enterprise. 

*If the installer still tries to force a Microsoft account on you :*

Press Shift+F10 on the account screen --> opens a command prompt --> type OOBE\BYPASSNRO --> system reboots --> then on the network screen pick I don't have internet --> fall through to local account creation. (Microsoft keeps changing this trick's name, if BYPASSNRO is gone in a future build, search for the current bypass.)
 
<img width="991" height="618" alt="image" src="https://github.com/user-attachments/assets/a0a7af5a-26b5-496a-afc0-678b05a01ff2" />

Now for the Disk , again : 

<img width="889" height="617" alt="image" src="https://github.com/user-attachments/assets/d92f309a-3a40-4c75-a844-d7b243b0de55" />

We choose the virtio driver : 

<img width="973" height="598" alt="image" src="https://github.com/user-attachments/assets/f45bc481-9b54-43e5-ae2b-101191f23360" />

Just go to **E:\amd64\w11** 

<img width="943" height="640" alt="image" src="https://github.com/user-attachments/assets/5b521ee3-f2ec-4314-9b21-ce14bfe13e93" />

And finally we'll choose the SCSI controller (not the pass-through controller) :

<img width="1142" height="639" alt="image" src="https://github.com/user-attachments/assets/263ddb2e-aef4-4dca-a7ca-83c6830d17fd" />

Finally we'll start the installation : 

<img width="910" height="508" alt="image" src="https://github.com/user-attachments/assets/897a965b-6907-4e3f-97be-1e44db69d348" />

Now we just wait : 

<img width="925" height="291" alt="image" src="https://github.com/user-attachments/assets/aa4907a1-acf6-4e73-99ff-13a0a9221e34" />

Once it's done we'll just clone it and rename it something else, for the second machine , unless you want to redo all these steps of course :)

Now once done , pick you language , Keyboard layout : 

<img width="1096" height="686" alt="image" src="https://github.com/user-attachments/assets/434926e0-ba38-4e5d-804a-4aed29cf8a7d" />

After that it will ask you for the name of the local account and his password :

A<img width="1105" height="642" alt="image" src="https://github.com/user-attachments/assets/a468209b-413a-4f3e-92ad-b0c8cc3e6a85" />

After that you just need to answer few questions , just say No to all of them :)

<img width="1226" height="764" alt="image" src="https://github.com/user-attachments/assets/0e374243-f442-4b5d-a640-e43997f0decc" />

Just like that our Windows machine is set .

Now just turn it off , go to Virt-manager and clone it : 

<img width="869" height="530" alt="image" src="https://github.com/user-attachments/assets/ce9efeef-99e9-4d17-9de4-45ab4deacd17" />

We'll name it something else and make sure we clone every storage we got.

<img width="747" height="541" alt="image" src="https://github.com/user-attachments/assets/e70608a3-cb40-4ede-bfd5-5a45758cf5ac" />

Once done we should see all of our machines inside the Virt-manager : 

<img width="872" height="452" alt="image" src="https://github.com/user-attachments/assets/b7f42775-1573-4a65-8c35-98df13ccdad8" />

Now we can move to the next step which is setting up the Domain : 


### Setting up the Domain : 

For this lab we will create a Domain called **2SEC** , we will first setup the DC and the CA , join the CA machine , from there we will join both Windows machines onto the domain as well . 











