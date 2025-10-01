## This is for installing the minimal kde on Debian , which is working as of now (6.3).

### Step 1: Install bare server distro, I'm choosing debian netinst, without any DE, just ssh and system utility.
### Step 2: After installation, install the `kde-plasma-desktop` package from your distribution's repository.
### Step 3: Reboot. Now you should see the login screen. Enter your password and you're now in KDE Plasma Desktop.
### Step 4: Many times, during server installation, the distribution sets some network config. Which can cause problem to connect using wifi icon in panel. To fix that:
 - Go to `/etc/NetworkManager/NetworkManager.conf` and set managed to true.
 - If that doesn't work, go to /etc/network/interfaces, comment out your interface and reboot. 

### Step 5: Some extra fonts can be missing, install them by installing the `fonts-recommended` metapackage.

### Step 6: From version 12, Debian now includes non-free and proprietary firmwares/drivers in the official iso, but you need to tell Debian to use the non-free repo. To do that:
 - Edit the `/etc/apt/sources.list` file.
 - Add `non-free` and `contrib` in every entry of the repo.
 - Run `sudo apt update`
