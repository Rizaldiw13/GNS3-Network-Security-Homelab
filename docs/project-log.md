# Project Log

This log documents how I built the homelab, the problems I ran into, what I tried, and how I verified that each solution worked.

---

## Initial Environment Setup

*Date: August 18, 2026*

Well.. Here we go! The start of my very own virtual network engineering and security homelab, using GNS3.

My initial setup consisted of:

- Windows 11 Home
- GNS3 Desktop 2.2.61
- GNS3 VM 2.2.61
- VMware Workstation Pro

At first, I did not fully understand why GNS3 Desktop, the GNS3 VM, and VMware were all needed at the same time.

After working through the setup, I learned that they each serve a different purpose:

- **GNS3 Desktop** will be the front-end that we interact with. It lets us create topologies, add devices, connect interfaces, open consoles, and start or stop nodes.
- **GNS3 VM** will be the Linux-based backend that runs more resource-intensive virtual network appliances.
- **VMware Workstation**, in my original setup, acted as the hypervisor that ran the GNS3 VM.

The original architecture looked like this:

```text
Physical PC
│
├── Windows 11
│
├── GNS3 Desktop
│
└── VMware Workstation
      │
      └── GNS3 VM
            │
            └── QEMU-based network appliances
```

### Configuring the GNS3 Local Server

During the initial setup, GNS3 Desktop had trouble communicating reliably with the local GNS3 server.

Because of that, I configured the local server to bind directly to the IPv4 loopback address:

```text
127.0.0.1
```

using port:

```text
3080
```

This means that GNS3 Desktop is now communicating with a server running on the same computer.

Using the loopback address also avoided potential hostname or IPv6 resolution issues (which may be the reason why it had trouble communicating initially)

After making this change, GNS3 successfully validated the local server.

### Configuring VMware Integration

GNS3 initially could not find VMware's `vmrun` utility automatically.

I found that VMware Workstation had installed it at:

```text
C:\Program Files\VMware\VMware Workstation\vmrun.exe
```

and I manually configured this path inside GNS3.

After doing that, GNS3 was able to detect VMware Workstation and the imported GNS3 VM.

---

## Basic GNS3 Connectivity Test

*Date: August 18, 2026*

Before trying to run virtual routers or firewalls, I wanted to confirm that the basic GNS3 environment was actually working.

I created a very simple Layer 2 topology:

```text
PC1 -------- Switch1 -------- PC2
```

I used two VPCS nodes and GNS3's built-in Ethernet switch.

I assigned the following static IPv4 addresses:

```text
PC1: 10.0.0.1/24
PC2: 10.0.0.2/24
```

From PC1, I tested connectivity using:

```text
ping 10.0.0.2
```

and.. the ping succeeded!

This confirmed that:

- GNS3 Desktop was working
- the local GNS3 server was working
- VPCS nodes were working
- the virtual links were working
- the built-in switch was correctly forwarding Ethernet frames

I also learned that I did not need to configure a default gateway in this test because both PCs were on the same `10.0.0.0/24` subnet.

---

## Issue — KVM Acceleration Was Unavailable

*Date: August 18, 2026*

When I first booted the GNS3 VM under VMware Workstation, I noticed that it reported:

```text
Virtualization: vmware
KVM support available: False
```

At first, I did not know whether this was actually a problem.

After looking into how GNS3 runs more advanced appliances, I learned that many of them run through QEMU (Quick Emulator) and that KVM provides hardware acceleration for those virtual machines.

Without KVM, QEMU can fall back to software-based CPU emulation, which could make larger labs significantly slower.

### Investigating Nested Virtualization

The virtualization structure I was trying to create looked like this:

```text
My Physical AMD Ryzen CPU: 7800x3d
        │
        │ AMD-V
        ▼
VMware Workstation
        │
        ▼
GNS3 VM
        │
        │ KVM/QEMU
        ▼
Virtual network appliances
```

This meant that I was effectively trying to run virtual machines inside another virtual machine, which requires **nested virtualization**.

Inside VMware, I enabled:

