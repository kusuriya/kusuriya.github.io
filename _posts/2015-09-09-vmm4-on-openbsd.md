---
layout: default
title: vmm(4) on OpenBSD
tags: 
    - OpenBSD
---
# {{ page.title }}  

Thanks to work by mlarkin@ OpenBSD is that much closer to having a native hypervisor and honestly I think this is a
game changer.

<!--more-->

It is still in its rather early stages but according to various reports and showings from his
twitter it is going well. I am very excited to see this and am trying to stay tuned for this.
my hope is that it is eventually compatible with libvirt (or that libvirt gets some modules
for it) but it is a very exciting prospect to have a stripped and secure by default host especially
since OpenBSDs network layer has proven to be robust and reliable.
So far it seems that he has mad some great progress and according to his [PasteBin](http://pastebin.com/B6bs3FB4) and 
[Twitter](https://twitter.com/mlarkin2012/status/640755032875360256) he has had quite a bit of success so far and has got as far as getting one of the RAMDISK kernels to run within vmm(4).  
When it gets a bit more stable and fleshed out I will for sure be doing some write ups on how to start taking advantage of it. In the mean time I would highly suggest 
for sure donating to the [OpenBSD Foundation](http://www.openbsdfoundation.org/) to support his work and see more cool things come, maybe we can get solaris like zones and run 
vmm inside zones for added protection.

