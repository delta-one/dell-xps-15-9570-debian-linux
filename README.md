# Debian Linux on the Dell XPS 15 9570

***Disclaimer:*** *I'm using Debian Testing [as of April 2020], so some aspects might not apply to your distribution.*

### Configuration
* CPU: Intel® Core™ i7-8750H
* RAM: 16 GB
* HDD: 512 GB
* Video card: NVIDIA® GeForce® GTX 1050 Ti
* Screen: FHD non-touch
* WIFI: ~~Killer 1535~~ ~~Intel 9260~~ Killer 1535
* Battery: 97 Whr

### Before installation
3 things need to be done in the BIOS before installing Linux:
* Disable *Secure Boot* to allow Linux to boot.
* Change the SATA Mode to `AHCI` to allow Linux to detect the NVMe SSD.
* Change Fastboot to `Thorough` in *POST Behaviour*.

Debian has support for Secure Boot and you can install Debian with Secure Boot enabled. However Secure Boot has certain limitations and might require to be disabled. For details, see the [Debian Wiki](https://wiki.debian.org/SecureBoot).

### Installation
If you need Wifi during installtion, you need to grab an image with non-free firmware, since the official Debian-image doesn't contain the driver for the Killer 1535-chip (or maybe another chip if you have switched). During the installation the installer may complain about missing files for the Wifi-firmware, but this warning can be ignored.

### After installation
If you run into CPU lockups when e.g. running `lspci` or when your computer won't restart/successfully logout, you can add the kernel parameter `nouveau.modeset=0`, which should fix these issues.

### Kernel
In case you want to compile your own kernel, you can use [my kernel-configs](kernel-config) as a base. Depending on what you do, you might have to adjust the configs slightly, but they should provide a working base-configuration and can be used with the [vanilla kernel-sources](https://www.kernel.org/). <br>
**Note:** The configs work with both the Killer 1535-chip and the Intel 9260-chip. I'm also not using the Nouveau driver, but the proprietary one from NVIDIA.

If you only want to include those modules that you are really using, run a distro-kernel for a while and simply record all the modules you are using with [modprobed-db](https://github.com/graysky2/modprobed-db) for example. You can then take the distro-configuration and run `make localmodconfig` while proving the module-list from *modprobed-db*.

### Kernel parameters
The following kernel parameters can be useful:

* `loglevel=2` -- suppresses some error messages
* `acpi_rev_override=1 acpi_osi=Linux` -- makes some chnages to how ACPI works
* `mem_sleep_default=deep` -- uses a more efficient suspend mode. However the CPU may get stuck in a high power state after resuming. Clicking the touchpad after resuming should pull the CPU out of this state as described [here](https://www.dell.com/community/Precision-Mobile-Workstations/High-load-heating-up-fan-noise-after-S3-suspend-and-resume-on/td-p/7441933).
* `nouveau.modeset=0 nouveau.runpm=0` -- prevents the Nouveau-driver from managing the Nvidia card and disables the power management
* `scsi_mod.use_blk_mq=1` -- enables block multiqueue for better NVMe performance
* `pcie_aspm=force` -- enables Active-State Power Management, which sets a lower power state for PCIe links when the devices to which they connect are not in use
* `drm.vblankoffdelay=1` -- reduces wakeup events and saves minimal power

### Wifi + Bluetooth
The drivers needed for the Killer 1535-chip are in the `firmware-atheros`-package, which should be installed if you used an image with non-free firmware. Bluetooth should be working out of the box.<br>
While the speed of the Killer-chip was nothing to complain about, I saw a lot of connection drops and switched the chip with an Intel 9260-card. The drivers for this card are in the `firmware-iwlwifi`-package. If you run the distribution-kernel, you are fine and the new card should then work out of the box. If you run a custom kernel, you need to add a couple of modules (see my [my kernel-configs](kernel-config) for details). `config-4.18.0-rc6-pfd1-nouveau-killer-intel` is my last config, which works with both chipsets.<br>
However, the latest firmware-package available in Debian for the Intel-chip (`firmware-iwlwifi_20190717-2`) unfortunately borks the Wifi completely (Wifi crashes completely + significant speed-loss). Fixes mentioned [here](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=940813) provide no more than a short temporary fix, so I went back to the Killer-chip. So far the connection seems stable and I haven't had the dropd I had earlier. All kernel-configs from 5.4.6 on will work with both the Killer 1535-chip and the Intel 9260-chip.

### Power Management
Dell removed the S3 sleep-state with BIOS 1.3.0. If you want to use S3, you need to stay on BIOS 1.2.2, though I find the current standby-state does its job well enough.

Suspend works out of the box. Unfortunately there is no indicator, if the computer is in suspend-mode.

### Video card
The integrated Intel card works out of the box - a bit trickier was the installation of [bumblebee](https://wiki.debian.org/Bumblebee) for the discrete NVIDIA card. I managed to get it working with the proprietary NVIDIA-driver and there are probably different ways to get it working, but the following worked for me. Credit goes to the people on the [Arch forum](https://bbs.archlinux.org/viewtopic.php?pid=1826641#p1826641).

* Install the following packages: `apt install bumblebee-nvidia primus-nvidia nvidia-smi` (This should pull all necessary packages. The packgae `nvidia-smi` is not necessarily needed, but used in the script to enable to NVIDIA card.)
* Add your user to the `bumblebee`-group: `usermod -a -G bumblebee YOURUSERNAME`
* Edit `/etc/bumblebee/bumblebee.conf` in the `[driver-nvidia]`-section:
```
[driver-nvidia]
KernelDriver=nvidia
PMMethod=none
```
* Create the file `/etc/tmpfiles.d/nvidia_pm.conf` and add the following to allow the GPU to poweroff on boot:
```
w /sys/bus/pci/devices/0000:01:00.0/power/control - - - - auto
```
* Create the file `/etc/X11/xorg.conf.d/01-noautogpu.conf` and add the following:
```
Section "ServerFlags"
	Option "AutoAddGPU" "off"
EndSection
```
* Create the file `/etc/X11/xorg.conf.d/20-intel.conf` and add the following:
```
Section "Device"
	Identifier  "Intel Graphics"
	Driver      "modesetting"
EndSection
```
* Several modules need to be blacklisted in order to prevent them from being loaded on boot. Add the following to `/etc/modprobe.d/blacklist.conf`:
```
blacklist nouveau
blacklist rivafb
blacklist nvidiafb
blacklist rivatv
blacklist nv
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
blacklist nvidia-uvm
blacklist ipmi_msghandler
blacklist ipmi_devintf
```
* Create the file `/etc/modprobe.d/disable-nvidia.conf` and add the following:
``` bash
install nvidia /bin/false
```
* If you are using *TLP*, you might need to blacklist the discrete NVIDIA card by adding/uncommenting the following line in `/etc/default/tlp`:
```
RUNTIME_PM_BLACKLIST="01:00.0"
```
Double-check the address with `lspci`. Similarly, if you are using *powertop*, you might have to disable/blacklist the card there as well.
* In order to enable and disable the video card create the following 2 scripts:
#### `enableGPU.sh`
``` bash
#!/bin/sh
# allow to load nvidia module
if [ ! -f /etc/modprobe.d/disable-nvidia.conf ]; then
	printf "File /etc/modprobe.d/disable-nvidia.conf does not exist.\n"
	printf "Is the GPU already enabled ?\n"
	exit 1
fi
printf "Allowing to load NVIDIA modules...\n"
mv /etc/modprobe.d/disable-nvidia.conf /etc/modprobe.d/disable-nvidia.conf.disable
printf "Changing power control...\n"
# remove NVIDIA card (currently in power/control = auto)
echo -n 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
sleep 1
# change PCIe power control
echo -n on > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1
# rescan for NVIDIA card (defaults to power/control = on)
echo -n 1 > /sys/bus/pci/rescan
if [ -x "$(command -v nvidia-smi)" ]; then
	printf "\n"
	nvidia-smi
fi
printf "\nNVIDIA CARD IS NOW ENABLED.\n"
```
#### `disableGPU.sh`
``` bash
#!/bin/sh
printf "Unloading NVIDIA modules...\n"
modprobe -r nvidia_drm
modprobe -r nvidia_uvm
modprobe -r nvidia_modeset
modprobe -r nvidia
printf "Changing power control...\n"
# change NVIDIA card power control
echo -n auto > /sys/bus/pci/devices/0000\:01\:00.0/power/control
sleep 1
# change PCIe power control
echo -n auto > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1
# lock system from loading nvidia module
if [ -f /etc/modprobe.d/disable-nvidia.conf.disable ]; then
	mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf
	printf "\nNVIDIA CARD IS NOW DISABLED.\n"
else
	printf "\nFile /etc/modprobe.d/disable-nvidia.conf.disable does not exist.\n"
	printf "Is the GPU already disabled ?\n"
fi
```
* If the video card is not disabled on shutdown, then the modules will be loaded again at next boot even though they are blacklisted. Therefore we need to create a service which shuts down the NVIDIA card at shutdown. Create the file `/etc/systemd/system/disable-nvidia-on-shutdown.service` and add the following:
``` bash
[Unit]
Description=Disables Nvidia GPU on OS shutdown
[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/true
ExecStop=/bin/bash -c "mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf || true"
[Install]
WantedBy=multi-user.target
```
* Finally we need to enable the service:
``` bash
systemctl daemon-reload
systemctl enable disable-nvidia-on-shutdown
```
* Reboot.

After rebooting, doublecheck that the `nvidia`-module is not loaded: `lsmod | grep nvidia`.<br>
Now you can enable the NVIDIA card by running the aforementioned script `enableGPU.sh`. If you did install `nvidia-smi`, then the script will verify if the Nvidia card is enabled. Otherwise you will have to do so manually if you wish. Finally you can run a command with `optirun` :
``` bash
$ glxinfo | grep "OpenGL renderer"
OpenGL renderer string: Mesa DRI Intel(R) UHD Graphics 630 (Coffeelake 3x8 GT2)

$ optirun glxinfo | grep "OpenGL renderer"
OpenGL renderer string: GeForce GTX 1050 Ti with Max-Q Design/PCIe/SSE2
```
Disable the card with `disableGPU.sh` to lower the power consumption.

#### Various options for the integrated Intel-card
*tlp* is recommended to save some power. However several more options can be activated for the Intel card in order to save power or prevent screen flickering. Add the following line to `/etc/modprobe.d/i915.conf` :
```
options i915 enable_fbc=1 disable_power_well=0 fastboot=1 enable_psr=0
```
Some guides suggest the option `enable_guc=3`, however my computer got stuck at boot with that option. Before you add it to `/etc/modprobe.d/i915.conf`, try it first as a command-line option before you boot.

### Battery
My battery initially showed a capacity of 87 Whr. Draining the battery completely until the computer shuts down automatically and then fully recharging it a couple of times (as [suggested by Dell](https://dell.to/2JJejor)) increased the capacity to 91.5 Whr.

### Touchpad
The touchpad should work out of the box. Depending on how much you want to configure your touchpad, you might need to install additional packages like `xserver-xorg-input-multitouch` or `xserver-xorg-input-synaptics`.

### Fn-keys
This might depend on your desktop environment. In KDE Plasma the keys for volume, brightness, search and screen worked out of the box, whereas I had to configure the music-player keys in the settings.<br>
**Tip:** `Fn+Insert` puts the computer in suspend-mode. `Fn+F7` turns off the screen and mutes the computer (though that option has to be activated in the BIOS).

### Fan control
**INFO: This does not work with BIOS 1.7.0 or 1.8.1, but works again with BIOS 1.9.1 or higher.**

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
set config(1) {{1 0} 50 60 50 60}
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
Sound should work out of the box incl. the internal microphone. Recording audio via the microphone of a headset also works out of the box, though it might require mixer adjustments.

### Card Reader
The SD-card reader should work out of the box. The device will be `/dev/mmcblk0` and the partitions are called `/dev/mmcblk0p1` and so on. They can then be mounted as usual.

### BIOS update
There are 2 ways to update the BIOS:

1. The XPS 15 supports [LVFS](https://fwupd.org/) (Linux Vendor Firmware Service) and thus updates can be installed with `fwupd`. A short tutorial can be found [here](https://github.com/hughsie/fwupd#basic-usage-flow-command-line).
2. Download the BIOS-file from the [Dell support page](https://www.dell.com/support/home/us/en/19/product-support/product/xps-15-9570-laptop/drivers) (`XPS_9570...exe`) and copy it into `/boot/efi/EFI/Dell/Bios` (the path may vary slightly). Reboot, press `F12`, choose `BIOS Flash Update` and then choose the downloaded file to start the update.

Here is a (non-comprehensive) list of BIOS-versions for the XPS 15:
* [1.16.2](https://www.dell.com/support/home/en-us/drivers/driversdetails?driverid=p6pcn)
* [1.15.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=gxff7)
* [1.14.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=9d5fr)
* [1.13.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=fpcc0)
* [1.12.0](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverid=WCGY9)
* [1.11.2](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=m6ffy)
* [1.10.1](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverid=kkwch)
* [1.9.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=wghf7)
* [1.8.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=n61vd)
* [1.7.0](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverId=1WN0H)
* [1.6.0](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverId=DDNHT)
* [1.5.0](https://www.dell.com/support/home/us/en/19/drivers/driversdetails?driverid=5g45w)
* [1.4.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=n54j5)
* [1.3.1](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=82mk9)
* [1.3.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=9d1j8)
* [1.2.2](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=rvxyr)
* [1.1.4](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=r8d1d)
* [1.0.5](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=tty8p)
* [1.0.0](https://www.dell.com/support/home/us/en/04/drivers/driversdetails?driverid=312d7)


### Undervolting
[Tests have shown](https://www.notebookcheck.net/Dell-XPS-15-9570-i7-UHD-GTX-1050-Ti-Max-Q-Laptop-Review.332758.0.html#toc-performance), that the Intel® Core™ i7-8750H can be undervolted to gain up to 15% more performance under heavy workloads. Under Linux a tool that can undervolt Intel CPUs is [intel-undervolt](https://github.com/kitsunyan/intel-undervolt). My XPS seems to be running stable at -0.150V for the CPU and -0.100V for the GPU.