```text
Virtualize Intel VT-x/EPT or AMD-V/RVI
```

However, VMware returned an error saying that virtualized AMD-V/RVI was not available.

At first, I assumed that VMware would simply be able to pass AMD-V directly through to the GNS3 VM. That turned out to be more complicated because Windows was already using hardware virtualization for its own security features.

---

## Investigating Windows VBS and Memory Integrity

*Date: August 18, 2026*

I discovered that Windows was running **Virtualization-Based Security (VBS)** and **Memory Integrity**.

Because these features use Microsoft's own hypervisor, I suspected that they were interfering with VMware's ability to expose AMD-V to the nested GNS3 VM.

I checked the Windows boot configuration using:

```cmd
bcdedit /enum {current}
```

I also checked the Device Guard and VBS state using PowerShell:

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard `
-Namespace root\Microsoft\Windows\DeviceGuard |
Format-List VirtualizationBasedSecurityStatus,SecurityServicesConfigured,SecurityServicesRunning
```

The system reported:

```text
VirtualizationBasedSecurityStatus : 2
```

which indicated that VBS was running.

I also checked Windows System Information (`msinfo32`) and saw:

```text
A hypervisor has been detected
```

This confirmed that Microsoft's hypervisor was active.

### Attempting to Disable the Hypervisor

As part of troubleshooting, I temporarily tested disabling some Windows virtualization features.

I used:

```cmd
bcdedit /set hypervisorlaunchtype off
```

and:

```cmd
bcdedit /set vsmlaunchtype off
```

I also checked whether Windows features such as Virtual Machine Platform and Windows Hypervisor Platform were enabled.

Even after these changes and multiple restarts, Windows continued to report that VBS and the Microsoft hypervisor were running.

At this point, I decided that continuing to disable more Windows security features just to make VMware nested virtualization work was not worth it. Therefore I decided i will change the virtualization architecture from using VMWare Workstation Pro, to Windows' Hyper-V.

---

## Restoring Windows Security Settings

*Date: August 19, 2026*

Before moving on, I restored the Windows security settings that I had temporarily changed during my previous troubleshooting.

I restored the boot configuration using:

```cmd
bcdedit /set hypervisorlaunchtype auto
bcdedit /set vsmlaunchtype auto
```

I also re-enabled VBS in the registry:

```powershell
Set-ItemProperty `
-Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" `
-Name "EnableVirtualizationBasedSecurity" `
-Value 1
```

I turned **Memory Integrity** back on through:

```text
Windows Security
→ Device Security
→ Core Isolation
→ Memory Integrity
```

After restarting Windows, I verified the configuration again with:

```cmd
bcdedit /enum {current}
```

..and the output showed:

```text
hypervisorlaunchtype    Auto
vsmlaunchtype           Auto
```

I also checked Device Guard again:

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard `
-Namespace root\Microsoft\Windows\DeviceGuard |
Format-List VirtualizationBasedSecurityStatus,SecurityServicesConfigured,SecurityServicesRunning
```

..and the result was:

```text
VirtualizationBasedSecurityStatus : 2
SecurityServicesConfigured        : {2}
SecurityServicesRunning           : {2}
```

This confirmed that VBS and Memory Integrity were running again.

---

## Migrating from VMware to Hyper-V

*Date: August 19, 2026*

Since Windows was already using Microsoft's hypervisor for VBS, I decided that it made more sense to use Hyper-V directly instead of trying to make VMware coexist with it for nested virtualization.

My computer was originally running Windows 11 Home, which does not include the full Hyper-V feature set.

Because I also plan to use virtualization for future networking, active directory, server, and security homelabs, I upgraded the host operating system to Windows 11 Pro.

After upgrading, I enabled Hyper-V through:

```text
Turn Windows features on or off
→ Hyper-V
```

I restarted Windows and verified the installation by opening Hyper-V Manager.

The new architecture became:

```text
My Physical AMD Ryzen CPU: 7800x3d
        │
        ▼
    Hyper-V
        │
        ▼
    GNS3 VM
        │
        │ KVM/QEMU
        ▼
Virtual network appliances
```

