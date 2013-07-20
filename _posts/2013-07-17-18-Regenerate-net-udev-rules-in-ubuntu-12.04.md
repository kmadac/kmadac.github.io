---
layout: post
title: "Regenerate net udev rules in Ubuntu"
category: posts
---

Today, I finally found solution to regenerate udev rules to `/etc/udev/rules.d/70-persistent-net.rules` very quickly.
When I tried to install Openstack Grizzly on my cloned VM with Ubuntu 12.04, I found out that my network interfaces where not detected in correct order. I tried to modify udev rules file, but it was empty. Reason for that was, that I cloned that VM from existing VM and udev rules where not regenerated after reboot. Here are the steps which I used to regenerate those rules.

In Ubuntu and probably in Debian as well, there is a script `/lib/udev/write_net_rules` which takes care of generating records to file `70-persistent-net.rules`.
This script does not accept arguments. You have to set 
environment variables to pass data to the script.

Two important variables are `$INTERFACE` and `$MATCHADDR`
Lets say you want to generate udev rule for eth1. First step is to find out mac address which will go to `$MATCHADDR` variable. Set name of interface to `$INTERFACE` and MAC address of interface to `$MATCHADDR`.

    export INTERFACE=eth1
    export MATCHADDR=`ip addr show $INTERFACE | grep ether | awk '{print $2}'`

Now run the script and check /etc/udev/rules.d/70-persistent-net.rules file whether rules appeared.

    /lib/udev/write_net_rules
    cat /etc/udev/rules.d/70-persistent-net.rules

If rules are there, you can adjust names of interfaces if you want and reboot the machine.
  
