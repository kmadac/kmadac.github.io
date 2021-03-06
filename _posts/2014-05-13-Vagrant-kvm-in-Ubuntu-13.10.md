---
layout: post
title: "Vagrant-kvm in Ubuntu 13.10"
category: posts
---

Vagrant is very valuable tool for every developer who would like to automate creation of testing environments. I wanted to use it on my development Linux machine, but it is primarly written for Virtualbox.
Because I used to use kvm hypervisor and I didn't want to mix different virtualization platforms on my development machine, I was looking for a tool which can provision VMs in KVM as simple as Vagrant can.
Finally I found plugin for vagrant called vagrant-kvm, which can utilize KVM for virtualization, yet use Vagrant comfort and simplicity.  

## Install vagrant

I downloaded and installed most recent Vagrant version (1.6.2 x64) of .deb package for Ubuntu directly from Vagrant downloads [page][2].

    # cd ~/Downloads
    # dpkg -i vagrant_1.6.2_x86_64

Then I installed kvm plugin

    # vagrant plugin install vagrant-kvm

Installation of plugin was unsuccessful, with following error:

    An error occurred while installing ruby-libvirt (0.4.0), and Bundler cannot continue.
    Make sure that `gem install ruby-libvirt -v '0.4.0'` succeeds before bundling.

I solved the issue by installing `libvirt-dev` package

    # apt-get install libvirt-dev

After this, plugin was installed successfuly.

## Get Vagrant KVM image

It is not possible to use Vagrant images prepared for Virtualbox, so we have to convert images for KVM, or download already prepared images.
    
I downloaded KVM box directly from [KVM Boxes][1] Github page

    # vagrant box add trusty64 https://vagrant-kvm-boxes-si.s3.amazonaws.com/trusty64-kvm-20140418.box
    # vagrant box list
    trusty64 (kvm, 0)

Now, we can prepare `Vagrantfile` where we have VM configuration

    # nano Vagrantfile

and put there following lines

```
Vagrant.configure("2") do |config|
  config.vm.box = "trusty64"
  config.vm.network :private_network, ip: "192.168.123.10"
end
```

This file specifies that we want to run `trusty64` box and it'll have internal IP address 192.168.123.10. 

Now run following command to run this machine in directory where you have Vagrantfile file:

```
# vagrant up --provider=kvm
Bringing machine 'default' up with 'kvm' provider...
==> default: Importing base box 'trusty64'...
==> default: Matching MAC address for NAT networking...
==> default: Preparing network interfaces based on configuration...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 192.168.123.10:22
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: Warning: Connection timeout. Retrying...
==> default: Machine booted and ready!
==> default: Creating shared folders metadata...
==> default: Rsyncing folder: /home/kmadac/vagrant/ => /vagrant

# vagrant ssh
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-24-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Sat Apr 19 01:03:51 2014 from 10.0.2.2
vagrant@ubuntu-14:~$ 

```

and we are in our new Ubuntu 14.04 server machine :-)

We have to specify provider otherwise we will get error message that VirtualBox is not installed.
There is also possibility to specify default provider. Set environment variable `VAGRANT_DEFAULT_PROVIDER` to kvm and you don't need to specify provider on command line.

{% highlight bash %}
export VAGRANT_DEFAULT_PROVIDER=kvm
{% endhighlight %}

Now we can simply run

    # vagrant up

and our VM will be provisioned by vagrant using KVM.

If you want destroy the VM, run 

    # vagrant destroy


### How to convert Virtualbox image to KVM 
I tried to convert existing images with vagrant-mutate plugin, but it always finished with error:

    # vagrant mutate precise32 kvm
    Vagrant-mutate does not support 0 for input or output

It seems it is incompatibility between Vagrant 1.6.2 and mutate plugin. I opened [issue #55][3] and author of plugin recommended to upgrade mutate plugin to version 0.3.0.
Version 0.3.0 was not uploaded in rubygems, so I cloned vagrant-mutable github repo, generated gem package and installed it:

```  
cd /tmp; git clone https://github.com/sciurus/vagrant-mutate.git
cd vagrant-mutate
rake build
cd pkg
vagrant plugin install vagrant-mutate-0.3.0.gem
```

After upgrade of plugin, I addded Virtualbox image from [Vagrantbox.es][4]

```
# vagrant box add saucy64 http://cloud-images.ubuntu.com/vagrant/saucy/current/saucy-server-cloudimg-amd64-vagrant-disk1.box
==> box: Adding box 'saucy64' (v0) for provider: 
    box: Downloading: http://cloud-images.ubuntu.com/vagrant/saucy/current/saucy-server-cloudimg-amd64-vagrant-disk1.box
==> box: Successfully added box 'saucy64' (v0) for 'virtualbox'!

# vagrant box list
saucy64  (virtualbox, 0)
trusty64 (kvm, 0)
```

Now we are ready to convert saucy64 from virtualbox to kvm

```
# vagrant mutate saucy64 kvm
Converting saucy64 from virtualbox to kvm.
    (100.00/100%)
The box saucy64 (kvm) is now ready to use.

# vagrant box list
saucy64  (kvm, 0)
saucy64  (virtualbox, 0)
trusty64 (kvm, 0)
```

Now, we can prepare `Vagrantfile` for sauce64 and run it with

    vagrant up 


[1]: https://github.com/adrahon/vagrant-kvm/wiki/List-boxes "KVM-Boxes"
[2]: https://www.vagrantup.com/downloads.html "Vagrant downloads"
[3]: https://github.com/sciurus/vagrant-mutate/issues/55 "Issue 55"
[4]: http://www.vagrantbox.es/ "vagrantbox.es"
