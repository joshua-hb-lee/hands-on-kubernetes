# Let’s Kubernetes! 

Build a bare-metal Kubernetes cluster from scratch.
> using `kubeadm` (bare-metal)

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

5. **Other Configurations**

- Activate ip forwarding on each node.
```shell
$ vim /etc/sysctl.conf
net.ipv4.ip_forward=1 # activate

$ sudo sysctl -p
```

- Set up the private IP address as this Kubernetes node’s IP.
```shell
$ vim /etc/default/kubelet
KUBELET_EXTRA_ARGS=--node-ip=[PRIVATE_IP]
```

## Creating a Kubernetes Cluster

### 1. Initialize Control Plane (only control plane)

- initialize new Kubernetes cluster using `kubeadm`.
```shell
$ sudo kubeadm init \
--apiserver-advertise-address=[NODE_PRIV_IP] \
--pod-network-cidr=10.244.0.0/16

...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join [CONTROL_PRIV_IP]:6443 --token [token] \
	--discovery-token-ca-cert-hash sha256:...
```
- set up kube environment.
```shell
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- check nodes
```shell
$ kubectl get nodes
NAME             STATUS   ROLES           AGE   VERSION
ubuntu-alpha-a   Ready    control-plane   76s   v1.36.2
```

### 2. Join the Cluster on Each Worker Node.
- join the kubernetes cluster on each worker node.
```shell
$ kubeadm join [CONTROL_PRIV_IP]:6443 --token [token] \
	--discovery-token-ca-cert-hash sha256:...
```

- check if worker nodes joined or not.
```shell
$ kubectl get nodes -o wide
NAME             STATUS   ROLES           AGE     VERSION
ubuntu-alpha-a   Ready    control-plane   5m43s   v1.36.2
ubuntu-alpha-b   Ready    <none>          28s     v1.36.2
ubuntu-alpha-c   Ready    <none>          6s      v1.36.2
```

## Container Network Interface (CNI) Installation

> Calico v3.32.1

- [Calico Installation Document](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [CNCF Landscape - Cloud Native Network (Calico)](https://landscape.cncf.io/?item=runtime--cloud-native-network--project-calico)

```shell
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

$ curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
```

- update CIDR in ip pools (Pod CIDR: 10.244.0.0/16)
```shell
$ vim custom-resources.yaml
```
[calico/custom-resources.yaml](https://github.com/joshua-hb-lee/hands-on-kubernetes/calico/custom-resources.yaml)
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 10.244.0.0/24 ##### !!IMPORTANT!!
        # ...
```

- apply calico resources
```shell
$ kubectl create -f custom-resources.yaml
$ kubectl -n calico-system get pods
```

## Application Networking

- Ingress Nginx Controller (Deprecated)
- **Gateway API**

### Ingress Nginx Controller (Deprecated)

```shell
# use YAML manifest
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml

# use helm chart
$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

```shell
$ kubectl -n ingress-nginx get pods
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7c444fc6cf-spr87   1/1     Running   0          34s

$ kubectl -n ingress-nginx get svc
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.103.69.117   <pending>     80:32078/TCP,443:31970/TCP   86s
ingress-nginx-controller-admission   ClusterIP      10.101.12.159   <none>        443/TCP                      86s
```
