---
layout: default
title: FreeBSD on the System76 Galago Pro
tags:
    - Hardware
    - System76
    - FreeBSD
---
# FreeBSD on the System76 Galago Pro

Hey all, It's been a while since I last posted but I thought I would hammer something out here. My most recent purchase was a [System76](https://system76.com) Galago Pro. I thought, afer playing with POP! OS a bit, is there any reason I couldn't get BSD on this thing. Turns out the answer is no, no there isnt and it works pretty decently.

<!--more-->

To get some accounting stuff out of the way I tested this all on FreeBSD Head and 11.1, and all of it is valid as of May 10, 2018. Head is a fast moving target so some of this is only bound to improve.

## The hardware
- Intel Core i5 Gen 8
- UHD Graphics 620
- 16 GB DDR4 Ram
- RTL8411B PCI Express Card Reader
- RTL8111 Gigabit ethernet controller
- Intel HD Audio
- Samsung SSD 960 PRO 512GB NVMe


## The caveats
There are a few things that I cant seem to make work straight out of the box, and that is the SD Card reader, the backlight, and the audio is a bit finicky. Also the trackpad doesn't respond to two finger scrolling.
The wiki is mostly up to date, there are a few edits that need to be made still but there is a bug where I cant register an account yet so I haven't made all the changes.

## Processor
It works like any other Intel processor. Pstates and throttling work.

## Graphics
The boot menu sets itself to what looks like 1024x768, but works as you expect in a tiny window.  
The text console does the full 3200x1800 resolution, but the text is ultra tiny. There isnt a font for the console that covers hidpi screens yet.  
As for X Windows it requres the `drm-kmod-next` package. Once installed follow the directions from the package and it works with almost no fuss. I have it running on X with full intel acceleration, but it is running at it's full 3200x1800 resolution, to scale that down just do `xrandr --output eDP-1 --scale 0.5x0.5` it will blow it up to roughly 200%. Due to limitations with X windows and hidpi it is harder to get more granular.

## Intel Wireless 8265
The wireless uses the iwm module, as of right now it does not seem to automagically load right now. Adding `iwm_load="YES"` will cause the module to load on boot and `kldload iwm`

## Battery
I seem to be getting about 5 hours out of the battery, but everything reports out of the box as expected. I could get more by throttling the CPU down speed wise.

## Overall impression
It is a pretty decent experience. While not as polished as a Thinkpad there is a lot of potential with a bit of work and polishing. The laptop itself is not bad, the keyboard is responsive. The build quality is pretty solid. My only real complaint is the trackpad is stiff to click and sort of tiny. They seem to be a bit indifferent to non linux OSes running on the gear but that isnt anything new.
I wont have any problems using it and is enough that when I work through this laptop, but I'm not sure at this stage if my next machine will be a System76 laptop, but they have impressed me enough to put them in the running when I go to look for my next portable machine but it hasn't yet replaced the hole left in my heart by lenovo messing with the thinkpad.