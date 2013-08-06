---
layout: post
title: "Building Virtual Openstack Quantum Testing Environment"
category: posts
---


If I'm talking about `VM`, it is virtual machine in KVM in my notebook where Openstack and his services are installed and Netapp simulator is installed. In case of `instances`, I mean virtual machines created within OpenStack, so it is VM in VM.  
  
### Why I did'n use DevStack?

I wanted to simulate my original real environment and real installation steps as close to reality as possible. Installation of [DevStack](http://devstack.org/) is automated and it was hard for me to guess what is going on in background.

Here you can see my environment which I built in my notebook:

![My environment](/images/openstack_vm_testing_environment.png)

You can see two VMs which simulates two physical nodes. One node `ostack-grizzly` which was used for installation of Grizzly single node installation and `ubuntu-ontap-sim7` which was Ubuntu 12.04 LTS with Netapp Simulator in 7-mode.

### Virtual switch for connecting VMs
I created two virtual switch in order to interconnect my two VMs. One switch `ovsbr0` for which will be used for connection between OpenStack nodes (as this article is only about single-node installation, I created it only for possible future use) and `ovsstor` used for simulation of physical connectivity between OpenStack instances and Netapp (provider network)

Create two openvswitch switches:

    ovs-vsctl add-br ovsbr0
    ovs-vsctl add-br ovsstor

## ostack-grizzly VM

I created VM with 2GB of RAM and 6GB of disk space. I installed Ubuntu 12.04 LTS and used KVM as Hypervisor and 
configured 3 interfaces for VM in following order - eth0, eth1, eth2. Here is snippet from output of command

    virsh dumpxml ostack-grizzly

{% highlight xml %}
    <interface type='bridge'>
      <mac address='52:54:00:5c:eb:4a'/>
      <source bridge='ovsbr0'/>
      <virtualport type='openvswitch'>
        <parameters interfaceid='784ee2ba-6cd6-948a-922a-bda1abcdf01c'/>
      </virtualport>
      <target dev='vnet1'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>

    <interface type='network'>
      <mac address='52:54:00:2c:e0:52'/>
      <source network='default'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

    <interface type='bridge'>
      <mac address='52:54:00:9b:83:16'/>
      <source bridge='ovsstor'/>
      <virtualport type='openvswitch'>
        <parameters interfaceid='0707a806-bf64-ac37-c548-1a57c65bc60b'/>
      </virtualport>
      <target dev='vnet4'/>
      <alias name='net2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </interface>
{% endhighlight %}

**eth0** - interface used by OpenStack services. Openstack Services and APIs are listening on this interface. In case of multi-node installation, compute node and network node communicates with control node over this interface.

**eth1** - interface used for external connectivity to OpenStack instances and also management interface for KVM VM. I used IP addresse assigned on this interface to ssh to VM. Connected to virbr0.

**eth2** - interface used for provider network connectivity. IP addresses from this network will be assigned to instances created in OpenStack 

I used **Virtual Machine Manager** to configure VMs. There is a trick to connect created interface to openvswitch. Firstly, create interface as you would like to create normal single interface.

![Create interface 1](/images/ovs-virtualmachinemanager1.png)

Finish it, Select it in the left menu, change source mode to **Bridge** and put `openwswitch` to Virtual port type

![Create interface 2](/images/ovs-virtualmachinemanager2.png)

Then select **Specify shared device name** in **Source device** and enter Bridge name (`ovsstor` and `ovsbr0` in my case).

![Create interface 3](/images/ovs-virtualmachinemanager3.png)

After boot up of VM, 3 interfaces were detected by Linux kernel. I had to reorder it them by modification of udev rules in */etc/udev/rules.d/70-persistent-net.rules*. In case you cannot see any rule, you can see how to generate them in [another article](/posts/18-Regenerate-net-udev-rules-in-ubuntu-12.04/) in my blog. It is important to know which interface is connected to which virtual switch.

I used following configuration in */etc/networking/interfaces* to have everything prepared for OpenStack installation.

{% highlight bash %}
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 10.10.100.51
netmask 255.255.255.0

auto eth1
iface eth1 inet manual
up ifconfig $IFACE 0.0.0.0 up
up ip link set $IFACE promisc on
down ip link set $IFACE promisc off
down ifconfig $IFACE down

auto br-ex
iface br-ex inet static
address 192.168.122.200
netmask 255.255.255.0
gateway 192.168.122.1
dns-nameservers 8.8.8.8

auto eth2
iface eth2 inet manual
up ifconfig $IFACE 0.0.0.0 up
up ip link set $IFACE promisc on
down ip link set $IFACE promisc off
down ifconfig $IFACE down

auto br-stor
iface br-ex inet manual
up ifconfig br-stor up
{% endhighlight %}

According to [this](http://four-eyes.net/2013/06/open-vswitch-basic-initial-setup-on-ubuntu-12-04/) article I used workaround to turn off networking boot delay, because */etc/network/interfaces* is not compatible with OpenvSwitch. I modified */etc/init/failsafe.conf* and decreased sleep time to 1:

{% highlight bash %}
$PLYMOUTH message --text="Waiting for network configuration..." || :
sleep 40
$PLYMOUTH message --text="Waiting up to 60 more seconds for network configuration..." || :
sleep 59
$PLYMOUTH message --text="Booting system without full network configuration..." || :
{% endhighlight %}

to

{% highlight bash %}
$PLYMOUTH message --text="Waiting for network configuration..." || :
sleep 1
$PLYMOUTH message --text="Waiting up to 60 more seconds for network configuration..." || :
sleep 1
$PLYMOUTH message --text="Booting system without full network configuration..." || :
{% endhighlight %}









