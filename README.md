
# 14 Easy Steps to Install Kubernetes in an Offline Server

The Kubernetes installation in a server involves the following steps:-
1. Letting iptables see bridged traffic
2. Installing runtime
3. Installing kubeadm, kubelet and kubectl
4. Configuring a cgroup driver

During these steps, the server needs to have internet connection as it needs to download
all the required libraries.

This document will guide you with 14 easy steps to setup Docker and Kubernetes in a server with
no internet access.
## Installation

Step 1 - Disable Swap in both machines, if this is a single node installation

```bash
    swapoff -a; sed -i '/swap/d' /etc/fstab
```

Step 2 - Update sysctl settings for Kubernetes networking in both machines

```bash
    cat >>/etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
```

Step 3 - Install Docker and get docker packages in Online Machine

You will notice the apt install command with the --download-only switch. This
will result in the download of the package files in /var/cache/apt/archives location

```bash
    apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common --download-only
    apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    apt install -y docker-ce=5:19.03.10~3-0~ubuntu-bionic containerd.io --download-only
    apt install -y docker-ce=5:19.03.10~3-0~ubuntu-bionic containerd.io
```

Step 4 - Install Kubeadm, Kubectl, Kubelet and get their packages in Online Machine

```bash
    apt update
    apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00 --download-only
    apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00
```

Step 5 - Copy the packages from /var/cache/apt/archives of Online machine and move to same location in Offline machine

Step 6 - In your Offline Server, Run commands to install Docker, KubeAdm, Kubelet, Kubectl (the packages will be picked from local)

```bash
    apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
    apt install -y docker-ce=5:19.03.10~3-0~ubuntu-bionic containerd.io
    apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00
```
    
Step 7 - Get list of container images in Online Server

```bash
    kubeadm config images list 
```

Step 8 - Pull container images with docker command in Online Server

```bash
    for image in k8s.gcr.io/kube-apiserver:v1.18.5 \
        k8s.gcr.io/kube-controller-manager:v1.18.5 \
        k8s.gcr.io/kube-scheduler:v1.18.5 \
        k8s.gcr.io/kube-proxy:v1.18.5 \
        k8s.gcr.io/pause:3.2 \
        k8s.gcr.io/etcd:3.4.3-0 \
        k8s.gcr.io/coredns:1.6.7; do
        sudo docker pull $image;
    done
```

Step 9 - Save images as .tar files

```bash
    mkdir ~/k8s-images
    docker save k8s.gcr.io/kube-apiserver:v1.18.5 > ~/k8s-images/kube-apiserver.tar
    docker save k8s.gcr.io/kube-controller-manager:v1.18.5 > ~/k8s-images/kube-controller-manager.tar
    docker save k8s.gcr.io/kube-scheduler:v1.18.5 > ~/k8s-images/kube-scheduler.tar
    docker save k8s.gcr.io/kube-proxy:v1.18.5 > ~/k8s-images/kube-proxy.tar
    docker save k8s.gcr.io/pause:3.2 > ~/k8s-images/pause.tar
    docker save k8s.gcr.io/etcd:3.4.3-0 > ~/k8s-images/etcd.tar
    docker save k8s.gcr.io/coredns:1.6.7 > ~/k8s-images/coredns.tar
```

Step 10 - Import tar image files into Docker of the Offline Server

```bash
    cd k8s-images/
    ls * | while read image; do sudo docker load < $image; done
```

Step 11 - Start Kubernetes and check status

```bash
    kubeadm init --ignore-preflight-errors=all
    kubectl cluster-info
```

Step 12 - Get Calico Network related images. You can use any other network plugin

```bash
    docker pull calico/cni:v3.14.2
    docker pull calico/pod2daemon-flexvol:v3.14.2
    docker pull calico/node:v3.14.2
    mkdir calico-images
    cd calico-images
    docker pull calico/kube-controllers:v3.14.2
    docker save calico/cni:v3.14.2 > calico_cni.tar
    docker save calico/pod2daemon-flexvol:v3.14.2 > calico_pod2daemon.tar
    docker save calico/node:v3.14.2 > calico_node.tar
    docker save ull calico/kube-controllers:v3.14.2 > calico_kube-controllers.tar
```

Step 13 - Get Calico Network related yaml

```bash
    wget https://docs.projectcalico.org/v3.14/manifests/calico.yaml > calico.yaml
```

Step 14 - Calico Network installation in Offline server

```bash
    cd calico-images/
    ls * | while read image; do sudo docker load < $image; done
    kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f calico.yaml
```
