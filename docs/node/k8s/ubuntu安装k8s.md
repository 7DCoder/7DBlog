
# ubuntu安装k8s



# 虚拟机地址

* 10.100.2.200 master200
* 10.100.2.201 master201
* 10.100.2.202 node202
* 10.100.2.203 node203
* 10.100.2.204 node204
* 10.100.2.205 node205



**参考博客** 
* [1.1 基于ubuntu 部署最新版 k8s 集群 — 图解K8S documentation (iswbm.com)](https://k8s.iswbm.com/c01/p01_depoly-kubernetes-cluster-with-kubelet.html#id1)
* [K8S的安装(Ubuntu 20.04) - 简书 (jianshu.com)](https://www.jianshu.com/p/520d6414a4ab) 
* [Kubernetes第二篇：从零开始搭建k8s集群（亲测可用）51CTO博客_如何搭建k8s集群](https://blog.51cto.com/u_15287666/5269619)  

## 0. k8s配置文件总结
```sh
# ################################################################################################
# k8s配置文件总结
/var/lib/kubelet/config.yaml
/var/lib/kubelet/kubeadm-flags.env # --cgroup-driver=systemd
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
/etc/kubernetes/kubelet
/lib/systemd/system/kubelet.service
```

## 1. 设置代理

把下述内容写入 `/etc/profile` 文件中。
```shell
vim /etc/profile
# /etc/profile 文件最后写入
export http_proxy=http://10.100.0.10:808
export https_proxy=http://10.100.0.10:808
# k8s相关的地址必须不经过代理，否则k8s无法初始化
export no_proxy=127.0.0.1,localhost,10.100.2.0/24,10.96.0.0/12,10.100.2.200,10.100.2.201,10.100.2.202,10.100.2.203,10.100.2.204,10.100.2.205,

# 使其生效
source /etc/profile

# 配置新建连接也生效
vim ~/.bashrc
# 文件最后增加下面命令，保存
source /etc/profile


# 验证是否成功
# 验证方法一：
curl www.baidu.com
# 验证方法二：查看代理
> env|grep -i proxy
https_proxy=http://10.100.0.10:808
http_proxy=http://10.100.0.10:808
```

## 2. 配置软件仓库
**参考连接** 
* [ubuntu镜像_ubuntu下载地址_ubuntu安装教程-阿里巴巴开源镜像站 (aliyun.com)](https://developer.aliyun.com/mirror/ubuntu) 
* [ubuntu | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirrors-i.tuna.tsinghua.edu.cn/help/ubuntu/) 

```sh
# #####################################################################################
# 打开sources.list文件
vim /etc/apt/sources.list
deb https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

# deb https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse

```


## 3. 系统配置

**参考** 
* [kubeadm集群化部署多master节点（生产环境适用）kubeadm部署多master_andboby的博客-CSDN博客](https://blog.csdn.net/wangqiubo2010/article/details/114582436)

```sh
# ###########################################################################################
# 时区修改
timedatectl set-timezone Asia/Shanghai

# ###########################################################################################
# 关闭swap
vim /etc/fstab
# 最后一行 `/swap.img` 已经注释掉

# 重启ubuntu
reboot

# 验证命令：free -h
root@envops:/home/envops# free -h
              total        used        free      shared  buff/cache   available
Mem:          1.9Gi       149Mi       1.5Gi       1.0Mi       295Mi       1.6Gi
Swap:            0B          0B          0B

# 如果启用了Swap分区则会显示Swap分区的总大小和使用情况。

# ###########################################################################################
# 关闭防火墙
ufw disable
# 验证命令
ufw status

# ###########################################################################################
# 其他一些配置
# Enable kernel modules
 modprobe overlay &&  modprobe br_netfilter
 
# Add some settings to sysctl
tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# 开启ip转发  
vim /etc/sysctl.conf
net.ipv4.ip_forward=1
 
# Reload sysctl
 sysctl --system

# 查看状态
sysctl -p

# ###########################################################################################
# 修改主机名
hostnamectl set-hostname master200
# 验证
root@envops:~# hostname
master200

# ###########################################################################################
# 修改 /etc/hosts 文件
	vim /etc/hosts
# 添加
10.100.2.200 master200
10.100.2.201 master201
10.100.2.202 node202
10.100.2.203 node203
10.100.2.204 node204
10.100.2.205 node205

```


## 4.  apt更新安装基础apt包
```sh
apt-get update
apt-get install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

## 5. docker安装
```sh
# ###########################################################################################
# 1. 首先，更新软件包索引，并且安装必要的依赖软件，来添加一个新的 HTTPS 软件源
apt update
apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

# ###########################################################################################
# 2. 使用下面的 `curl` 导入源仓库的 GPG key：
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 3. 将 Docker APT 软件源添加到你的系统：
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# 现在，Docker 软件源被启用了，你可以安装软件源中任何可用的 Docker 版本。

# 4.   （最新版本安装）安装 Docker 最新版本，运行下面的命令。如果你想安装指定版本，跳过这个步骤，并且跳到下一步。
apt update 
apt install docker-ce docker-ce-cli containerd.io

# 5.(指定版本安装)想要安装指定版本，首先列出 Docker 软件源中所有可用的版本：
apt update
apt list -a docker-ce

# 6 安装指定的版本 `apt install docker-ce=<VERSION> docker-ce-cli=<VERSION> containerd.io `
apt install docker-ce=5:19.03.15~3-0~ubuntu-focal  docker-ce-cli=5:19.03.15~3-0~ubuntu-focal  containerd.io

# 7. docker-compose（可以不做）
# apt install -y docker-compose

# 8. 将docker设置为开机自启 
systemctl enable docker


# 9. 修改镜像源,创建文件`/etc/docker/daemon.json`
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": [
		"https://registry.docker-cn.com",
		"https://docker.mirrors.ustc.edu.cn",
		"http://hub-mirror.c.163.com",
		"https://cr.console.aliyun.com/"
	]
}

# 8. （因为我的虚拟机需要代理上网，因此需要在docker中配置代理，无需代理就不用执行这一步）
vim /etc/systemd/system/multi-user.target.wants/docker.service
# 然后在service下面加入代理的配置，例如：
Environment=HTTP_PROXY=http://10.100.0.10:808
Environment=HTTPS_PROXY=http://10.100.0.10:808
Environment=NO_PROXY=localhost,127.0.0.1

# 10. 重启docker
systemctl daemon-reload
systemctl restart docker

# 11. 验证
docker run hello-world
systemctl status docker

# 12 阻止 Docker 自动更新，锁住它的版本
apt-mark hold docker-ce

```

## 6. k8s安装
* [安装最新版Calico-CSDN博客](https://blog.csdn.net/wvxvsuizhong/article/details/124068957)
* [Calico官网所有版本](https://docs.tigera.io/archive) 

**主节点、从节点都执行**
```sh

###########################################################################################
# 添加证书
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

# 添加apt源
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# apt更新
apt-get update

# ###########################################################################################
# 查看可安装版本
apt-cache madison kubelet 

# ###########################################################################################
# 安装指定版本(1.18.4)
apt-get install -y kubelet=1.18.4-00 kubeadm=1.18.4-00 kubectl=1.18.4-00

# 设置apt不更新 docker-ce kubelet kubeadm kubectl
apt-mark hold docker-ce kubelet kubeadm kubectl

# 设置开机启动(1.18.4)
systemctl enable kubelet &&  systemctl start kubelet

# 查看所需镜像(1.18.4)
kubeadm config images list --kubernetes-version=v1.18.4
 
k8s.gcr.io/kube-apiserver:v1.18.4
k8s.gcr.io/kube-controller-manager:v1.18.4
k8s.gcr.io/kube-scheduler:v1.18.4
k8s.gcr.io/kube-proxy:v1.18.4
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7

# 下载需要的镜像(1.18.4)
kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.4

```

**主节点(master)初始化**
```sh

# （主节点执行）k8s主节点执行初始化
kubeadm init --pod-network-cidr=10.100.2.0/24 --apiserver-advertise-address=10.100.2.200 --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.4 --ignore-preflight-errors=Swap


# #########################################################################################
# 初始化没有成功时，使用如下命令查看详细错误信息、
journalctl -xeu kubelet
journalctl -xeu kubelet -l
journalctl -f -u kubelet
systemctl status kubelet
# 使用kubeadm reset恢复
kubeadm reset



# #########################################################################################
# 成功示例，
# 执行1 ` mkdir -p $HOME/.kube` 
# 执行2 ` sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
# 执行3 ` sudo chown $(id -u):$(id -g) $HOME/.kube/config`
# 保存 `kubeadm join 10.100.2.201:6443 --token txi20a.77rm6gz5gb2r5wpk \
#    --discovery-token-ca-cert-hash sha256:2544cc7e4be705bd67f3d0521c1125ab4995bc7304b80366c0619177d585eaeb`
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.100.2.201:6443 --token txi20a.77rm6gz5gb2r5wpk \
    --discovery-token-ca-cert-hash sha256:2544cc7e4be705bd67f3d0521c1125ab4995bc7304b80366c0619177d585eaeb

# 对于root用户添加，在/etc/profile 加入下面一行
export KUBECONFIG=/etc/kubernetes/admin.conf

# 验证主节点成功
kubectl get pods -n kube-system
# 健康检查（不要怀疑，就是healthz）
curl -k https://localhost:6443/healthz

```


**从节点执行**
```sh
# #########################################################################################
# 从节点执行
# node节点机器配置
# 步骤：
# 1. 设置代理
# 2. 软件仓库配置
# 3. 系统配置
# 4. apt更新安装基础apt包
# 5. docker安装
# 6. k8s安装（执行步骤到“下载需要的镜像(1.18.4)”即可）


# 后面我们要将 worker 节点加入集群，就要执行这条命令
# （主节点）这条命令是有有效期的，需要的时候，可以执行如下命令进行获取
kubeadm token create --print-join-command


# 部署 node节点
kubeadm join 10.100.2.201:6443 --token txi20a.77rm6gz5gb2r5wpk \
    --discovery-token-ca-cert-hash sha256:2544cc7e4be705bd67f3d0521c1125ab4995bc7304b80366c0619177d585eaeb

# 主节点查看
kubectl get nodes

```



**主节点(master)安装calico网络插件** 
参考博客：
* [calico网络安装和删除_k8s删除calico_开发运维玄德公的博客-CSDN博客](https://blog.csdn.net/xingzuo_1840/article/details/119580448)
```sh
# 安装 calico 的v3.18版本（对应k8s的1.18/1.19/1.20版本）
wget https://docs.projectcalico.org/archive/v3.18/manifests/calico.yaml
```
官网地址：[Install Calico networking and network policy for on-premises deployments (tigera.io)](https://docs.tigera.io/archive/v3.18/getting-started/kubernetes/self-managed-onprem/onpremises) 
```sh
# 修改配置文件
vim calico.yaml
# 1.关闭IPIP模式,修改CALICO_IPV4POOL_IPIP 为 off
            - name: CALICO_IPV4POOL_IPIP
              #value: "Always"
              value: "off"
# 2. 修改pod网段 CALICO_IPV4POOL_CIDR（k8s初始化命令kubeadm init --pod-network-cidr=10.100.2.0/24对应的网段）
            - name: CALICO_IPV4POOL_CIDR
              value: "10.100.2.0/24"
```

创建calico网络
```sh
kubectl create -f calico.yaml
```
查看pod,calico-node和calico-kube-controllers都启动起来了，coredns 也变为Running
```sh
root@master200:~# kubectl get pod -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-7b5bcff94c-bjvq7   1/1     Running   0          13m
kube-system   calico-node-mq8xw                          1/1     Running   0          13m
kube-system   calico-node-pjndz                          1/1     Running   0          13m
kube-system   calico-node-x8rb7                          1/1     Running   0          13m
kube-system   coredns-7ff77c879f-8nndv                   1/1     Running   0          16s
kube-system   coredns-7ff77c879f-m8ncm                   1/1     Running   0          16s
kube-system   etcd-master200                             1/1     Running   0          2d18h
kube-system   kube-apiserver-master200                   1/1     Running   0          2d18h
kube-system   kube-controller-manager-master200          1/1     Running   0          2d18h
kube-system   kube-proxy-6795x                           1/1     Running   0          2d17h
kube-system   kube-proxy-8t874                           1/1     Running   0          2d18h
kube-system   kube-proxy-gzv8z                           1/1     Running   0          2d17h
kube-system   kube-scheduler-master200   
```
查看node
```sh
root@master200:~# kubectl get node
NAME        STATUS   ROLES    AGE     VERSION
master200   Ready    master   2d18h   v1.18.4
node204     Ready    <none>   2d17h   v1.18.4
node205     Ready    <none>   2d17h   v1.18.4
```

查看路由
```sh
root@master200:~# ip route
default via 10.100.0.1 dev ens3 proto static 
10.100.0.0/22 dev ens3 proto kernel scope link src 10.100.2.200 
10.100.2.64/26 via 10.100.2.204 dev ens3 proto bird 
10.100.2.128/26 via 10.100.2.205 dev ens3 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```


**增加一个master节点** 
[如何向Kubernetes集群中添加master节点（原集群只有一个master节点） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/460794489) 
首次添加节点需要执行
```sh
# 在初始化的节点中查看文件kubeadm-config.yaml
kubectl -n kube-system get cm kubeadm-config -oyaml
# 查看是否有`controlPlaneEndpoint`,如果没有，则执行下一行命令
kubectl -n kube-system edit cm kubeadm-config
# 添加 controlPlaneEndpoint: 10.100.2.200:6443（主节点ip地址）
kind: ClusterConfiguration  
kubernetesVersion: v1.18.4
controlPlaneEndpoint: 172.16.64.2:6443 # 在此处添加


# （master节点）执行命令 kubeadm init phase upload-certs --upload-certs
root@envops:~# kubeadm init phase upload-certs --upload-certs
I0407 17:14:25.402390   29313 version.go:252] remote version is much newer: v1.26.3; falling back to: stable-1.18
W0407 17:14:26.134663   29313 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
6d22c82254ce677dacf0cea76d304cad420ba20dc0d0924dbccf518cbec09050

# 保存certificate key:： 6d22c82254ce677dacf0cea76d304cad420ba20dc0d0924dbccf518cbec09050

# （master节点）执行
root@envops:~# kubeadm token create --print-join-command
kubeadm join 10.100.2.200:6443 --token vyz985.9fgo1zwfpl47tsz3 --discovery-token-ca-cert-hash sha256:d3d729f64601e5f7889d6b86a2a37f2cb2acbbf5d802eaa3407fbda97792edde 

# 将得到的token和certificate key进行拼接，得到如下命令：
# > 注意事项：  
#1.  不要使用 --experimental-control-plane，会报错
#2.  要加上--control-plane --certificate-key ，不然就会添加为node节点而不是master
#3.  join的时候节点上不要部署，如果部署了kubeadm reset后再join
kubeadm join 10.100.2.200:6443 --token vyz985.9fgo1zwfpl47tsz3 --discovery-token-ca-cert-hash sha256:d3d729f64601e5f7889d6b86a2a37f2cb2acbbf5d802eaa3407fbda97792edde  --control-plane --certificate-key 6d22c82254ce677dacf0cea76d304cad420ba20dc0d0924dbccf518cbec09050
```


## 99. 删除node在重新加入集群
* [k8s集群进行删除并添加node节点 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1888163) 
* [(4条消息) K8S集群增加新node - kubeadm 生成 token_kubeadm token gene_lswzw的博客-CSDN博客](https://blog.csdn.net/lswzw/article/details/90168428) 

```sh
# master节点执行###########################################
# 查看节点
kubectl get nodes
# 删除节点
kubectl delete nodes k8s-node1(节点名称)


# node节点###########################################
# 在被删除的node节点中清空集群数据信息。
kubeadm reset

# iptables 必须执行。否则重新加入集群会报错
iptables -nL
iptables -F
iptables -X
iptables -Z

# 重启机器（不重启的情况下，再次加入节点总是出问题）
reboot


# master节点执行###########################################
# 生成新的token（直接执行kubeadm token create --print-join-command即可）
kubeadm token create
# 查看新的token
kubeadm token list
# 获取集群加入的命令
kubeadm token create --print-join-command



# node节点执行###########################################
# 将node节点重新添加到k8s集群中
kubeadm join 10.100.2.201:6443 --token txi20a.77rm6gz5gb2r5wpk \
    --discovery-token-ca-cert-hash sha256:2544cc7e4be705bd67f3d0521c1125ab4995bc7304b80366c0619177d585eaeb


# 如果还是node节点还是notReady状态master节点重新安装calico网络
kubectl delete -f calico.yaml
kubectl apply -f calico.yaml

```
