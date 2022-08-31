---
title: KuBerryPi
description: Setup of a Kubernetes Cluster on a set of Raspberry Pi 3 machines
date: 2017-02-14
tags:
- kubernetes
- raspberry pi
- home lab
- cluster
---

I'm going to run through what is needed to get Kubernetes running on a cluster
of Raspberry Pi 3 machines. Thankfully with the addition of `kubeadm` this has
become fairly trivial. The getting started guide for [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
can give you more information about the tool itself, I'll outline the steps here
but feel free to go to that getting started guide.

There are a few blog posts out there that discuss doing this and suggest using
[hypriot](https://blog.hypriot.com/) (because docker didn't officially support
the arm architecture) as the OS to use. I'm going to do this with Raspbian Jessie
Lite (because docker now officially supports the arm architecture) which can be
downloaded here. I'm running Debian so the commands will work for the majority
of Linux systems.

---

![Raspberry Pi 3 Boxes](/images/pi-boxes.jpeg "Raspberry Pi 3 Boxes")

You can setup a single node or multi-node cluster of Pi. The Pi 3 is recommended
because it comes with 1GB of RAM which leaves you more RAM to run your own systems
on the cluster than the 512MB version.

To get started you'll need to have an SD Card (I've used 64GB cards without issue)
and load the Raspbian Jessie Lite image onto the card. You can do this using the
`dd` command. First let's make sure we're using the right device on the system as
we don't want to overwrite our OS drive.

```bash
$ df -h # This command is showing mounted volumes, you should see your sdcard listed here.
Filesystem                    Size  Used Avail Use% Mounted on
/dev/sda1                     8.2G  7.0G  801M  90% /
udev                           10M     0   10M   0% /dev
tmpfs                         3.2G  9.2M  3.2G   1% /run
tmpfs                         7.9G  507M  7.4G   7% /dev/shm
tmpfs                         5.0M  4.0K  5.0M   1% /run/lock
tmpfs                         7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/sda2                     237M   42M  183M  19% /boot
tmpfs                         1.6G   56K  1.6G   1% /run/user/1000
/dev/sdb                       64G     0   64G   0% /mnt/nalum/sd
```

In this we can see that there are 3 mounted partitions for me, the drive I'm
looking for is `/dev/sdb`. For you this could be something like `/dev/mmcblk0`.
If you see `sdb1` rather than sdb you'll want to remove the digit or in the case
of `mmcblk0p1` you'll want to remove the `p1`. We can double check that this is
correct by running the command `ls -lah /dev/mmcblk0*` this will result in something
like the following output:

```bash
$ ls -lah /dev/mmcblk0*
brw-rw---- 1 root disk 8, 0 Dec 15 09:43 /dev/mmcblk0
brw-rw---- 1 root disk 8, 1 Dec 15 09:43 /dev/mmcblk0p1
```

As you can see we have `mmcblk0` and `mmcblk0p1`, `mmcblk0` is the whole device
and `mmcblk0p1` is a partition on that device, there can be multiple partitions
and there will be two after the Raspbian image is applied to the SDCard. We apply
the Raspbian image to the SD card by using the following command:

> **NOTE**
>
> This command will replace the contents of a drive (HDD, SSD, SDCard, etc), be
> absolutely certain that you are running it against the correct drive.

```
$ sudo dd bs=4M if=/path/to/raspbian.image of=/dev/sdb
$ sync
```

The Raspberry Pi site notes that if `bs=4M` doesn't work change it to `bs=1M` and
that it will take longer with that setting. Running `sync` will make sure that
it is safe to unmount the SD Card. Check here for full installation instructions
for Linux, Mac OS and Windows.

The next step is to boot the Raspberry Pi using the SD Cards you've setup. You
will need to plug a monitor and keyboard directly into at least one Pi as SSH has
been disabled by default on the Raspbian since November 2016.

The following set of commands need to be run on each of the Raspbian installs.
They will install any available updates, vim, docker and kubernetes and make
changes to some files enabling ssh and setting a cgroup config for kubernetes/docker.

```bash
$ sudo su
$ apt-get update
$ apt-get upgrade -y
$ apt-get install -y vim
$ vim /etc/hostname
## Replace contents with kubenode[0-9]{2,}
```

So if you run `cat /etc/hostname` you should see something like the following: `kubenode00`

```bash
$ touch /boot/ssh
$ vim /boot/cmdline.txt
## Add `cgroup_enable=cpuset` before `elevator=deadline`
```

If you run `cat /boot/cmdline.txt` you should see something like the following:

```text
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 cgroup_enable=cpuset elevator=deadline fsck.repair=yes rootwait quiet init=/usr/lib/raspi-config/init_resize.sh
```

Now we will install docker and kubernetes and reboot:

> **NOTE**
>
> The first `curl` command can be very dangerous, 1) because we ran `sudo su` at
> the beginning of this and 2) because you don't know what is in the file you
> piped into `sh`. So I very much recommend that you download the file and have
> a look at what it is doing to make sure you are happy running it.

```bash
$ curl -sSL https://get.docker.com | sh
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
$ apt-get update
$ apt-get install -y kubelet kubeadm kubectl kubernetes-cni
$ reboot now
```

The following set of commands should be run on one Raspbian install which will
setup the master Kubernetes node:

```bash
$ sudo su
$ kubeadm init --pod-network-cidr=10.244.0.0/16 --use-kubernetes-version=v1.5.2
$ curl -sSL "https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml?raw=true" | sed "s/amd64/arm/g" | kubectl create -f -
```

The `kubeadm init` command will produce output like the following:

```log
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[init] Using Kubernetes version: v1.5.1
[tokens] Generated token: <token>
[certificates] Generated Certificate Authority key and certificate.
[certificates] Generated API Server key and certificate
[certificates] Generated Service Account signing keys
[certificates] Created keys and certificates in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 61.317580 seconds
[apiclient] Waiting for at least one node to register and become ready
[apiclient] First node is ready after 6.556101 seconds
[apiclient] Creating a test deployment
[apiclient] Test deployment succeeded
[token-discovery] Created the kube-discovery deployment, waiting for it to become ready
[token-discovery] kube-discovery is ready after 6.020980 seconds
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node:

kubeadm join --token=<token> <master-ip>
```

The `curl` command run will setup flannel on the cluster which is required to
allow the pods to talk to each other. There are other options that you can use
instead of flannel if you want.

You need to note the `kubeadm join` command at the bottom of the output as you
will need it for the other nodes. The last step in setup of the cluster is to
add the other nodes which you can do by running the noted kubeadm join command
on each of the other Raspberry Pi:

```bash
$ sudo su
$ kubeadm join --token=a123c6.d61342c6501bf2c8 192.168.168.210
```

With the above command run on each of the nodes you should start to see them show
in the output of `kubectl get nodes`, you can run this on the master node and you
should see something like the following:

```bash
$ kubectl get nodes
NAME         STATUS         AGE
kubemaster   Ready,master   2m
kubenode01   Ready          2m
kubenode02   Ready          2m
```

To be able to control the cluster from any machine, not just the master, copy the
file `/etc/kubernetes/admin.conf` from the master node to any other machine that
you want to be able to access the cluster from. This can be combined with your
existing `kubectl` config or you can have `kubectl` look at the file separate to
your config by using the flag `--kubeconfig`.

![3 Pi](/images/3-pi.jpeg "3 Pi")

So with that done you now have a Kubernetes cluster running on your Raspberry Pi.
I haven't really done anything more with my cluster since setting it up but feel
free to ask questions and I'll do my best to answer. I think the next thing I'll
be doing is setting up CephFS or NFS to act as persistent storage for the cluster.

