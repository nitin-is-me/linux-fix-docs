# Understanding QEMU, KVM, Virt-Manager, and Networking

Before we dive in, here’s what each piece does:

- **QEMU** – an open-source machine emulator/virtualizer. It runs guest OSes and can emulate different hardware architectures.
- **KVM** – a Linux kernel module that lets QEMU use hardware virtualization features (Intel VT-x/AMD-V) for *near-native performance*.
- **libvirt** – a service and API that manages QEMU/KVM VMs.
- **Virt-Manager** – a GUI tool built on libvirt that makes creating and managing VMs way easier.
- **Networking** – by default, libvirt creates a virtual NAT network so your VMs can access the internet without touching your host’s network config. You can also create bridged or isolated networks if needed.

---

## Step 1: Install QEMU, KVM, Virt-Manager, and Networking Tools
```bash
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```
- `qemu-kvm`: provides the virtualization backend  
- `libvirt-daemon-system` + `libvirt-clients`: manage VMs without root and provide the `virsh` CLI  
- `bridge-utils`: for advanced networking (bridged mode)  
- `virt-manager`: the graphical VM manager

---

## Step 2: Add Your User to Required Groups
```bash
sudo adduser $USER libvirt
sudo adduser $USER kvm
```
This lets you manage VMs without using `sudo`. **Log out and log back in** (or reboot) for the changes to apply.

---

## Step 3: Verify Installation
```bash
sudo systemctl status libvirtd
```
If it's inactive, run:
```bash
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```
This ensures the libvirt daemon is running and auto-starts on boot.

---

## Step 4: Launch Virt-Manager
```bash
virt-manager
```
Open from your application menu or terminal. Virt-Manager gives you a graphical interface to create and manage VMs.

---

## Step 5: Configure Networking
- **Default NAT network:** Should be active by default. Check with:
```bash
virsh net-list --all
```
If "default" shows as inactive:
```bash
sudo virsh net-start default
sudo virsh net-autostart default
```
- **Bridged network (optional):** Use this if you want VMs to appear as separate devices on your LAN. Requires creating a bridge interface using `bridge-utils` and selecting it inside Virt-Manager.

---

## Step 6: Create Your First Virtual Machine
1. In Virt-Manager, click **Create a new virtual machine**.
2. Choose installation method (ISO, PXE, import disk image).
3. Assign CPU, RAM, and disk space.
4. Finish and install the OS inside the VM.

---

## Step 7: Useful Extras

### QEMU Guest Agent
Improves communication between host and guest:
```bash
sudo apt install qemu-guest-agent
```
Install **inside your VM** for features like graceful shutdown and better host control.

### Snapshots
Take VM snapshots from Virt-Manager: useful for testing without breaking things. Make sure you have enough disk space.

### USB Device Passthrough
Attach USB devices directly in Virt-Manager under **VM → Details → Add Hardware → USB Host Device**.

### CLI Management with Virsh
```bash
virsh list --all          # List VMs
virsh start <vm_name>     # Start a VM
virsh shutdown <vm_name>  # Gracefully shut down
virsh dominfo <vm_name>   # Get VM info
```

