# Notes

Unstructured notes made during various attempts to get Kubernetes running on the Odroid MC1 cluster.

# Armbian
I'm staring with the Armbian Stretch image - for information on the available options see the ![README.md](https://github.com/selste/odroid-cluster/blob/master/README.md).

## Basic Setup
You'll want to use static IP addresses ...
* Run [armbian-config](https://docs.armbian.com/User-Guide_Armbian-Config/) (or edit the file manually)
* Change the hostname(s)
* Add all the cluster nodes to /etc/hosts
* Turn swap off: `swapoff -a`, then disable the entry in /etc/fstab

Run `apt-get update && apt-get upgrade` to fetch all the updates.

As a side note: Even though it takes some time getting used to [tmux](https://github.com/tmux/tmux/wiki) makes dealing with 4 shells at the same time really easy:
![Cluster shells](https://github.com/selste/odroid-cluster/blob/master/images/odroid-tmux.png)

## Kubernetes
I'm basically following the official documentation: [Installing kubeadm](https://kubernetes.io/docs/tasks/tools/install-kubeadm/)

### Docker
Officially supported with the current version of Kubernetes (1.10.3) is Docker CE 17.03, 17.06+ _might work_ but hasn't been tested yet.
The canonical Docker repository already contains 18.03, which probably won't work - so we're going to deviate from the official guide, download the oldest available DEB package from the repository and install it manually (might be a good idea to pin it as well):
* `curl -LO https://download.docker.com/linux/debian/dists/stretch/pool/stable/armhf/docker-ce_17.06.2~ce-0~debian_armhf.deb`
* Install a necessary dependency first: `apt-get install libltdl7`
* `dpkg -i docker-ce_17.06.2~ce-0~debian_armhf.deb`
* Copy the package to the nodes: `scp docker-ce_17.06.2~ce-0~debian_armhf.deb odroidmc1-node1:/tmp odroidmc1-node2:/tmp odroidmc1-node3:/tmp` and repeat the installation on each one

### cri-tools
Kubernetes 1.10.3 requires the [CLI and validation tools for Kubelet Container Runtime Interface (CRI)](https://github.com/kubernetes-incubator/cri-tools).
It's possible to build the binary from scratch, but there's an easier way - simply download the [appropriate archive](https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.0.0-beta.1/crictl-v1.0.0-beta.1-linux-arm.tar.gz):
* `curl -LO https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.0.0-beta.1/crictl-v1.0.0-beta.1-linux-arm.tar.gz`
* `tar -xzf crictl-v1.0.0-beta.1-linux-arm.tar.gz`, `chown root:root crictl`and `cp crictl /usr/local/bin`
* Copy the binary to all nodes: `scp crictl odroidmc1-node1:/tmp odroidmc1-node2:/tmp odroidmc1-node3:/tmp`

### kubeadm, kubelet and kubectl
Finally back to the installation guide (_Installing kubeadm, kubelet and kubectl_).

There's still one catch - the cgroup driver used by kubelet is not the same as the one used by Docker. As a matter of fact, there is no configuration entry at all present in the configuration file.
I followed [this](https://stackoverflow.com/questions/45708175/kubelet-failed-with-kubelet-cgroup-driver-cgroupfs-is-different-from-docker-c) thread on stackoverflow and added another line to _/etc/systemd/system/kubelet.service.d/10-kubeadm.conf_:
`Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"`

## Kubernetes Setup
Running `kubeadm init` on the master node is disappointing:
```
root@odroidmc1-master:~# kubeadm init
[init] Using Kubernetes version: v1.10.3
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 17.06.2-ce. Max validated version: 17.03
[preflight] Some fatal errors occurred:
        [ERROR KubeletVersion]: couldn't get kubelet version: exit status 2
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```

Did a simple test, ran `kubelet --help` and ...
```
root@odroidmc1-master:~# kubelet --help
unexpected fault address 0x15d71470
fatal error: fault
[signal SIGSEGV: segmentation violation code=0x2 addr=0x15d71470 pc=0x15d71470]

goroutine 1 [running, locked to thread]:
runtime.throw(0x2a84a9e, 0x5)
	/usr/local/go/src/runtime/panic.go:605 +0x70 fp=0x16533e98 sp=0x16533e8c pc=0x3efa4
runtime.sigpanic()
	/usr/local/go/src/runtime/signal_unix.go:374 +0x1cc fp=0x16533ebc sp=0x16533e98 pc=0x5517c
k8s.io/kubernetes/vendor/github.com/appc/spec/schema/types.SemVer.Empty(...)
	/workspace/anago-v1.10.3-beta.0.74+2bba0127d85d5a/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/appc/spec/schema/types/semver.go:68
k8s.io/kubernetes/vendor/github.com/appc/spec/schema/types.NewSemVer(0x1618a560, 0x20945b4, 0x2a8fbcf, 0xb, 0x15f91170)
	/workspace/anago-v1.10.3-beta.0.74+2bba0127d85d5a/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/appc/spec/schema/types/semver.go:41 +0x90 fp=0x16533f58 sp=0x16533ec0 pc=0x206c8d8

goroutine 5 [chan receive]:
k8s.io/kubernetes/vendor/github.com/golang/glog.(*loggingT).flushDaemon(0x4551f48)
	/workspace/anago-v1.10.3-beta.0.74+2bba0127d85d5a/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go:879 +0x70
created by k8s.io/kubernetes/vendor/github.com/golang/glog.init.0
	/workspace/anago-v1.10.3-beta.0.74+2bba0127d85d5a/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go:410 +0x1a0

goroutine 25 [syscall]:
os/signal.signal_recv(0x2bd146c)
	/usr/local/go/src/runtime/sigqueue.go:131 +0x134
os/signal.loop()
	/usr/local/go/src/os/signal/signal_unix.go:22 +0x14
created by os/signal.init.0
	/usr/local/go/src/os/signal/signal_unix.go:28 +0x30
```
Well, that is **really** disappointing!

# Ubuntu
Starting all over again, this time using the Hardkernel Ubuntu image.

Because there is as of now no official Kubernetes release available for the current Ubuntu LTS version (Bionic Beaver) i'm sticking with the older one.

## Basic Setup
Follow the instructions provided by Hardkernel to update the system and the kernel.

You'll want to use static IP addresses ...
* Edit /etc/network/interfaces
* Edit /etc/hostname
* Edit /etc/hosts and add all the cluster nodes

## Kubernetes

### Docker
Using the official Docker repository this time, i'm simply following [Get Docker CE for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/) to install _17.03.2\~ce-0\~ubuntu-xenial_: `apt-get install docker-ce=17.03.2~ce-0~ubuntu-xenial`

Successfully verified that Docker is installed correctly:

```
root@odroidmc1-master:~# docker run hello-world

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

Might be a good idea to pin this package.

### kubeadm, kubelet and kubectl
Switching to the Kubernetes installation guide.

Again, following [this](https://stackoverflow.com/questions/45708175/kubelet-failed-with-kubelet-cgroup-driver-cgroupfs-is-different-from-docker-c) thread on stackoverflow to add another line to _/etc/systemd/system/kubelet.service.d/10-kubeadm.conf_:
`Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"`

Restarting _kubelet_ afterwards
```
systemctl daemon-reload
systemctl restart kubelet
```
does not lead to any error messages (at least on the console).

## Kubernetes Setup
Trying _kubeadm init_ on the master node ... the output is not encouraging:
```
root@odroidmc1-master:~# kubeadm init
[init] Using Kubernetes version: v1.10.3
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[preflight] Some fatal errors occurred:
	[ERROR KubeletVersion]: couldn't get kubelet version: exit status 2
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```

Basically the same error again!

Hmmm ... the issue [kubelet - 1.10.3-00 - segmentation violation](https://github.com/kubernetes/kubernetes/issues/64234) seems to describe the problem.
All right, i'll downgrade to 1.10.2, by doing `apt install kubelet=1.10.2-00`.

Initialize Kubernetes, this time using `kubeadm init --pod-network-cidr=10.244.0.0/16` (because we'll be using _Flannel_ as pod network):
```
root@odroidmc1-master:~# kubeadm init --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.10.4
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [odroidmc1-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.178.110]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [odroidmc1-master] and IPs [192.168.178.110]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 111.007031 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node odroidmc1-master as master by adding a label and a taint
[markmaster] Master odroidmc1-master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: b4jj31.i9y3tdewbrd6l606
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.178.110:6443 --token [secret token] --discovery-token-ca-cert-hash sha256:[secret hash]
```

Yay!!!

With hindsight ... Armbian would have worked as well after downgrading kubelet, i suppose.

Probably needs pinning as well, to prevent nasty surprises.

### cri-tools
To get rid of the warning regarding _cri-tools_:
* get the archive (`curl -LO https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.0.0-beta.1/crictl-v1.0.0-beta.1-linux-arm.tar.gz`)
* extract (`tar -xzf crictl-v1.0.0-beta.1-linux-arm.tar.gz`)
* change ownership (`chown root:root crictl`) and move (`mv crictl /usr/local/bin`)

the binary.

### Pinning
Opted for a file - in _etc/apt/preferences.d/_ - that contains entries for the (currently) two packages that need to be pinned:
```
Package: docker-ce
Pin: version 17.03.2~ce-0~ubuntu-xenial
Pin-Priority: 1000

Package: kubelet
Pin: version 1.10.2-00
Pin-Priority: 1000
```

### Node(s)
ToDo - short description of the installation steps for each node

### Installing pod network
As stated in the documentation
* download the manifest manually: `curl -LO https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml`
* replace the occurences of _amd64_ with _arm_

Running _kubectl apply -f kube-flannel.yml_:
```
root@odroidmc1-master:~# kubectl apply -f /etc/kubernetes/kube-flannel.yml 
clusterrole.rbac.authorization.k8s.io "flannel" created
clusterrolebinding.rbac.authorization.k8s.io "flannel" created
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset.extensions "kube-flannel-ds" created
```
looks pretty good.

Check that is is indeed working:
```
root@odroidmc1-master:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY     STATUS     RESTARTS   AGE
kube-system   etcd-odroidmc1-master                      0/1       Pending    0          3s
kube-system   kube-apiserver-odroidmc1-master            0/1       Pending    0          3s
kube-system   kube-controller-manager-odroidmc1-master   0/1       Pending    0          3s
kube-system   kube-dns-686d6fb9c-76258                   0/3       Pending    0          9m
kube-system   kube-flannel-ds-9ccnj                      0/1       Init:0/1   0          21s
kube-system   kube-proxy-wzrnc                           1/1       NodeLost   0          9m
kube-system   kube-scheduler-odroidmc1-master            0/1       Pending    0          3s
```

### Adding node(s)
According to the manual it should now be possible to join a node to the master ... except it doesn't:
```
root@odroidmc1-node1:~# kubeadm join 192.168.178.110:6443 --token [secret token] --discovery-token-ca-cert-hash sha256:[secret hash]
[preflight] Running pre-flight checks.
[preflight] Some fatal errors occurred:
	[ERROR CRI]: unable to check if the container runtime at "/var/run/dockershim.sock" is running: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with \`--ignore-preflight-errors=...\`
```

Oh well, this issue [Installing crictl requires use of --ignore-preflight-errors=cri](https://github.com/kubernetes/kubeadm/issues/657) maybe?!

Let's try and see:
```
root@odroidmc1-node1:~# kubeadm join 192.168.178.110:6443 --token [secret token] --discovery-token-ca-cert-hash sha256:[secret hash] --ignore-preflight-errors=cri
[preflight] Running pre-flight checks.
	[WARNING CRI]: unable to check if the container runtime at "/var/run/dockershim.sock" is running: exit status 1
[discovery] Trying to connect to API Server "192.168.178.110:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.178.110:6443"
[discovery] Requesting info from "https://192.168.178.110:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.178.110:6443"
[discovery] Successfully established connection with API Server "192.168.178.110:6443"

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

'export KUBECONFIG=/etc/kubernetes/admin.conf'

Master seems happy:
```
root@odroidmc1-master:~# kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
odroidmc1-master   Ready     master    26m       v1.10.2
odroidmc1-node1    Ready     <none>    1m        v1.10.2
root@odroidmc1-master:~# 
```

