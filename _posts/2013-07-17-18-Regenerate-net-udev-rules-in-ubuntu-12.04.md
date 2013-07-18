---
layout: post
title: "Regenerate net udev rules in Ubuntu"
category: posts
---

Today I finally found solution to regenerate udev rules to `/etc/udev/rules.d/70-persistent-net.rules` very quickly.
When I tried to install Openstack Grizzly on my cloned VM with Ubuntu 12.04, I found out that my network interfaces where not in correct order.
I cloned that VM from existing VM and udev rules where regenerated after reboot. Here are steps to regenrate those rules.

There is a script `/lib/udev/write_net_rules` which takes care of generating records to file `70-persistent-net.rules`.
This script does't accept arguments, but you have to set two environment variables to pass data for the script.

Two important variables as `$INTERFACE` and `$MATCHADDR`
Lets say you want to generate udev rule for eth1. First step is to find out mac address which will go to $MATCHADDR variable.

    export INTERFACE=eth1
    export MATCHADDR=`ip addr show $INTERFACE | grep ether | awk '{print $2}'`

 Now run the script and check /etc/udev/rules.d/70-persistent-net.rules file whether rules appeared.

    /lib/udev/write_net_rules
    cat /etc/udev/rules.d/70-persistent-net.rules

Now you can adjust names of interfaces and reboot the machine.
  
