# Understanding QEMU, KVM, libvirt, and Virt-Manager

## 1. The Big Picture

Linux virtualization is not a single monolithic application like VirtualBox. Instead, it is a **layered stack of technologies**, where each component has a different responsibility.

A useful mental model is:

```text
+-------------------------------------------------------------+
|                    Management Layer                         |
|                                                             |
|     Virt-Manager (GUI)       virsh / virt-install (CLI)     |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                         libvirt                             |
|       VM management API, configuration, networking,        |
|                    storage, and lifecycle                   |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                           QEMU                              |
|        Userspace VM process + virtual hardware              |
|                                                             |
|  Virtual CPU | RAM | Disk | NIC | GPU | USB | BIOS/UEFI    |
+------------------------------+------------------------------+
                               |
                         KVM acceleration
                               |
                               v
+-------------------------------------------------------------+
|                    Linux Kernel / KVM                       |
|       Hardware-assisted CPU virtualization (VT-x/AMD-V)    |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                    Physical Hardware                        |
|                         CPU / RAM                           |
+-------------------------------------------------------------+
```

The most important thing to understand is:

> **QEMU is the userspace process that actually represents the virtual machine. KVM is the Linux kernel subsystem that accelerates the VM's CPU execution using hardware virtualization. libvirt manages QEMU, while Virt-Manager provides a graphical interface to libvirt.**

---

# 2. Component Breakdown

## KVM — Kernel-based Virtual Machine

**KVM** is a Linux kernel virtualization subsystem.

It uses hardware virtualization extensions such as:

* **Intel VT-x**
* **AMD-V**

to allow guest operating systems to execute CPU instructions directly on the physical CPU whenever possible, rather than having every instruction emulated in software.

KVM also provides the kernel-side mechanisms needed for virtual CPUs and memory virtualization.

### Is KVM a Type 1 or Type 2 hypervisor?

This is slightly more nuanced than the usual "KVM is a Type 1 hypervisor" explanation.

KVM is integrated directly into the **Linux kernel**, so the Linux kernel performs the hypervisor role. Because the virtualization layer operates directly with the hardware rather than running on top of another conventional host OS, KVM-based Linux virtualization is commonly classified as **Type 1**.

However, Linux is still functioning as a complete operating system with normal userspace applications.

A better mental model is:

```text
Linux Kernel
    ├── Normal OS functionality
    └── KVM virtualization functionality
             │
             └── Guest virtual CPUs
```

---

## QEMU — Quick Emulator

**QEMU** is a userspace virtualization and emulation program.

QEMU provides the VM's virtual hardware, including things such as:

* Virtual disks
* Virtual network cards
* Virtual storage controllers
* Virtual graphics devices
* USB controllers
* Audio devices
* BIOS/UEFI firmware
* Virtual CPUs
* Other virtual peripherals

When KVM is available, QEMU can use KVM to accelerate CPU virtualization.

Therefore, a useful simplified relationship is:

```text
QEMU
 │
 ├── Provides virtual hardware
 │
 └── Uses KVM
       │
       └── Accelerates CPU virtualization
```

Without KVM, QEMU can perform much more CPU emulation in software, but performance is generally much worse for x86 virtual machines.

---

## libvirt

**libvirt** is a virtualization management framework and API.

It provides a consistent interface for managing virtualization technologies, with QEMU/KVM being one of its most common backends.

Instead of manually constructing long QEMU command lines, libvirt manages VM definitions and configuration.

VM definitions are represented using XML, for example:

```xml
<domain type='kvm'>
    ...
</domain>
```

libvirt can manage:

* VM creation
* VM startup and shutdown
* CPU and RAM configuration
* Virtual disks
* Virtual networks
* Storage pools
* Snapshots
* VM autostart
* Device configuration

On modern Linux distributions, libvirt may use several modular daemons rather than one monolithic `libvirtd` process. For QEMU/KVM, `virtqemud` is the relevant modular daemon.

---

## Virt-Manager

**Virt-Manager** is a graphical frontend for libvirt.

It allows you to:

* Create VMs
* Configure CPU and RAM
* Attach ISO files
* Manage virtual disks
* Configure networking
* View VM consoles
* Monitor resource usage
* Start, stop, and delete VMs

Virt-Manager itself is **not the hypervisor**.

The relationship is:

