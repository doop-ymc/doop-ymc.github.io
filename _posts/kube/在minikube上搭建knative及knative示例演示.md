# 1. 什么是`serverless`?什么是`knative`? 
什么是`severless`, 下面是[CNCF](https://www.cncf.io/blog/2018/02/14/cncf-takes-first-step-towards-serverless-computing/)对`serverless`架构给出的定义：

``
“Serverless computing refers to the concept of building and running applications that do not require server management. It describes a finer-grained deployment model where applications, bundled as one or more functions, are uploaded to a platform and then executed, scaled, and billed in response to the exact demand needed at the moment”
``

从定义中可以看出`serverless`架构应该下面的几个特点：

* 构建及运行应用的基础设施环境
* 无需进行服务的状态管理
* 足够细粒度的部署模式
* 可扩展且按使用量付费

上面的几个特点，除去**足够细粒度的部署模式**这点外，`kubernetes`都能够提供非常好的支持。然后幸运的是，不论是为了让`kubernetes`完整支持`serverless`架构,还是google在cloud上更加吸引开发者，Google在Google Cloud Next 2018上，发布了`knative`，将其称为: “基于`kubernetes`的平台，用来构建、部署和管理现代`serverless`架构”。`knative`的主要角色如下图中所描述：
![knative角色](/media/files/tech/knative_role.jpg)
`knative`致力于提供可重用的“通用模式和最佳实践组合”实现，目前可用的组件包括：

* `Build`: Cloud-native source to container orchestration
* `Eventing`: Management and delivery of events
* `Serving`：Request-driven compute that can scale to zero

## 1.1. Build构建系统
knative的构建工作都是被设计于`kubernetes`中进行，和整个`kubernetes`生态结合更紧密；另外，它旨在提供一个通用的标准化的构建组件，使其可以在广泛的场景得以被使用。正如官方文档中的说`Build`构建系统，更多是为了定义标准化、可移植、可重用、性能高效的构建方法。`knative`提供了 `Build CRD` 对象，让用户可以通过yaml文件定义构建过程。一个典型的`Build`配置文件如下：

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: kaniko-build
spec:
  serviceAccountName: build-bot
  source:
    git:
      url: https://github.com/my-user/my-repo
      revision: master
  template:
    name: kaniko
    arguments:
    - name: IMAGE
      value: us.gcr.io/my-project/my-app
```

## 1.2. Serving：服务系统
`serving`的核心功能是让应用运行起来提供服务。其提供的基本功能包括：
* 自动化启动和销毁容器
* 根据名字生成网络访问相关的 service、ingress等对象
* 监控应用的请求，并自动扩缩容
* 支持蓝绿发布、回滚功能，方便应用方法流程
  
`knative serving` 功能是基于 `kubernetes` 和 `istio` 开发的，它使用 `kubernetes` 来管理容器（deployment、pod），`istio` 来管理网络路由（VirtualService、DestinationRule）。
下面这张图介绍了 `knative serving` 各组件之间的关系。
![knative角色](/media/files/tech/knative_serve.jpg)

## 1.3. Eventing：事件系统
`knative`定义了很多事件相关的概念。介绍一下：
* `EventSource`：事件源，能够产生事件的外部系统
* `Feed`：把某种类型的EventType和EventSource和对应的Channel绑定到一起
* `Channel`：对消息实现的一层抽象，后端可以使用 kafka、RabbitMQ、Google PubSub作为具体的实现。channel name 类似于消息集群中的topic，可以用来解耦事件源和函数。事件发生后sink到某个channel中，然后channel中的数据会被后端的函数消费
* `Subscription`：把channel和后端的函数绑定的一起，一个channel可以绑定到多个`knative service`

目前支持的事件源有三个：`github`（比如 merge 事件，push 事件等），`kubernetes`（events），`Google PubSub`（消息系统），后面还会不断接入更多的事件源。
## 1.4. Auto-scaling
`auto-scaling`其实本质上是用于提高云上使用资源的弹性、提供按照使用量计费能力，以提代给用户高性价比的云服务的，其两个特点：
* `Request-driving`：根据请求量动态伸缩，目前通过统计系统当前并发请求量、和配置中的基准值比较，做出伸缩决策
* `Scale to zero`：无流量时完全释放资源，有请求时重新唤醒

`Knative Serving`中抽象了一系列用于定义和控制应用行为的资源对象，称为`Kubernetes Custom Resource Definitions (CRDs)`。
* `Service`：app/function生命周期管理
* `Route`：路由管理
* `Configuration`：定义了期望的运行状态
* `Revision`: 某一时刻code + configuration ，Revision是不可变对象，修改代码或配置生成新的revision
4者间的交互如下图示：
![knative auto-scaing](/media/files/tech/auto-scaling.png)
Revision生命周期有三种运行状态：
* `Active`：Revision启动，可以处理请求
* `Reserve`：一段时间未请求为0后，Revision被标记为Reserve状态，并释放占用的资源、伸缩至零
* `Retired`: Revision废弃，不再收到请求
其具体的`auto-scaling`的过程，这里就不介绍了，可以自行了解。

# 2. knative实践

在上面大致了解`knative`后，本节将详细介绍如何完成`knative`的部署。为方便大家能按指引同样完成`knative`的部署，因此选择滴滴云提供的基本的云服务器完成，大家可在滴滴云上申请云服务器按下面的步骤，完成`knative`的基本部署。若未注册滴滴云账号的，可以通过此[链接](https://i.didiyun.com/2duVxlaPt3Y)完成注册，有券^_^。

## 2.1 云服务器申请
在滴滴云注册好账号后，申请一个`16核32G内存，带80G本地盘及500G EBS数据盘`的云服务器，然后申请一按流量计费的`公网IP`。之所以申请这样的配置，是为后续完成整个部署过程更为顺畅。首先登录服务器，滴滴云的出于安全的考虑默认登录账户是`dc2-user`,并且**禁止了root用户的直接登录**，登录命令如下示：
```zsh
$Code ssh dc2-user@116.85.49.244
Warning: Permanently added '116.85.49.244' (ECDSA) to the list of known hosts.
dc2-user@116.85.49.244's password:
[dc2-user@10-255-1-243 ~]$
[dc2-user@10-255-1-243 ~]$ sudo su
[root@10-255-1-243 dc2-user]#
```
服务器登录成功，也使用`sudo su`命令完成了到root账户的切换，购买云服务器时，我们购买了一块500G的数据盘，由于从未挂载过，则需要先 `格式化云盘`，才能开始使用该云盘。初始化的过程如下示：
```zsh
[root@10-255-1-243 dc2-user]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0   80G  0 disk
└─vda1 253:1    0   80G  0 part /
vdb    253:16   0  500G  0 disk
```
`vdb`即为那块新买的`ebs`盘。详细的挂载流程可见[挂载云盘](https://help.didiyun.com/hc/kb/article/1121818/)。通过教程的指引完成了数据盘的挂载，如下示：
```zsh
[root@10-255-1-243 dc2-user]# df -h
文件系统        容量  已用  可用 已用% 挂载点
/dev/vda1        80G  1.6G   79G    2% /
devtmpfs        3.8G     0  3.8G    0% /dev
tmpfs           3.9G     0  3.9G    0% /dev/shm
tmpfs           3.9G   17M  3.9G    1% /run
tmpfs           3.9G     0  3.9G    0% /sys/fs/cgroup
tmpfs           783M     0  783M    0% /run/user/1001
tmpfs           783M     0  783M    0% /run/user/0
/dev/vdb1       500G   33M  500G    1% /data
```
到目前为此，云服务器的准备OK了，下面开始knative的部署。

## 2.2 knative部署
由于我们买的是一台裸的云服务器，因此要完成整个knative的部署，大致需要下面的几个步骤：
* go的安装
* docker安装
* kubectl安装
* minikube安装
* istio部署
* knative serving/knative build部署

ok,下面依次完成上面几个相关组件的安装。
### 2.2.1 go安装
安装go环境,先使用yum 安装一个低版本的golang,如下：
```zsh
[root@10-255-1-243 dc2-user]# yum install golang
已加载插件：fastestmirror
Repository base is listed more than once in the configuration
Repository updates is listed more than once in the configuration
Repository extras is listed more than once in the configuration
Repository centosplus is listed more than once in the configuration
Loading mirror speeds from cached hostfile
正在解决依赖关系
--> 正在检查事务
---> 软件包 golang.x86_64.0.1.8.3-1.el7 将被 安装
...
...
作为依赖被安装:
  golang-bin.x86_64 0:1.8.3-1.el7                                                                                                           golang-src.noarch 0:1.8.3-1.el7

完毕！
[root@10-255-1-243 dc2-user]# mkdir ~/workspace
[root@10-255-1-243 dc2-user]# echo 'export GOPATH="$HOME/workspace"' >> ~/.bashrc
[root@10-255-1-243 dc2-user]# source ~/.bashrc
[root@10-255-1-243 dc2-user]# go version
go version go1.8.3 linux/amd64
```
完成了基本版本的安装，但发现是`1.8`版本的，太低了后面安装`kube`相关工具时，后面也会发现确实版本太低了，直接进行升级，如下：
但因为`kubectl`必须要大于go.1.11版本的golang，因此需要升级golang,如下示：
```zsh
[root@10-255-1-243 dc2-user]# wget https://dl.google.com/go/go1.11.2.linux-amd64.tar.gz
[root@10-255-1-243 dc2-user]# tar vxf go1.11.2.linux-amd64.tar.gz
[root@10-255-1-243 dc2-user]# cd go/src
[root@10-255-1-243 src]# sh all.bash
Building Go cmd/dist using /usr/lib/golang.
Building Go toolchain1 using /usr/lib/golang.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for linux/amd64.

##### Testing packages.
ok  	archive/tar	0.021s
...
...
##### API check
Go version is "go1.11.2", ignoring -next /home/dc2-user/go/api/next.txt

ALL TESTS PASSED
---
Installed Go for linux/amd64 in /home/dc2-user/go
Installed commands in /home/dc2-user/go/bin
*** You need to add /home/dc2-user/go/bin to your PATH.
[root@10-255-1-243 src]# export PATH=/home/dc2-user/go/bin:$PATH
[root@10-255-1-243 src]# go version
go version go1.11.2 linux/amd64
```
也可以在此[地址](https://golang.org/dl/)下载对应的go版本安装。这里基本完成了go的安装。

### 2.2.2 docker安装
docker的安装是为后面的集群搭建做准备的，如下：
```zsh
[root@10-255-1-243 src]# cd -
[root@10-255-1-243 dc2-user]# sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
已加载插件：fastestmirror
adding repo from: http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
grabbing file http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
[root@10-254-167-111 dc2-user]# yum list docker-ce --showduplicates | sort -r
已加载插件：fastestmirror
可安装的软件包
Loading mirror speeds from cached hostfile
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
...
[root@10-255-1-243 dc2-user]# yum makecache fast
[root@10-255-1-243 dc2-user]# yum install  -y docker-ce-18.06.0.ce-3.el7
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
正在解决依赖关系
--> 正在检查事务
---> 软件包 docker-ce.x86_64.0.18.06.1.ce-3.el7 将被 安装
--> 正在处理依赖关系 container-selinux >= 2.9，它被软件包 docker-ce-18.06.1.ce-3.el7.x86_64 需要
--> 正在处理依赖关系 libltdl.so.7()(64bit)，它被软件包 docker-ce-18.06.1.ce-3.el7.x86_64 需要
...
...
完毕！
[root@10-255-1-243 dc2-user]# docker version
Client:
 Version:           18.06.1-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        e68fc7a
 Built:             Tue Aug 21 17:23:03 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.1-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       e68fc7a
  Built:            Tue Aug 21 17:25:29 2018
  OS/Arch:          linux/amd64
  Experimental:     false
[root@10-255-1-243 dc2-user]# service docker start
Redirecting to /bin/systemctl start docker.service
```
通过上面的步骤，即完成了docker的安装。下面继续安装其它组件。

### 2.2.3 kubectl安装
因为`knative`依赖`kubernates`，刚只在滴滴云只买了一个dc2云服务器，在开始之前还需要一个kubernates集群，由于只有一台云服务器，就直接选择安装[minikube](https://github.com/kubernetes/minikube)。安装`minikube`前可以先安装`kubectl`及相关驱动，这里选择通过源代码编译安装，编译源码需要有 Git、Golang 环境的支撑。安装试试：
```zsh
[root@10-255-1-243 dc2-user]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
[root@10-255-1-243 dc2-user]# yum install -y kubectl
[root@10-255-1-243 dc2-user]# kubectl version
Client Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.2", GitCommit:"17c77c7898218073f14c8d573582e8d2313dc740", GitTreeState:"clean", BuildDate:"2018-10-24T06:54:59Z", GoVersion:"go1.10.4", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server 10.254.150.215:8443 was refused - did you specify the right host or port?
```
以上完成了`kubectl`工具的安装。也可以不通过此方式安装，但需要注意版本正确，否则下面启动`minikube`时会报错。

### 2.2.4 minikube安装
下一步即开始安装`minikube`, minikube的安装，如下示：
```zsh
[root@10-255-1-243 dc2-user]# curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.30.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
[root@10-255-1-243 dc2-user]# export PATH=/usr/local/bin/:$PATH
[root@10-255-1-243 dc2-user]# minikube version
minikube version: v0.30.1
```
因为`minikube`的启动其实也需要依赖一些墙外的镜像的，为了顺利安装通过需要对相应的镜像提前准备好，然后以`docker tag`方式标记，相关的命令，已经准备好，放在了github中，准备过程如下示：
```zsh
[root@10-255-1-243 dc2-user]# wget https://raw.githubusercontent.com/doop-ymc/gcr/master/docker_tag.sh
--2018-11-09 15:11:30--  https://raw.githubusercontent.com/doop-ymc/gcr/master/docker_tag.sh
正在解析主机 raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.108.133
正在连接 raw.githubusercontent.com (raw.githubusercontent.com)|151.101.108.133|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：1340 (1.3K) [text/plain]
正在保存至: “docker_tag.sh”

100%[===============================================================================================================================================================================================================================================================================================================================================================================================>] 1,340       --.-K/s 用时 0s

2018-11-09 15:11:31 (116 MB/s) - 已保存 “docker_tag.sh” [1340/1340])
[root@10-255-1-243 dc2-user]# ls
docker_tag.sh  go  go1.11.2.linux-amd64.tar.gz  go1.11.2.linux-amd64.tar.gz.1  kubernetes-master  master.zip
[root@10-255-1-243 dc2-user]# sh docker_tag.sh
```
执行完上面的命令，无报错的情况，通过下面的命令即可完成启动`minikube`, 如下示：
```zsh
[root@10-255-1-243 dc2-user]#  minikube start  --registry-mirror=https://registry.docker-cn.com  --vm-driver=none  --kubernetes-version=v1.12.1   --bootstrapper=kubeadm   --extra-config=apiserver.enable-admission-plugins="LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook"
========================================
kubectl could not be found on your path. kubectl is a requirement for using minikube
To install kubectl, please run the following:

curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo cp kubectl /usr/local/bin/ && rm kubectl

To disable this message, run the following:

minikube config set WantKubectlDownloadMsg false
========================================
Starting local Kubernetes v1.12.1 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Downloading kubelet v1.12.1
Downloading kubeadm v1.12.1
Finished Downloading kubeadm v1.12.1
Finished Downloading kubelet v1.12.1
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
===================
WARNING: IT IS RECOMMENDED NOT TO RUN THE NONE DRIVER ON PERSONAL WORKSTATIONS
	The 'none' driver will run an insecure kubernetes apiserver as root that may leave the host vulnerable to CSRF attacks

When using the none driver, the kubectl config and credentials generated will be root owned and will appear in the root home directory.
You will need to move the files to the appropriate location and then set the correct permissions.  An example of this is below:

	sudo mv /root/.kube $HOME/.kube # this will write over any previous configuration
	sudo chown -R $USER $HOME/.kube
	sudo chgrp -R $USER $HOME/.kube

	sudo mv /root/.minikube $HOME/.minikube # this will write over any previous configuration
	sudo chown -R $USER $HOME/.minikube
	sudo chgrp -R $USER $HOME/.minikube

This can also be done automatically by setting the env var CHANGE_MINIKUBE_NONE_USER=true
Loading cached images from config file.
[root@10-255-1-243 dc2-user]# minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 10.255.1.243
[root@10-255-1-243 dc2-user]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-6c66ffc55b-l2hct                1/1     Running   0          3m53s
kube-system   etcd-minikube                           1/1     Running   0          3m8s
kube-system   kube-addon-manager-minikube             1/1     Running   0          2m54s
kube-system   kube-apiserver-minikube                 1/1     Running   0          2m46s
kube-system   kube-controller-manager-minikube        1/1     Running   0          3m2s
kube-system   kube-proxy-6v65g                        1/1     Running   0          3m53s
kube-system   kube-scheduler-minikube                 1/1     Running   0          3m4s
kube-system   kubernetes-dashboard-6d97598877-6g528   1/1     Running   0          3m52s
kube-system   storage-provisioner                     1/1     Running   0          3m52s
```
经历上面的过程，`minikube`基本是准备好了，下面开始安装knative相关组件

### 2.2.5 istio部署
应用下面的命令开始安装
```zsh
[root@10-255-1-243 dc2-user]# curl -L https://raw.githubusercontent.com/knative/serving/v0.2.0/third_party/istio-1.0.2/istio.yaml \
   | sed 's/LoadBalancer/NodePort/' \
   | kubectl apply --filename -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0namespace/istio-system created
