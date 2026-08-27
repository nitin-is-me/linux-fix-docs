# Debian KDE: External Monitor Brightness

## Why this happens

A laptop's built-in display usually exposes a kernel backlight control, so KDE can directly change the panel brightness.

An external monitor uses **DDC/CI** instead. KDE/PowerDevil needs access to the monitor's I²C/DDC interface to control its real hardware brightness. Without that access, KDE may fall back to software dimming or fail to change the monitor at all.

## What to install/configure

```bash
sudo apt install ddcutil
sudo modprobe i2c-dev
sudo usermod -aG i2c $USER
reboot
```

- **ddcutil** — communicates with the monitor using DDC/CI.
- **i2c-dev** — provides the Linux I²C device interface used for DDC.
- **i2c group** — gives your user permission to access `/dev/i2c-*`.

Verify:

```bash
ddcutil detect
ddcutil getvcp 10
```

If the monitor is detected and VCP `0x10` (Brightness) works, KDE/PowerDevil can use the DDC path for normal hardware brightness control.