---

## Installing the Hyper-V Version of the GNS3 VM

*Date: August 19, 2026*

I downloaded the Hyper-V version of the GNS3 VM and stored it in:

```text
C:\GNS3-VM\GNS3.VM.Hyper-V.2.2.61
```

The package contained:

```text
gns3vm-disk1.vhdx
gns3vm-disk2.vhdx
create-vm.ps1
install-vm.bat
```

### Smart App Control Blocking the Installer

When I first tried to run `install-vm.bat`, Windows Smart App Control blocked the downloaded scripts.

Since I had downloaded the package from the official GNS3 source, I chose to unblock those specific files rather than disable Smart App Control.

I opened PowerShell as Administrator and ran:

```powershell
Get-ChildItem "C:\GNS3-VM\GNS3.VM.Hyper-V.2.2.61" -Recurse |
Unblock-File
```

After that, I ran the installer from an elevated Command Prompt:

```cmd
cd /d C:\GNS3-VM\GNS3.VM.Hyper-V.2.2.61
install-vm.bat
```

The script successfully created a Hyper-V virtual machine named:

```text
GNS3 VM
```

and I verified this in Hyper-V Manager.

---

## Configuring Nested Virtualization

*Date: August 19, 2026*

The main reason I moved to Hyper-V was to get nested virtualization working properly.

I checked whether Hyper-V was exposing the CPU virtualization extensions to the GNS3 VM using:

```powershell
Get-VMProcessor -VMName "GNS3 VM" |
Select-Object Count, ExposeVirtualizationExtensions
```

The output showed:

```text
Count  ExposeVirtualizationExtensions
-----  -------------------------------
1      True
```

The important part here was:

```text
ExposeVirtualizationExtensions = True
```

This meant that Hyper-V was exposing the physical CPU's virtualization capabilities to the GNS3 VM.

I then increased the VM resources to:

```text
4 vCPUs
8 GB RAM
```

I also disabled Dynamic Memory so that the GNS3 VM would receive a predictable fixed RAM allocation.

After booting the VM, I finally saw:

```text
Virtualization: hyperv
KVM support available: True
```

This confirmed that moving to Hyper-V had solved the original KVM problem.

---

## Issue — GNS3 Desktop Could Not Control the Hyper-V VM

*Date: August 19, 2026*

After getting the GNS3 VM running successfully under Hyper-V, I ran into another problem.

GNS3 Desktop could not detect or control the VM.

GNS3 reported:

```text
The Windows account running GNS3 does not have the required permissions for Hyper-V
```

Because of this, the VM name dropdown in GNS3 remained empty.

### Adding My Account to Hyper-V Administrators

I opened PowerShell as Administrator and first stored my current Windows account name:

```powershell
$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
```

I then added that account to the local Hyper-V Administrators group:

```powershell
Add-LocalGroupMember `
-Group "Hyper-V Administrators" `
-Member $currentUser
```

I verified the change with:

```powershell
Get-LocalGroupMember -Group "Hyper-V Administrators"
```

My account appeared in the group.

However, GNS3 still reported that I did not have permission.

### Diagnosing the Permission Problem

I wanted to determine whether the problem was actually GNS3 or whether my normal Windows session still lacked Hyper-V permissions.

I opened a normal, non-administrator PowerShell window and ran:

```powershell
whoami
```

I then checked whether my current login token contained the Hyper-V Administrators group:

```powershell
whoami /groups | findstr /i "Hyper-V"
```

Initially, this returned nothing.

I also tried querying Hyper-V directly:

```powershell
Get-VM
```

This produced a permissions error.

That told me the problem was not actually GNS3. My account had been added to the group, but my current Windows login session had not yet received the updated permissions.

So I restarted Windows completely.

After restarting, I ran:

```powershell
whoami /groups | findstr /i "Hyper-V"
```

This time, the result included:

```text
BUILTIN\Hyper-V Administrators
```

Then I ran:

