---
title: kubeadm概述
date: 2018-02-13 08:55:30
tags:
  - Kubernetes
categories: Kubernetes
---
Kubeadm是Kubernetes社区提供的一键快速部署Kubernetes的工具。在天朝之外排除了防火墙的干扰的地方，用户确实只需要执行``kubeadm init``即可一键安装kuberentes。那么kubeadm工具具体做了那些事？其部署Kubernetes的原理是什么？本文涉及到一些Kubernetes的概念（如静态Pod、ConfigMap、Secret、Service、DaemonSet等等），需要读者先行找相关的K8S入门文章了解，本文不再赘述。
<!-- more -->

# 1. init

## 1. 流程概述

``kubeadm init``主要进行了如下操作：

1. pre check。首先会检查当前环境是否满足安装Kubernetes的要求，比如前文提到swap检查等等。
2. 生成自签名的证书和公私钥。用作ssl加密通信。保存在``/etc/kubernetes/pki/``中。
3. 根据上一步的证书，生成组件间交互以及访问API时需要的鉴权配置文件，包括``controller-manager.conf``、``kubelet.conf``、``scheduler.conf``以及``admin.conf``。保存在``/etc/kubernetes/``中。
4. ``apiserver``、``controller-manager``和``scheduler``这三个服务是``kubelet``进程在初始化过程中，以静态Pod的方式拉起的，因此这里要先生成对应的Pod yaml文件。如果没有可访问的etcd服务时，还会自动生成``etcd``服务的yaml文件。保存在``/etc/kubernetes/manifests``中。
5. 第四步之后，kubernetes的主要进程已经运行好了，下面是一些配置工作。先给该节点打上``master``对应的标签，标明该节点是master节点。
6. 生成一个令牌，新node节点使用该令牌（``kubeadm join``命令）加入到该kubernetes集群中。为了保证新node可以join，还要在master节点上做一些配置，如创建对应的ConfigMap等等。
7. 创建``kube-dns``和``kube-proxy``两个动态Pod。当然在部署对应的CNI之前，DNS Pod是起不起来的。

可以看到kubeadm的精髓在于``使用Kubernetes部署Kubernetes``。``Kubelet``不依赖其他Kubernetes组件就能创建Pod，从而使得apiserver等K8S服务容器化，以容器的方式提供容器编排服务，使人眼前一亮。

## 2. 参数

使用kubeadm init提供的参数可以定制化部署K8S集群，这些参数说明如下：

