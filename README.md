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
3 things need to be done in the BIOS before installing Linux:
* Disable *Secure Boot* to allow Linux to boot.
* Change the SATA Mode to `AHCI` to allow Linux to detect the NVMe SSD.
* Change Fastboot to `Thorough` in *POST Behaviour*.

### Installation
If you need Wifi during installtion, you need to grab an image with non-free firmware, since the official Debian-image doesn't contain the driver for the Killer 1535-chip (or maybe another chip if you have switched). During the installation the installer will complain about missing files for the Wifi-firmware, but this warning can be ignored.

### After installation
If you run into CPU lockups when e.g. running `lspci` or when your computer won't restart/successfully logout, you can add the kernel parameter `nouveau.modeset=0`, which should fix these issues.

### Kernel
In case you want to compile your own kernel, you can use [my kernel-configs](kernel-config) as a base. Depending on what you do, you might have to adjust the configs slightly, but they should provide a working base-configuration and can be used with the [vanilla kernel-sources](https://www.kernel.org/).

If you only want to include those modules that you are really using, run a distro-kernel for a while and simply record all the modules you are using with [modprobed-db](https://github.com/graysky2/modprobed-db) for example. You can then take the distro-configuration and run `make localmodconfig` while proving the module-list from *modprobed-db*.

### Wifi + Bluetooth
The drivers needed for the Killer 1535-chip are in the `firmware-atheros`-package, which should be installed if you used an image with non-free firmware. Bluetooth should be working out of the box.<br>
While the speed of the Killer-chip was nothing to complain about, I saw a lot of connection drops and switched the chip with an Intel 9260-card. The drivers for this card are in the `firmware-iwlwifi`-package. ~~not in the Debian archives yet,
so you need to copy it manually into `/lib/firmware` (the Wifi-firmware) and `/lib/firmware/intel` (the Bluetooth-firmware). The details are in [this bug-report](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=899101).~~ If you run the distribution-kernel, you are fine and the new card should then work out of the box. If you run a custom kernel, you need to add a couple of modules (see my [my kernel-configs](kernel-config) for details). `config-4.18.0-rc6-pfd1-nouveau-killer-intel` is the my config, that works with both chipsets. All future kernel-configs will only work with the Intel-chip.

### Power Management
~~[upower](https://packages.debian.org/sid/upower) has a bug in version `0.99.8-1`, which prevents recognizing if the AC adapter gets plugged in or out (this affects KDE Plasma, Gnome and possibly even more desktop environments). One solution is to downgrade [upower](https://snapshot.debian.org/package/upower/0.99.7-2/#upower_0.99.7-2) and [libupower-glib3](https://snapshot.debian.org/package/upower/0.99.7-2/#libupower-glib3_0.99.7-2) to version `0.99.7-2`, then the AC adapter should get recognized again and your power settings should be applied accordingly. Alternatively you can stay on `0.99.8-1` and follow the advice in [this bugreport](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=902644#24).~~ Fixed in *upower 0.99.8-2*.

Dell removed the S3 sleep-state with BIOS 1.3.1. If you want to use S3, you need to stay on BIOS 1.2.2.

### Suspend
Suspend works out of the box. Unfortunately there is no indicator, if the computer is in suspend-mode.

### Video card
The nouveau-driver works, but the dGPU needs some power even when it's idle. One alternative is to install [bumblebee](https://wiki.debian.org/Bumblebee) to switch off the dGPU, when it's not needed.<br>
While installing went without any problems, I haven't been able to get the dGPU working with `optirun`:
```
[ERROR]Cannot access secondary GPU - error: Could not enable discrete graphics card

[ERROR]Aborting because fallback start is disabled.
```
I tried several kernel parameters like `pcie_port_pm=off`, but I'm always getting the same result. The only good news is, that the card is turned off and not using any power. Since I'm not really using the dGPU anyway, I can live with that for now, but I might investigate the issue again in the future.

### Battery
My battery initially showed a capacity of 87 Whr. Draining the battery completely until the computer shuts down automatically and then fully recharging it a couple of times (as [suggested by Dell](https://dell.to/2JJejor)) increased the capacity to 94 Whr.

### Touchpad
The touchpad should work out of the box. Which one of the packages `xserver-xorg-input-libinput` and `xserver-xorg-input-synaptics` you need (or possibly both), might depend on your desktop environment. Configuring the touchpad also depends on your DE. In KDE Plasma the touchpad can be configured in the system-settings (mouse-click emulation, gestures, multitouch etc.).

### Fn-keys
This might again depend on your DE. In KDE Plasma the keys for volume, brightness, search and screen worked out of the box, whereas I had to configure the music-player keys in the settings.<br>
**Tip:** `Fn+Insert` puts the computer in suspend-mode. `Fn+F7` turns off the screen and mutes the computer (though that option has to be activated in the BIOS).

### Sound
Sound should work out of the box. I haven't really tested the microphone yet though.

### BIOS update
Format a USB-drive with FAT32, download and copy the BIOS-file from the Dell support page (`XPS_9570...exe`) onto the USB-drive and reboot your computer. Press `F12`, choose `BIOS Flash Update` and then choose the downloaded file to start the update.
