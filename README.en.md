# Amlogic Boot Scripts for Armbian

**Language / Idioma:** [🟢 English](README.en.md) | [Português](README.md)

## Table of Contents
- [Overview](#overview)
- [Setup](#setup)
- [Customizing `aml_autoscript`](#customizing-aml_autoscript)
- [Supported Devices](#supported-devices)
- [Troubleshooting — Running `aml_autoscript` Manually](#troubleshooting--running-aml_autoscript-manually)
- [How It Works Internally](#how-it-works-internally)

---

## Overview

Armbian images for Amlogic TV Boxes normally rely on secondary u-boot blobs to boot the mainline kernel. In practice, these are unnecessary: the factory u-boot that came with your box is already capable of doing this on its own. All it takes are a few modifications to the Armbian boot scripts.

> **Prerequisite:** the vendor u-boot must be running on eMMC. If your box was reflashed with a different bootloader, restore the stock Android image using the [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool) before continuing.

> **Mainline focus:** this project's scripts remove the Android variables from U-Boot. If you want to keep using Android, use the original autoscripts by devmfc, which this project is a fork of. Details in [Customizing `aml_autoscript`](#customizing-aml_autoscript).

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

> The first time, holding reset is what makes `aml_autoscript` run and write the new variables. After that, the box looks for SD → USB → eMMC on its own. If reset has no effect, see [Troubleshooting](#troubleshooting--running-aml_autoscript-manually).

---

## Customizing `aml_autoscript`

This project's `aml_autoscript` is a **fork of devmfc's original code, focused on mainline Linux**. Besides setting the U-Boot variables needed to boot Armbian, it removes dozens of variables that only exist for Android (recovery, burning, Dolby Vision, A/B slots, etc.). This keeps the U-Boot environment clean, making debugging and understanding the code much easier.

> ⚠️ **Android no longer boots from eMMC.** `bootcmd` becomes just `run start_autoscript` and the `storeboot` variable is removed. If you want to keep using Android, use the original autoscripts by **devmfc**. To go back to Android after running this script, restore the stock image with the [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool).

The editable source is `aml_autoscript.command`. The `aml_autoscript` file (no extension) is the compiled version, the one that goes on the boot partition.

> **Any change only takes effect after you recompile the script and run it again on the box** (by holding reset, or manually via serial console — see the troubleshooting section). The variables are only written by `saveenv` at the end of the run.

### Recompiling the script

```bash
sudo apt install u-boot-tools   # provides mkimage (Debian/Ubuntu)
mkimage -C none -A arm -T script -d aml_autoscript.command aml_autoscript
```

Copy the generated `aml_autoscript` to the FAT boot partition, overwriting the existing one.

---

### U-Boot bootlogo on mainline Linux

`aml_autoscript` injects a bootlogo function into U-Boot, something normally only Android provides. The logo shows as soon as the box powers on, before the kernel loads.

All you need to do is place a file named **`bootlogo.bmp`** on **partition 1 (the FAT boot partition)** of the media. U-Boot looks for the file in this order and uses the first one found:

1. USB drive (ports 0 to 3)
2. SD card
3. eMMC

If no file is found, the box simply boots without a logo. To use a different name, change the `bootlogo_filename` variable in `aml_autoscript.command` (without the `.bmp` extension) and recompile.

**Accepted format**

The box's U-Boot only displays BMPs in a specific format. In most cases it is this one (RGB565, 16 bits):

```bash
file bootlogo.bmp
# bootlogo.bmp: PC bitmap, Windows 3.x format, 320 x 388 x 16, 3 compression, image size 248320, cbSize 248386, bits offset 66
```

**Converting a PNG or JPEG to the correct format**

```bash
ffmpeg -i bootlogo.png -pix_fmt rgb565 -compression_level 0 bootlogo.bmp
```

> ⚠️ **This format is the most common, not a guaranteed standard.** Each U-Boot may have its own quirks (resolution, color depth, etc.). If the logo does not show or looks distorted, you will need to adapt the conversion to your case — there is no single solution that covers every box.

---

### Showing the bootlogo on CVBS output

By default, the bootlogo is shown over HDMI only. If your box has a CVBS (composite video) output, you can enable it:

1. In `aml_autoscript.command`, change:
   ```bash
   setenv cvbs_boot 0
   ```
   to:
   ```bash
   setenv cvbs_boot 1
   ```
2. [Recompile the script](#recompiling-the-script).
3. Copy the new `aml_autoscript` to the boot partition and run it again on the box.

---

### Reusing another `aml_autoscript`

By default, this feature comes **disabled**: U-Boot no longer looks for a new `aml_autoscript` at power-on, which keeps boot simpler and more predictable.

To re-enable it, edit `aml_autoscript.command`: **uncomment** the four lines in the indicated block and **comment out** the `setenv update` line right below it.

```bash
# Uncomment these lines:
setenv check_update_button ${upgrade_key}
setenv update 'run load_aml_autoscript'
setenv load_aml_autoscript 'if mmcinfo; then if fatload mmc 0 1020000 aml_autoscript; then autoscr 1020000; fi; fi; if usb start; then for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 aml_autoscript; then autoscr 1020000; fi; done; fi'
setenv bootcmd 'run check_update_button; run start_autoscript'

# And comment out this one (further down in the file):
#setenv update
```

With this, `bootcmd` checks the reset button again and `load_aml_autoscript` looks for an `aml_autoscript` on SD and USB. Recompile and run the script to apply.

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

## Troubleshooting — Running `aml_autoscript` Manually

> ⚠️ **This section is for when holding the reset button does not run `aml_autoscript`.** If the main method worked, you do not need this.
>
> You no longer need to type U-Boot variables by hand: `aml_autoscript` already does all the configuration. The only thing left is running it manually by interrupting U-Boot over the serial console.

### Prerequisites

- **Serial TTL adapter (3.3V UART):** ⚠️ **Use 3.3V only. 5V will damage the device.** Requires soldering TX/RX/GND pads on the board.
- **Serial terminal software:** PuTTY, Minicom, or picocom.
- **A USB drive formatted as FAT32** with the `aml_autoscript` file in its root.

### 🔒 Back Up eMMC Before Anything Else

`aml_autoscript` wipes the factory U-Boot environment (`defenv`) and removes the Android variables. If there is any chance you will want to go back, back up first (from an ARM Linux system running from the USB drive):

```bash
# Compressed backup (a 16GB backup becomes 2-4GB)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# To restore:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

### Step 1 — Connect the serial cable

Solder TX, RX, and GND to the device's UART pads and connect to your PC.

### Step 2 — Open the serial console

```bash
ls -la /dev/ttyUSB*

picocom -b 115200 /dev/ttyUSB0
# or:
minicom -D /dev/ttyUSB0 -b 115200
```

### Step 3 — Interrupt U-Boot

With the USB drive connected, power on the device and quickly press `Ctrl+C` or `Enter` to interrupt U-Boot before it boots.

### Step 4 — Run `aml_autoscript`

In the U-Boot console, run:

```bash
usb start
fatload usb 0 $loadaddr aml_autoscript
autoscr $loadaddr
```

> ⚠️ **Keep only 1 USB drive connected** during this procedure. The command above reads the first USB device (`0`).

The script rewrites the variables, runs `saveenv`, and ends with `run start_autoscript`, booting Armbian right after. From then on, you will no longer need the serial console or the reset button.

---

## How It Works Internally

For those who want to understand what happens under the hood — the role of each file in the boot chain.

### `aml_autoscript` — The Route Injector

Runs **only once** per installation, at the moment you force recovery mode (by holding the reset button while powering on) or run it manually over the serial console. Since the variables are written with `saveenv`, the result persists across later boots. In order, it:

1. **Restores the factory environment** (`defenv`, `env default -a`, and `saveenv`), starting from a clean base.
2. **Defines the new boot route** (`start_autoscript`): SD card → USB → eMMC, redirecting the flow to the scripts below. `bootcmd` becomes just `run start_autoscript`, without Android's `storeboot`.
3. **Sets up the bootlogo and video output** (`init_display`, run from `preboot`): picks the output mode (HDMI or CVBS), looks for `bootlogo.bmp` on USB, SD, and eMMC, and displays it.
4. **Removes the Android variables** (recovery, burning, Dolby Vision, networking, A/B slots, etc.), keeping the environment lean.
5. **Saves everything** with `saveenv` and calls `run start_autoscript` to start booting immediately.

Optionally, you can keep the ability to load another `aml_autoscript` on future boots, as explained in [Reusing another `aml_autoscript`](#reusing-another-aml_autoscript).

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

Feito com 🐧 no IFSP Salto · Tecnologia a serviço da educação pública
