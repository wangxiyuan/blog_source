---
title: 天朝之Kubernetes安装--Kubeadm篇
date: 2018-02-12 06:04:12
tags:
  - Kubernetes 
categories: Kubernetes 
---
# 0 . 环境准备

1. Ubuntu 16.04节点两台，一个台部署master，一台部署node。如果只想all-in-one的话，一台就够，可跳过第四大步的node安装。
2. Master能访问外网。如不能访问，则需要配置好ubuntu和docker的内部源。本文不讲述配制方法。
3. 本文以Kubernetes 1.9.3为例。如果后续Kubeadm没有大的架构变化，本文依旧适用，修改对应版本号即可。
<!-- more -->

# 1. 安装docker 17.03

kubeadm要求docker的版本是17.03。使用其他版本的docker不保证能安装成功, 安装命令如下：

```shell
apt-get update

apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

add-apt-repository \
   "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable"

apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```

如果想在非root用户下使用docker，则需要修改权限，使用root用户的话可以跳过此步：

```shell
adduser {username} docker
```

安装后验证，保证版本正确，并且可执行docker命令：

```shell
docker version
```

下载docker镜像

```shell
docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.9.3 

docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.9.3

docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.9.3

# 注意，随着K8S版本的更迭，对pause和etcd的版本的要求也会发生变化。我这里的做法是：
# 1. pause，下载全部版本，目前有3.0和3.1。kubeadm1.9.3使用的是3.0。
# 2. etcd，查看/etc/kubernetes/manifests/etcd.yaml文件，找到对应的image版本号。该文件在执行kubeadm init后才会生成，请参考第三大步。
docker pull mirrorgooglecontainers/pause-amd64:3.0

docker pull mirrorgooglecontainers/etcd-amd64:3.1.11
```

修改镜像名的prefix为gcr.io

```shell
docker tag mirrorgooglecontainers/kube-apiserver-amd64:v1.9.3 gcr.io/google_containers/kube-apiserver-amd64:v1.9.3

docker tag mirrorgooglecontainers/kube-controller-manager-amd64:v1.9.3 gcr.io/google_containers/kube-controller-manager-amd64:v1.9.3

docker tag mirrorgooglecontainers/kube-scheduler-amd64:v1.9.3 gcr.io/google_containers/kube-scheduler-amd64:v1.9.3

docker tag mirrorgooglecontainers/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0

docker tag mirrorgooglecontainers/etcd-amd64:3.1.11 gcr.io/google_containers/etcd-amd64:3.1.11
```

# 2. 安装kubeadm、kubelet和kubectl

先添加对应的源，由于天朝自带防火墙的原因，无法获取官网的key，我复制了一份，请先[下载](/files/kube_apt_key.gpg)，下载后执行：

```shell
cat kube_apt_key.gpg | apt-key add -

echo "deb [arch=amd64] https://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-$(lsb_release -cs) main" >> /etc/apt/sources.list

apt-get update && apt-get install -y kubelet kubeadm kubectl
```

安装后验证，保证都是v1.9.3：

```shell
kubeadm version
kubelet --version
kubectl version
```

# 3. 使用kubeadm安装kubernetes maser节点

关闭swap。

```shell
swapoff -a
```

停止kubelet服务。

```shell
service kubelet stop
```

删除/etc/kubernetes目录。

```shell
rm -rf /etc/kubernetes
```

执行kubeadm初始化命令，这里必须明文指定要安装的K8S版本号，否则kubeadm会去官网查找版本号，然后你懂得。

```shell
kubeadm init --kubernetes-version v1.9.3
```

执行成功后，记录下屏幕出打印出的kubeadm join命令，后面会用到。

```
kubeadm join --token d7d192.8e9115ee698a1c0c 10.3.150.28:6443 --discovery-token-ca-cert-hash sha256:c81f7f8511adbbcf3985f0951510f115286008737ecbd2cb6472d6fe1591eb29
```

然后需要给当前用户赋予访问K8S的权限：

