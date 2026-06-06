# Amlogic Boot Scripts for Armbian

**Language / Idioma:** [🟢 English](README.en.md) | [Português](README.md)

## Table of Contents
- [Overview](#overview)
- [Setup](#setup)
- [Supported Devices](#supported-devices)
- [Advanced Troubleshooting — Bootloader Modification](#advanced-troubleshooting--bootloader-modification)
- [How It Works Internally](#how-it-works-internally)

---

## Overview

Armbian images for Amlogic TV Boxes normally rely on secondary u-boot blobs to boot the mainline kernel. In practice, these are unnecessary: the factory u-boot that came with your box is already capable of doing this on its own. All it takes are a few modifications to the Armbian boot scripts.

> **Prerequisite:** the vendor u-boot must be running on eMMC. If your box was reflashed with a different bootloader, restore the stock Android image using the [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool) before continuing.

---

## Setup

### Step 1 — Download the Armbian image

Download the latest Armbian for s9xxx-box. We recommend [bookworm minimal](https://dl.armbian.com/aml-s9xx-box/Bookworm_current_minimal).

---

### Step 2 — Prepare the installation media

Flash the image to the USB drive using **[balenaEtcher](https://etcher.balena.io/)** — the simplest option — or via command line:

```bash
sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
```

> ⚠️ Replace `/dev/sdX` with your USB drive. Use `lsblk` or `fdisk -l` to confirm the correct device. With `dd`, writing to the wrong device will erase its data without any confirmation prompt.

Mount the FAT partition of the USB drive and replace the boot scripts with the modified ones, overwriting the existing files:

- **[aml_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/aml_autoscript)**
- **[s905_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/s905_autoscript)**
- **[emmc_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/emmc_autoscript)**
- **[gxl-fixup.scr](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/gxl-fixup.scr)** *(only if your SoC is GXBB/S905 or GXL/S905X/W/L)*

> **Compiling your own images or preparing multiple USB drives?** You can modify the `.img` file directly before flashing, avoiding the need to edit each drive individually. Mount the image with `losetup`:
>
> ```bash
> sudo losetup -fP Armbian_*.img
> lsblk | grep loop          # identify the device and the FAT partition (usually loopXp1)
> sudo mount /dev/loop0p1 /mnt/armbian_boot
> ```
>
> Copy the scripts normally to `/mnt/armbian_boot/` and **continue through the next steps as usual**, editing `armbianEnv.txt` and other parameters with the image still mounted. Only after completing all configurations, unmount and flash:
>
> ```bash
> sudo umount /mnt/armbian_boot
> sudo losetup -d /dev/loop0
> sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
> ```

---

### Step 3 — Configure `armbianEnv.txt`

The `armbianEnv.txt` file controls essential boot parameters. Before editing, back up the original:

```bash
sudo cp /mnt/your_usb/armbianEnv.txt /mnt/your_usb/armbianEnv.txt.bak
```

Edit with nano or your preferred editor:

```bash
sudo nano /mnt/your_usb/armbianEnv.txt
```

**Reference content:**

```bash
extraargs=earlycon=meson,0xfe07a000 console=ttyS0,921600n8 rootflags=data=writeback rw no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 watchdog.stop_on_reboot=0 pd_ignore_unused clk_ignore_unused rootdelay=5
bootlogo=false
verbosity=7
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
console=both

# DTB file for this tvbox
# fdtfile=amlogic/meson-gxl-s905x-nexbox-a95x.dtb
fdtfile=amlogic/meson-sm1-x96-air-gbit.dtb

# set this to the UUID of the root partition (value can be found with blkid or in fstab)
#rootdev=UUID=92139c84-3871-41d7-a3f2-e8a943cbfa87
# or use the default partition label:
#rootdev=LABEL=ROOTFS

# Enable ONLY for gxbb (S905) / gxl (S905X/L/W) to create fake u-boot header
#soc_fixup=gxl-
```

> ⚠️ **This file is a starting point, not a universal configuration.**
>
> The content above works for many devices, but may not work for yours. Different TV boxes, SoCs, and Armbian versions may require different parameters — especially the `extraargs` line.
>
> **Before replacing the file**, compare it against the original `armbianEnv.txt` from the Armbian image and merge carefully. Parameters present in the original and absent here may be required for your hardware. When in doubt, start from the original and apply only the changes you understand. If the system fails to boot, restoring the backup (`armbianEnv.txt.bak`) is the first step to diagnose the issue.

---

### Step 4 — Set the correct `fdtfile`

Change the `fdtfile` line to the DTB that matches your box. Available files can be found in `/boot/dtb/amlogic/` inside the Armbian image.

---

### Step 5 — Set `rootdev` *(optional since version 3)*

By default, `rootdev` is commented out and the system uses the `ROOTFS` label automatically. If you need to specify it manually:

| Media | Value |
|-------|-------|
| USB flash drive | `/dev/sda2` |
| SD card | `/dev/mmcblk0p2` |
| By UUID *(recommended)* | `UUID=<your-uuid>` |
| By label | `LABEL=ROOTFS` |

**How to get the root partition UUID:**

With the system running from the USB drive or SD card, run:

```bash
blkid
```

Expected output:

```
/dev/sda2: UUID="92139c84-3871-41d7-a3f2-e8a943cbfa87" TYPE="ext4" PARTUUID="..."
```

Copy the `UUID=` value of the root partition (usually `sda2` or `mmcblk0p2`) and paste it into `armbianEnv.txt`. The UUID can also be found in `/etc/fstab` or in the original `armbianEnv.txt` from the image, if already filled in.

---

### Step 6 — Enable SoC fixup *(GXBB/GXL only)*

If your box uses a GXBB (S905) or GXL (S905X/W/L) SoC, uncomment the line:

```
soc_fixup=gxl-
```

---

### Step 7 — Boot from USB

1. Power off the box.
2. Insert the USB drive.
3. Press and **hold** the reset button.
4. Power on the box and keep holding for approximately **7 seconds**.
5. If everything is correct, Armbian will boot with a mainline kernel — without any secondary u-boot blobs.

---

## Supported Devices

**✅ Fully Tested & Working:**
- S905X, S905W, S912, S905X2, S922X, S905X3, S905X4 (HTV H8)

**⚠️ Partial Support:**
- S905: Boots only on first attempt (known limitation)

**❓ Untested:**
- S905W2: Likely compatible but untested (not currently supported by Armbian kernel)

All files and source files are available on [Github](https://github.com/projetotvbox/amlogic-bootscripts-Armbian).

---

## Advanced Troubleshooting — Bootloader Modification

> ⚠️ **This section is for cases where the scripts simply do not work.** If the main method worked, you do not need this.
>
> Some devices have factory bootloaders that do not support running external scripts by default. In those cases, it is possible to modify the bootloader variables directly via serial console to force that support. This is a low-level procedure with a real risk of bricking the device. **Proceed only if you know what you are doing.**

### Prerequisites

- **Functional ARM Linux system:** Armbian, Debian, or Ubuntu ARM running from USB/SD on the Amlogic device — required to access eMMC and the shell.
- **Serial TTL adapter (3.3V UART):** ⚠️ **Use 3.3V only. 5V will damage the device.** Requires soldering TX/RX/GND pads on the board.
- **Serial terminal software:** PuTTY, Minicom, or picocom.

### 🔒 Back Up eMMC Before Anything Else

```bash
# Compressed backup (a 16GB backup becomes 2-4GB)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# To restore:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

### Checking Bootloader Support

#### Step 1: Connect the serial cable
Solder TX, RX, and GND to the device's UART pads and connect to your PC.

#### Step 2: Open the serial console

```bash
ls -la /dev/ttyUSB*

picocom -b 115200 /dev/ttyUSB0
# or:
minicom -D /dev/ttyUSB0 -b 115200
```

#### Step 3: Interrupt U-Boot
Power on the device and quickly press `Ctrl+C` or `Enter` to interrupt U-Boot before it boots.

#### Step 4: Check bootloader variables

```bash
printenv bootcmd
```

Expected output:
```
bootcmd=run start_autoscript; run storeboot
```

Also check:
```bash
printenv start_usb_autoscript
printenv start_mmc_autoscript
printenv start_emmc_autoscript
```

> Variable names may differ slightly. Look for patterns like `start_*_autoscript`.

### Modifying the Variables

If the bootloader is writable, run in the U-Boot console:

```bash
setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then setenv devtype "mmc"; setenv devnum 1; autoscr 1020000; fi;'
setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then setenv devtype "mmc"; setenv devnum 0; autoscr 1020000; fi;'
setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then setenv devtype "usb"; setenv devnum 0; autoscr 1020000; fi; done'
setenv upgrade_step 2
setenv bootdelay 1
```

#### ⚠️ Setting `bootcmd` — Preserve the Original Command

**Do not simply use `run start_autoscript; run storeboot`** without checking your original `bootcmd` first. A generic command may brick your device if the original was different.

1. **Write down the original `bootcmd`:**
   ```bash
   printenv bootcmd
   ```

2. **Set it while preserving the original:**
   ```bash
   setenv bootcmd 'run start_autoscript; [YOUR ORIGINAL BOOTCMD HERE]'
   ```

**Real device examples:**

```bash
# Example 1 — Generic Amlogic box (original: run storeboot)
setenv bootcmd 'run start_autoscript; run storeboot'

# Example 2 — HTV H8 (original: run start_emmc_autoscript; run storeboot)
setenv bootcmd 'run start_autoscript; run start_emmc_autoscript; run storeboot'

# Example 3 — Complex bootcmd
# Original: if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot
setenv bootcmd 'run start_autoscript; if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot'
```

#### Save and verify

```bash
saveenv
reset
```

Interrupt U-Boot again and confirm:

```bash
printenv bootcmd
```

- **Variables saved** → writable bootloader, modifications applied successfully.
- **Variables not saved** → read-only bootloader; this method cannot be applied.

---

## How It Works Internally

For those who want to understand what happens under the hood — the role of each file in the boot chain.

### `aml_autoscript` — The Route Injector

Runs **only once**, at the moment you force recovery mode (by holding the reset button while powering on). It rewrites the factory U-Boot environment variables via `saveenv`, establishing a new boot order: SD card → USB → eMMC, redirecting the flow to the scripts below.

### `s905_autoscript` — The External Media Loader

Runs every time the board powers on with a USB drive or SD card connected. It reads `armbianEnv.txt`, loads the Kernel, DTB, and Initrd into RAM, prepares the `bootargs`, and hands control to the Kernel to start the operating system.

### `emmc_autoscript` — The Internal Storage Loader

Functionally identical to the previous script, but triggered when no bootable USB drive or SD card is connected. It points to the physical address of eMMC memory (`devnum 1`) and mounts the root filesystem from the internal partition.

### `gxl-fixup.scr` — The Fake Header Hack *(GXBB/GXL only)*

The factory bootloaders of GXBB (S905) and GXL (S905X/W/L) families only accept kernels in the legacy `uImage` format (using `bootm`). Modern Armbian uses the `Image` format (using `booti`), which those bootloaders simply refuse.

Instead of compiling legacy-format kernels, the script solves this at runtime:

1. Replaces the standard boot routine (`cmd_do_boot`).
2. Uses `mw.l` to write directly into memory a **fake** legacy u-boot header at address `0x1ffffc0`, just before the Kernel.
3. Injects a valid CRC into the fake header (`cmd_hdr_crc`).
4. Fires `bootm` — the bootloader sees the fake header, believes it's dealing with a valid `uImage`, and boots the modern Linux kernel normally.

> **Restriction:** the Kernel file cannot exceed **32MB** in size.

This is why Step 6 instructs you to uncomment `soc_fixup=gxl-` for these SoCs: without this hack, the original bootloader would stall at boot.

---
