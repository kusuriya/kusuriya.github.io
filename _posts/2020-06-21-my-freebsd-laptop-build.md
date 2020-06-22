---
layout: default
title: My FreeBSD Laptop Build
Date: 2015-10-26 21:44:00 +/-0000
published: yes
tags: 
    - FreeBSD
    - Laptops
    - Workstation
---
# My FreeBSD Laptop Build
I have always liked Thinkpad hardware and when I started to do more commuting I decided I needed something that had a decent sized screen but fit well on a bus. Luckily about this time Lenovo gave me a nice gift in the [Thinkpad X390](https://www.lenovo.com/us/en/laptops/thinkpad/thinkpad-x/X390/p/22TP2TX3900). Its basically the famous X2xx series but with a 13" screen and smaller bezel.  

So with this laptop I figured it was time to actually put the docs together on how I got my FreeBSD workstation working on it. I will here in the near future have another post that will cover this for HardenedBSD as well since the steps are similar but have a few extra gotchas due to the extra hardening.

<!--more-->

## Hardware
The guts are:
- 8th Generation Intel Core i7 1.9 GHz
- 16 GB Ram
- Replaced the 512GB NVMe Disk with a 1TB Samsung Evo Pro NVMe disk
- 13.3" 1920x1080 IPS display
- Intel Graphics
- Intel Wireless 9560 802.11AC card (2x2)
- Bluetooth 5.0

Some of the more important parts of this is it supports USB-C and charging from USB-C. I've been moving more and more to all of my mobile tech charging from USB-C so I can strip down what I am carrying. It has became a requirement for me. For the people that start asking "Well how do you deal with needing high performance or high power processors for ______". My answer is what it has been for nearly 15 years now. I have 3 6 core 12 thread Xeon workstations at home. If the laptop cant do it in a reasonable time it was the wrong device for the job and the job gets offloaded to my workstations.

But I digress, really the only way this could get better is if it used open firmware like the System76 laptops that are coming out. Im keeping an eye on the newer System76 machines and honestly if I get the spare money to just experiment with a new toy or one magics itself onto my door step this laptop will live a full service life but the next one will probably be a System76 for various ethical and peace of mind reasons. But if a shiny new System76 laptop wills itself into existence in my fleet you can bet It will get BSD poured onto it and you all will hear about it.

What works:
- Wireless
- Keyboard
- Trackpad
- Trackpoint
- Backlight
- Sleep and Resume
- Sound

What I couldn't get working:
- MicroSD reader
- Hibernate

So with the hardware in hand the next part is the first step, bsdinstall.

## Installer
So this is based on using 13-Current or newer. The X390 needs 13-Current due to iwm changes. The iwm driver on release as of 2020-06-21 does not support the card in this. The standard high points to hit for the installer:
- ZFS on root
- Full disk encryption, My mobile machines with SSDs always get encrypted disks
- Hardening options
  - Random PIDs
    - This doesn't seem to break anything, take away any functionality I use, and provides some extra protection from stuff guessing PIDs for apps.
  - Disable syslog sockets
    - I don't ship my syslog off of my laptop
  - Clean tmp on startup
    - Clean your tmp where you can.
  - Disable sendmail
    - Its a laptop, not a mail server. Might as well lighten the load
- Disable sshd
  - again its a laptop lighten the load
- Enable powerd
- Enable moused
- Enable ntpd

With that on first boot I do a `pkg bootstrap` and pull down my core packages.

## Core Package List
Boiled down I install a few packages every time that are common to every role. These are mostly particular to me but they give me a KDE5 desktop.
- xorg
- kde5
- sddm
- devcpu-data
- intel-backlight
- u2f-devd
- firefox
- chromium
- libreoffice
- git
- subversion
- portlint
- diffstat
- base64
- sudo
- neovim
- drm-kmod

After these are loaded I make a few changes to some files around the system to enable hardware and make things work to my liking.
## Files
### loader.conf
```
# Lets not wait on the boot loader
autoboot_delay=0

# I know that my root will never be on a USB disk, if it is we went sideways somewhere around albuquerque 
hw.usb.no_boot_wait="1"

# Load the webcam drivers
cuse_load="YES"

# Pull in the ACPI video power control for backlight control
acpi_video_load="YES"

# Intel has problems, we know they do, lets load their patches.
cpu_microcode_load="YES"
cpu_microcode_name="/boot/firmware/intel-ucode.bin"

# This is something I borrowed from colin percival to deal with an annoying warning that will be fixed soon and I'll test axing it later.
compat.linuxkpi.i915_disable_power_well="0"

# Load temperature sensors.
coretemp_load="YES"

# My processor should support accelerated AES no reason to not use it.
aesni_load="YES"
```
### Sysctl.conf
```
# The hardware bell is deafening, I don't frequently want it, so go away!
kern.vt.enable_bell=0
```
### pf.conf
I use PF for my workstation firewall because I know it and I have a basic rule. The rule set is, block all ingress, ignore the loopback adapter, pass all egress.
```
block in all
set skip on lo0
pass out keep state
```
## Configuration

### Services
So these are things I have enabled and the flags they need.
```
# I want my webcam
sysrc webcamd_enable="YES"

# These are parts of having my DE actually work as expected
sysrc sddm_enable="YES"
sysrc dbus_enable="YES"
sysrc hald_enable="YES"
sysrc avahi_daemon_enable="YES"
sysrc avahi_dnsconfd_enable="YES"

# I do have a scanner that sometimes works, I should kill it off one day...
sysrc saned_enable="YES"

# I want to have power control enabled. When the new power control drivers come in this wont be needed and is broken since it stops exposing frequencies but that just means you can kill this
sysrc powerd_enable="YES"
sysrc powerd_flags="-N"

# Lets use the video drivers from packages/ports and not the kernel
sysrc kldlist="/boot/modules/i915kms.ko"

# I want my wireless to be ready for me to use.
sysrc wlans_iwm0="wlan0"
sysrc ifconfig_wlan0="WPA DHCP"
```

### User Groups
My user will need to be part of the following groups to use all the devices attached.
- webcamd
- u2f
- operator
  - This allows me to use power commands like halt and reboot without sudo.
- wheel
- sound
- video
- games

## Notes
### Backlight
Need to put the devd rules in place `cp /usr/local/share/examples/intel-backlight/acpi-video-intel-backlight.conf /usr/local/etc/devd/`

The keys for it sometimes work sometimes don't there is some tweaking of a patch that has been pending review for a while that fixes this. Once it makes it in the Thinkpad function keys will be less fiddly


### Future Wifi Improvements
Probably should consider trunking it
```
sysrc ifconfig_em0="ether MY:WI:FI:MAC:ADD:RESS"
sysrc ifconfig_ue0="ether MY:WI:FI:MAC:ADD:RESS"
sysrc cloned_interfaces="lagg0"
sysrc ifconfig_lagg0="laggproto failover laggport em0 laggport ue0 laggport wlan0 DHCP"
```
### USB Audio Devices
SDDM runs kmix as part of the accessibility stack. If you have a usb audio card plugged in when SDDM starts it causes the kernel to enter an eternal loop waiting for a kmix detach that is never going to come, see [this bug](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=194727) for some nitty gritty on it. Near as I can tell from experimenting if I cause the card to detach ahead of sleep or shutdown, OR kill kmix as part of a sleep script it goes away. Probably should circle back around at some code for this because IMO the correct behavior should be the kernel says "hey guys please release your FDs" then after some period of time the kernel comes back around with an angry face and goes "Okay asking didn't work, your FDs are not released, have a nice day" But handling that correctly could be complex and it may be a case where we accept that not all FDs are created equally and some should interrupt the kernel and others the software should be told to sod off.
