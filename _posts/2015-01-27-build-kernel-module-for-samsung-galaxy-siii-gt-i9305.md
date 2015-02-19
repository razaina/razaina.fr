---
layout: post
title: "Build Kernel Module for: Samsung Galaxy SIII (GT I9305)"
description: ""
category: 
permalink: "/:title"
comments: true
tags: []
---
{% include JB/setup %}

Here are the infos regarding the phone:

- Model Number: GT-I9305
- CyanogenMod version: 11-20150125-NIGHTLY-i9305
- Android Version: 4.4.4
- Baseband Version: I9305XXUENG1
- Kernel Version: 3.0.64-CM-gc426a2f build02@cyanogenmod #1 Sun Jan 25 13:36:34 PST 2015
- CPU: ARMv7 Processor rev0 (v7)
- Build Number: cm_i9305-userdebug 4.4.4 KTU84Q 00e9a874q7 test-keys


## Step 1: Get the sources of the kernel
Sources code can be found on <a href="http://wiki.cyanogenmod.org/w/I9305_Info" target="_blank">Cyanogendmod's website</a>

## Step 2: Prepare the kernel

### Configure the kernel

If the running kernel has been configured to have /proc/config.gz then pull it in order to
extract the configuration file. Otherwise check in the following directory the one that will fit
your kernel: ../android_kernel_samsung_smdk4412/arch/arm/configs.

I picked the following default configuration file "cyanogenmod_i9305_defconfig".

Then run the make command as follows:
{% highlight ruby %}
export
PATH=$PATH:$NDK_HOME/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86/bin
make ARCH=arm CROSS_COMPILE=arm-linux-androideabi- cyanogenmod_i9300_defconfig
{% endhighlight %}


### Compile the kernel

First of all, I had to modify the Makefile:

- At the beginning of the file set the correct version so all modules will be compiled with the
  right version number.

{% highlight ruby %}
VERSION = 3
PATCHLEVEL = 0
SUBLEVEL = 64
EXTRAVERSION = -CM-gc426a2f
NAME = Sneaky Weasel
{% endhighlight %}

- Based on this <a
  href="http://blog.umbrellaj.com/blog/2013/03/15/trick-on-the-verson-magic-number-of-linux-kernel/"
  target="_blank">blog post</a> I also added the proper parameter while the "setlocalversion" script
  is called so the EXTRAVERSION is not ovewritten by a default version:

  {% highlight ruby %}
  # Store (new) KERNELRELASE string in include/config/kernel.release
include/config/kernel.release: include/config/auto.conf FORCE
	$(Q)rm -f $@
	$(Q)echo "$(KERNELVERSION)$$($(CONFIG_SHELL) $(srctree)/scripts/setlocalversion --save-scmversion $(srctree))" > $@
  {% endhighlight %}


- Based on this <a
  href="https://github.com/CyanogenMod/android_kernel_samsung_tuna/commit/ec1ac589d49081286164246fbaaba0012ae7ecac">
  diff</a> I modified accordingly my Makefile. Otherwise I had the following error:


{% highlight ruby %}
arm-linux-androideabi-ld: error: arch/arm/boot/compressed/piggy.gzip.o: unknown CPU architecture
arm-linux-androideabi-ld: error: arch/arm/boot/compressed/lib1funcs.o: unknown CPU architecture
make[4]: *** [arch/arm/boot/compressed/vmlinux] Error 1
make[3]: *** [arch/arm/boot/compressed/vmlinux] Error 2
{% endhighlight %}

- Finally we can compile the kernel;

{% highlight ruby %}
make ARCH=arm CROSS_COMPILE=arm-linux-androideabi-
{% endhighlight %}

## Step 3: Compile the module

The source code of my module is as follows:

{% highlight c %}
#include <linux/module.h>
#include <linux/kernel.h>

int init_module(void)
{
    printk(KERN_INFO "+++++++++++++++++++++++++  Hello Kernel !!! +++++++++++++++++++++\n");
    return 0;
}


void cleanup_module(void)
{
    printk(KERN_INFO "++++++++++++++++++++++++++++ Goodbye Kernel !!!+++++++++++++++++++++\n");

}

MODULE_AUTHOR("razaina");
MODULE_LICENSE("GPL");
MODULE_VERSION("1");
MODULE_DESCRIPTION("Prints hello world!");
{% endhighlight %}


My Makefile looks like:

{% highlight c %}
obj-m := hello.o
KDIR := /home/razaina/Android-kernel-sources/android_kernel_samsung_smdk4412/
PWD := $(shell pwd)
CCPATH := /home/razaina/android-ndk/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86/bin/
default:
	$(MAKE) CFLAGS_MODULE=-fno-pic ARCH=arm CROSS_COMPILE=$(CCPATH)/arm-linux-androideabi- -C $(KDIR) M=$(PWD) modules
{% endhighlight %}

Do not forget the "-fno-pic" flag otherwise your module won't work and you'll get the following
error message in dmesg:

{% highlight c %}
c0 your_module_name: unknown relocation: 3
{% endhighlight %}

You can also get the following:

{% highlight c %}
Unknown symbol _GLOBAL_OFFSET_TABLE_ (err 0)
{% endhighlight %}

This is because if you do not explicitely tell the compiler to build the module with "NO
Position-Independent Code", then your module will try to look for the "__GLOBAL_OFFSET_TABLE__" while
trying to call some shared functions.

Then compile external modules in order to create the file "Module.symvers" in the kernel's
root directory.


{% highlight ruby %}
make ARCH=arm CROSS_COMPILE=arm-linux-androideabi- modules_prepare
make ARCH=arm CROSS_COMPILE=arm-linux-androideabi- modules
{% endhighlight %}

Finally...compile your module using the make command.

<div id="disqus_thread"></div>
<script type="text/javascript">
/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
var disqus_shortname = 'razaina'; // required: replace example with your forum shortname

/* * * DON'T EDIT BELOW THIS LINE * * */
(function() {
var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
(document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
