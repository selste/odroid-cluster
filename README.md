# Odroid Kubernetes Cluster
Setting up Kubernetes on a Odroid MC1 cluster

## Hardware
Yes, there's the Raspberry Pi ... it's very popular, there's a pretty good distribution [Hypriot](https://blog.hypriot.com/) that sets You up (with Docker Swarm at least) in no time at all.

But the [Odroid MC1](http://www.hardkernel.com/main/products/prdt_info.php?g_code=G150152508314) is ever so cute, comes preassembled and the hardware is significantly better than that of the Raspberry Pi 3(+):
* Octocore CPU, 4 Cores run at 2 GHz
* 2 GByte RAM
* GBit Ethernet (no, the Rapsberry can't do /real/ GBit Ethernet ... look it up)
* Formfactor is much smaller, the size of the 4 Boards assembled, including the fan, is only 112 x 93 x 72mm
* Aluminium case to assemble the units acts as a (very substantial) heatsink - You can probably get away with only passive cooling

## Software
Hardkernel provide images based on Ubuntu running Kernel 4.14 ... which is pretty nice, unfortunately there's no headless image based on Ubuntu 18.04 at the time of this writing (June 2018).
Instead i opted for [Armbian](https://www.armbian.com/) - they provide three images:
* Armbian Jessie (Kernel 4.9)
* Armbian Stretch (Kernel 4.9)
* Armbian Bionic (Kernel 4.9)
I'm using the Armbian Stretch image right now.

### Basic Setup
You'll want to use static IP addresses ...
* Run [armbian-config](https://docs.armbian.com/User-Guide_Armbian-Config/) (or edit the file manually)
* Change the hostname(s)
* Add all the cluster nodes to /etc/hosts

Run `apt-get update && apt-get upgrade` to fetch all the updates.

As a side note: Even though it takes some time getting used to [tmux](https://github.com/tmux/tmux/wiki) makes dealing with 4 shells at the same time really easy!

### Kubernetes
I'm basically following the official documentation: [Installing kubeadm](https://kubernetes.io/docs/tasks/tools/install-kubeadm/)

#### Docker
Officially supported with the current version of Kubernetes (1.10) is Docker CE 17.03, 17.06+ _might work_ but hasn't been tested yet.
The canonical Docker repository already contains 18.03, which probably won't work - so we're going to deviate from the official guide, download the oldest available DEB package from the repository and install it manually (might be a good idea to pin it as well):
* `curl -LO https://download.docker.com/linux/debian/dists/stretch/pool/stable/armhf/docker-ce_17.06.2~ce-0~debian_armhf.deb`
* Install a necessary dependency first: `apt-get install libltdl7`
* `dpkg -i docker-ce_17.06.2~ce-0~debian_armhf.deb`
* Copy the package to the nodes: `scp docker-ce_17.06.2~ce-0~debian_armhf.deb odroidmc1-node1:/tmp odroidmc1-node2:/tmp odroidmc1-node3:/tmp` and repeat the installation on each one

