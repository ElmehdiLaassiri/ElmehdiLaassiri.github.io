---
title: " Home Lab 1 : Minimal AD Setup on Debian "
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

