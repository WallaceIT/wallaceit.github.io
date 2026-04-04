# An embedded Linux journey

> **WARNING**: this is a Work In Progress! The content is subject to change with no notice.

## Introduction

This work will detail the integration activities performed on an embedded Linux running on a chosen
hardware platform for a specific use case, with a particular focus on boot-time optimization.
All the steps will be documented, including mistakes, dead ends and wrong choices, in the hope they
can serve as a guide for whoever will be doing the same job.

During the journey, any component or optimization that will be deemed useful for the open source
community will be sent as a patch for upstream inclusion.

### Disclaimer

Except when stated differently, the Author is not affiliated with any of the companies or
organizations mentioned or linked in this work.

All trademarks belong to their owners, including (but not limiting to):

- [Linux](http://kernel.org/) which is a trademark of Linux Torvalds
- [Yocto Project](https://www.yoctoproject.org/) which is a trademark of The Linux Foundation

## Acronyms

| Acronym | Meaning                 |
| ------- | ----------------------- |
| CI      | Continuous Integration  |
| OPT     | One-Time Programmable   |
| SoC     | System on Chip          |
| CAN     | Controller Area Network |

## Scope of the work

### Hardware

The work will focus on the i.MX93 FRDM board, which is widely available at a low cost from the
major distributors worldwide. This board is based on a i.MX93 SoC from NXP and features a rich set
of periphrals, including Ethernet, CAN, Bluetooth/Wifi, RTC and so on; it is also well equipped for
development, as it sports e.g. an on-board USB-serial converter as well as an SD-card for quick
BSP testing.

Detailed specifications can be found on the
[official product page](https://www.nxp.com/design/design-center/development-boards-and-designs/FRDM-IMX93).

A key point of both the chosen board and SoC, which will be fundamental for the work presented below,
is that most of the documentation for them is public, including the
[Technical Reference Manual](#referenced-documents) and the [board schematics](#referenced-documents).

> NOTE: the [Beagleplay](https://www.beagleboard.org/boards/beagleplay) platform from the homonym
> foundation was chosen initially for this work, it being of more widespread usage and covered by a
> number of high-quality courses; however, the Author believes that the i.MX93 FRDM has more
> learning opportunities and can be brought to more aggressive boot times (having e.g. an on-board
> eMMC flash device).

### Software

The software will be based on a GNU/Linux stack. The Yocto project will be used as build system,
with custom components developed at need. While most of the techniques presented are portable to any
build system, the Author believe to be fluent in Yocto and for this reason this framework has
been chosen.

Systemd will be used as init system, as it provides a set of functionalities any sufficiently
complex embedded product would probably need.

If possibile, upstream versions of BSP components (such as the Linux kernel, the bootloader, TF-A
and Op-TEE) will be used.

### Use case

By definition, an embedded system cannot cover all use cases. This work will thus focus on one,
arbitrally defined by the Author (hoping the various techniques can be easily(-_ish_) ported to
others). The following requirements will be considered:

- The system shall be able to communicate its identity through CAN bus in under 1000ms
- The system shall display a static splash screen in under 2000ms
- The system shall perform a DHCP request for a IPv4 address over Ethernet in under 25000ms
- The system shall be able to connect to WiFi networks
- The system shall support Bluetooth Low Energy (exact use cases TBD)
- The system shall work as as USB device, offering both the CDC-ACM and storage profiles
- The system shall perform a secure boot

Given this description, features will be tagged as being part of three categories:

- critical (to be brought up as soon as possible): CAN communication
- important (to be brought up early): Ethernet communication, display
- deferrable: Bluetooth/WiFi communication

As for the secure boot point, a full chain-of-trust will be designed, with the implementation
ending whenever blocked by closed tools or destructive actions (such as OTP burning).

## Required materials

- A i.MX93 FRDM board
- One USB cable, with one type-C connector at one end, to supply power to the board
- Another USB cable, with one type-C connector at one end, to connect to the on-board USB-serial
  converter
- A good quality USB power supply, capable of at least 2A of current (cheap power supplies might be
  able to supply sufficient power while not having a suitable inrush current capability)

### A note on brand new i.MX93 FRDM boards

Brand new i.MX93 FRDM boards include a full featured OS on the internal eMMC card. For this reason,
the boot from the microSD card that will be initially used in this article has to be forced.
This can be done modifying the on-board DIP switches before applying power to the device.

## The journey

### Planning

The work will logically follow four different phases, depicted above; in practice, some
back-and-forth will probably take place:

1. Setup of development environment
2. Initial assessment of functionalities and overall performances (including boot time)
3. Customization of the default configuration to implement the [use case](#use-case)
4. Optimizations

The "optimizations" stages will be further divided in sub-steps:

- Kernel optimizations
- Userspace optimizations
- Bootloader optimizations

### Setup of development environment

#### Build host

To build the system images using the Yocto project a container image coming from
[the CROPS project](https://github.com/crops/poky-container) will be used.
This approach, successfully leveraged by the Author during the last few years, allows to maintain a
controlled and fully reproducible build environment, that can be moved from one machine to another
with virtually zero effort. Moreover, it is CI-friendly and has few (if any) disadvantages.

Given a target location on the (real) host machine of `${HOME}/YOCTO`, the following command is used
to start the development container using the `Docker` containerization technology:

```sh
docker run --rm -i -t --name playgrond_${USER} --network="host" -v ${HOME}/YOCTO:${HOME}/YOCTO --workdir=${HOME}/YOCTO crops/poky:ubuntu-22.04
```

Let's analyze the different parameters supplied to `docker run`:

- `--rm`: the container image is removed at exit; this is definitely wanted, as it discourages any
  runtime customization to the image and helps in maintaining a reproducible environment;
- `-i`: keep STDIN open even if not attached;
- `-t`: allocate a pseudo-TTY (allows for a console to be run inside the container);
- `--name playgrond_${USER}`: give the `playgrond_${USER}` name to the container, allowing an
  easier re-attach after a detach (which can be used in case of long running builds, especially if
  performed on a remote machine); the `${USER}` suffix further helps in a multi-user environment
  (such as a build server);
- `--network="host"`: use the network interface(s) from the host instead of allocating a dedicated
  NAT'ed one; can simplify operations behind corporate firewalls and proxies and helps debugging
  fetching failures;
- `-v ${HOME}/YOCTO:${HOME}/YOCTO`: mount the `${HOME}/YOCTO` directory of the host (real) machine
  into the container under the same; this provides persistence of the build artifacts (and it's the
  reason `--rm` can be used); the same path is used to be able to access and use git repositories
  from outside the conatiner;
- `--workdir=${HOME}/YOCTO`: automatically move to `${HOME}/YOCTO` once the container is started;
- `crops/poky:ubuntu-22.04`: image to be used for the container.

It should be noted that under some circumstances, such as sources hosted on private servers,
additional configurations might be required:

- `-v ${HOME}/.ssh:/etc/skel/.ssh:ro`: mount the user's `.ssh` directory under `/etc/skel/.ssh` in
  read-only mode; at container startup, the user created inside the container will have this copied
  into its home directory. **THIS MAY LEAD TO THE LEAK GIT SSH CREDENTIALS AND KEYS** in a
  multi-user environment (as other users may be able to attach to the container and exfiltrate them).
- `-v ~/.gitconfig:/etc/skel/.gitconfig:ro`:  mount the user's `.gitconfig` file under
  `/etc/skel/.gitconfig` in read-only mode; at container startup, the user created inside the
   container will have this copied into its home directory.
   **THIS MAY LEAD TO THE LEAK OF GIT CREDENTIALS** in a multi-user environment (as other users may
   be able to attach to the container and exfiltrate them).

Once attached with the command above, the user can detach from the running container using the
`CTL-p` `CTL-q` escape sequence, then re-attach with:

```sh
docker attach playground_${USER}
```

#### Clone of meta layers

Here are the Yocto meta layers, along with the chosen git hash, that will be used as a basis.
The `master` (or equivalent) branch will be used for layers that provide it, with frequent updates
during the journey, to be able to back-feed contributions without any overhead.

| Layer                                                                  | Used sub-layers                  | Branch | Here because...                                          |
| ---------------------------------------------------------------------- | -------------------------------- | ------ | -------------------------------------------------------- |
| [openembedded-core](https://git.yoctoproject.org/openembedded-core)    | `meta`                           | master | Basic layer                                              |
| [meta-yocto](https://git.openembedded.org/meta-yocto)                  | `meta-poky`                      | master | Provider of the `poky` distro                            |
| [meta-openembedded](https://git.openembedded.org/meta-openembedded)    | `meta-oe`                        | master | Dependency of meta-ti                                    |
| [meta-arm](https://git.yoctoproject.org/meta-arm)                      | `meta-arm`, `meta-arm-toolchain` | master | Dependency of `meta-freescale` and provider op `OP-TEE`  |
| [meta-freescale](https://github.com/Freescale/meta-freescale)          | No sublayers                     | master | Contains NXP-specific components                         |
| [meta-linux-mainline](https://github.com/betafive/meta-linux-mainline) | No sublayers                     | master | Allows to build the mainline Linux kernel                |

It is important to note that not all of the sub-layers contained inside the cloned meta layers are
used; this is the case for example of `meta-yocto/meta-yocto-bsp`, which contains machine
configurations for the Yocto reference machines and has thus no use in the target configuration.

Meta layers can be cloned directly inside the [running build container](#build-host); once this is
done, the [build directory](#setup-of-the-build-directory) shall be created.

For sake of order and clarity, the layers should be cloned in a dedicated directory such as `layers`
and not straight into the work directory.

#### Setup of the build directory

Once the [meta layers](#clone-of-meta-layers) have been cloned, the build directory can be created
using:

```sh
source layers/openembedded-core/oe-init-build-env build.imx9
```

This will initialize the build environment will all the required variables and will make `bitbake`
accessible to the user. The first time it is launched, it will also create the `build.imx9`
directory and copy a skeleton configuration under it `conf/` subdirectory, where it can be
customized.

##### `bblayers.conf`

The `bblayers.conf` file contains the list of meta layers used by the project; according to what has
been defined [before](#clone-of-meta-layers), it has to be modified to look more or less like this:

```
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

LAYERDIR ?= "/home/user/YOCTO/layers"

BBLAYERS ?= " \
  ${LAYERDIR}/openembedded-core/meta \
  ${LAYERDIR}/meta-yocto/meta-poky \
  ${LAYERDIR}/meta-openembedded/meta-oe \
  ${LAYERDIR}/meta-freescale \
  ${LAYERDIR}/meta-arm/meta-arm-toolchain \
  ${LAYERDIR}/meta-arm/meta-arm \
  ${LAYERDIR}/meta-linux-mainline \
"
```

##### `local.conf`

The `local.conf` file holds the build-specific configurations. The explanation of all the different
key-value pairs would add no value to the current work, and will thus be left to more autoritative
sources, such as the [official Yocto documentation](https://docs.yoctoproject.org/).

To obtain the initial build, the following values shall however be added at the bottom of this file:

```
# Target machine is the i.MX93 FRDM, which is not existent: for the time being, fall back to i.MX93 EVK
MACHINE = "imx93-11x11-lpddr4x-evk"

# The Poky reference distro will be used
DISTRO = "poky"

# Allow empty passwords, even for the root user, and allow root to login.
# These are development configuration and _shall_ be removed on a production system!

EXTRA_IMAGE_FEATURES = "allow-empty-password empty-root-password allow-root-login"

# Use systemd as init manager
INIT_MANAGER = "systemd"
```

If the host machine has limited storage, the following configuration can be added to remove all the
intermediate files that are generated during the build of the software components:

```
INHERIT += "rm_work"
```

This will somewhat increase the build time, since multiple files will be deleted after each build,
but dramatically decrease the storage space required for a fill build.

#### Image build

Once all the configurations have been performed, it is possible to build a complete image for the
system. The `core-image-full-cmdline` will be used at first, as it provides a sufficient complete
system without introducing too much overhead.

```sh
bitbake core-image-full-cmdline
```

Depending on the speed of the Internet connection and the capabilities of the host machine, this
will take an amount of time that can vary from a few minutes to several hours [^1].

#### Image flashing

Once built, the gzip'ed image can be found under
`build.imx9/tmp/deploy/images/${MACHINE}/core-image-full-cmdline-${MACHINE}.rootfs.wic.gz`.
Such image can be written to a microSD card using a number of tools; all of them have to be used on
the (real) host machine, since the container has no access to it.

##### Flash using `Balena Etcher`

The `Balena Etcher` graphical application from [balena.io](https://etcher.balena.io/) can be used
to flash the built image to a microSD. Since the Author has no real experience using it, its usage
will no be treated here.

#### Flash using `bmaptool`

The `bmaptool` Python application can be used to flash the built image to a microSD:

```sh
sudo bmaptool copy ${HOME}/YOCTO/build.imx9/tmp/deploy/images/${MACHINE}/core-image-full-cmdline-${MACHINE}.rootfs.wic.gz /dev/mmcblk0
```

> Here the microSD is assumed to be accessible through `/dev/mmcblk0`; this might need to be
> adapted to the system at hand.

The `bmaptool` application is available for most GNU/Linux distributions under the `bmap-tools`
package. Along with the `.wic.gz` image file, it uses the accompanying `.bmap` file to write to the
target microSD only the blocks that are really needed (i.e., skipping the empty ones).

#### Flash using `bmap-writer`

The `bmap-writer` application can be used to flash the built image to a microSD:

```sh
sudo bmap-writer ${HOME}/YOCTO/build.imx9/tmp/deploy/images/${MACHINE}/core-image-full-cmdline-${MACHINE}.rootfs.wic.gz /dev/mmcblk0
```

> Here the microSD is assumed to be accessible through `/dev/mmcblk0`; this might need to be
> adapted to the system at hand.

The `bmap-writer` application would probably need to be compiled from scratch following instructions
from [its git repository](https://github.com/embetrix/bmap-writer). Along with the `.wic.gz` image
file, it uses the accompanying `.bmap` file to write to the target microSD only the blocks that are
really needed (i.e., skipping the empty ones).

### Day 1: Assessment of the situation

**TL;DR**: the initial assessment of the `meta-freescale` layer showed that support for the i.MX93
FRDM is absent; before adding it, solving a build issue was deemed necessary.
In the end, a pull request towards `meta-freescale` was opened with the fix.

A first assessment shows that the i.MX93 FRDM board is not yet supported by the `meta-freescale`
layer, while being present both in mainline Linux kernel and U-Boot (with the Author being one of
the reviewer for their inclusion). This support can eventually be included, but it's wise to start
with a supported target to understand if the build system is set up correclty.

One of the i.MX93 EVKs is the obvious choice, so let's use it inside `local.conf`:

```
MACHINE = "imx93-11x11-lpddr4x-evk"
```

A build can now be performed:

```sh
bitbake core-image-full-cmdline
```

...with the first error of the journey!

```sh
| make -f /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/sources/u-boot-imx-2025.04/scripts/Makefile.build obj=dts/upstream/src/arm64 dtbs
|   aarch64-poky-linux-objdump -t u-boot > u-boot.sym
|   cert-to-efi-sig-list /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/sources/u-boot-imx-2025.04/CRT.crt dts/upstream/src/arm64/capsule_esl_file
| /bin/sh: line 1: cert-to-efi-sig-list: command not found
|   tools/relocate-rela
| make[3]: *** [scripts/Makefile.lib:392: dts/upstream/src/arm64/capsule_esl_file] Error 127
| make[2]: *** [/home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/sources/u-boot-imx-2025.04/dts/Makefile:60: arch-dtbs] Error 2
| make[1]: *** [/home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/sources/u-boot-imx-2025.04/Makefile:1169: dts/dt.dtb] Error 2
| make[1]: *** Waiting for unfinished jobs....
| make[1]: Leaving directory '/home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/build/imx93_11x11_evk_defconfig-sd'
| make: Leaving directory '/home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/sources/u-boot-imx-2025.04'
| make: *** [Makefile:177: sub-make] Error 2
| ERROR: oe_runmake failed
| WARNING: /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364:303 exit 1 from 'exit 1'
| WARNING: Backtrace (BB generated script):
|       #1: bbfatal_log, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 303
|       #2: die, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 287
|       #3: oe_runmake, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 243
|       #4: uboot_compile_config, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 226
|       #5: do_compile, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 180
|       #6: main, /home/francesco/YOCTO/build.imx9/tmp/work/imx93_11x11_lpddr4x_evk-poky-linux/u-boot-imx/2025.04/temp/run.do_compile.2050364, line 316
ERROR: Task (/home/francesco/YOCTO/layers/meta-freescale/recipes-bsp/u-boot/u-boot-imx_2025.04.bb:do_compile) failed with exit code '1'
```

The `cert-to-efi-sig-list` tool should be provided by the `efitools` package, a collection of tools
used for manipulation of EFI artifact. Despite the fact that it is quite old (at the time of
writing, the last commit on the [official repository](https://git.kernel.org/pub/scm/linux/kernel/git/jejb/efitools.git)
is ~5 years old), there is no sign of a recipe neither in `openembedded-core` nor in
`meta-openembedded`, meaning that it's probably not much used in a typical build flow.

The [OpenEmbedded Layer Index](https://layers.openembedded.org/layerindex/) shows that an `efitools`
recipe is provided by the [`meta-efi-secure-boot`](https://layers.openembedded.org/layerindex/branch/master/layer/meta-efi-secure-boot/)
layer; further researches lead to the conclusion that the same recipe has also been imported into
`meta-imx`, a layer from NXP which provides further support (and dependencies, and bloat) for the
i.MX family of processors.

However, the recipe being present elsewhere does not change the fact that the i.MX93 EVK machine is
unbuildable; let's fix that first, importing only the relevant part of the `efitools` recipe inside
`meta-freescale` and proposing it for inclusion into mainline.
According to the README file of the layer, contributions shall be made through Github pull requests,
hence that flow will be followed.

With the fixes contained into the [pull request](https://github.com/Freescale/meta-freescale/pull/2485),
the build can now be completed successfully.

Time to start working on proper support for the real target (i.e., the i.MX93 FRDM).

### Day 2: Adding support for the i.MX93 FRDM

**TL;DR**: support for the i.MX93 FRDM device is added to `meta-freescale`; this leads to a complete
boot of the board.

While not (yet?) supported by `meta-freescale`, the i.MX93 FRDM board is not really different from
the EVK based on the same SoC; a basic machine file can be derived quite easily:

```
require include/imx93-evk.inc

IMX_DEFAULT_BOOTLOADER:imx-nxp-bsp = "u-boot-imx"
IMX_DEFAULT_KERNEL:imx-nxp-bsp = "linux-imx"

KERNEL_DEVICETREE_BASENAME = "imx93-11x11-frdm"

UBOOT_CONFIG_BASENAME = "imx93_11x11_frdm"

DDR_FIRMWARE_NAME = " \
    lpddr4_dmem_1d_v202201.bin \
    lpddr4_dmem_2d_v202201.bin \
    lpddr4_imem_1d_v202201.bin \
    lpddr4_imem_2d_v202201.bin \
"
```

Here it should be noted that the `-imx` variants have been selected both for the Linux kernel and
the U-Boot bootloader; the community-maintained `-fslc` variants are in fact (at the time of
writing) based on too-old versions and lack support for the FRDM board. Also considering that the
goal is to rely on upstream components for both the U-Boot and the kernel, this "limitation" can
be accepted for the time being.

Once an image compiled for this machine (that will be called `imx93-11x11-frdm`) has been compiled
and flashed to an SD card, a first boot is obtained. Four main stages can be observed from the
output of the USB-to-serial adapter (at `/dev/ttyACM0` on the Author's machine):

1. U-Boot SPL

```
U-Boot SPL 2025.04-g4ddbad60eff3 (Nov 19 2025 - 07:56:58 +0000)
PMIC: PCA9451A
PMIC: Over Drive Voltage Mode
DDR: 3733MTS
DDR: 3733MTS
found DRAM 2GB DRAM matched
M33 prepare ok
Normal Boot
Trying to boot from BOOTROM
Boot Stage: Primary boot
image offset 0x8000, pagesize 0x200, ivt offset 0x0
Load image from 0x57800 by ROM_API
```

2. TF-A (that is, Trusted Firmware for A processors)

```
NOTICE:  TRDC init done
NOTICE:  BL31: v2.12.0(release):lf-6.18.2-1.0.0-dirty
NOTICE:  BL31: Built : 07:53:18, Feb 10 2026
INFO:    GICv3 without legacy support detected.
INFO:    ARM GICv3 driver initialized in EL3
INFO:    Maximum SPI INTID supported: 991
INFO:    BL31: Initializing runtime services
INFO:    BL31: Initializing BL32
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x80200000
INFO:    SPSR = 0x3c9
```

3. U-Boot full (full output omitted for brevity)

```
U-Boot 2025.04-g4ddbad60eff3 (Nov 19 2025 - 07:56:58 +0000)

Reset Status: POR

CPU:   NXP i.MX93(52) Rev1.1 A55 at 1700 MHz
CPU:   Industrial temperature grade  (-40C to 105C) at 27C
Model: NXP FRDM-IMX93
DRAM:  2 GiB
BOARD: V1.0(ADC2:689,ADC3:358)
TCPC:  Vendor ID [0x1fc9], Product ID [0x5110], Addr [I2C2 0x52]
SNK.Default on CC2
tcpc_pd_receive_message: Polling ALERT register, TCPC_ALERT_RX_STATUS bit failed, ret = -62
TCPC:  Vendor ID [0x1fc9], Product ID [0x5110], Addr [I2C2 0x50]
Core:  229 devices, 32 uclasses, devicetree: separate
MMC:   FSL_SDHC: 0, FSL_SDHC: 1

<snip>

Booting from mmc ...
61368 bytes read in 3 ms (19.5 MiB/s)
## Flattened Device Tree blob at 83000000
   Booting using the fdt blob at 0x83000000
Working FDT set to 83000000
   Using Device Tree in place at 0000000083000000, end 0000000083011fb7
Working FDT set to 83000000

Starting kernel ...
```

4. Linux kernel (again, full output omitted)

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x412fd050]
[    0.000000] Linux version 6.18.2-1.0.0-gf49f45233f7b (oe-user@oe-host) (aarch64-poky-linux-gcc (GCC) 15.2.0, GNU ld (GNU Binutils) 2.46) #1 SMP PREEMPT Wed Feb 11 18:37:47 UTC 2026
[    0.000000] KASLR disabled due to lack of seed
[    0.000000] Machine model: NXP FRDM-IMX93

<snip>

[    3.764456] Freeing unused kernel memory: 2112K
[    3.769088] Run /sbin/init as init process
```

5. systemd init system (again, full output omitted)

```
Welcome to Poky (Yocto Project Reference Distro) 5.3.99+snapshot-fff0a8fb3ee6925883b8e662ee2ca6ea33990c10 (wrynose)!

[  OK  ] Created slice Slice /system/getty.
[  OK  ] Created slice Slice /system/modprobe.
[  OK  ] Created slice Slice /system/serial-getty.
[  OK  ] Created slice User and Session Slice.
[  OK  ] Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Forward Password Requests to Wall Directory Watch.
         Expecting device /dev/ttyLP0...
[  OK  ] Reached target NFS client services.
[  OK  ] Reached target Path Units.
[  OK  ] Reached target Preparation for Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[  OK  ] Reached target Slice Units.
[  OK  ] Reached target Swaps.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Reached target RPC Port Mapper.
[  OK  ] Listening on Syslog Socket.

<snip>

         Starting Network Management...
[  OK  ] Started Network Management.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Network.
         Starting Enable Persistent Storage in systemd-networkd...
         Starting Wait for Network to be Online...
[  OK  ] Finished Enable Persistent Storage in systemd-networkd.
[  OK  ] Reached target Hardware activated USB gadget.
         Starting Virtual Console Setup...
[  OK  ] Finished Virtual Console Setup.
```

6. The login prompt

```
Poky (Yocto Project Reference Distro) 5.3.99+snapshot-fff0a8fb3ee6925883b8e662ee2ca6ea33990c10 imx93-11x11-frdm ttyLP0

imx93-11x11-frdm login:
```

While not printing on its own, traces of the Op-TEE also being present can be found inside the
Linux kernel message buffer (a.k.a. dmesg):

```
[    1.644824] optee: probing for conduit method.
[    1.649327] optee: revision 4.8 (e7ed997213779e3d)
[    1.649762] optee: dynamic shared memory is enabled
[    1.659822] optee: initialized driver
```

The very basic machine file, coupled with the support already present inside the various software
components, did its magic and brought the system to a complete boot. Something is still missing
(e.g., kernel modules, for which the directive `IMAGE_INSTALL:append = " kernel-modules" can be
added to the `local.conf` as a quick-and-dirty solution), but it's satisfying to reach the login
prompt without fighting with the hardware debugger, for once.

Shall the machine file be contributed to `meta-freescale`? Maybe, but let's wait for the first PR to
be accepted before rushing to it (or the support wouldn't even be functional).

### Day 3: Boot time measurement

**TL;DR**: boot time of the booting system is measured as 22 seconds, of which ~7 spent in the 
bootloading phase.

Now that the device is booting, it's time to measure the value of the boot time to use as a
starting point for future optimizations.

Among all the methods that can be used to measure the boot time, the "external" one based on
`grabserial` is probably the simplest - even if not the most accurate - one. It is based on output
on a serial line and in particular on the time difference between two chosen prints; it is thus
clear that everything that is not printed cannot be measured. Its major selling point is that no
instrumentation is required on a system that already prints to a serial line its boot trace, and is
thus perfect for the device under analysis.

Starting `grabserial` as:

```sh
grabserial --device=/dev/ttyACM0 --baudrate=115200 --time
```

it can be observed that the total time between the first print and the latest (that is, the login
prompt) reaches almost 23 seconds:

```
[0.000001 0.000001]
[0.000114 0.000113] U-Boot SPL 2025.04-g4ddbad60eff3 (Nov 19 2025 - 07:56:58 +0000)
[0.011566 0.011452] PMIC: PCA9451A
[0.011966 0.000400] PMIC: Over Drive Voltage Mode

<snip>

[22.842030 2.022011] Poky (Yocto Project Reference Distro) 5.3.99+snapshot-fff0a8fb3ee6925883b8e662ee2ca6ea33990c10 imx93-11x11-frdm ttyLP0
[22.843699 0.001669]
[22.843709 0.000010] Type 'root' to login with superuser privileges (no password will be asked).
[22.852661 0.008952]
[22.852698 0.000037] imx93-11x11-frdm login:
```

This is more or less expected, since the system is printing a lot (and printing to a serial line is
usually an expensive operation!) and moreover some delay waiting for user input is still present in
U-Boot.

Speaking of U-Boot, it can be observed that 1/4 of the total time is spent in thebootloading phase,
while 3/4 is Linux kernel and userspace:

```
[6.167836 0.000458] Running BSP bootcmd ...
[6.292699 0.124863] switch to partitions #0, OK
[6.293553 0.000854] mmc1 is current device
[6.417961 0.124408] ** No boot file defined **
[6.802913 0.384952] 36211200 bytes read in 379 ms (91.1 MiB/s)
[6.803877 0.000964] Booting from mmc ...
[6.817215 0.013338] 61368 bytes read in 2 ms (29.3 MiB/s)
[6.818098 0.000883] ## Flattened Device Tree blob at 83000000
[6.818721 0.000623]    Booting using the fdt blob at 0x83000000
[6.819060 0.000339] Working FDT set to 83000000
[6.839662 0.020602]    Using Device Tree in place at 0000000083000000, end 0000000083011fb7
[6.840230 0.000568] Working FDT set to 83000000
[6.856365 0.016135] 
[6.856410 0.000046] Starting kernel ...
[6.856706 0.000296] 
[6.875977 0.019271] [    0.000000] Booting Linux on physical CPU 0x0000000000 [0x412fd050]
```

and in particular, jump to userspace is around the 11 seconds mark:

```
[11.303824 0.009562] [    3.780903] Freeing unused kernel memory: 2112K
[11.304881 0.001057] [    3.785588] Run /sbin/init as init process
```

To summarize, a verbose boot from SD card can roughly be divided in:

| Stage        | Start time (s) | Duration (s) |
| ------------ | -------------- | ------------ |
| U-Boot SPL   |  0             |  0.43        |
| TF-A         |  0.43          |  0.63        |
| U-Boot full  |  1.06          |  5.85        |
| Linux kernel |  6.91          |  4.50        |
| Userspace    | 11.40          | 11.45        |

The full boot captured with `grabserial` can be found [here](files/grabserial_imx_bsp.txt).

### To be continued...

## Referenced documents

- [i.MX93 Technical Reference Manual](https://www.nxp.com/webapp/Download?colCode=IMX93RM)
- [i.MX93 FRDM design files](https://www.nxp.com/webapp/Download?colCode=FRDM-iMX93-DESIGNFILES)

## License

### Content license

With the exception of source code, all content is released under the
[CC BY-SA](https://creativecommons.org/licenses/by-sa/4.0/) license.

### Source code license

Except when stated differently, all source code contained in this work is released under the
MIT license:

```
Copyright 2025-2026 Francesco Valla

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the “Software”), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

[^1]: But this [counts as work time](https://xkcd.com/303), right?
