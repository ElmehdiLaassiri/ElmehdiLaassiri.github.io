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


**Add Hardware --> Storage (for the virtio-win CD):**

- Click Add Hardware --> Storage.
- Select or create custom storage --> Browse --> pick ~/Lab/ISOs/virtio-win.iso.
- Device type: CDROM device. Bus: SATA.
- Click Finish. We now have two CD-ROMs attached to the VM: the Windows installer + the virtio drivers.







