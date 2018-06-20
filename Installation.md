# Armbian
I'm following the official documentation: [Getting Started](https://docs.armbian.com/User-Guide_Getting-Started/)

* run `apt update && apt upgrade`
* switch to static IP address afterwards - can be done manually or with the help of `armbian-config`
* change hostname and add cluster members to '/etc/hosts'
* turn swap off - `swapoff -a` - and permanently disable the entry in '/etc/fstab'

# Docker
Official documentation for Docker is [here](https://docs.docker.com/install/linux/docker-ce/debian/)
Kubernetes recommends using Docker 17.03, unfortunately 17.06 is the oldest available one for Debian Stretch ... so i'm using that:
`apt install docker-ce=17.06.2~ce-0~debian`

Pin this version to make sure it won't get updated!

Verification that Docker is up and running:
```
root@odroidmc1-master:~# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
61ddb93a5f93: Pull complete 
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm32v5)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

