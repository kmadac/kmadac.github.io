---
layout: post
title: "Single user mode in OpenStack (Grizzly) Linux intances"
category: posts
---

Occasionally, during our administration of internal OpenStack cloud, we and our users went into situation when we needed to login into instance which was not accessible over the network (screwed iptables/ssh/key configuration). Dashboard VNC console is only way how to login into VM. Normally, one would expect that when you add the `single` keyword into at the end of kernel parameters, he would see single user mode prompt in the Dashboard web console, and will be able to set root password. Indeed, it is not true in standard image configuration. Common configuration for consoles is to have following kernel parameters:

    console=ttyS0,115200 console=tty console=ttyS0

Your kernels parameters may be bit different, so you either can see single user console prompt in the log window, or you won't see single user console at all. I have seen several advices accross the internet which recommends to edit kvm xml domain file, but it is something which cannot be done by ordinary cloud user - by the way, these advices won't work for me anyway.

The solution found by one of my colleagues is to modify the kernel parameters the way that only one console is specified - virtual console:

    console=tty

so final line for single user mode for CentOS looks like:

    kernel /boot/vmlinuz-2.6.32-431.el6.x86_64 ro root=UUID=8a057bba-2787-46a7-9436-fd6dc8a3aa85 rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD crashkernel=auto rd_NO_DM SYSFONT=latarcyrheb-sun16  KEYBOARDTYPE=pc KEYTABLE=us console=tty notsc single

#### SLES
In SLES image you have to use `init=/bin/sh` instead of `single` kernel parameter.

#### Ubuntu
In official version of Ubuntu cloud images, it is no possible to get to GRUB menu to edit kernel parameters. You need to prepare image for that. There are two files to edit to ensure Grub menu will be accessible during the boot:

- /etc/default/grub
- /etc/default/grub.d/50-cloudimg-settings.cfg

###### /etc/default/grub

Here is snippet of the file, which needs to be changed:

{% highlight bash %}
GRUB_HIDDEN_TIMEOUT=5
GRUB_HIDDEN_TIMEOUT_QUIET=false  # countdown will appear
GRUB_TIMEOUT=5                   # overriden by the value in /etc/default/grub
{% endhighlight %}


###### /etc/default/grub

{% highlight bash %}
# Cloud Image specific Grub settings for Generic Cloud Images
#  These settings are set by the Cloud Image build process

# Set the recordfail timeout
GRUB_RECORDFAIL_TIMEOUT=0

GRUB_TIMEOUT=5			# how long will grub wait for you keypress in grub menu

# Set the default commandline
GRUB_CMDLINE_LINUX_DEFAULT="console=tty"

# Set the grub console type
GRUB_TERMINAL=console
{% endhighlight %}

Run `update-grub` as root after modification, and after the upgrade press `Shift` key during first 5 seconds of boot. You will see Grub menu and you can add `single` kernel parameter by pressing `e` key on Ubuntu grub item and boot up kernel with `F10` key.
