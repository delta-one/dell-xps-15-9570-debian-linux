# Debian Linux on the Dell XPS 15 9570

***Disclaimer:*** *I'm using Debian Unstable, so some aspects might not apply to your distribution.*

### Configuration
* CPU: Intel® Core™ i7-8750H
* RAM: 16 GB
* HDD: 512 GB
* Video card: NVIDIA® GeForce® GTX 1050 Ti
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
In case you want to compile your own kernel, you can use [my kernel-configs](kernel-config) as a base. Depending on what you do, you might have to adjust the configs slightly, but they should provide a working base-configuration and can be used with the [vanilla kernel-sources](https://www.kernel.org/). <br>
**Note:** I have switched the Wifi-chip from the Killer 1535 to an Intel 9260-chip and I'm not using the Nouveau driver, but the proprietary one from NVIDIA.

If you only want to include those modules that you are really using, run a distro-kernel for a while and simply record all the modules you are using with [modprobed-db](https://github.com/graysky2/modprobed-db) for example. You can then take the distro-configuration and run `make localmodconfig` while proving the module-list from *modprobed-db*.

### Wifi + Bluetooth
The drivers needed for the Killer 1535-chip are in the `firmware-atheros`-package, which should be installed if you used an image with non-free firmware. Bluetooth should be working out of the box.<br>
While the speed of the Killer-chip was nothing to complain about, I saw a lot of connection drops and switched the chip with an Intel 9260-card. The drivers for this card are in the `firmware-iwlwifi`-package. If you run the distribution-kernel, you are fine and the new card should then work out of the box. If you run a custom kernel, you need to add a couple of modules (see my [my kernel-configs](kernel-config) for details). `config-4.18.0-rc6-pfd1-nouveau-killer-intel` is my last config, that works with both chipsets. All future kernel-configs will only work with the Intel-chip.

### Power Management
Dell removed the S3 sleep-state with BIOS 1.3.0. If you want to use S3, you need to stay on BIOS 1.2.2.

Suspend works out of the box. Unfortunately there is no indicator, if the computer is in suspend-mode.

### Video card
The integrated Intel-card works out of the box - a bit trickier was the installation of [bumblebee](https://wiki.debian.org/Bumblebee) for the discrete NVIDIA card. I managed to get it working with the proprietary NVIDIA-driver and there are probably several different ways to get it working, but the following worked for me:

* Install `bumblebee-nvidia` for the proprietary NVIDIA-driver as well as the proprietary NVIDIA-driver.
* Deinstall `xserver-xorg-video-nouveau`.
* Install `xserver-xorg-input-mouse`.
* Add the following kernel-parameter to your configuration: `pcie_port_pm=on`
* Edit `/etc/bumblebee/bumblebee.conf` by setting the `Driver` to `nvidia` and by setting `PMMethod` to `none` in the `[driver-nvidia]`-section.
* Add the following snippet to `/etc/bumblebee/xorg.conf.nvidia`:
```
Section "Screen"
    Identifier "Default Screen"
    Device "DiscreteNvidia"
EndSection
```
* If you are using *TLP*, you might need to blacklist the discrete NVIDIA-card by adding/uncommenting the following line in `/etc/default/tlp`:
```
RUNTIME_PM_BLACKLIST="01:00.0"
```
Double-check the address with `lspci`. Similarly, if you are using *powertop*, you might have to disable/blacklist the card there as well.

That should enable the card when running a command with `optirun`:
```
$ glxinfo|grep "OpenGL renderer"
OpenGL renderer string: Mesa DRI Intel(R) UHD Graphics 630 (Coffeelake 3x8 GT2)
$ optirun glxinfo|grep "OpenGL renderer"
OpenGL renderer string: GeForce GTX 1050 Ti with Max-Q Design/PCIe/SSE2
```
It's possible you might have to install additional packages like `libgl1-mesa-glx`.

**Note:** The NVIDIA card is now permanently on, thus drawing some power.

### Battery
My battery initially showed a capacity of 87 Whr. Draining the battery completely until the computer shuts down automatically and then fully recharging it a couple of times (as [suggested by Dell](https://dell.to/2JJejor)) increased the capacity to 91.5 Whr.

### Touchpad
The touchpad should work out of the box. Which one of the packages `xserver-xorg-input-libinput` and `xserver-xorg-input-synaptics` you need (or possibly both), might depend on your desktop environment. Configuring the touchpad also depends on your DE. In KDE Plasma the touchpad can be configured in the system-settings (mouse-click emulation, gestures, multitouch etc.).

### Fn-keys
This might again depend on your DE. In KDE Plasma the keys for volume, brightness, search and screen worked out of the box, whereas I had to configure the music-player keys in the settings.<br>
**Tip:** `Fn+Insert` puts the computer in suspend-mode. `Fn+F7` turns off the screen and mutes the computer (though that option has to be activated in the BIOS).

### Fan control
**INFO: This does not work with BIOS 1.7.0, since the BIOS fan control cannot be overriden atm. You need to stay on BIOS 1.6.0 or lower, if you want a custom fan control.**

The BIOS fan control is a bit too trigger-happy for my taste, i.e. the fan is running too often at low temperatures. Luckily you can set up your own fan control though it requires some configuration:

1. You need to enable the kernel module `I8K`.
2. Install `i8kutils`.
3. `i8kutils` installs a sample confuration at `/etc/i8kmon.conf`, which can be adjusted. My configuration file looks like this:


```
# Sample i8kmon configuration file (/etc/i8kmon.conf, ~/.i8kmon).

# External program to control the fans
set config(i8kfan)      /usr/bin/i8kfan

# Report status on stdout, override with --verbose option
set config(verbose)     0

# Status check timeout (seconds), override with --timeout option
set config(timeout)     2

# Temperature display unit (C/F), override with --unit option
set config(unit)        C

# Temperature threshold at which the temperature is displayed in red
set config(t_high)      80

# Minimum expected fan speed
#set config(min_speed)   2000

# Temperature thresholds: {fan_speeds low_ac high_ac low_batt high_batt}
# These were tested on the I8000. If you have a different Dell laptop model
# you should check the BIOS temperature monitoring and set the appropriate
# thresholds here. In doubt start with low values and gradually rise them
# until the fans are not always on when the cpu is idle.
set config(0) {{0 0} -1 55 -1 55}
set config(1) {{0 1} 50 60 50 60}
set config(2) {{1 1} 55 75 55 75}
set config(3) {{2 2} 70 128 70 128}

# Speed values are set here to avoid i8kmon probe them at every time it starts.
#set status(leftspeed)  "0 1000 2000 3000"
#set status(rightspeed) "0 1000 2000 3000"

# end of file
```
4. Depending on your usage and preferences, you might have to adjust the lines starting with `set config(0)` and specify at which temperatures which fan shall be active and at what speed.
5. Unfortunately the `i8k`-package was not designed for the XPS 15 9570 originally, so the kernel module needs to be force-loaded: `sudo modprobe i8k force=1`
6. Even if the daemon is running, the BIOS will still override it. Therefore the BIOS control must be disabled. An easy way to do that is using the tool [dell-bios-fan-control](https://github.com/TomFreudenberg/dell-bios-fan-control). After installing it the BIOS control can be disabled with `sudo dell-bios-fan-control 0`.
7. Start the `i8k`-daemon with `sudo service i8kmon start` and enjoy more silence.

### Sound
Sound should work out of the box. I haven't really tested the microphone yet though.

### BIOS update
There are 2 ways to update the BIOS:

1. The XPS 15 supports [LVFS](https://fwupd.org/) (Linux Vendor Firmware Service) and thus updates can be installed with `fwupd`. A short tutorial can be found [here](https://github.com/hughsie/fwupd#basic-usage-flow-command-line).
2. Format a USB-drive with FAT32, download and copy the BIOS-file from the [Dell support page](https://www.dell.com/support/home/us/en/19/product-support/product/xps-15-9570-laptop/drivers) (`XPS_9570...exe`) onto the USB-drive and reboot your computer. Press `F12`, choose `BIOS Flash Update` and then choose the downloaded file to start the update.

Here is a (non-comprehensive) list of BIOS-versions for the XPS 15:
* [1.7.0](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverId=1WN0H&osCode=WT64A&productCode=xps-15-9570-laptop)
* [1.6.0](https://www.dell.com/support/home/uk/en/ukdhs1/drivers/driversdetails?driverId=DDNHT&osCode=WT64A&productCode=xps-15-9570-laptop)
* [1.5.0](https://www.dell.com/support/home/uk/en/ukdhs1/drivers/driversdetails?driverid=5g45w)
* [1.4.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=n54j5)
* [1.3.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=82mk9)
* [1.3.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=9d1j8)
* [1.2.2](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=rvxyr)
* [1.1.4](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=r8d1d)


### Undervolting
[Tests have shown](https://www.notebookcheck.net/Dell-XPS-15-9570-i7-UHD-GTX-1050-Ti-Max-Q-Laptop-Review.332758.0.html#toc-performance), that the Intel® Core™ i7-8750H can be undervolted to gain up to 15% more performance under heavy workloads. Under Linux a tool that can undervolt Intel CPUs is [intel-undervolt](https://github.com/kitsunyan/intel-undervolt). My XPS seems to be running stable at -0.150V for the CPU and -0.100V for the GPU.
