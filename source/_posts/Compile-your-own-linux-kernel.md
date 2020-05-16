---
title: Compile your own linux kernel
date: 2016-03-23 14:33:15
tags: [kernel, linux]
categories: Technology
---

As we know, linux is one of the greatest open source projects in the world and serves millions of enterprises. An open source project means that you can define your own features catering to different application scenarios. All big Internet firms such as Google, Facebook and Aamazon recompile the linux kernel so that features can be added to or removed from the official kernel release version.
Compiling the kernel for linux kernel developers is also unavoidable. In the rest part of this post, attention will focus on tutorials on compiling a linux kernel.



### 1. Getting the kernel source of official release

Nothing comes from nothing. So the first thing before compiling a customized kernel is getting source code.
I strongly recommend using `Git` to download and manage the linux kernel source:

```
git clone source_git_link
```

<!-- more -->


Surely, you can also download the compressed package of source code and then uncompress it.
Go to the source code root directory, there exists a number of directories under it.

{% asset_img kernel-directory.png kernel src file dir %}

The following table[1] illustrates explanation about these directories.

|   Directory   |                  Description                  |
| :-----------: | :-------------------------------------------: |
|     arch      |         Architecture-specific source          |
|     block     |                Block I/O layer                |
|     certs     |             SSL/TLS certification             |
|    crypto     |                  Crypto API                   |
| Documentation |          Kernel source documentation          |
|    drivers    |                drivers Device                 |
|   firmware    | Device firmware needed to use certain drivers |
|      fs       |    The VFS and the individual filesystems     |
|    include    |                Kernel headers                 |
|     init      |        Kernel boot and initialization         |
|      ipc      |        Interprocess communication code        |
|    kernel     |    Core subsystems, such as the scheduler     |
|      lib      |                Helper routines                |
|      mm       |    Memory management subsystem and the VM     |
|      net      |             Networking subsystem              |
|    samples    |          Sample, demonstrative code           |
|    scripts    |       Scripts used to build the kernel        |
|   security    |             Linux Security Module             |
|     sound     |                Sound subsystem                |
|      usr      |   Early user-space code (called initramfs)    |
|     tools     |      Tools helpful for developing Linux       |
|     virt      |         Virtualization infrastructure         |

### 2. Building the kernel source code

After the first step, you come here. Now what you should do is configuring the kernel before compiling. As mentioned previously, it is possible to compile support into your kernel for only the specific features and drivers you want. Configuring the kernel is a required process before building it. By default, the kernel of official release version provides myriad features and supports a varied basket of hardware.

#### (1). Configuration

when you change your current working directory to the root directory of linux kernel source code, you will find there is a file named **.config**. Using command such as `cat .config | more` you can take a glimpse of its content.

```
cat .config | more
```

{% asset_img kernel-configuration.png kernel config file %}

As shown in the picture, kernel configuration is controlled by configuration options, which are prefixed by **CONFIG** in the form **CONFIG_FEATURE**. That is to say, asynchronous IO is controlled by the configuration option **CONFIG_AIO**. This option enables POSIX asynchronous I/O which may by used by some high performance threaded applications[Reference][2]. When this option is set, AIO is enabled; if unset, AIO is disabled.
Configuration options that control the build process are either Booleans or tristates. A Boolean option is either yes or no. Kernel features, such as CONFIG_PREEMPT, are usually Booleans. A tristate option is one of yes, no, or module.The module setting represents a configuration option that is set but is to be compiled as a module (that is, a separate dynamically loadable object). In the case of tristates, a yes option explicitly means to compile the code into the main kernel image and not as a module. Drivers are usually represented by tristates[Reference][3].
Configuration options can also be strings or integers.These options do not control the build process but instead specify values that kernel source can access as a preprocessor macro. For example, a configuration option can specify the size of a statically allocated array[Reference][4].
Kernel provides multiple choices for you to facilitate configurations. A straightfoward way is using a graphical interactive interface: `make menuconfig`.

```
make menuconfig
```

After typing this command, a graphical interactive interface will appears in your screen like this:

{% asset_img kernel-menuconfig.png kernel menuconfig file %}

And you can move the cursor to different options to set them. Because of space, how to configure these options correctly can not be presented. For more detailed knowledge, you can find them in the linux orgnization.

#### (2). Compile and build

Now, it is time to get into the marrow of the second part: Compile && Build. Please make sure that command `make` and `gcc` is installed on your machine firstly.
Just type `make` and all related source code about kernel will be compiled and built, the default Makefile rule will handle everything.

```
make
```

In general, one flaw about the `make` method is that this action spawns only a single job because Makefiles all too often have incorrect dependency information. With incorrect dependencies, multiple jobs can step on each other’s toes, resulting in errors in the build process. However, The kernel’s Makefile have correct dependency information, so spawning multiple jobs does not result in failures. To build the kernel with multiple make jobs, use

```
make -jn
```

The n here is number of jobs to spawn. Usual practice is to spawn one or two jobs per processor. If you have 16 processors in you machine, then you might do

```
make -j32
```

The resulting kernel file is “arch/x86/boot/bzImage” (in x86 platform).

### 3. Installation

After the kernel is built, you can install it. It is possible that the kernel you install cannot boot successfully, so in case of that, you should have at least two kernel installed on you machine so that you can choose the another one to boot.

#### (1). Install modules

Installing modules, thankfully, is automated and architecture-independent. As root, simply run

```
make modules_install
```

After this, you will find a module file under **/lib/modules/a.b.c** where a.b.c is the kernel version.

#### (2). Install kernel

As root user, simply run

```
make install
```

After this, a new kernel file and a new boot image will appear in the **/boot** directory.

#### (3). Set booting order

If you execute all the steps normally, new content about the new installed kernel has been added to **/boot/grub/grub.conf** file. And you can edit the **grub.conf** file to choose to use which kernel when booting.

{% asset_img kernel-grub.png kernel grub file %}

Reboot the machine, and then you will find the new installed kernel in the booting screen.

{% asset_img kernel-booting.png booting file %}

[1]: Love, Robert Love. (2003). Linux Kernel Development, 3, 40-42

[2]: http://cateee.net/lkddb/web-lkddb/AIO.html

[3]: Love, Robert Love. (2003). Linux Kernel Development, 3, 42-43
[4]: Love, Robert Love. (2003). Linux Kernel Development, 3, 43-45
