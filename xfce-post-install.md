Xfce is pretty lightweight, modular (being able to replace things) and minimalistic dekstop environment, but it's **insanely customizable**. So you can do some stuffs yourselfs to set it up. <br>
I'm not documenting visual customization, because it can be changed really easily. Here's my current Debian Xfce:

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/35196a8c-00ea-4385-82eb-b09c271643cc" />


## Redshift (for eye protection):

1. Install redshift:
   ```
   sudo apt install redshift redshift-gtk
   ```
2. Make a config file named `redshift.conf`:
   ```
   ~/.config/redshift.conf
   ```
3. Paste this inside that file and save it:
   ```
    [redshift]
    temp-day=3500
    temp-night=3500
    brightness=1.0
    gamma=1.0
    transition=0
    location-provider=manual
    
    [manual]
    lat=0
    lon=0
   ```
4. Now start redshift either from start menu or by typing `redshift-gtk` in the terminal.
5. Click on the icon in panel to check if autostart is enabled or not.

## xfce4-clipman (for clipboard history):

1. Install xfce4-clipman
   ```
   sudo apt install xfce4-clipman
   ```
2. Start clipman either from start menu or by typing `clipman` in the terminal.
3. Click on the icon in panel to check if autostart is enabled or not.

## light-locker (in replacement of xfce4-screensaver for auto locking, nothing more):

1. Install light-locker
   ```
   sudo apt install light-locker
   ```
2. That's it, now it's available in the power manager of xfce. We can also download a separate software for managing its settings: `sudo apt install light-locker-settings`, but it's not needed in most cases.

## arc-theme and papirus-icon-theme (or Orchid-theme and Kora icons from internet, these two sets go well along):
 
1. Install arc-theme and papirus-icon-theme
   ```
   sudo apt install arc-theme papirus-icon-theme
   ```
2. Apply them both from appearance settings.

## fonts-noto-color-emoji (for emoji and other characters support):

1. Install fonts-noto-color-emoji:
   ```
   sudo apt install fonts-noto-color-emoji
   ```
2. That's it.

## Enable USB file transfer (if not working):

1. Install these tools:
   ```
   sudo apt install gvfs gvfs-backends gvfs-fuse mtp-tools jmtpfs
   ```
2. That's it! Happy file transferring.

## Customize lock screen (lightdm):

1. Edit `/etc/lightdm/lightdm.conf` to change lock screen behaviour:
   - Uncomment `greeter-hide-users=false`
   - Uncomment `greeter-session=` and set its value to `lightdm-gtk-greeter`
2. Install `lightdm-gtk-greeter-settings`:
   ```
   sudo apt install lightdm-gtk-greeter-settings
   ```
