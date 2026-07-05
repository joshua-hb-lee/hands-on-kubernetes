# Let’s Kubernetes! 

Build a Kubernetes cluster from scratch.
> using `kubeadm`

## Specification

- kubeadm
- kubernetes

## Prerequisite

Server Instance X 3
(Ubuntu 24.04, RAM>=8Gi)

## Pre-Configuration

1. **swap off**

```shell
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^/#/' /etc/fstab

free -m
        total     used     free   shared  buff/cache   available
Mem:    .....     ....      ...       ..        ....        ....
Swap:       0        0        0
```

2. **cgroup** 

- check the version of cgroup on an instance.
```shell
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

> Kubernetes recommends cgroup v2

```shell
$ sudo vim /etc/default/grub
...
GRUB_CMDLINE_LINUX="cgroup_memory=1 cgroup_enable=memory swapaccount=1"
...

$ sudo update-grub
```
> The cgroup configuration may vary depending on the server environment.
