---
layout: post
draft: true
title: Building a Custom Linux distribution for the Raspberry Pi (Part 1)
---

[Raspbian](https://www.raspbian.org/) is the debian based distribution usually used on Raspberry Pi boards. From there, it can be customized with new packages like a traditional linux desktop. If, for whatever reasons, you're not interested in using this default distribution, this blog post will demonstrate how an entirely custom distribution can be built using [Yocto](https://www.raspbian.org/).

## 1. Yocto Primer
If you're already familiar with Yocto, skip right to section 2.

Yocto is a build system used to build images (as in disk images) that can be installed on a target machine. Usually, the target is an embedded system. We will use Yocto to define the content of a custom Linux distribution and build the disk image that will be installed on our target machine (Raspberry Pi).

Yocto uses the concept of layers to let users customize the distribution's content. Conceptually, we could say that we start from a base layer that defines the distribution's core content and add layers on top of it to define the final content.

Thanks to the concept of layers, not every distribution has to start from scratch. The community around the Yocto Projet maintains a layer called "poky" which contains all the tool needed to build a custom distribution. Namely, it contains `bitbake` the command line utility used to build images and packages. This is the "base" layer we'll use. The community also maintains the meta-openembedded layer which contains the recipes (bitbake scripts) to build most packages commonly included in distributions. We'll also be using this layer.

## 2. Environment setup

Yocto requires a few dependencies on the build host. They are listed in Yocto's [quick start guide](https://www.yoctoproject.org/docs/3.1.1/brief-yoctoprojectqs/brief-yoctoprojectqs.html#brief-compatible-distro).

## 3. Setting up the project

A way to track the different layers combined to build a distribution is to use git submodules. The final git repository of this tutorial can be found [here](https://www.google.com).

First, create an empty git repository and clone within it poky and meta-openembedded.

```
git init ~/yocto-rpi
git clone git://git.yoctoproject.org/poky.git -b dunfell ~/yocto-rpi/poky
git clone git://git.openembedded.org/meta-openembedded -b dunfell ~/yocto-rpi/meta-openembedded
```
poky and meta-openembedded were checkked out on their dunfell branches, which are the latest releases as of writing this.

The next layer to add is meta-raspberrypi. This layer will customize in a machine specific way the recipes building the necesary artifacts to boot Linux on a Raspberry Pi board. Having a machine specifc layer is a common pattern in Yocto. If a different board was targeted, another layer such as meta-freescale could be used.

```
git clone git://git.yoctoproject.org/meta-raspberrypi -b dunfell ~/yocto-rpi/meta-raspberrypi
```

## 4. Building a first image

By reusing layers made by the community, we're already at the point were we can build an image that can be copied on a SD card used to boot a raspberry pi. The image won't contain much, but it'll be enough to bring the board alive. 

The content of this image will be defined in a fourth layer. The image definition could be written in any layer, but mainting it in a separate layer is a useful abstraction. That way, to target a new board, simply swap meta-raspberrypi for meta-mynewboard.

Creating new layers is automated by a script contained in poky. To have access to it (and to many others), source poky/oe-init-build-env. This script, amongst other things, rewrites the `PATH` environnement variable so that we can usethese helper scripts without having to specify their path directly.

```
source poky/oe-init-build-env ~/builds/rpi
```

The argument passed to the script is the build directory used. It can be an empty directory or a directory used in a previous yocto build. If an empty directory is used, it'll be populated with a few folders. The `conf` folder contains configuration information for the build and the `tmp` folder will contain the build artifacts.

Now, multiple bitbake-XYZ and oe-XYZ scripts are available. To create a new layer, use `bitbake-layers`.

```
bitbake-layers create-layer ~/yocto-rpi/meta-blueberry
```

bitbake kindly tells us that it created a new layer, but that it is not currently considered by the build configuration.

```bash
bitbake-layers add-layer ~/yocto-rpi/meta-blueberry
```

This command added the layer in the list of layers to consider for the build located in ~/build/rpi/conf/bblayers.conf. By opening the file, we notice that it contains meta-blueberry, but not the other layers cloned early. We simply add them using the same command.

--> add meta-oe core only

```bash
bitbake-layers add-layer ~/yocto-rpi/meta-openembedded/meta-oe
bitbake-layers add-layer ~/yocto-rpi/meta-raspberrypi
```

As you can see, bitbake doesn't care where layers are or they are tracked. Any layer can be added to a build configuration, wherever they might be. The way I proposed to setup the project was only out of personal preference.

In meta-blueberry, create the the first recipe file within this layer name `basic-image.bb`

```bash
mkdir meta-blueberry/images
touch meta-blueberry/base-image.bb
```

base-image.bb will contain the bare minimum:

```
SUMMARY = "A base image that doesn't do much"
LICENSE = "MIT"
inherit core-image
```

--> modify layer.conf

The key is `inherit core-image`. core-image is a class provided by poky which already defines everything needed to build a basic image. Much like in object oriented programming, a class is inherited to obtain some generic traits. Finally, build the image. Beware, this will take a _while_ since everything is built from scratch on the first run (entire toolchain, libc, everything that ends up the image):

```
$ bitbake base-image
```