```powershell
Get-VM
```

and successfully received:

```text
Name       State
----       -----
GNS3 VM    Off
```

This confirmed that my normal Windows account could now manage Hyper-V without running applications as Administrator.

### Connecting GNS3 Desktop to Hyper-V

I reopened GNS3 normally and went to:

```text
Edit
→ Preferences
→ GNS3 VM
```

I configured:

```text
Enable the GNS3 VM:       Enabled
Virtualization engine:    Hyper-V
VM name:                  GNS3 VM
Port:                     80
```

After applying the settings, GNS3 successfully detected and started the Hyper-V VM.

The Servers Summary showed both:

```text
GNS3 VM      Green
Local Server Green
```

At this point, GNS3 Desktop could automatically start, stop, and communicate with the GNS3 VM without me manually opening Hyper-V Manager.

---

## What I Learned About the Virtualization Architecture

*Date: August 19, 2026*

One thing I did not understand when I started this project was why I needed both Hyper-V and the GNS3 VM.

I originally thought Hyper-V itself was the virtual machine.

However, I learned that Hyper-V is actually the **hypervisor** responsible for creating and running virtual machines, while the GNS3 VM is one of the virtual machines running on top of Hyper-V.

So the final, working architecture for this homelab is:

```text
Physical PC
│
├── Windows 11 Pro
│
├── GNS3 Desktop
│
└── Hyper-V
      │
      └── GNS3 VM
            │
            ├── KVM
            ├── QEMU
            │
            └── Virtual network appliances
```

The roles are:

```text
GNS3 Desktop
→ Frontend and topology controller

Hyper-V
→ Host hypervisor

GNS3 VM
→ Linux-based backend used by GNS3

KVM/QEMU
→ Runs virtual network appliances inside the GNS3 VM
```

This is also an example of **nested virtualization**:

```text
Physical CPU
    ↓
Hyper-V
    ↓
GNS3 VM
    ↓
KVM/QEMU
    ↓
Network appliance VM
```

An interesting thing here is that, the entire installation and troubleshooting process has helped me connect some of the virtualization concepts I had previously learned in my OS course, now with an actual practical use case.

---

## Lab Design Decision

*Date: August 19, 2026*

Another issue I ran into was Cisco virtual images.

Originally, I planned to use Cisco IOSv and IOSvL2 inside GNS3.

However, I learned that legally obtaining and using Cisco images outside Cisco's own platforms can involve additional licensing restrictions.

Rather than relying on unofficial images, I decided to split my portfolio into two complementary homelabs.

### GNS3 Network Security Homelab

This project will focus on mixed-vendor networking and security.

The planned technologies and topics include:

- IP addressing and subnetting
- static routing
- OSPF
- VLANs and segmentation
- inter-VLAN routing
- VyOS / FRRouting
- pfSense
- NAT
- VPNs
- Python network automation
- Ansible
- SNMP monitoring
- IDS/IPS
- Wireshark packet analysis

### Cisco CML Homelab

I will also build a separate lab using Cisco Modeling Labs for Cisco-specific practice (CCNA in.. 3 months? Hopefully!)

That project will focus on:

- Cisco IOS / IOS-XE CLI
- Cisco switching
- VLAN configuration
- 802.1Q trunks
- STP
- EtherChannel
- OSPF
- ACLs
- Cisco troubleshooting commands

This way, the GNS3 project can demonstrate broader mixed-vendor network engineering, automation, and security skills while the CML project demonstrates Cisco-specific experience.

---

## Current Status

*Date: August 19, 2026*

The virtualization environment is now working.

Current status:

```text
Windows 11 Pro            Working
Hyper-V                   Working
Nested virtualization     Working
GNS3 Desktop              Working
GNS3 VM                   Working
KVM acceleration          Available
GNS3 ↔ Hyper-V control    Working
Local GNS3 server         Working
```

The next milestone is to begin building the actual network topology.

My first networking objective will be to build a small routed network, configure IP addressing and static routes, and verify end-to-end connectivity before moving on to OSPF and larger network designs.