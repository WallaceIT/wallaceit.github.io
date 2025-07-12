# An embedded Linux journey

## Introduction

This work will detail the integration activities performed on an embedded Linux running on a chosen hardware platform for a specific use case, with a particular focus on boot-time optimization. All the steps will be documented, including mistakes, dead paths and wrong choices, in the hope they can serve as a guide for whoever will be doing the same job.

### Disclaimer

Except when stated differently, the Author is not affiliated with any of the companies or organizations mentioned or linked in this work.

All trademarks belong to their owners, including (but not limiting to):

- [Beagleplay](https://www.beagleboard.org/boards/beagleplay) which is a trademark of the BeagleBoard.org Foundation
- [Linux](http://kernel.org/) which is a trademark of Linux Torvalds
- [Yocto Project](https://www.yoctoproject.org/) which is a trademark of The Linux Foundation

## Acronyms

| Acronym | Meaning               |
| ------- | --------------------- |
| OPT     | One-Time Programmable |
| SoC     | System on Chip        |
| CAN     | Controller Area Network |

## Scope of the work

### Hardware

The work will focus on the Beagleplay platform, which is widely available at a low cost and already used in a number of different courses, such as the ones [from](https://bootlin.com/training/embedded-linux/) [Bootlin](https://bootlin.com/doc/training/boot-time/) and [from](https://rootcommit.com/training/embedded-linux/) Root Commit. The Beagleplay is based on a AM625 SoC from Texas Instruments and features a rich set of periphrals, including Etherne, Bluetooth/Wifi, RTC and so on. Detailed specifications can be found at the official product page.

A key point of the AM625 SoC, which will be fundamental for the work presented below, is that most of the documentation for it is public, including the Technical Reference Manual. In the same way, the Beagleplay schematics are public and released under an open license.

### Software

The software will be based on a GNU/Linux stack. The Yocto project will be used as build system, with custom components developed at need. While most of the techniques presented are portable to any build system, the Author believe they are fluent in Yocto and for this reason this framework has been chosen.

Systemd will be used as init system, as it provides a set of functionalities any sufficiently complex embedded product would probably need.

### Use case

By definition, an embedded system cannot cover all use cases. This work will thus focus on one, arbitrally defined by the Author (hoping the various techniques can be easily(-_ish_) ported to others). The following requirements will be considered:

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

As for the secure boot point, a full chain-of-trust will be designed, with the implementation ending whenever blocked by closed tools or destructive actions (such as OTP burning).

## The journey

### Planning

The work will logically follow four different phases, depicted above; in practice, some back-and-forth will probably take place:

1. Setup of development environment
2. Customization of the default configuration to implement the [use case](#use-case)
3. Initial assessment of functionalities and overall performances (including boot time)
4. Optimizations

The "optimizations" stage will be further divided in sub-steps:

- Kernel optimizations
- Userspace optimizations
- Bootloader optimizations

> While logically the bootloader comes first in the boot chain, removing functionalities from it equals to have much less control on what happens next (for example, removing the ability to set kernel command line parameters on-the-fly); for this reason, it will be left as the last task.

### Setup of development environment

#### Build host

To build the system images using the Yocto project a container image coming from [the CROPS project](https://github.com/crops/poky-container) will be used. This approach, successfully leveraged by the Author during the last few years, allows to maintain a controlled and fully reproducible build environment, that can be moved from one machine to another with virtually zero effort. Moreover, it is CI-friendly and has few (if any) cons.

Given a target location on the (real) host machine of `${HOME}/YOCTO`, the following command is used to start the development container using the `Docker` containerization technology:

```sh
docker run --rm -i -t --name beagleplay_${USER} --network="host" -v ${HOME}/YOCTO:/work --workdir=/work crops/poky:ubuntu-22.04
```

Let's analyze the different parameters supplied to `docker run`:

- `--rm`: the container image is removed at exit; this is definitely wanted, as it discourages any runtime customization to the image and helps in maintaining a reproducible environment;
- `-i`: Keep STDIN open even if not attached;
- `-t`: Allocate a pseudo-TTY (allows for a console to be run inside the container);
- `--name beagleplay_${USER}`: give the `beagleplay_${USER}` name to the container, allowing an easier re-attach after a detach (which can be used in case of long running builds, especially if performed on a remote machine); the `${USER}` suffix further helps in a multi-user environment (such as a build server);
- `--network="host"`: use the network interface(s) from the host instead of allocating a dedicated NAT'ed one; can simplify operations behind corporate firewalls and proxies and helps debugging fetching failures;
- `-v ${HOME}/YOCTO:/work`: mount the `${HOME}/YOCTO` directory of the host (real) machine into the container under `/work`; this provides persistence of the build artifacts (and it's the reason `--rm` can be used);
- `--workdir=/work`: automatically move to `/work` once the container is started;
- `crops/poky:ubuntu-22.04`: image to be used for the container.

It should be noted that under some circumstances, such as sources hosted on private servers, additional configurations might be required:

- `-v ${HOME}/.ssh:/etc/skel/.ssh:ro`: mount the user's `.ssh` directory under `/etc/skel/.ssh` in read-only mode; at container startup, the user created inside the container will have this copied into its home directory. **THIS MAY LEAD TO THE LEAK GIT SSH CREDENTIALS AND KEYS** in a multi-user environment (as other users may be able to attach to the container and exfiltrate them).
- `-v ~/.gitconfig:/etc/skel/.gitconfig:ro`:  mount the user's `.gitconfig` file under `/etc/skel/.gitconfig` in read-only mode; at container startup, the user created inside the container will have this copied into its home directory. **THIS MAY LEAD TO THE LEAK OF GIT CREDENTIALS** in a multi-user environment (as other users may be able to attach to the container and exfiltrate them).

Once attached with the command above, the user can detach from the running container using the `CTL-p` `CTL-q` escape sequence, then re-attach with:

```sh
docker attach beagleplay_${USER}
```

#### Clone of meta layers

Here are the Yocto meta layers, along with the chosen git hash, that will be used as a basis.
The `walnascar` branch will be used for layers that provide it, being this the most recent released version of Yocto at the time of writing.

| Layer                                                               | Used sub-layers                  | Branch    | Hash                                     | Here because...                                 |
| ------------------------------------------------------------------- | -------------------------------- | --------- | ---------------------------------------- | ----------------------------------------------- |
| [poky](https://git.yoctoproject.org/poky)                           | `meta`, `meta-poky`              | walnascar | 41038342a471b4a8884548568ad147a1704253a3 | Basic layer                                     |
| [meta-openembedded](https://git.openembedded.org/meta-openembedded) | `meta-oe`                        | walnascar | 3c6844219a69d9ca8cb0f1579114efe9a9e2a646 | Dependency of meta-ti                           |
| [meta-arm](https://git.yoctoproject.org/meta-arm)                   | `meta-arm`, `meta-arm-toolchain` | walnascar | c63ce2117ed9ad720c61a1e2df5d381ca1ac54a2 | Dependency of meta-ti                           |
| [meta-ti](https://git.yoctoproject.org/meta-ti)                     | `meta-ti-bsp`, `meta-beagle`     | master    | 19aba166544a558e3596de3376f80375e0bd9758 | Contains the `beagleplay` machine configuration |

Meta layers can be cloned directly inside the [running build container](#build-host); once this is done, the [build directory](#setup-of-the-build-directory) shall be created.

#### Setup of the build directory

Once the [meta layers](#clone-of-meta-layers) have been cloned, the build directory can be created using:

```sh
source poky/oe-init-build-env build.beagleplay
```

This will initialize the build environment will all the required variables and will make `bitbake` accessible to the user. The first time it is launched, it will also create the `build.beagleplay` directory and copy a skeleton configuration under it `conf/` subdirectory, where it can be customized. 

##### `bblayers.conf`

The `bblayers.conf` file contains the list of meta layers used by the project; according to what has been defined [before](#clone-of-meta-layers), it has to be modified to look more or less like this:

```
POKY_BBLAYERS_CONF_VERSION = "2"  
  
BBPATH = "${TOPDIR}"  
BBFILES ?= ""  
  
BBLAYERS ?= " \  
 /work/poky/meta \  
 /work/poky/meta-poky \  
 /work/meta-openembedded/meta-oe \  
 /work/meta-arm/meta-arm \  
 /work/meta-arm/meta-arm-toolchain \  
 /work/meta-ti/meta-ti-bsp \  
 /work/meta-ti/meta-beagle \  
 "
```

It is important to note that not all of the sub-layers contained inside the cloned meta layers are used; this is the case for example of `poky/meta-yocto-bsp`, which contains machine configurations for the Yocto reference machines and has thus no use in the defined configuration (i.e., a system running on the Beagleplay).

##### `local.conf`

The `local.conf` file holds the build-specific configurations. The explanation of all the different key-value pairs would add no value to the current work, and will thus be left to more autoritative sources, such as the [official Yocto documentation](https://docs.yoctoproject.org/).

To obtain the initial build for the Beagleplay, the following values shall however be added at the bottom of this file:

```
# Target machine is the Beagleplay
MACHINE = "beagleplay"

# The Poky reference distro will be used
DISTRO = "poky"

# Allow empty passwords, even for the root user, and allow root to login.
# These are development configuration and _shall_ be removed on a production system!

EXTRA_IMAGE_FEATURES = "allow-empty-password empty-root-password allow-root-login"

# Use systemd as init manager
INIT_MANAGER = "systemd"

# Use systemd-boot as EFI stub
EFI_PROVIDER = "systemd-boot"
```

If the host machine has limited storage, the following configuration can be added to remove all the intermediate files that are generated during the build of the software components:

```
INHERIT += "rm_work"
```

#### Image build

Once all the configurations have been performed, it is possible to build a complete image for the system. The `core-image-full-cmdline` will be used at first, as it provides a sufficient complete system without introducing too much overhead.

```sh
bitbake core-image-full-cmdline
```

> Depending on the speed of the Internet connection and the capabilities of the host machine, this can take an amount of time that can vary from a few minutes to several hours.

#### Image flashing

Once built, the gzip'ed image can be found under `build.beagleplay/deploy-ti/images/beagleplay/core-image-full-cmdline-beagleplay.rootfs.wic.gz`. Such image can be written to a microSD card using a number of tools; all of them have to be used on the (real) host machine, since the container has no access to it.

##### Flash using `Balena Etcher`

The `Balena Etcher` graphical application from [balena.io](https://etcher.balena.io/) can be used to flash the built image to a microSD. Since the Author has no real experience using it, its usage will no be treated here.

#### Flash using `bmaptool`

The `bmaptool` Python application can be used to flash the built image to a microSD:

```sh
sudo bmaptool copy ${HOME}/YOCTO/build.beagleplay/deploy-ti/images/beagleplay/core-image-full-cmdline-beagleplay.rootfs.wic.xz /dev/mmcblk0
```

> Here the microSD is assumed to be accessible through `/dev/mmcblk0`; this might need to be adapted to the system at hand.

The `bmaptool` application is available for most GNU/Linux distributions under the `bmap-tools` package. Along with the `.wic.gz` image file, it uses the accompanying `.wks` file to write to the target microSD only the blocks that are really needed (i.e., skipping the empty ones).

#### Flash using `bmap-writer`

The `bmap-writer` application can be used to flash the built image to a microSD:

```sh
sudo bmap-writer ${HOME}/YOCTO/build.beagleplay/deploy-ti/images/beagleplay/core-image-full-cmdline-beagleplay.rootfs.wic.xz /dev/mmcblk0
```

> Here the microSD is assumed to be accessible through `/dev/mmcblk0`; this might need to be adapted to the system at hand.

The `bmap-writer` application would probably need to be compiled from scratch following instructions from [its git repository](https://github.com/embetrix/bmap-writer).  Along with the `.wic.gz` image file, it uses the accompanying `.wks` file to write to the target microSD only the blocks that are really needed (i.e., skipping the empty ones).

## Referenced documents

- [AM625 Technical Reference Manual](https://www.ti.com/lit/pdf/spruiv7)
- [Beagleplay Design and Specifications](https://docs.beagleboard.org/boards/beagleplay/03-design.html)

## License

### Content license

With the exception of source code, all content is released under the [CC BY-SA](https://creativecommons.org/licenses/by-sa/4.0/) license.

### Source code license

Except when stated differently, all source code contained in this work is released under the MIT license:

```
Copyright 2025 Francesco Valla

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```