```text
Virt-Manager
      │
      ▼
   libvirt
      │
      ▼
    QEMU
      │
      ▼
     KVM
      │
      ▼
   Hardware
```

---

# 3. What Actually Happens When You Start a VM?

Suppose you click **Start** in Virt-Manager.

The rough sequence is:

```text
You click "Start"
       │
       ▼
Virt-Manager
       │
       ▼
libvirt
       │
       ▼
QEMU process is started
       │
       ├── Creates virtual disk
       ├── Creates virtual NIC
       ├── Creates virtual GPU
       ├── Creates virtual CPU
       ├── Loads BIOS/UEFI
       │
       ▼
QEMU uses KVM for CPU virtualization
       │
       ▼
Linux kernel
       │
       ▼
Physical CPU / RAM
```

So:

* **Virt-Manager** = GUI
* **libvirt** = management layer
* **QEMU** = VM userspace process + virtual hardware
* **KVM** = kernel-side hardware virtualization
* **CPU/RAM** = physical resources

This distinction is important because these technologies are often incorrectly treated as interchangeable.

---

# 4. Virtual Networking

libvirt commonly provides a default NAT-based virtual network.

You will usually see a bridge such as:

```text
virbr0
```

## NAT Networking

The default libvirt network generally works like this:

```text
                Internet
                   │
                   ▼
             Physical NIC
                   │
                Host OS
                   │
              libvirt NAT
                   │
                virbr0
              /         \
             /           \
          VM 1          VM 2
```

The VM receives a private IP address and its traffic is translated through the host.

### Advantages

* Works with minimal configuration.
* VMs get internet access.
* Does not require giving each VM a LAN address.

### Limitation

Devices elsewhere on your LAN generally cannot directly initiate connections to the VM unless you configure appropriate port forwarding or other networking rules.

---

# 5. Bridged Networking

With a bridge, the VM can appear directly on the physical network.

Conceptually:

```text
             Router
                │
                │
        Physical Network
                │
          Host Network Bridge
             /       \
            /         \
         Host          VM
                       │
                  Own LAN IP
```

The VM can receive its own IP address from your router's DHCP server.

This is useful when you want the VM to behave like another physical computer on the network.

### NAT vs Bridge

| Feature                                  | NAT          | Bridge                     |
| ---------------------------------------- | ------------ | -------------------------- |
| Internet access                          | Yes          | Yes                        |
| VM gets LAN IP                           | Usually no   | Yes                        |
| Other LAN devices can directly access VM | Not normally | Yes                        |
| Configuration difficulty                 | Easy         | More involved              |
| Good for                                 | General VMs  | Servers / network services |

> **Note:** Modern Linux systems often configure bridges using NetworkManager or `systemd-networkd`. `bridge-utils` is an older utility package and is not required for the standard libvirt NAT setup.

---

# 6. Step 1: Install the Virtualization Stack

On Debian-based systems:

```bash
sudo apt update
sudo apt install qemu-system-x86 libvirt-daemon-system libvirt-clients virt-manager virtinst
```

For a desktop system, this provides the core QEMU/KVM + libvirt + Virt-Manager stack.

## What the Packages Provide

| Package                 | Purpose                                                     |
| ----------------------- | ----------------------------------------------------------- |
| `qemu-system-x86`       | QEMU system emulator for x86/x86-64 virtual machines        |
| `libvirt-daemon-system` | System-level libvirt infrastructure and service integration |
| `libvirt-clients`       | Client tools such as `virsh`                                |
| `virt-manager`          | Graphical VM management application                         |
| `virtinst`              | CLI tools such as `virt-install` and `virt-xml`             |

### Optional: Guest Agent Package

You install `qemu-guest-agent` **inside the guest OS**, not necessarily on the host.

For example, on a Debian/Ubuntu guest:

```bash
sudo apt install qemu-guest-agent
```

---

# 7. Step 2: Configure User Permissions

Add your normal user to the groups commonly used for KVM/libvirt access:

```bash
sudo usermod -aG libvirt,kvm $USER
```

Then **log out and log back in** for the new group membership to take effect.

You can verify your groups with:

```bash
groups
```

Or specifically:

```bash
groups | grep libvirt
```

> Depending on the distribution and libvirt configuration, exact group requirements can vary. If you still receive permission errors after re-login, check the relevant libvirt socket permissions and your distribution's libvirt documentation.

---

