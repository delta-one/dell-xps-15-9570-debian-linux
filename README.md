# Dell XPS 15 9570

***Disclaimer:*** *I'm using Debian Unstable, so some aspects might not apply to your distribution.*

### Configuration
* CPU: Intel® Core™ i7-8750H
* RAM: 16 GB
* HDD: 512 GB
* Video card: NVIDIA® GeForce® GTX 1050Ti
* Screen: FHD non-touch
* WIFI: Killer 1535
* Battery: 97 Whr

### Before installation
2 things need to be done in the BIOS before installing Linux:
* Disable *Secure Boot* to allow Linux to boot.
* Change the SATA Mode to `AHCI` to allow Linux to detect the NVMe SSD.

### Installation
If you need Wifi during installtion, you need to grab an image with non-free firmware, since the official Debian-image doesn't contain the driver for the Killer 1535-chip. During the installation the installer will complain about missing files for the Wifi-firmware, but this warning can be ignored.

### After installation
When you boot the first time, press `e` in Grub and add the kernel-parameter `nouveau.modeset=0` before you boot. This will prevent CPU-lockups when running `lspci` e.g. or when trying to logout or reboot. I'd also strongly recommend to add this parameter permanently in `/etc/default/grub` (don't forget `update-grub2` afterwards).

### Wifi
The drivers needed for the Killer 1535-chip are in the `firmware-atheros`-package, which should be installed if you used an image with non-free firmware. At times the connection is a bit slow though. I might switch to an Intel-chip.


### Video card
The nouveau-driver works, but the dGPU needs quite a lot of power. Will work on getting `bumblebee` to work.

### Battery
My battery initially showed a capacity of 87 Whr. Draining the battery completely until the computer shuts down automatically and then fully recharging it a couple of times (as [suggested by Dell](https://dell.to/2JJejor)) increased the capacity to 94 Whr.

### Touchpad
The touchpad should work out of the box. Which one of the packages `xserver-xorg-input-libinput` and `xserver-xorg-input-synaptics` you need (or possibly both), might depend on your desktop environment. Configuring the touchpad also depends on your DE. In KDE Plasma the touchpad can be configured in the system-settings (mouse-click emulation, gestures, multitouch etc.).

### Fn-keys
This might again depend on your DE. In KDE Plasma the keys for volume, brightness, search and screen worked out of the box, whereas I had to configure the music-player keys in the settings.<br>
**Tip:** `Fn+Insert` puts the computer in suspend-mode.

### Sound
Sound should work out of the box. I haven't really tested the microphone yet though.

### Bluetooth
Worked out of the box.
