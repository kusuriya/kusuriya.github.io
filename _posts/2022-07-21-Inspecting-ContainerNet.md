---
layout: default
title: Inspecting ContainerNet
Date: 2022-07-21 09:00:00 +/-0000
published: yes
tags: 
    - Docker
    - Networking
---

# Inspecting ContainerNet

For various reasons there may be a time that you have to answer the question "What the heck are my containers saying on the network?" The answer 
I found to how to answer that question was a less than satisfying "well install `tcpdump` everywhere, dump the contents to a docker volume 
inspect later". That felt less than satisfying and produced a lot of work that just didnt feel needed. I know on a traditional network 
you can dump a machine into promiscuous mode and let the packets roll in so I wondered "Why cant I do that in docker!" turns out the answer was 
you can, if you know how. So lets walk down that path.

<!-- more -->

## Basic Setup

- So for this you will need to have some sort of container software, docker, podman, whatever take your pick, you just have to be able to run containers.
- a container with network utilities and tcp dump installed
- A dark heart wanting to read the latest gossip between your containers without them knowing Karen is telling you.

The reqs are honestly simple and if you are asking the question you likely already have them all.

## Setting up the snitch

This part is actually a critical piece to this. You need something that can sit on the same "network" as the containers and capture packets. so the basic 
example that I can dump here will do that with ubuntu as the base container. Im using docker here for simplicity of things people understand but you could
do this with just about any container framework that runs linux containers.

```dockerfile
FROM ubuntu 
RUN apt-get update && apt-get install -y tcpdump net-tools
CMD ifconfig eth0 promisc && tcpdump -i eth0 $FILTERS
```

The rundown of what you have here is grab the ubuntu container, install net-tools and tcpdump to it, set eth0 to promiscuous and then start grabbing packets. That little environment variable on the end, dont worry about 'em just yet.

Once you have the dockerfile somewhere you can build it like you would any other docker image `docker build -t tcpdump:latest - < Dockerfile`

## Embedding the snitch and running it

From here its like any other container run it, pick the net you want to watch and let the packets fly.

The example I have here connects to the host network and just dumps it to the screen. I'd like to note here it is required that you have at least `--privileged`
putting the network adapter into promisc requires the container to have permissions not normally granted to it so this container has to run in an elevated mode.
```shell
docker run -it --rm --privileged --net=host tcpdump
```

Now its running, and if you are running it like me in something like rancher desktop or you have a chatty program on that network youre going to see a flood. Now even with something like wireshark this flood could be just simply too much, or youre only interested in specific things and gee wouldnt it be nice to use `tcpdump` filters in there without having to make a new container each time?! Well this is where that environment variable I mentioned earlier comes in. So if we pass it into the container on start up it will apply filters to the TCP dump stream so as an example `docker run -it --rm --privileged --net=host --env C_TCPDUMP_FILTER="\!host host.lima.internal" tcpdump` will run the container then exclude `host.lima.internal` from the stream, why, because where I am testing this is a mac that uses rancher desktop, and I dont care about the traffic of the passed through docker sockets.

Now you may be thinking also what if I need to write this to disk. Well that can be handled a few ways. One you could capture the output of the container using pipes `docker run -it --rm --privileged --net=host --env C_TCPDUMP_FILTER="\!host host.lima.internal" tcpdump > capture.pcap` you could modify the `CMD` line of the docker file to be something like `CMD ifconfig eth0 promisc && tcpdump -i eth0 -w /data/capture.pcap $FILTERS` then `docker run -it --rm --privileged --net=host -v ~/caps:/data --env C_TCPDUMP_FILTER="\!host host.lima.internal" tcpdump` and youll start getting the capture written `~/caps`

Its ultimately as flexible as you can think of so long as you are comfortable dealing with `tcpdump` directly.