# 8. Step 3: Verify KVM and libvirt

## Check for `/dev/kvm`

```bash
ls -l /dev/kvm
```

If KVM is available, you should see the device node.

## Check loaded KVM modules

For Intel CPUs:

```bash
lsmod | grep kvm_intel
```

For AMD CPUs:

```bash
lsmod | grep kvm_amd
```

You can also check the generic KVM module:

```bash
lsmod | grep kvm
```

## Validate the Host

If available on your distribution:

```bash
virt-host-validate
```

This performs a series of checks for virtualization-related host configuration.

## Check libvirt

On systems using the traditional `libvirtd` service:

```bash
systemctl status libvirtd
```

If your distribution uses modular libvirt daemons, you may instead see services such as:

```text
virtqemud
virtlogd
virtnetworkd
```

Do not assume that every modern Linux distribution uses a single `libvirtd` daemon.

---

# 9. Step 4: Configure Virtual Networking

Check the available libvirt networks:

```bash
sudo virsh net-list --all
```

You will typically see something similar to:

```text
 Name      State      Autostart
---------------------------------
 default   active     yes
```

If the default network exists but is inactive:

```bash
sudo virsh net-start default
```

To make it start automatically:

```bash
sudo virsh net-autostart default
```

Then verify:

```bash
sudo virsh net-list --all
```

---

# 10. Step 5: Create a Virtual Machine

## Option A: Virt-Manager

Launch Virt-Manager:

```bash
virt-manager
```

Then:

1. Click **Create a new virtual machine**.
2. Select **Local install media (ISO image or CDROM)**.
3. Select your guest operating system ISO.
4. Allocate RAM.
5. Allocate virtual CPUs.
6. Create or select a virtual disk.
7. Review the VM configuration.
8. Start the installation.

For a lightweight server VM, something like:

```text
RAM: 512 MB – 1 GB
vCPU: 1
Disk: 8–20 GB
```

may be sufficient, depending on the operating system and workload.

For a desktop OS, allocate substantially more resources.

---

# 11. VM Disk Images

QEMU commonly uses disk image formats such as:

* `qcow2`
* `raw`

## qcow2

`qcow2` is commonly used for general-purpose VMs because it supports features such as:

* Dynamic allocation
* Snapshots
* Compression
* Encryption support
* Sparse storage

For example:

```text
10 GB qcow2 disk
```

does **not necessarily consume 10 GB of physical storage immediately**. The file can grow as data is written.

## raw

A raw disk image has a simpler representation and can provide excellent performance, but it lacks many of the management features of qcow2.

For normal libvirt desktop/server VMs, **qcow2 is a reasonable default**.

---

# 12. Option B: Create a VM from the CLI

For headless systems or SSH-based administration, `virt-install` can create VMs without Virt-Manager.

Example:

```bash
virt-install \
  --name ubuntu-server \
  --memory 1024 \
  --vcpus 1 \
  --disk size=10,format=qcow2 \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics none \
  --location /path/to/ubuntu-server.iso \
  --extra-args "console=ttyS0"
```

This creates a VM named:

```text
ubuntu-server
```

with:

```text
RAM:       1024 MB
vCPU:      1
Disk:      10 GB qcow2
Network:   libvirt default NAT network
Graphics:  Disabled
```

The exact `--os-variant` value should match an operating system variant supported by your installed version of `virt-install`.

You can check available variants with:

```bash
virt-install --osinfo list
```

---

# 13. Step 6: Essential `virsh` Commands

`virsh` is the primary command-line interface for managing libvirt VMs.

## List VMs

Running VMs:

```bash
virsh list
```

All VMs:

```bash
virsh list --all
```

## Start a VM

```bash
virsh start <vm_name>
```

## Gracefully Shut Down a VM

```bash
virsh shutdown <vm_name>
```

This asks the guest OS to shut down normally.

## Force Power Off a VM

```bash
virsh destroy <vm_name>
```

> Despite the name, `destroy` does **not** delete the VM. It is essentially equivalent to pulling the VM's power plug.

## Enable VM Autostart

```bash
virsh autostart <vm_name>
```

## Disable VM Autostart

```bash
virsh autostart --disable <vm_name>
```

## Connect to a Serial Console

For a VM configured with a serial console:

```bash
virsh console <vm_name>
```

To exit the console, the usual escape sequence is:

```text
Ctrl + ]
```

