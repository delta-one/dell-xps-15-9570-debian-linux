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
* Disable *Fast Boot* to allow Linux to boot.
* Change the SATA Mode to `AHCI` to allow Linux to detect the NVMe SSD.

### Installation
If you need Wifi during installtion, you need to grab an image with non-free firmware, since the official Debian-image doesn't contain the driver for the Killer 1535-chip. During the installation the installer will complain about missing files for the Wifi-firmware, but this warning can be ignored.

### After installation
When you boot the first time, press `e` in Grub and add the kernel-parameter `nouveau.modeset=0` before you boot. This will prevent CPU-lockups when running `lspci` e.g. or when trying to logout or reboot. I'd also strongly recommend to add this parameter permanently in `/etc/default/grub` (don't forget `update-grub2` afterwards).
