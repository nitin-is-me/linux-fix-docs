## This is for installing the minimal kde on Debian (or any other distro), which is working as of now (6.3).

### Step 1: Install bare server distro, I'm choosing debian netinst, without any DE, just ssh and system utility.
### Step 2: After installation, install the `kde-plasma-desktop` package from your distribution's repository.
### Step 3: Reboot. Now you should see the login screen. Enter your password and you're now in KDE Plasma Desktop.
### Step 4: Many times, during server installation, the distribution sets some network config. Which can cause problem to connect using wifi icon in panel. To fix that:
 - Go to `/etc/NetworkManager/NetworkManager.conf` and set managed to true.
 - If that doesn't work, go to /etc/network/interfaces, comment out your interface and reboot. 