## Delete a VM

To remove the VM definition:

```bash
virsh undefine <vm_name>
```

To also remove its managed storage:

```bash
virsh undefine <vm_name> --remove-all-storage
```

> Be careful with `--remove-all-storage`. It can permanently delete the VM's disks.

---

# 14. Step 7: Storage Management

libvirt can organize VM storage using **storage pools**.

List pools:

```bash
virsh pool-list --all
```

List volumes in a pool:

```bash
virsh vol-list <pool_name>
```

A storage pool can represent a directory, filesystem, logical volume setup, or other storage backend.

This gives libvirt a structured way of managing VM disks instead of treating every disk image as an unrelated file.

---

# 15. Step 8: Install QEMU Guest Agent

The **QEMU Guest Agent (QGA)** runs inside the guest OS and provides a communication channel between the host and guest.

It can enable functionality such as:

* Guest IP address discovery
* Improved shutdown coordination
* Filesystem freeze/thaw operations
* Better host/guest integration

## Debian / Ubuntu Guest

```bash
sudo apt install qemu-guest-agent
```

Then enable it:

```bash
sudo systemctl enable --now qemu-guest-agent
```

## Fedora / RHEL Guest

```bash
sudo dnf install qemu-guest-agent
```

Then:

```bash
sudo systemctl enable --now qemu-guest-agent
```

## Alpine Guest

```bash
apk add qemu-guest-agent
```

The exact service configuration can vary slightly between distributions.

---

# 16. Step 9: Performance Optimization

## Use VirtIO Devices

When configuring a VM, prefer **VirtIO** devices where supported.

Common examples include:

```text
Disk → VirtIO
Network → VirtIO
```

VirtIO provides paravirtualized devices designed specifically for virtualization and generally has much lower overhead than older fully emulated devices.

---

## Use an Appropriate CPU Configuration

For general VMs, libvirt/QEMU can expose a virtual CPU model to the guest.

For performance-sensitive workloads, CPU configuration can be tuned further, including options such as host CPU passthrough.

However, CPU passthrough can reduce VM portability between different physical hosts, so it is not always the best default.

---

## Remove Unnecessary Virtual Hardware

If a server VM does not need certain hardware, remove it.

For example:

* Sound card
* USB devices
* Unused display devices
* Unused controllers
* Other unnecessary peripherals

A minimal headless server can therefore look something like:

```text
1 vCPU
512 MB–1 GB RAM
VirtIO network device
VirtIO disk
No sound
No unnecessary USB devices
No graphical display
QEMU Guest Agent
```

This keeps the VM simple and reduces unnecessary overhead.

---

# 17. Useful Commands at a Glance

| Task                 | Command                          |
| -------------------- | -------------------------------- |
| List running VMs     | `virsh list`                     |
| List all VMs         | `virsh list --all`               |
| Start VM             | `virsh start <vm>`               |
| Shut down VM         | `virsh shutdown <vm>`            |
| Force power off      | `virsh destroy <vm>`             |
| Enable autostart     | `virsh autostart <vm>`           |
| Disable autostart    | `virsh autostart --disable <vm>` |
| Open VM console      | `virsh console <vm>`             |
| Show VM information  | `virsh dominfo <vm>`             |
| Show VM XML          | `virsh dumpxml <vm>`             |
| List networks        | `virsh net-list --all`           |
| List storage pools   | `virsh pool-list --all`          |
| List storage volumes | `virsh vol-list <pool>`          |
| Create VM from CLI   | `virt-install ...`               |
| Launch GUI           | `virt-manager`                   |
| Validate host        | `virt-host-validate`             |

---

# 18. The Mental Model to Remember

If you forget everything else, remember this:

```text
                  Virt-Manager
                       │
                       │ GUI
                       ▼
                    libvirt
                       │
                       │ manages
                       ▼
                     QEMU
                       │
             ┌─────────┴─────────┐
             │                   │
      Virtual Hardware      CPU Virtualization
             │                   │
             │                  KVM
             │                   │
             └─────────┬─────────┘
                       ▼
                  Linux Kernel
                       │
                       ▼
               Physical Hardware
```

### In one sentence:

**Virt-Manager is the GUI, libvirt is the management layer, QEMU is the userspace VM process and virtual hardware provider, and KVM is the Linux kernel subsystem that provides hardware-assisted CPU virtualization.**
