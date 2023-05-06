# Nested virtualization: Run Hyper-V in a Virtual Machine

## About
This document describes what nested virtualization is and how to run Hyper-V in a Hyper-V Virtual Machine.  

## Description
Nested virtualization is a feature that allows you to run Hyper-V inside of a Hyper-V virtual machine (VM).  

## Table of contents
1. [Prerequisites](#prerequisites)
    1. [Intel processor with VT-x and EPT technology](#intel-processor)
    2. [AMD EPYC/Ryzen processor or later](#amd-processor)
2. [Configure Nested Virtualization](#configure-nested-virtualization)
3. [Disable Nested Virtualization](#disable-nested-virtualization)
4. [Dynamic Memory and Runtime Memory Resize](#dynamic-memory-and-runtime-memory-resize)
5. [Networking Options](#networking-options)
    1. [MAC Address Spoofing](#mac-address-spoofing)
    2. [Network Address Translation (NAT)](#network-address-translation-nat)
6. [How nested virtualization works](#how-nested-virtualization-works)
7. [3rd Party Virtualization Apps](#3rd-party-virtualization-apps)
8. [Related links](#related-links)

## Prerequisites <a id="prerequisites"></a>

### Intel processor with VT-x and EPT technology <a id="intel-processor"></a>
- The Hyper-V host must be Windows Server 2016/Windows 10 or greater
- VM configuration version 8.0 or greater

### AMD EPYC/Ryzen processor or later <a id="amd-processor"></a>
- The Hyper-V host must be Windows Server 2022/Windows 11 or greater
- VM configuration version 10.0 or greater

> :warning: **Note:** The guest can be any Windows supported guest operating system. Newer Windows operating systems may support enlightenments that improve performance.

## Configure Nested Virtualization <a id="configure-nested-virtualization"></a>
1. Create a virtual machine. See the prerequisites above for the required OS and VM versions.
2. While the virtual machine is in the OFF state, run the following command on the physical Hyper-V host. This enables nested virtualization for the virtual machine.

```powershell
Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
```

3. Start the virtual machine.
4. Install Hyper-V within the virtual machine, just like you would for a physical server. For more information on installing Hyper-V 
see, [Install Hyper-V](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v).

## Disable Nested Virtualization <a id="disable-nested-virtualization"></a>
You can disable nested virtualization for a stopped virtual machine using the following PowerShell command:
```powershell
Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $false
```

## Dynamic Memory and Runtime Memory Resize <a id="dynamic-memory-and-runtime-memory-resize"></a>
When Hyper-V is running inside a virtual machine, the virtual machine must be turned off to adjust its memory. This means that even if dynamic memory is enabled, the 
amount of memory will not fluctuate. For virtual machines without dynamic memory enabled, any attempt to adjust the amount of memory while it's on will fail.  

Note that simply enabling nested virtualization will have no effect on dynamic memory or runtime memory resize. The incompatibility only occurs while Hyper-V 
is running in the VM.  

## Networking Options <a id="networking-options"></a>
There are two options for networking with nested virtual machines:
1. MAC address spoofing
2. NAT networking

### MAC Address Spoofing <a id="mac-address-spoofing"></a>
In order for network packets to be routed through two virtual switches, MAC address spoofing must be enabled on the first (L1) level of virtual switch. This is 
completed with the following PowerShell command.  
```powershell
Get-VMNetworkAdapter -VMName <VMName> | Set-VMNetworkAdapter -MacAddressSpoofing On
```

### Network Address Translation (NAT) <a id="network-address-translation-nat"></a>
The second option relies on network address translation (NAT). This approach is best suited for cases where MAC address spoofing is not possible, like in a public 
cloud environment.  

First, a virtual NAT switch must be created in the host virtual machine (the "middle" VM). Note that the IP addresses are just an example, and will vary across 
environments:
```powershell
New-VMSwitch -Name VmNAT -SwitchType Internal
New-NetNat –Name LocalNAT –InternalIPInterfaceAddressPrefix '192.168.100.0/24'
```

Next, assign an IP address to the net adapter:
```powershell
Get-NetAdapter 'vEthernet (VmNat)' | New-NetIPAddress -IPAddress 192.168.100.1 -AddressFamily IPv4 -PrefixLength 24
```

Each nested virtual machine must have an IP address and gateway assigned to it. Note that the gateway IP must point to the NAT adapter from the previous step.
```powershell
Get-NetAdapter 'vEthernet (VmNat)' | New-NetIPAddress -IPAddress 192.168.100.2 -DefaultGateway 192.168.100.1 -AddressFamily IPv4 -PrefixLength 24
```

You may also want to assign a DNS server:
```cmd
Netsh interface ip add dnsserver "vEthernet (VmNat)" address=<my DNS server>
```

## How nested virtualization works <a id="how-nested-virtualization-works"></a>
Modern processors include hardware features that make virtualization faster and more secure. Hyper-V relies on these processor extensions to run virtual 
machines (e.g. Intel VT-x and AMD-V). Typically, once Hyper-V starts, it prevents other software from using these processor capabilities. This prevents guest 
virtual machines from running Hyper-V.  

Nested virtualization makes this hardware support available to guest virtual machines.  

The diagram below shows Hyper-V without nesting. The Hyper-V hypervisor takes full control of the hardware virtualization capabilities (orange arrow), and 
does not expose them to the guest operating system.  

![Nested Virtualization Disabled](../../img/withoutnestedvirtualization.png "Nested Virtualization Disabled")

In contrast, the diagram below shows Hyper-V with nested virtualization enabled. In this case, Hyper-V exposes the hardware virtualization extensions to its 
virtual machines. With nesting enabled, a guest virtual machine can install its own hypervisor and run its own guest VMs.  

![Nested Virtualization Enabled](../../img/withnestedvirtualization.png "Ubuntu Enabled")

## 3rd Party Virtualization Apps <a id="3rd-party-virtualization-apps"></a>
Virtualization applications other than Hyper-V are not supported in Hyper-V virtual machines, and are likely to fail. This includes any software that requires 
hardware virtualization extensions.

## Related links <a id="related-links"></a>
[Nested Virtualization - microsoft.com](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization)  