configmap/istio-galley-configuration created
configmap/istio-statsd-prom-bridge created
...
...
destinationrule.networking.istio.io/istio-policy created
destinationrule.networking.istio.io/istio-telemetry created
[root@10-255-1-243 dc2-user]]# kubectl label namespace default istio-injection=enabled
namespace/default labeled
[root@10-255-1-243 dc2-user]# kubectl get pods --namespace istio-system
NAME                                        READY   STATUS              RESTARTS   AGE
istio-citadel-6959fcfb88-scskd              0/1     ContainerCreating   0          58s
istio-cleanup-secrets-xcc7w                 0/1     ContainerCreating   0          59s
istio-egressgateway-5b765869bf-7vxs5        0/1     ContainerCreating   0          58s
istio-galley-7fccb9bbd9-p2r5v               0/1     ContainerCreating   0          58s
istio-ingressgateway-69b597b6bd-pqfq9       0/1     ContainerCreating   0          58s
istio-pilot-7b594977cf-fv467                0/2     ContainerCreating   0          58s
istio-policy-59b7f4ccd5-dqstb               0/2     ContainerCreating   0          58s
istio-sidecar-injector-5c4b6cb6bc-p2nwk     0/1     ContainerCreating   0          57s
istio-statsd-prom-bridge-67bbcc746c-mcb74   0/1     ContainerCreating   0          58s
istio-telemetry-7686cd76bd-8f4l6            0/2     ContainerCreating   0          58s
```
过几分钟后，各`pods`的状态均会变为`running`或者`completed`,如下
```zsh
[root@10-255-1-243 dc2-user]# kubectl get pods --namespace istio-system
NAME                                        READY   STATUS      RESTARTS   AGE
istio-citadel-6959fcfb88-scskd              1/1     Running     0          6m11s
istio-cleanup-secrets-xcc7w                 0/1     Completed   0          6m12s
istio-egressgateway-5b765869bf-7vxs5        1/1     Running     0          6m11s
istio-galley-7fccb9bbd9-p2r5v               1/1     Running     0          6m11s
istio-ingressgateway-69b597b6bd-pqfq9       1/1     Running     0          6m11s
istio-pilot-7b594977cf-fv467                2/2     Running     0          6m11s
istio-policy-59b7f4ccd5-dqstb               2/2     Running     0          6m11s
istio-sidecar-injector-5c4b6cb6bc-p2nwk     1/1     Running     0          6m10s
istio-statsd-prom-bridge-67bbcc746c-mcb74   1/1     Running     0          6m11s
istio-telemetry-7686cd76bd-8f4l6            2/2     Running     0          6m11s
```
到这`istio`基本部署好。

### 2.2.6 knative serving/knative build部署
下面开始部署`knative`相关的组件
```bash
curl -L https://github.com/knative/serving/releases/download/v0.2.0/release-lite.yaml \
  | sed 's/LoadBalancer/NodePort/' \
  | kubectl apply --filename -
