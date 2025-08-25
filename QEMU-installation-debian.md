Before diving into the steps, it’s important to understand the roles of QEMU, KVM, and Virt-Manager. QEMU is an open-source machine emulator and virtualizer that allows you to run operating systems and software designed for a different architecture. KVM is a virtualization module in the Linux kernel that allows the kernel to function as a hypervisor. Lastly, Virt-Manager is a graphical interface for managing virtual machines through libvirt.

## Step 1: Install QEMU and Virt-Manager:
Install QEMU, KVM, and Virt-Manager to set up your virtualization environment. These tools will allow you to create and manage virtual machines with ease. Execute the following command:
```
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

## Step 2: Add Your User to Necessary Groups:
To manage virtual machines without root privileges, add your user to the ‘libvirt’ and ‘kvm’ groups by running:
```
sudo adduser $USER libvirt
sudo adduser $USER kvm
```

## Step 3: Verify Installation:
Check that the libvirt service is running with:
```
sudo systemctl status libvirtd
```
If it’s not running, start and enable it at boot with:
```
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```
This step ensures that the libvirt daemon is active and set to start automatically on boot, which is necessary for managing virtual machines.

## Step 4: Launch Virt-Manager:
Open Virt-Manager either from your applications menu or by executing the following command in a terminal:
```
virt-manager
```
Virt-Manager provides a user-friendly graphical interface for creating, configuring, and managing virtual machines. It simplifies the process of VM management, making it accessible even to those new to virtualization.

## Step 5: Create a New Virtual Machine:
In Virt-Manager, click on the “Create a new virtual machine” button. Follow the wizard to select the installation method (ISO image, network installation, or importing existing disk), allocate resources (CPU, memory, disk space), and complete the setup by installing the operating system.

## Step 6: Using Your Virtual Machine:
Once the operating system installation is complete, you can start, stop, pause, and configure your virtual machine settings through Virt-Manager. Additionally, you can connect to the VM’s console to interact with it directly.

---

# Extras

After setting up your virtualization environment with QEMU, KVM, and Virt-Manager, there are several advanced features and best practices you can employ to enhance your virtual machine management and performance. Here are some tips and tricks to help you get the most out of your virtualized setup.

## Utilizing QEMU Guest Agent
The QEMU Guest Agent facilitates improved communication between the host and the guest VMs, enabling functionalities such as file transfers, graceful shutdowns, and system information queries. To take advantage of these features, install the guest agent in your VMs:
```
sudo apt install qemu-guest-agent
```
Once installed, make sure the guest agent service is enabled and running within your VMs for enhanced performance and management capabilities.

## Mastering Snapshots
Snapshots are a powerful feature that allows you to save the state of a VM at any given point in time. This is incredibly useful for testing software or updates without risking your primary system state. To create a snapshot in Virt-Manager, select your VM, click on the “Snapshots” section, and then “Take Snapshot”. Always ensure you have enough disk space available before taking snapshots to avoid performance degradation.

## Keyboard Shortcuts and Sending Keys
Virt-Manager provides keyboard shortcuts for common actions such as sending the ‘Ctrl+Alt+Delete’ command to a VM. This can be done through the VM’s menu in Virt-Manager. Familiarizing yourself with these shortcuts can significantly streamline your workflow.

## Attaching USB Devices
To attach USB devices directly to a VM, go to the VM’s hardware details in Virt-Manager, add a USB host device, and select the device you wish to attach. This is particularly useful for software testing on different operating systems or accessing data stored on external drives directly within your VM.

## Command-Line Administration
While Virt-Manager provides a user-friendly GUI for managing VMs, knowing how to perform tasks from the command line can be invaluable, especially for remote administration or scripting. Here are some basic commands:
- List all running VMs: `sudo virsh list --all`
- Start a VM: `sudo virsh start vm_name`
- Shutdown a VM: `sudo virsh shutdown vm_name`
- Get VM information: `sudo virsh dominfo vm_name`
