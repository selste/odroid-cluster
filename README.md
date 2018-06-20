# Odroid Kubernetes Cluster
Setting up Kubernetes on a Odroid MC1 cluster

## Hardware
Yes, there's the Raspberry Pi ... it's very popular, there's a pretty good distribution - [Hypriot](https://blog.hypriot.com/) - that sets You up (with Docker Swarm at least) in no time at all.

But the [Odroid-MC1](http://www.hardkernel.com/main/products/prdt_info.php?g_code=G150152508314) is ever so cute, comes preassembled and the hardware is significantly better than that of the Raspberry Pi 3(+):
* Octocore CPU, 4 Cores run at 2 GHz
* 2 GByte RAM
* GBit Ethernet (no, the Raspberry can't do /real/ GBit Ethernet ... look it up)
* Formfactor is much smaller, the size of the 4 Boards assembled, including the fan, is only 112 x 93 x 72mm
* Aluminium case to assemble the units acts as a (very substantial) heatsink - You can probably get away with only passive cooling

![Odroid-MC1](https://github.com/selste/odroid-cluster/blob/master/images/odroid-mc1.png)

## Software
Hardkernel provide images based on Ubuntu running Kernel 4.14 ... which is pretty nice.
Still, i opted for [Armbian](https://www.armbian.com/) - they provide a number of images:
* Armbian Jessie (Kernel 4.9)
* Armbian Stretch (Kernel 4.9)
* Armbian Bionic (Kernel 4.9)

These images are considered 'Stable', additionally two 'Testing' images are available:
* Armbian Stretch (Kernel 4.14)
* Armbian Bionic (Kernel 4.14)