| 参数 <类型>                                  | 功能说明                                     |
| :--------------------------------------- | ---------------------------------------- |
| --apiserver-advertise-address <string >  | API服务的IP地址，默认是0.0.0.0                    |
| --apiserver-bind-port <int32>            | API服务的端口号，默认是6443                        |
| --apiserver-cert-extra-sans <stringSlice> | 证书需要包含的额外内容。[sans](https://en.wikipedia.org/wiki/Subject_Alternative_Name)与X.509证书格式相关。 |
| --cert-dir <string>                      | 证书存储路径，默认是``/etc/kubernetes/pki``        |
| --config <string>                        | 读取配置文件，指定更多的kubeadm参数。下一节讲详细说明该参数用法。     |
| --cri-socket <string>                    | 容器运行时的socket，默认是``/var/run/dockershim.sock`` |
| --dry-run                                | 只打印出``kubeadm init``的流程，但不执行任何改变。        |
| --feature-gates <string>                 | 一组键值对，使能对应的功能与否。                         |
| --ignore-preflight-errors <stringSlice>  | 在pre check时，忽略对应的errror，改为只打warning。     |
| --kubernetes-version <string>            | 需要安装的K8S版本，默认是stable-1.9                 |
| --node-name <string>                     | 节点的名字                                    |
| --pod-network-cidr <string>              | Pod对应虚拟网络的CIDR段。                         |
| --service-cidr <string>                  | Pod对应的对外暴露的服务的CIDR段。默认是``10.96.0.0/12``  |
| --service-dns-domain <string>            | 服务的dns域名。默认是``cluster.local``            |
| --skip-token-print                       | ``kubeadm init``过程中，不打印令牌。               |
| --token <string>                         | 指定master和node之间的令牌，不再自动生成。               |
| --token-ttl <duration>                   | 自动生成的token的过期时间，为0表示永不过期。默认是``24h0m0s``。 |

这里需要特别说下``--feature-gates``这个参数，支持的功能有：

```shell
# 使用CoreDns，不再使用默认的kube-dns。
CoreDNS=true|false (ALPHA - default=false)
# 使用动态的ConfigMap来管理配置项。
DynamicKubeletConfig=true|false (ALPHA - default=false)
# kubelet拉起的静态Pod将转为DaemonSet的Pod。
SelfHosting=true|false (ALPHA - default=false)
# 使用Secret存储证书，而不是本地存储。
StoreCertsInSecrets=true|false (ALPHA - default=false)
```

默认都是关闭状态，比如若想使用CoreDns，则执行命令：

```
kubeadm init --feature-gates=CoreDNS=true
```

## 3. --config 用法

Kubeadm默认从google官网拉取docker镜像，那么遇到天朝这种访问不了谷歌的情况，甚至只能访问内部docker源的情况时，该怎么办？[前文](http://www.wangxiyuan.top/2018/02/12/kubernetes-install/)使用了手动拉镜像再打tag的土鳖方法。现在可以使用``kubeadm init --config``的方式手动指定镜像源。当然``--config``包含了更多功能。一般一个config文件的内容与格式如下：

```yaml
# 可以看到该方式还是alpha版本。说明其稳定性或功能并不完善，后续改动的可能非常大。
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  # 对应到参数--apiserver-advertise-address和--apiserver-bind-port，以下很多地方也对应到相应的参数，
  # 不再细说。
  advertiseAddress: <address|string>
  bindPort: <int>
etcd:
  endpoints:
  - <endpoint1|string>
  - <endpoint2|string>
  caFile: <path|string>
  certFile: <path|string>
  keyFile: <path|string>
  dataDir: <path|string>
  extraArgs:
    <argument>: <value|string>
    <argument>: <value|string>
  # etcd 镜像的地址，可以专门指定。
  image: <string>
kubeProxy:
  config:
    mode: <value|string>
networking:
  dnsDomain: <string>
  serviceSubnet: <cidr>
  podSubnet: <cidr>
kubernetesVersion: <string>
cloudProvider: <string>
nodeName: <string>
authorizationModes:
- <authorizationMode1|string>
- <authorizationMode2|string>
token: <string>
tokenTTL: <time duration>
selfHosted: <bool>
# 这里可以增加apiserver、controller-manager、scheduler服务启动时的额外参数。因此想要用好config，是需
# 要对K8S有一定掌握的。
apiServerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
controllerManagerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
schedulerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
# 因为apisever等服务是以Pod的方式运行的，因此可以给它们挂载卷。
apiServerExtraVolumes:
- name: <value|string>
  hostPath: <value|string>
  mountPath: <value|string>
controllerManagerExtraVolumes:
- name: <value|string>
  hostPath: <value|string>
  mountPath: <value|string>
schedulerExtraVolumes:
- name: <value|string>
  hostPath: <value|string>
  mountPath: <value|string>
apiServerCertSANs:
- <name1|string>
- <name2|string>
certificatesDir: <string>
# 镜像源，这里可以配置成本地源，或aliyun、ustc等镜像源。
imageRepository: <string>
# 有时为了方便或需要，apiserver、controller-manage等master节点的服务被打在一个镜像内，这时只需要指定
# 一个unifiedControlPlaneImage即可。
unifiedControlPlaneImage: <string>
featureGates:
  <feature>: <bool>
  <feature>: <bool>
```

# 2. join

在Node节点执行``Kubeadm join``命令，可以把该节点加入到对应的Kubernetes集群中。join的流程是:

1. 访问apiserver获取K8S集群信息。
2. 根据集群信息，启动Kubelet进程
3. Node和Master打通后，自动在node上拉起``kube-proxy`` Pod。

# 3. reset

可以在Master或Node节点上分别执行``kubeadm reset``命令，用来回滚对应的``kubeadm init``和``kubeadm join``操作。

# 4. upgrade

``kubeadm upgrade``命令用来升级当前kubernetes集群。

首先执行``kubeadm upgrade plan``命令，检查当前可升级的版本信息。确定了要升级的版本后，执行``kubeadm upgrade apply [version]``进行在线升级。

# 5. 其他命令

1. ``kubeadm config``：提供``kubeadm config upload``、``kubeadm config view``两个子命令，分别用来备份和查询当前K8S集群的配置信息。
2. ``kubeadm completaion``：打印出kubeadm每个命令的shell代码。
3. ``kubeadm version``：打印当前kubeadm的版本信息。
4. ``kubeadm token``：集群初始化令牌的CRUD，包括``kubeadm token create``、``kubeadm token delete`` 、``kubeadm token list``和``kubeadm token generate``四个命令。

# 6. alpha集合

kubeadm提供了很多alpha测的命令，属于尝鲜、不稳定的功能。统一以``kubeadm alpha``命令提供。感兴趣的同学请自行尝试。
