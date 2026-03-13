# Amlogic boot scripts for Armbian

## Table of Contents
- [Overview](#overview)
- [Setup](#setup)
- [Supported Devices](#supported-devices)
- [Advanced - Bootloader Customization](#advanced---bootloader-customization)

## Overview

The Armbian images for Amlogic TV Boxes use secondary, chain loaded u-boot blobs to boot mainline kernel images.
The vendor u-boot bootloaders can however boot mainline Linux perfectly without them. So they are not needed.

All it takes are some simple modifications of some of the Armbian u-boot scripts.

## Setup
assumption: you have vendor u-boot (the one that came with the box) running on eMMC. If you don't, you can just restore the stock Android image with Amlogic USB Burning tool.

+ **Step 1:** Download latest Armbian for s9xxx-box, let's use [bookworm minimal](https://dl.armbian.com/aml-s9xx-box/Bookworm_current_minimal)  
+ **Step 2:** Burn the image to a USB flash drive  
+ **Step 3:** Copy the modified boot scripts (**[aml_autoscript](https://github.com/devmfc/amlogic-bootscripts-Armbian/blob/main/aml_autoscript)**, **[s905_autoscript](https://github.com/devmfc/amlogic-bootscripts-Armbian/blob/main/s905_autoscript)**, **[emmc_autoscript](https://github.com/devmfc/amlogic-bootscripts-Armbian/blob/main/emmc_autoscript)** ) to the fat partition on the USB drive. Overwrite the existing files.  
+ **Step 4:** If you have a GXBB (S905) or GXL (S905X/W/L) soc, you also need **[gxl-fixup.scr](https://github.com/devmfc/amlogic-bootscripts-Armbian/blob/main/gxl-fixup.scr)**  
+ **Step 5:** Add an armbianEnv.txt file with the following content (file is also on github):  
```bash
extraargs=earlycon=meson,0xfe07a000 console=ttyS0,921600n8 rootflags=data=writeback rw no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 watchdog.stop_on_reboot=0 pd_ignore_unused clk_ignore_unused rootdelay=5
bootlogo=false
verbosity=7
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
console=both

# DTB file for this tvbox
# fdtfile=amlogic/meson-gxl-s905x-nexbox-a95x.dtb
fdtfile=amlogic/meson-sm1-x96-air-gbit.dtb

# set this to the UUID of the root partition (value can be found 
# in /extlinux/extlinux.conf after APPEND root= or with blkid)
rootdev=UUID=92139c84-3871-41d7-a3f2-e8a943cbfa87

# Enable ONLY for gxbb (S905) / gxl (S905X/L/W) to create fake u-boot header
#soc_fixup=gxl-
```
+ **Step 6:** Change *fdtfile* to the DTB for your box.  
+ **Step 7:** (optional since version 3:) Change *rootdev* to the right UUID for the rootfs for your image or change to /dev/sda2 when booting from USB or /dev/mmcblk0p2 when booting from SDCARD  
+ **Step 8:** Only if your box has a GXBB (S905) or GXL (S905X/W/L) soc, uncomment the line *soc_fixup=gxl-*  
+ **Step 9:** Power off the the box.  
+ **Step 10:** Put the USB disk in your box.  
+ **Step 11:** Push the reset button and hold the button  
+ **Step 12:** power up your box while holding the reset button for approx 7 seconds.  
+ **Step 13:** If you're lucky, it will now boot Armbian with a mainline kernel. Without any secondary u-boot blobs.  

## Supported Devices

**✅ Fully Tested & Working:**
- S905X, S905W, S912, S905X2, S922X, S905X3, S905X4 (HTV H8)

**⚠️ Partial Support:**
- S905: Boots only on first attempt (known limitation)

**❓ Untested:**
- S905W2: Likely compatible but untested (not currently supported by Armbian kernel)

All used files and source files can be found on [Github](https://github.com/devmfc/amlogic-bootscripts-Armbian).

---

## Advanced - Bootloader Customization

### ⚠️ Disclaimer & Prerequisites

**DISCLAIMER:** Modifying your device's bootloader can result in a bricked device. Any damage or data loss is your sole responsibility. Proceed only if you understand the risks.

**Required Prerequisites:**

- **Functional ARM Linux System:** Armbian, Debian, or Ubuntu ARM running from USB/SD on your Amlogic device
  - Necessary to access internal eMMC and run analysis/extraction commands
  - System must boot correctly to provide shell access
  
- **Serial TTL Adapter (3.3V UART):** High-quality USB serial adapter
  - ⚠️ **CRITICAL:** Use 3.3V only. 5V will damage the device!
  - Requires soldering skills to connect TX/RX/GND to the board
  
- **Serial Terminal Software:** PuTTY, Minicom, or picocom
  
- **Patience & Methodology:** Follow each step carefully

### 🔒 Mandatory: Backup Your eMMC

Before ANY experiment, create a complete backup:

```bash
# Bit-by-bit backup with compression (saves space)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# To restore in case of disaster:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

Why gzip? A 16GB backup becomes 2-4GB, saving significant space.

### Checking Vendor Bootloader Support

#### Step 1: Connect Serial Cable
Solder TX, RX, GND to your device's UART pads and connect to your PC.

#### Step 2: Open Serial Console
Using picocom as example:

```bash
# Find your serial device
ls -la /dev/ttyUSB*

# Connect at 115200 baud (adjust if different)
picocom -b 115200 /dev/ttyUSB0

# Or with minicom:
minicom -D /dev/ttyUSB0 -b 115200
```

#### Step 3: Interrupt U-Boot
Power on the device and quickly press `Ctrl+C` or `Enter` to interrupt U-Boot before it boots.

#### Step 4: Check Bootloader Variables
Once in U-Boot console, type:

```bash
printenv bootcmd
```

Expected output should be similar to:
```
bootcmd=run start_autoscript; run storeboot
```

Check for related variables:
```bash
printenv start_usb_autoscript
printenv start_mmc_autoscript
printenv start_emmc_autoscript
```

**Note:** Variable names may differ slightly. Look for patterns like `start_*_autoscript`.

### Modifying Vendor Bootloader (Advanced Users Only)

If your bootloader is writable and you want to force script support, execute these commands in U-Boot console:

```bash
setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then setenv devtype "mmc"; setenv devnum 1; autoscr 1020000; fi;'
setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then setenv devtype "mmc"; setenv devnum 0; autoscr 1020000; fi;'
setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then setenv devtype "usb"; setenv devnum 0; autoscr 1020000; fi; done'
setenv bootdelay 1
```

#### ⚠️ CRITICAL: Setting bootcmd - Preserve Your Original Command

**DO NOT simply use `run start_autoscript; run storeboot`** - This is generic and may brick your device if your original bootcmd was different!

**Step-by-step approach:**

1. **First, WRITE DOWN your original bootcmd:**
   ```bash
   printenv bootcmd
   # Write down the EXACT output here:
   # _________________________________
   ```

2. **Then set bootcmd to preserve it:**
   ```bash
   setenv bootcmd 'run start_autoscript; [PASTE YOUR ORIGINAL BOOTCMD HERE]'
   ```

**Examples from real devices:**

**Example 1 - Generic Amlogic Box:**
```bash
# Original was:
# bootcmd=run storeboot

# So you do:
setenv bootcmd 'run start_autoscript; run storeboot'
```

**Example 2 - Different Vendor (HTV H8):**
```bash
# Original was:
# bootcmd=run start_emmc_autoscript; run storeboot

# So you do:
setenv bootcmd 'run start_autoscript; run start_emmc_autoscript; run storeboot'
```

**Example 3 - Complex Bootcmd:**
```bash
# Original was:
# bootcmd=if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot

# So you do:
setenv bootcmd 'run start_autoscript; if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot'
```

#### Final step - Save & Verify Changes

```bash
saveenv
reset
```

Interrupt U-Boot again and verify variables were saved:

```bash
printenv bootcmd
```

**Success indicators:**
- Variables were saved → Bootloader is writable and modifications should work
- Variables weren't saved → Read-only bootloader; cannot apply this method

---
