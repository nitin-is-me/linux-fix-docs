# Installing macOS (Catalina) on QEMU/KVM via virt-manager

So this assumes you've already installed qemu/kvm on your linux. Also I'm installing Catalina because this had .xml in Apple's official repo.

## Step 1: Clone OSX-KVM

This clones the repo to set up the mac fetching for us.

```bash
git clone https://github.com/kholia/OSX-KVM.git
cd OSX-KVM
```

## Step 2: Grab the actual macOS image

```bash
python3 fetch-macOS-v2.py
```

Picked **Catalina (option 3)** again because of the same reasons, it has .xml file.
This downloads `BaseSystem.dmg` + `.chunklist` directly.

## Step 3: Convert the dmg to img

```bash
dmg2img BaseSystem.dmg BaseSystem.img
```

QEMU can't boot a raw `.dmg`, needs it as an `.img`.

## Step 4: Set up the VM in virt-manager (GUI)

**Enable XML editing first:** Edit -> Preferences -> check "Enable XML editing".

**New VM -> Import existing disk image** (not Local install media because macOS doesn't
come as a normal bootable ISO, it boots through OpenCore (bootloader like grub in linux) instead which then boots to the BaseSystem.img).

- Point it at `OpenCore/OpenCore.qcow2` from the repo
- OS type: **Generic or unknown OS** (no idea why isn't it listed lol)
- Allocate memory and cores

**Customize before install -> Overview tab:**
- Chipset: `i440FX` → **`Q35`** (newer chipset, works properly with UEFI)
- Firmware: BIOS → **UEFI x86_64 (OVMF)** (crazy you can run uefi on a pc which itself uses legacy bios lol).

**Storage: ended up with 3 SATA disks total:**
1. OpenCore boot disk (bootloader) (was IDE by default, changed to SATA)
2. `BaseSystem.img` added as SATA. This is the actual installer.
3. A blank qcow2 disk (created via Add Hardware → new storage volume) — this
   is where macOS actually gets permanently installed

All three need to be bus type SATA specifically since macOS doesn't ship IDE or
VirtIO storage drivers.

**NIC:** virt-manager will likely default this to `e1000e` or `virtio`, doesn't
matter what it picks here, it gets replaced entirely in Step 5 by `vmxnet3` (which wasn't available in GUI for me).

## Step 5: XML editing part (the actual "why does macOS need this" part)

Because Windows/Linux can run on literally any PC hardware, no checks. macOS can
only run on genuine Apple hardware and it checks CPU vendor
strings, an "OSK" key baked into real Mac firmware, SMBIOS data claiming to be
a real Mac model. Editing this isn't available through GUI.

Pulled the actual reference XML from the repo
(`macOS-libvirt-Catalina.xml`).

Edits to the auto-generated virt-manager XML:

**1. Add the namespace declaration** to the top `<domain>` tag (without this
libvirt has no idea what `<qemu:commandline>` even is):

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
```

**2. Remove `<cpu mode="host-passthrough"/>`** because it conflicts with manually
specifying `-cpu Penryn` below, and CPU spoofing has to be the Penryn string,
not the raw real CPU.

**3. Fix the input devices :** virt-manager defaults to PS/2 keyboard/mouse.
PS/2 works fine at the OpenCore bootloader, but macOS's
kernel doesn't bind a driver to PS/2 once it finishes booting,
so keyboard/mouse input dies right after boot. Use USB for both instead:

```xml
<input type="tablet" bus="usb">
  <address type="usb" bus="0" port="1"/>
</input>
<input type="keyboard" bus="usb">
  <address type="usb" bus="0" port="4"/>
</input>
```

Delete any `<input type="mouse" bus="ps2"/>` and `<input type="keyboard" bus="ps2"/>`
lines entirely.

**4. Fix the network interface.** virt-manager's default NIC models
(`e1000e`, `virtio`) either have no macOS driver at all (`virtio`) or don't get
recognized as "built-in" ethernet by macOS, so `ifconfig`
inside macOS shows no `en0` interface. The xml reference from the repo suggested this:

```xml
<interface type="network">
  <mac address="52:54:00:c5:38:5e"/>
  <source network="default"/>
  <model type="vmxnet3"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x06" function="0x0"/>
</interface>
```

- `model type="vmxnet3"` — this is the NIC model the official OSX-KVM
  `macOS-libvirt-Catalina.xml` actually ships with, not `e1000`/`e1000e`.
- The PCI address matters just as much as the model: it must sit directly on
  `bus="0x00"`, **not** behind a `pcie-root-port`
  (which is where virt-manager normally places NICs, you'd see something
  like `bus="0x02"` there instead). Pick any slot on bus `0x00` that isn't already used by
  another device in your XML (check for conflicts: `0x1d`, `0x1f`, `0x02`,
  and `0x01` were already taken in mine, so I used `0x06`).

**5. Add this block right before `</domain>`:**

```xml
<qemu:commandline>
  <qemu:arg value="-device"/>
  <qemu:arg value="isa-applesmc,osk=ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc"/>
  <qemu:arg value="-smbios"/>
  <qemu:arg value="type=2"/>
  <qemu:arg value="-cpu"/>
  <qemu:arg value="Penryn,kvm=on,vendor=GenuineIntel,+invtsc,vmware-cpuid-freq=on,+ssse3,+sse4.2,+popcnt,+avx,+aes,+xsave,+xsaveopt,check"/>
</qemu:commandline>
```

## Done, now install and boot