```shell
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# root用户可以不执行此步。
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

最后，验证K8S环境：

```shell
# kubectl get nodes

NAME       STATUS    ROLES     AGE       VERSION
node-6-5   Ready     master    13m       v1.9.3

# kubectl get pods --namespace kube-system

NAME                               READY     STATUS              RESTARTS   AGE
etcd-node-6-5                      1/1       Running             0          13m
kube-apiserver-node-6-5            1/1       Running             8          12m
kube-controller-manager-node-6-5   1/1       Running             0          13m
kube-dns-6f4fd4bdf-ktz6s           0/3       ContainerCreating   0          13m
kube-proxy-tg57q                   0/1       ImagePullBackOff    0          13m
kube-scheduler-node-6-5            1/1       Running             0          12m
```

可以看到kube-dns和kube-proxy两个pods没有running，这是因为所需要的image在google中，天朝无法访问。解决方法和第一步一致，手动pull docker 镜像，并打上对应标签。

```shell
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.9.3
docker tag mirrorgooglecontainers/kube-proxy-amd64:v1.9.3 gcr.io/google_containers/kube-proxy-amd64:v1.9.3

docker pull mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7
docker pull mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker pull mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7

docker tag mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7 gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
docker tag mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7 gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker tag mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7 gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
```

对于上面的dns命令，我们如何知道它版本号是1.14.7的？执行如下命令：

```shell
kubectl describe pods kube-dns-6f4fd4bdf-ktz6s --namespace kube-system
```

在返回信息中可以看到``Image``标签，其对应的值就是我们要pull的镜像。

以上步骤全部完成后，执行``kubectl get pods -n kube-system``命令，我们可以看到``kube-proxy``已经running了，但``kube-dns``还是有问题，为什么？因为kubernetes集群的网络组件还未安装。kubeadm支持CNI标准的网络，包括``Calico``、``Canal``、``Flannel``、``Kube-router``、``Romana``和``Weave Net``，不支持``kubenet``。

本文以``Calico``为例，命令如下：

```
kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```

稍等片刻，执行``kubectl get pods -n kube-system``，直到所有pods都running。至此K8S master节点安装完毕。

# 4. 安装kubernetes nodes

在node节点上先添加K8S源。

```
cat kube_apt_key.gpg | apt-key add -

echo "deb [arch=amd64] https://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-$(lsb_release -cs) main" >> /etc/apt/sources.list
```

再安装docker，并拉取镜像，此时只需要拉取kube-proxy和pause，方法如上。

还记得在第三步中，kubeadm init执行成功后，屏幕中打印的一句:

```
kubeadm join --token d7d192.8e9115ee698a1c0c 10.3.150.28:6443 --discovery-token-ca-cert-hash sha256:c81f7f8511adbbcf3985f0951510f115286008737ecbd2cb6472d6fe1591eb29
```

镜像准备完毕后，在node节点上执行如上命令即可。此时执行kubectl get nodes，会发现该node已经成功添加。

# 5. 注意事项

1. kubeadm安装失败后，再次安装前需要停止kubelet服务， 并删掉/etc/kubernetes目录
2. 在执行``kubeadm init``时，如果docker image的版本不对，则会无限卡在``[init] This might take a minute or longer if the control plane images have to be pulled.``这一步。此时请打开``/etc/kubernetes/manifests/etcd.yaml``文件核对版本号，重开一个session拉取镜像并打tag。
3. 本文用的方法很土，需要手动pull镜像并打tag，高级用法是在执行``kubeadm init``时指定配置文件，让kubeadm从配置文件中指定的镜像源自动pull镜像，比如指定aliyun或ustc等等。可以免去手动tag的大量操作。但config这种方法在kubeadm中被标记为实验性方式，意味着不稳定，使用方式可能会经常改变。因此为了保证K8S的成功安装，我这里选择了土鳖的手动方式。我会在下一篇文章中详细说明``kubeadm init --config``的详细用法。
4. TBD...如果你在安装过程中遇到了什么其他问题，请留言。