```
官方提供了上面的部署命令，但是因为`科学上网`的问题，最后是不可能装成功的，下载上面的`release-lite.yaml`其实部分依赖的`image`文件是在`gcr.io`等地方，如：
```
    gcr.io/knative-releases/github.com/knative/build/cmd/creds-init@sha256:c1c11fafd337f62eea18a1f02b78e6ae6949779bedgcr72d53d19b2870723a8f104
    gcr.io/knative-releases/github.com/knative/build/cmd/git-init@sha256:6fa8043ed114920cd61e28db3c942647ba48415fe1208acde2fb2ac0746c9164
    gcr.io/knative-releases/github.com/knative/build/cmd/nop@sha256:f94e6413749759bc3f80d33e76c36509d6a63f7b206d2ca8fff167a0bb9c77f2
    ...
```
除去上面的几个外，还有一部分这里不依依给出了，且上面的镜像地址是带`digest`引用的，直接用`docker tag`其实是解决不了问题的，如下,会报*refusing to create a tag with a digest reference*的错误：
```zsh
[root@10-255-1-243 dc2-user]# docker tag doopymc/knative-queue gcr.io/knative-releases/queue-7204c16e44715cd30f78443fb99e0f58@sha256:2e26a33aaf0e21db816fb75ea295a323e8deac0a159e8cf8cffbefc5415f78f1
refusing to create a tag with a digest reference
```
这里得想其它办法，一个比较简单的办法是利用`Docker Hub`可在国内`pull`,但它同时能拉取国外镜像的特点，选择在`Docker Hub`上构建立一个以*目标镜像*为`base`镜像的方式，然后将上面的`release-lite.yaml`上的目标镜像替换为在`Docker HUb`上建立的镜像的地址即可。如下为一个`Dockerfile`的示例：
```zsh
FROM gcr.io/knative-releases/github.com/knative/build/cmd/webhook@sha256:58775663a5bc0d782c8505a28cc88616a5e08115959dc62fa07af5ad76c54a97
MAINTAINER doop
```
`Docker Hub`的构建示例如图示：
![Docker Hub构建](/media/files/tech/docker-hub.png)
这里我已经完成了对`release-lite.yaml`不可使用镜像的替换，也放在github上了，但没替换完，只替换了这里需要用到的部分，下面安装`knative`相关组件过程如下：
```zsh
[root@10-255-1-243 dc2-user]# wget https://raw.githubusercontent.com/doop-ymc/gcr/master/release-lite.yaml
[root@10-255-1-243 dc2-user]# kubectl apply --filename release-lite.yaml
namespace/knative-build created
clusterrole.rbac.authorization.k8s.io/knative-build-admin created
serviceaccount/build-controller created
clusterrolebinding.rbac.authorization.k8s.io/build-controller-admin created
customresourcedefinition.apiextensions.k8s.io/builds.build.knative.dev created
customresourcedefinition.apiextensions.k8s.io/buildtemplates.build.knative.dev created
...
...
clusterrole.rbac.authorization.k8s.io/prometheus-system unchanged
rolebinding.rbac.authorization.k8s.io/prometheus-system unchanged
rolebinding.rbac.authorization.k8s.io/prometheus-system unchanged
rolebinding.rbac.authorization.k8s.io/prometheus-system unchanged
rolebinding.rbac.authorization.k8s.io/prometheus-system unchanged
clusterrolebinding.rbac.authorization.k8s.io/prometheus-system unchanged
service/prometheus-system-np unchanged
statefulset.apps/prometheus-system unchanged
[root@10-255-1-243 dc2-user]# kubectl get pods --namespace knative-serving
NAME                         READY   STATUS            RESTARTS   AGE
activator-59966ffc65-4l75t   0/2     PodInitializing   0          59s
activator-59966ffc65-98h5c   0/2     PodInitializing   0          59s
activator-59966ffc65-w8kdv   0/2     PodInitializing   0          59s
autoscaler-7b4989466-hpvnz   0/2     PodInitializing   0          59s
controller-6955d8bcc-xn72w   1/1     Running           0          59s
webhook-5f75b9c865-c5pdf     1/1     Running           0          59s
```
同样过几分钟后，所有的`pod`均会变为`Running`状态，如下示：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get pods --namespace knative-serving
NAME                         READY   STATUS    RESTARTS   AGE
activator-59966ffc65-4l75t   2/2     Running   0          8m31s
activator-59966ffc65-98h5c   2/2     Running   0          8m31s
activator-59966ffc65-w8kdv   2/2     Running   0          8m31s
autoscaler-7b4989466-hpvnz   2/2     Running   0          8m31s
controller-6955d8bcc-xn72w   1/1     Running   0          8m31s
webhook-5f75b9c865-c5pdf     1/1     Running   0          8m31s
```
到这一步`knative`的部署基本完成，我们能看一下在整个集群中有那些`pod`及`svc`,及他们对应的状态，首先是`service`,如下：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get svc --all-namespaces
NAMESPACE            NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                   AGE
default              kubernetes                    ClusterIP      10.96.0.1        <none>        443/TCP                                                                                                                   26m
istio-system         istio-citadel                 ClusterIP      10.107.14.76     <none>        8060/TCP,9093/TCP                                                                                                         21m
istio-system         istio-egressgateway           ClusterIP      10.104.246.50    <none>        80/TCP,443/TCP                                                                                                            21m
istio-system         istio-galley                  ClusterIP      10.98.121.169    <none>        443/TCP,9093/TCP                                                                                                          21m
istio-system         istio-ingressgateway          NodePort       10.107.139.191   <none>        80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32043/TCP,8060:30461/TCP,853:31114/TCP,15030:30980/TCP,15031:31742/TCP   21m
istio-system         istio-pilot                   ClusterIP      10.101.106.132   <none>        15010/TCP,15011/TCP,8080/TCP,9093/TCP                                                                                     21m
istio-system         istio-policy                  ClusterIP      10.108.222.26    <none>        9091/TCP,15004/TCP,9093/TCP                                                                                               21m
istio-system         istio-sidecar-injector        ClusterIP      10.103.23.143    <none>        443/TCP                                                                                                                   21m
istio-system         istio-statsd-prom-bridge      ClusterIP      10.103.76.13     <none>        9102/TCP,9125/UDP                                                                                                         21m
istio-system         istio-telemetry               ClusterIP      10.96.92.153     <none>        9091/TCP,15004/TCP,9093/TCP,42422/TCP                                                                                     21m
istio-system         knative-ingressgateway        LoadBalancer   10.97.114.164    <pending>     80:32380/TCP,443:32390/TCP,31400:32400/TCP,15011:31302/TCP,8060:32414/TCP,853:31653/TCP,15030:32327/TCP,15031:30175/TCP   10m
knative-build        build-controller              ClusterIP      10.103.97.112    <none>        9090/TCP                                                                                                                  10m
knative-build        build-webhook                 ClusterIP      10.110.178.246   <none>        443/TCP                                                                                                                   10m
knative-monitoring   grafana                       NodePort       10.104.107.125   <none>        30802:32144/TCP                                                                                                           10m
knative-monitoring   kube-controller-manager       ClusterIP      None             <none>        10252/TCP                                                                                                                 10m
knative-monitoring   kube-state-metrics            ClusterIP      None             <none>        8443/TCP,9443/TCP                                                                                                         10m
knative-monitoring   node-exporter                 ClusterIP      None             <none>        9100/TCP                                                                                                                  10m
knative-monitoring   prometheus-system-discovery   ClusterIP      None             <none>        9090/TCP                                                                                                                  10m
knative-monitoring   prometheus-system-np          NodePort       10.97.205.54     <none>        8080:32344/TCP                                                                                                            10m
knative-serving      activator-service             NodePort       10.103.75.164    <none>        80:30003/TCP,9090:30015/TCP                                                                                               10m
knative-serving      autoscaler                    ClusterIP      10.101.229.196   <none>        8080/TCP,9090/TCP                                                                                                         10m
knative-serving      controller                    ClusterIP      10.109.222.174   <none>        9090/TCP                                                                                                                  10m
knative-serving      webhook                       ClusterIP      10.101.155.150   <none>        443/TCP                                                                                                                   10m
kube-system          kube-dns                      ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP                                                                                                             26m
kube-system          kubernetes-dashboard          ClusterIP      10.104.60.66     <none>        80/TCP                                                                                                                    26m
```
再来看一下`pod`,如下：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get pod --all-namespaces
NAMESPACE            NAME                                        READY   STATUS             RESTARTS   AGE
istio-system         istio-citadel-6959fcfb88-scskd              1/1     Running            0          22m
istio-system         istio-cleanup-secrets-xcc7w                 0/1     Completed          0          22m
istio-system         istio-egressgateway-5b765869bf-7vxs5        1/1     Running            0          22m
istio-system         istio-galley-7fccb9bbd9-p2r5v               1/1     Running            0          22m
istio-system         istio-ingressgateway-69b597b6bd-pqfq9       1/1     Running            0          22m
istio-system         istio-pilot-7b594977cf-fv467                2/2     Running            0          22m
istio-system         istio-policy-59b7f4ccd5-dqstb               2/2     Running            0          22m
istio-system         istio-sidecar-injector-5c4b6cb6bc-p2nwk     1/1     Running            0          22m
istio-system         istio-statsd-prom-bridge-67bbcc746c-mcb74   1/1     Running            0          22m
istio-system         istio-telemetry-7686cd76bd-8f4l6            2/2     Running            0          22m
istio-system         knative-ingressgateway-84d56577db-flz59     1/1     Running            0          11m
knative-build        build-controller-644d855ff4-t4w72           1/1     Running            0          11m
knative-build        build-webhook-5f68d76c49-wjvx9              1/1     Running            0          11m
knative-monitoring   grafana-787566b4f6-4rlmk                    1/1     Running            0          11m
knative-monitoring   kube-state-metrics-f5446fc8c-2l94v          3/4     ImagePullBackOff   0          11m
knative-monitoring   node-exporter-kbzc6                         2/2     Running            0          11m
knative-monitoring   prometheus-system-0                         1/1     Running            0          11m
knative-monitoring   prometheus-system-1                         1/1     Running            0          11m
knative-serving      activator-59966ffc65-4l75t                  2/2     Running            0          11m
knative-serving      activator-59966ffc65-98h5c                  2/2     Running            0          11m
knative-serving      activator-59966ffc65-w8kdv                  2/2     Running            0          11m
knative-serving      autoscaler-7b4989466-hpvnz                  2/2     Running            0          11m
knative-serving      controller-6955d8bcc-xn72w                  1/1     Running            0          11m
knative-serving      webhook-5f75b9c865-c5pdf                    1/1     Running            0          11m
kube-system          coredns-6c66ffc55b-l2hct                    1/1     Running            0          27m
kube-system          etcd-minikube                               1/1     Running            0          27m
kube-system          kube-addon-manager-minikube                 1/1     Running            0          26m
kube-system          kube-apiserver-minikube                     1/1     Running            0          26m
kube-system          kube-controller-manager-minikube            1/1     Running            0          26m
kube-system          kube-proxy-6v65g                            1/1     Running            0          27m
kube-system          kube-scheduler-minikube                     1/1     Running            0          27m
kube-system          kubernetes-dashboard-6d97598877-6g528       1/1     Running            0          27m
kube-system          storage-provisioner                         1/1     Running            0          27m
```
从上可以看出`pod`的状态基本处于`Running`及`completed`,到此`knative`基本搭建完成，下面开始在在`knative`上跑一下官方的示例。

