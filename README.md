# U-571
This github repository contains a collection of u-boot macros which make the u-boot usage more beneficial. Almost every SBC vendor introduces his very own way how u-boot is going to be used. I've tried to consolidate these different approaches into one consistent way how a device can be started with u-boot.

# Warning
Installing this collection of u-boot macros will change the stored u-boot environment on your device. This might render your SBC unusable. Especially when the u-boot environment is saved on a non-removable storage device (e.g. the Utilite) you might brick your device.

# Motivation
When I started development on my different SBC, I was quite unhappy with the existing boot mechanisms I've found. I missed the ability to choose between different configurations during boot. For example I wanted to test a newly build Linux kernel without overwriting the existing one. Or I wanted to boot various Linux flavours stored on different partitions on the same storage device instead of swapping a bunch of µSD cards.

# Single partition layout
Many 'standard' images provided by the SBC vendors come with a two partition layout: One partition containing the Linux root filesystem and one small vfat partition carrying the Linux kernel image. In the very first days this boot partition was necessary due to u-boot's limitation being able to read files from fat filesystems only. Nowadays u-boot supports reading files from ext4 partitons, so I've decided to skip the boot partition in favour of a single ext4 partition layout, yielding a relocatable and self-contained root filesystem. Multiboot environments are easier to implement since they won't have side effects outside their root file system. Furthermore the directory layout inside `/boot` makes use of symbolic links which aren't supported by vfat filesystems.

# `/boot` directory layout
I've decided to tune up the `/boot` directory a little bit:
- `/boot/kernel.d/` is used to store different kernel images in separate directories, including
  - kernel image binary (`[z|u]Image`)
  - device tree binary (`*.dtb`)
  - initial ramdisk image (`[u]Initrd`)
  - additions like `System.map` or `config` (not needed for boot)
- `/boot/conf.d` is used to store different u-boot configurations in subdirectories:
  - `/boot/conf.d/default/`, the default u-boot configuration
  - `/boot/conf.d/test/`, for testing a new kernel or uEnv.txt or whatever
  - `/boot/conf.d/failsafe/`, the fail safe configuration which is supposed to boot under any circumstances

Each configuration has its own overlay for the default u-boot environment(`uEnv.txt`), a `kernel` directory (usually a link to a directory in `/boot/kernel.d/...`) and a pre-compiled u-boot script (`boot.scr`, optional). The configuration directory itself can be a symbolic link, so resetting `/boot/conf.d/default/` will change the default u-boot configuration.

For compatibility reasons a `config-$(make kernelrelease)` might be still present in `/boot` (needed for tools like `update-initramfs`).

# Custom u-boot environment (`uEnv.txt`)
U-Boot loads it's persistent environment during startup. There is a particular initial default environment for each board which already contains all necessary settings to boot up the default configuration, i.e. the board will boot even if the `uEnv.txt` is not present or has been accidentally deleted. If this file is present in the configuration directory, it's contents will be merged on the fly into the actual u-boot environment (but not be written back to the persistent u-boot environment).

# Some conventions about u-boot variables

1. There are two different ways to define a variable in the u-boot shell / monitor:

  1.  `setenv foo bar`: adds the variable `foo` to the u-boot environment
  1.  `foo=bar`: sets the shell-only variable `foo` which is not part of the u-boot environment (might be helpful if the environment size is rather small), but cannot be used for calling macros.
1. Shell-only variables are used for passing parameters to macros (`p_...`) and local variables (`v_...`)
2.  I've decided to use the prefix `m_` for all u-boot variables containing macros which helps to distinguish code parts from 'real' u-boot variables.
3.  All variables using the naming prefix `k_` are used by the macro `m_set_bootargs` to compose the kernel command line (aka `bootargs`). So instead of altering `bootargs` directly you might only want to change `k_rootfs`.

# Default u-boot variables

The following u-boot variables are defined in the initial default environment and can be overwritten:

| variable | semantics | typical value |
|:---------|:----------|:--------------|
|bootcmd   | default command to be executed | `run m_autoboot;`|
|bootenv   | file containing custom u-boot environment | `uEnv.txt`|
|bootdelay | time to interrupt non-interactive booting | `5` |
|bootscr   | compiled u-boot macro to be executed automatically | `boot.scr` |
|dbglevel  | turn on/off debugging messages| `0` |
|errlevel  | turn on/off error messages| `0` |
|fdt       | file containing the device tree binary | board specific |
|ramdisk   | file containing initial ramdisk content | `kernel/uInitrd` |
|kernel    | file containing kernel image | `zImage` |
|mmcdev    | MMC device:partion to boot from | `0:1` |
|prefix    | directory containing u-boot configurations | `/boot/conf.d` |
|run_pre_boot| execute the `m_pre_boot` macro immediately before `boot?` is executed | `0` |
|satadev   | SATA device:partition to boot from | `0:1` |
|usbdev    | USB device:partition to boot from | `0:1` |


# Boot sequence

## Typical boot sequence

### [`m_autoboot`](https://github.com/umiddelb/u-571/blob/master/macro/m_autoboot.uboot)
The macro `m_autoboot` implements the boot priority. You can tell u-boot to try to boot from USB first (`run m_autoboot_usb`) then to try from SATA (`run m_autoboot_sata`) and finally to boot from MMC (`run m_autoboot_usb`).

### [`m_autoboot_*`](https://github.com/umiddelb/u-571/blob/master/macro/m_autoboot_mmc.uboot)
This macro initializes the boot device and calls `m_boot_conf`.

