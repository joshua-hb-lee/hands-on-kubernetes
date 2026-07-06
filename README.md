# Let’s Kubernetes! 

Build a Kubernetes cluster from scratch.
> using `kubeadm`

## Specification

- kubeadm, kubectl, kubelet: v1.36

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

3. **Installing Command Tools**

[Kubernetes docs (install-kubeadm)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

```shell
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl gpg

$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"36", ...
```

4. **CRI (Container Runtime Interface)**


[Kubernetes Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- Docker
- containerd
- CRI-O

```shell
$ sudo apt install -y containerd

$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# activate systemd cgroup
$ sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

$ sudo systemctl restart containerd
```