# 3. `knative`示例演示
## 3.1 应用访问演示
按官方提供的[示例](https://github.com/knative/docs/blob/master/install/getting-started-knative-app.md), 简单修改了`service.yaml`, 如下：
```yaml
apiVersion: serving.knative.dev/v1alpha1 # Current version of Knative
kind: Service
metadata:
  name: hellodidiyun-go # The name of the app
  namespace: default # The namespace the app will use
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: doopymc/helloworld-go
            env:
            - name: TARGET # The environment variable printed out by the sample app
              value: "hello, didiyun"
```
下面是此应用的启动过程，如下：
```zsh
[root@10-255-1-243 dc2-user]# wget https://raw.githubusercontent.com/doop-ymc/helloworld-go/master/service.yaml
[root@10-255-1-243 dc2-user]# kubectl apply --filename service.yaml
service.serving.knative.dev/hellodidiyun-go created
[root@10-255-1-243 dc2-user]# kubectl get pods
NAME                                               READY   STATUS            RESTARTS   AGE
hellodidiyun-go-00001-deployment-d9489b84b-ws8br   0/3     PodInitializing   0          16s
```
过几分钟后，应该会被正常拉起，如下：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
hellodidiyun-go-00001-deployment-d9489b84b-ws8br   3/3     Running   0          58s
```
下面开始访问此应用，首先找到此服务的IP地址，如下：
```zsh
[root@10-255-1-243 dc2-user]#  kubectl get svc knative-ingressgateway --namespace istio-system
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                                   AGE
knative-ingressgateway   LoadBalancer   10.97.114.164   <pending>     80:32380/TCP,443:32390/TCP,31400:32400/TCP,15011:31302/TCP,8060:32414/TCP,853:31653/TCP,15030:32327/TCP,15031:30175/TCP   23m
```
可以看到`EXTERNAL-IP`为`<pending> `状态，大概是因为无外部的`LoadBalancer`，因此采用示例中的第二种方式，获取IP,如下：
```zsh
[root@10-255-1-243 dc2-user]# export IP_ADDRESS=$(kubectl get node  --output 'jsonpath={.items[0].status.addresses[0].address}'):$(kubectl get svc knative-ingressgateway --namespace istio-system   --output 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')
[root@10-255-1-243 dc2-user]# echo $IP_ADDRESS
10.255.1.243:32380
```
下一步获取服务的访问地址，如下：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get ksvc hellodidiyun-go  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
NAME              DOMAIN
hellodidiyun-go   hellodidiyun-go.default.example.com
```
访问服务如下：
```zsh
[root@10-255-1-243 dc2-user]# curl -H "Host: hellodidiyun-go.default.example.com" http://${IP_ADDRESS}
Hello World: hello, didiyun!
```
成功返回了**Hello World: hello, didiyun!** 符合预期。

## 3.2 `auto-scaling`演示
上面介绍`knative`时，提到了其非常重要的一个机制`auto-scaling`。这里看一下，在上面访问应用一段时间后，`hellodidiyun-go`应用的`pod`会慢慢被`Terminate`，如下示：
```zsh
[root@10-255-1-243 dc2-user]# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
hellodidiyun-go-00001-deployment-d9489b84b-6zssr   3/3     Running   0          2m43s
[root@10-255-1-243 dc2-user]# kubectl get pods
NAME                                               READY   STATUS        RESTARTS   AGE
hellodidiyun-go-00001-deployment-d9489b84b-6zssr   2/3     Terminating   0          5m42s
[root@10-255-1-243 dc2-user]# kubectl get pods
No resources found.
```
下面我们重新发起一次请求，然后在看一下`pod`的状态，如下：
```zsh
[root@10-255-1-243 dc2-user]# curl -H "Host: hellodidiyun-go.default.example.com" http://${IP_ADDRESS}
Hello World: hello, didiyun!
[root@10-255-1-243 dc2-user]# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
hellodidiyun-go-00001-deployment-d9489b84b-vmcg4   3/3     Running   0          11s
```
服务又启动了，符合预期。

# 4. 结语
以前`serverless`架构更多只能在公有云上才可运行及使用，目前`knative`出现后，相信会有更多服务独立维护小的`serverless`服务，当然`knative`发布时间不长，坑肯定不少，一起来趟吧。