### [`m_boot_conf`](https://github.com/umiddelb/u-571/blob/master/macro/m_boot_conf.uboot)
This is the main loop which boots a certain configuration (`p_conf`) from a specific device.

#### [`m_load_env`](https://github.com/umiddelb/u-571/blob/master/macro/m_load_env.uboot)
This macro looks for a text file called `uEnv.txt` inside of the configuration directory (`$prefix/$p_conf`), loads its contents and merges them with the existing u-boot environment by overriding values of existing variables.

#### [`m_run_bootscr`](https://github.com/umiddelb/u-571/blob/master/macro/m_run_bootscr.uboot)
This macro looks for a file called `boot.scr` inside of the configuration directory, load the file and execute its contents (usually without returning back). `boot.scr` contains compiled u-boot commands in a binary format. This step is optional.

#### [`m_set_bootargs`](https://github.com/umiddelb/u-571/blob/master/macro/m_run_bootscr.uboot)
This macro composes the kernel command line by setting the u-boot variable `bootargs`.

#### [`m_load_kernel`](https://github.com/umiddelb/u-571/blob/master/macro/m_load_kernel.uboot)
This macro loads the kernel image from `$prefix/$p_conf/kernel/$kernel`.

#### [`m_load_ramdisk`](https://github.com/umiddelb/u-571/blob/master/macro/m_load_ramdisk.uboot)
This macro loads the initial ramdisk archive from `$prefix/$p_conf/$ramdisk`.
The archive is created and updated by using the `update-initramfs` utility, which is usually done when a new kernel image has been installed. Although the kernel will boot without an initial ramdisk, most up-to-date Linux distributions will benefit from it.

#### [`m_load_fdt`](https://github.com/umiddelb/u-571/blob/master/macro/m_load_fdt.uboot)
This macro loads the device tree binary from `$prefix/$p_conf/kernel/$fdt`.
The Linux kernel on ARM needs a low level device description (device tree) in binary format, either appended to the kernel image or as a separate file. Many platforms prefer loading the device tree binary as a separate file, which offers more flexibility, allowing distribution of a single installation image for different platforms, or tweaking of the device tree for different use cases (see `m_pre_boot`).
The variable `ftd` may contain a [list of names](https://github.com/umiddelb/u-571/blob/master/board/odroid-xu4/board_init.env#L11) (delimited by a white space), which will be probed subsequently.

#### [`m_pre_boot`](https://github.com/umiddelb/u-571/blob/master/board/odroid-c1/uEnv.txt#L11)
In some case you want to modify the device tree after being loaded into memory.
This macro is called immediately before the `boot*` command is issued.
Some use case require a modification of the already loaded device tree. Using this macro will let you do so.


#### Depending on the kernel format (denoted by the actual file name), the kernel image will be booted via `booti`, `bootz` or `bootm`.


# Use Cases

## Non Permanent Changes
Running non permanent changes require access to the serial console in order to interrupt the automated u-boot boot procedure and to issue own u-boot shell commands.

### Boot a different kernel
Booting a freshly compiled Linux kernel without bricking my device was one of the reasons why I've build U-571. This can be achieved by creating another configuration directory, e.g. `/boot/conf.d/test`, create a `uEnv.txt` file and a `kernel` link (e.g. pointing to /boot/kernel.d/test, assuming that your test kernel image has been copied there).

Having prepared `/boot/...` you can now issue a reboot and enter the interactive u-boot shell. By issuing

	p_conf=test; run m_autoboot_mmc;

u-boot will start from MMC and uses `test` instead of `default` for loading the configuration.




### boot a rescue/fail save configuration

### boot from (by u-boot) unsupported storage devices
Create a particular configuration on the first internal storage device accessible by u-boot and modify the `k_rootfs` setting in `uEnv.txt`
pointing to `/dev/sda1` or `UUID=...`

Suggestion
- /boot/conf.d/u0p1/ for USB0 Partiton 1
- /boot/conf.d/s0p2/ for SATA0 Partition 2
- on the first partition , usually eMMC or uSD



## Permanent Changes (affecting the default configuration)
### boot a different kernel
Let `/boot/conf.d/default/kernel` point to a different directory within `/boot/kernel.d/`

### boot with different kernel settings
Modify `/boot/conf.d/default/uEnv.txt`.

### boot from the 2nd partition
Modify the u-boot environment variable `mmcdev`, `satadev` or `usbdev` from `0:1` to `0:2` and make sure that `/boot/...` is populated accordingly on `0:2`.

## boot from USB device first
Modify the u-boot environment variable `m_autoboot` in a way that `... run m_autoboot_usb ...` is called first.

# Installation

## Toolset
I've built a small toolset into the [Makefile](https://github.com/umiddelb/u-571/blob/master/macro/Makefile) targets aiming to make the u-boot macro coding a little bit more convenient. U-boot macros are stored as one large character string into an u-boot environment variable which makes editing not very intuitive, e.g. no line feeds, no indentation, no comments. You will find all this elements in the input files (`*.uboot`). The make procedure will remove them in order to produce a clean u-boot macro.

## Userland access to the u-boot environment
The u-boot environment is stored on dedicated space outside the filesystem. The u-boot-tools package contains the tool fw_setenv/fw_printenv to access the u-boot environment from the userland. This tool needs to be configured to your specific board, otherwise the make target `install` will fail. Please refer to [this article](https://github.com/umiddelb/armhf/wiki/Get-more-out-of-%22Das-U-Boot%22) for more details.

## [Choose your board](https://github.com/umiddelb/u-571/tree/master/board)

## `make install`
