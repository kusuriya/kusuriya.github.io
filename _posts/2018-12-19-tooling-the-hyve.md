---
layout: default
title: Tooling the hyve
---

Hey all, It's been a while here so I figured I would write up some stuff I have been playing with. This go around it is BHYVE on FreeBSD. 
BHYVE is a great hypervisor project that started with the idea of FreeBSD needing its own hypervisor like KVM, but if we were going to make KVM today what would we do?
There are all sorts of features with it and it gets better every day but as a Sysadmin my major gripe with it has been simple, tooling. Well Libvirt to the rescue.

<!-- more -->

## Installation
So installing on FreeBSD is pretty easy `pkg install libvirt` it will spin around, install the latest it has, and be ready to go. Now the release notes to mention setting up the default networking but honestly unless you have the desire to use NAT networking you dont need to pull this in. All in all this part was super painless.

## Initial access and config of the daemon
So the package info also notes that you should add yourself to the libvirt group, do it and log out and back in. This will make dealing with libvirt and virsh easier to deal with. The socket permissions are already configured for the group that is made so make use of it, I will never actually condone running as root as a valid solution to a permissions problem. 

So after that take a look through `/usr/local/etc/libvirt/libvirtd.conf` there are a few nice tidbits in there if you are running a fleet like having libvirtd listen on a port on the network, ACLs, and other things. Going into best practices and ideals about what you should set any of this too is a bit beyond the scope here, just know the defaults do work on FreeBSD

## Networking
Honestly the default network mentioned above does not work right out of the box. libvirt knows how to make the bridges and make the DHCP server work, but doesnt seem to configure the nat correctly. So honestly for now there are a few more manual steps to getting the network configured as compared to linux. So I will split this up into a couple parts, one involving NAT, one involving a direct bridge. 

### NAT
So for my local semi isolated labs that I still like having access to the internet with I love to hide them behind a NAT gateway and firewal. For this one here you really need just a couple things

1. `ls /usr/local/share/examples/libvirt/networks/default.xml` which is the template for the default natted network
2. pf enabled and installed

Now the reason I go for PF here is it is the easiest for me to configure and get going. you could use natd or ipfw or anything else that will nat network traffic, FreeBSD has a lot of ways to skin that cat and is great for giving you options for high performance networks. 

So the basics of setting this up are pretty simple. First you want to copy the default network template into `/usr/local/etc/libvirt/qemu/networks/default.xml` now with that you will have a basic natted network configured in libvirt. When you start it it will create a bridge named `vir-br0` start dnsmasq up and server DHCP from the IP ranges listed in `default.xml` and the guests if you were to start them now could speak with eachother and the host. The part missing here is the actual NAT. So for that it is just a quick trip into the pf.conf file and adding a nat-to directive like `from 192.168.0.1/24 to (egress:0) nat-to egress`. With that you should be able to either start PF or just reload the ruleset and get natting. No fuss or muss, libvirt does most of this correctly and just doesnt know how to handle PF. For a more flexible setup I would consider looking at making rule anchors in an external file that automation has access to modify and add new networks as needed. This will allow you to not have a giant unwieldy pf.conf and make automation less prone to errors.

### Bridged

This guy is actually fairly simple here. Create a bridge like you would for any other applicaiton, add an adapter that will serve as the bridged port, and generate a config telling libvirt that you want bridged networking. That is the gist of what is needed for the bridged networking part. Now the thoughts might be churning about the bridge and why a bridge and so on and so on. I honestly find it best to treat it and think of it as a virtual switch. But lets get a bit more detaail here.

#### Generating the bridge
This part is honestly easy as can be and you can find the instructions in the [handbook](https://www.freebsd.org/doc/handbook/network-bridging.html). But first you will want to create the bridge itself `ifconfig bridge create` the output from that will tell you the bridges name in this case since the test box has no other bridges its `bridge0`. Now we need to add the uplink to the bridge `ifconfig bridge0 addm em0` and set Spanning Tree Protocol on the uplink `ifconfig bridge0 stp em0` and there you go, you should have a bridge that is ready to pretend to be a virtual switch, just add tap interfaces for everything that needs to communicate on the switch.

#### Configuring libvirt to talk on the bridge.
This in itself is pretty easy, you can use something like virsh to create the bridged network, or just create the XML by hand.
I did the inital XML creation by hand and it honestly is just
```xml
<network>
  <name>default</name>
  <uuid>8acd39f7-f050-11e8-8d5f-00259069ef52</uuid>
  <forward mode='bridge'/>
  <bridge name='bridge0'>
</network>
```

With that you will nave a network named default that you can set your VMs up on and they will be bridged out. It is worth noting if you do this by hand that UUID must be unique for every different network you have.

## My first VM
So basically it builds all to this. Now everyone that is used to using libvirt on other platforms will know this is the sort of sucky part where there are tools to help you since libvirt does everything in XML. Not all of the tools understand the BHYVE driver, because of that I elect to manually modify templates. For some brevity I will link the [bhyve driver page](https://libvirt.org/drvbhyve.html), you can steal what I am using as a template from there. Once you have that template going its really exactly like you are used to on any other platform, just 'virsh create /tmp/sweet-linux-uefi-vm.xml' and out comes your VM. The bhyve driver supports UEFI and VNC so you get all of the goodies there.

## Caveats
- The tried their best but this is still a community supported driver that has been mainlined, as such if bhyve has added features there is no promise that the libvirt driver will understand them. 
- Libvirt is still libvirt, and youll need to secure it appropriately, the underlined OS doesnt nessicarily improve the security of the software running on it.
- The biggest downer for me is this one, nobody seems to have updated virt-manager's creator, so while you can use virt-manager to interact with existing virts you cant create new ones. what I have done to get around this guy is I just do the create with virsh and spin them up then use virt-manager to manage them. No fuss no mess.

## Overall impression
Honestly other than the XML this is probably the most polished familiar way of tooling bhyve that with minor overhad would allow the porting of lots of very familiar tools for non BSD admins. The bus factor is low on it so more hands would make light work and having the BHYVE team and the enthusiast such as my self adopt the driver would take us a long way in having a really neat supportable toolable virtualizer based on FreeBSD.