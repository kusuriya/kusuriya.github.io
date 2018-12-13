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

1. `/usr/local/share/libvirt/quemu/networks/default.xml` which is the template for the default natted network
2. pf enabled and installed

Now the reason I go for PF here is it is the easiest for me to configure and get going. you could use natd or ipfw or anything else that will nat network traffic, FreeBSD has a lot of ways to skin that cat and is great for giving you options for high performance networks. 

So the basics of setting this up are pretty simple. First you want to copy the default network template into `/usr/local/etc/libvirt/qemu/networks/default.xml` now with that you will have a basic natted network configured in libvirt. When you start it it will create a bridge named `vir-br0` start dnsmasq up and server DHCP from the IP ranges listed in `default.xml` and the guests if you were to start them now could speak with eachother and the host. The part missing here is the actual NAT. So for that it is just a quick trip into the pf.conf file and adding a nat-to directive like `from 192.168.0.1/24 to (egress:0) nat-to egress`. With that you should be able to either start PF or just reload the ruleset and get natting. No fuss or muss, libvirt does most of this correctly and just doesnt know how to handle PF. For a more flexible setup I would consider looking at making rule anchors in an external file that automation has access to modify and add new networks as needed. This will allow you to not have a giant unwieldy pf.conf and make automation less prone to errors.

### Bridged

This guy is actually fairly simple here. Create a bridge like you would for any other applicaiton, add an adapter that will serve as the bridged port, and generate a config telling libvirt that you want bridged networking. That is the gist of what is needed for the bridged networking part. Now the thoughts might be churning about the bridge and why a bridge and so on and so on. I honestly find it best to treat it and think of it as a virtual switch. But lets get a bit more detaail here.

#### Generating the bridge
This part is honestly easy as can be and you can find the instructions in the [handbook](https://www.freebsd.org/doc/handbook/network-bridging.html). But first you will want to create the bridge itself `ifconfig bridge create` the output from that will tell you the bridges name in this case since the test box has no other bridges its `bridge0`. Now we need to add the uplink to the bridge `ifconfig bridge0 addm em0` and set Spanning Tree Protocol on the uplink `ifconfig bridge0 stp em0` and there you go, you should have a bridge that is ready to pretend to be a virtual switch, just add tap interfaces for everything that needs to communicate on the switch.

#### Configuring libvirt to talk on the bridge.