layout: post
draft: true
title: Building a Custom Linux distribution for the Raspberry Pi (Part 1)
---

[Raspbian](https://www.raspbian.org/) is the debian based distribution usually used on Raspberry Pi boards. From there, it can be customized with new packages like a traditional linux desktop. If, for whatever reasons, you're not interested in using this default distribution, this blog post will demonstrate how an entirely custom distribution can be built using [Yocto](https://www.raspbian.org/).

## 1. Yocto Primer
If you're already familiar with Yocto, skip right to section 2.

Yocto is a build system used to build images (as in disk images) that can be installed on a target machine. Usually, the target is an embedded system. We will use Yocto to define the content of a custom Linux distribution and build the disk image that will be installed on our target machine (Raspberry Pi).

Yocto uses the concept of layers to let users customize the distribution's content. Conceptually, we could say that we start from a base layer that defines the distribution's core content and add layers on top of it to define the final content.

Thanks to the concept of layers, not every distribution has to start from scratch. The community around the Yocto Projet maintains a layer called "poky" which contains all the tool needed to build a custom distribution. Namely, it contains `bitbake` the command line utility used to build images and packages. This is the "base" layer we'll use. The community also maintains the meta-openembedded layer which contains the recipes (files that bitbake parses) to build most packages commonly included in distributions. We'll also be using this layer.

## 2. Environment setup

Yocto requires a few dependencies on the build host. They are listed in Yocto's [quick start guide](https://www.yoctoproject.org/docs/3.1.1/brief-yoctoprojectqs/brief-yoctoprojectqs.html#brief-compatible-distro).

## 3. Setting up the project

One way to track the layers combined to build a distribution is to use git submodules. The final git repository for this tutorial can be found [here](https://www.google.com).

First, create an empty git repository and clone within it poky and meta-openembedded.

```
git init ~/yocto-rpi
git clone git://git.yoctoproject.org/poky.git -b dunfell ~/yocto-rpi/poky
git clone git://git.openembedded.org/meta-openembedded -b dunfell ~/yocto-rpi/meta-openembedded
```
poky and meta-openembedded are checked out on their `dunfell` branches. `dunfell` is latest release's name as of writing this.

The next layer used is meta-raspberrypi. Mainly, it customizes in a machine specific way the recipes which build the artifacts used to boot Linux. Adding a machine specifc layer is a common pattern with Yocto, because it makes it easier to port the distribution to a new board cince all the machine specifc tweaks are circumscribed to a single layer.

```
git clone git://git.yoctoproject.org/meta-raspberrypi -b dunfell ~/yocto-rpi/meta-raspberrypi
```

## 4. Building a first image

By reusing layers made by the community, we're already at the point were we can build an image that can be copied on a SD card used to boot a raspberry pi. The image won't contain much, but it'll be enough to bring the board alive.

The content of this image will be defined in a fourth layer. The image definition could be written in any layer, but mainting it in a separate layer is a useful abstraction. That way, to target a new board, simply swap meta-raspberrypi for meta-mynewboard.

Creating new layers is automated by a script contained in poky. To have access to it (and to many others), source `poky/oe-init-build-env`. This script, amongst other things, rewrites the `PATH` environnement variable to give direct access to these helper scripts.

```
source poky/oe-init-build-env ~/builds/rpi
```

The argument passed to the script is the build directory used. It can be an empty directory or a directory used in a previous yocto build. If an empty directory is used, it'll be populated with a few folders. The `conf` folder contains configuration information for the build and the `tmp` folder will contain the build artifacts.

Now, multiple bitbake-XYZ and oe-XYZ scripts are available. To create a new layer, use `bitbake-layers`.

```
bitbake-layers create-layer ~/yocto-rpi/meta-blueberry
```

bitbake kindly tells us that it created a new layer, but that it is not currently considered by the build configuration.

```
bitbake-layers add-layer ~/yocto-rpi/meta-blueberry
```

This command added the layer in the list of layers to consider for the build located in ~/build/rpi/conf/bblayers.conf. By opening the file, we notice that it contains meta-blueberry, but not the other layers cloned earlier. Simply add them using the same command.

```
bitbake-layers add-layer ~/yocto-rpi/meta-openembedded/meta-oe
bitbake-layers add-layer ~/yocto-rpi/meta-raspberrypi
```

As you can see, bitbake doesn't care where the layers are or how they are tracked. Any layer can be added to a build configuration, wherever they might be. The way I proposed to setup the project was only out of personal preference.

Now, let's configure the first image. In meta-blueberry, create a first recipe file within named `basic-image.bb`

```
mkdir meta-blueberry/images
touch meta-blueberry/base-image.bb
```

For know, create base-image.bb with the bare minimum:

```
SUMMARY = "A base image"
LICENSE = "MIT"
inherit core-image
```

Adding a recipe in a layer renders it available as a build target with `bitbake`. How does it know to look in the images directory? By default, it doesn't. Add this new subdirectory in meta-blueberry/conf/layer.conf as such:

```
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend \
            ${LAYERDIR}/images/*.bb"
```

In the recipe file, the line that does all the work is `inherit core-image`. core-image is a class provided by poky which already defines everything needed to build a basic image. Much like in object oriented programming, a class is inherited to obtain some generic traits.

We must define for wich machine (rpi2, rpi3, etc) the image is built. This is done by setting the `MACHINE` [variable](https://www.yoctoproject.org/docs/latest/mega-manual/mega-manual.html#var-MACHINE) in `local.conf`.  This is another file that is part of the build configuration, so it is located in the build directory, `~/builds/rpi`. Open this file, and set MACHINE to your exact target board. You can see each supported raspberry pi macines in the folder `meta-raspberrypi/conf/machine/`.

```
MACHINE = "raspberrypi3"
```

There's a one last thing to think about before launching the build. Since the beginning, we've talked about building an "image", but what is the expected format of this image? The answer will change depeding on the target. For the raspberry pi, the SD card the board is booted on has to have 2 partitions, one containing the boot files and a second containing the root filesystem (rootfs). The boot partition is expected to be a fat32 filesystem, and the second one and ext2 or ext3 filesystem. Fortunately for us, all the hard work is already done within the meta-raspberrypi layer. There's an image class called `sdcard_image-rpi` that will create an image in just the right format so that we can simply `dd` it over an sd card. We want to use this new class for our image, but we don't really want to stain our meta-blueberry layer with machine specific informations. To achieve this, we will use the `IMGCLASSES` variable. This variable contains a list a additionnal classses to apply to every images that inherit image.bbclass (which base-image does throught core-image). For now, we will add also add this information in the `local.conf` file.

```
IMGCLASSES += "sdcard_image-rpi"
IMAGE_FSTYPES += "rpi-sdimg"
```

The first line will add the inherit to our image recipes and the second one will trigger the build of a "rpi-sdimg" image type.

Finally, launch the build. Beware, this will take a long time since everything is built from scratch on the first try. Eventually, the packages are automatically locally cached. A cache server can also be set and shared amongts build hosts.

```
bitbake base-image
```
