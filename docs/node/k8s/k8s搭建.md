
# ubuntu搭建高可用k8s集群多master



# 虚拟机地址

* 10.100.2.200 master200
* 10.100.2.201 master201
* 10.100.2.202 node202
* 10.100.2.203 node203
* 10.100.2.204 node204
* 10.100.2.205 node205






# k8s配置文件总结
```sh
# ################################################################################################
# k8s配置文件总结
/var/lib/kubelet/config.yaml
/var/lib/kubelet/kubeadm-flags.env # --cgroup-driver=systemd
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
/etc/kubernetes/kubelet
/lib/systemd/system/kubelet.service
```


# 环境准备

## 设置代理

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

## 配置软件仓库
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


## 系统配置

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


## apt更新安装基础apt包
```sh
apt-get update
apt-get install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

# docker环境

## docker安装
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


## docker安装haproxy

**参考**
* [k8s高可用部署：keepalived + haproxy_Kubernetes中文社区](https://www.kubernetes.org.cn/6964.html) 
* [docker中haproxy的安装以及负载均衡配置 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1876608) 
* [keepalived in docker - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1472255) 

**拉取镜像** 
```sh
docker pull haproxy
```

**配置目录** 
```sh
mkdir -p /docker/haproxy-master/ 
vim /docker/haproxy-master/haproxy.cfg
```

**配置haproxy.cfg文件**
```
defaults
    mode            tcp
    log             global
    option          tcplog
    option          dontlognull
    option http-server-close
    option          redispatch
    retries         3
    timeout http-request 10s
    timeout queue   1m
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    timeout http-keep-alive 10s
    timeout check   10s
    maxconn         3000
frontend    k8s
    bind        0.0.0.0:13307
    mode        tcp
    log         global
    default_backend k8s_apiserver

backend     k8s_apiserver
    balance roundrobin
    server master200 10.100.2.200:6443 check inter 5s rise 2 fall 3
    server master201 10.100.2.201:6443 check inter 5s rise 2 fall 3
    server master202 10.100.2.202:6443 check inter 5s rise 2 fall 3

listen stats
    mode    http
    bind    0.0.0.0:1080
    stats   enable
    stats   hide-version
    stats uri /haproxyamdin?stats
    stats realm Haproxy\ Statistics
    stats auth admin:admin
    stats admin if TRUE
```

**构建容器**
```sh
docker run --restart always -p 1080:1080 -p 13307:13307 -d --name haproxy-master -v /docker/haproxy-master/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg --privileged=true haproxy
```



## docker安装keepalievd
**参考**
* [k8s高可用部署：keepalived + haproxy_Kubernetes中文社区](https://www.kubernetes.org.cn/6964.html)
**拉取镜像**
```sh
docker pull osixia/keepalived
```
**创建文件**
```sh
mkdir -p  /docker/keepalived/
vim  /docker/keepalived/keepalived.conf
```

**配置文件**
```sh
global_defs {
   router_id node203 # keepalived 主机唯一标识，建议使用当前主机名
}

vrrp_instance VI_1 {
    state BACKUP # 当前节点在此虚拟路由器上的初始状态，状态为MASTER或者BACKUP
    interface ens3 # 绑定当前虚拟路由器使用的物理接口，可以不和VIP在同一个网卡
    virtual_router_id 51 # 每个虚拟路由器惟一标识，范围：0-255，同属一个虚拟路由器的多个 keepalived 节点此值必须相同
    priority 50 # 当前物理节点在此虚拟路由器的优先级，范围：1-254
    nopreempt # 非抢占式
    advert_int 1 # vrrp通告的时间间隔，默认1s
    authentication { # 认证机制
        auth_type PASS  # 认证类型，可以是AH或PASS，AH为IPSEC认证(不推荐),PASS为简单密码(建议使用)
        auth_pass 1111  # 预共享密钥，即相互认证密码
    }
    virtual_ipaddress { # 虚拟路由IP
                10.100.2.207 # 指定VIP的MASK和网卡，注意：不指定/prefix,默认为32
    }
    # 使用单播配置，按需求和上面的组播二选一即可
    unicast_src_ip 10.100.2.203        # 本机IP
    unicast_peer{
        10.100.2.204                    # 指向其他Keepalived主机IP
        10.100.2.205
    }
    track_script {
      chk_haproxy
    }
}

vrrp_script chk_haproxy {
        script "/bin/bash -c 'if [[ $(netstat -nlp | grep 13307) ]]; then exit 0; else exit 1; fi'"
        interval  1
        weight 3
}
```
**运行镜像**
```sh
docker run --restart always --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host --volume /docker/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf -d osixia/keepalived --copy-service
```
**查看日志**
```sh
docker logs 容器id
```



# k8s安装

**参考** 
* [安装最新版Calico-CSDN博客](https://blog.csdn.net/wvxvsuizhong/article/details/124068957)
* [Calico官网所有版本](https://docs.tigera.io/archive) 

## 主节点、从节点都执行
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

## 主节点(master)初始化
```sh

# （主节点执行）单节点k8s主节点执行初始化
kubeadm init --pod-network-cidr=10.100.2.0/24 --apiserver-advertise-address=10.100.2.200 --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.4 --ignore-preflight-errors=Swap


# （主节点执行）多节点k8s主节点执行初始化（注意，要在keepalived的主节点初始化）
kubeadm init --pod-network-cidr=10.100.2.0/24  --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.4 --ignore-preflight-errors=Swap --control-plane-endpoint=10.100.2.207:6443 --upload-certs



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

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.100.2.207:6443 --token xvf7to.6fl0gjdxmoadu62f \
    --discovery-token-ca-cert-hash sha256:2f86ed4492c0bc301186507f464a236ff80f9214e90ba132315fa7169e854197 \
    --control-plane --certificate-key b0f336b64d7a7e05093e797bb014156597b298d8770a1af83c8538fafa2145f5

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.100.2.207:6443 --token xvf7to.6fl0gjdxmoadu62f \
    --discovery-token-ca-cert-hash sha256:2f86ed4492c0bc301186507f464a236ff80f9214e90ba132315fa7169e854197 

# 对于root用户添加，在/etc/profile 加入下面一行
export KUBECONFIG=/etc/kubernetes/admin.conf

# 验证主节点成功
kubectl get pods -n kube-system
# 健康检查（不要怀疑，就是healthz）
curl -k https://localhost:6443/healthz

```


## node节点执行
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



## 主节点(master)安装calico网络插件
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

## 创建calico网络
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
## 查看node
```sh
root@master200:~# kubectl get node
NAME        STATUS   ROLES    AGE     VERSION
master200   Ready    master   2d18h   v1.18.4
node204     Ready    <none>   2d17h   v1.18.4
node205     Ready    <none>   2d17h   v1.18.4
```

## 查看路由
```sh
root@master200:~# ip route
default via 10.100.0.1 dev ens3 proto static 
10.100.0.0/22 dev ens3 proto kernel scope link src 10.100.2.200 
10.100.2.64/26 via 10.100.2.204 dev ens3 proto bird 
10.100.2.128/26 via 10.100.2.205 dev ens3 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```


## 增加一个master节点 

```sh
  kubeadm join 10.100.2.207:6443 --token xvf7to.6fl0gjdxmoadu62f \
    --discovery-token-ca-cert-hash sha256:2f86ed4492c0bc301186507f464a236ff80f9214e90ba132315fa7169e854197 \
    --control-plane --certificate-key b0f336b64d7a7e05093e797bb014156597b298d8770a1af83c8538fafa2145f5
```

# 其他
## 删除node重新加入集群

**参考** 
* [k8s集群进行删除并添加node节点 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1888163) 
* [K8S集群增加新node - kubeadm 生成 token_kubeadm token gene_lswzw的博客-CSDN博客](https://blog.csdn.net/lswzw/article/details/90168428) 

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


# 参考博客
* [1.1 基于ubuntu 部署最新版 k8s 集群 — 图解K8S documentation (iswbm.com)](https://k8s.iswbm.com/c01/p01_depoly-kubernetes-cluster-with-kubelet.html#id1)
* [K8S的安装(Ubuntu 20.04) - 简书 (jianshu.com)](https://www.jianshu.com/p/520d6414a4ab) 
* [Kubernetes第二篇：从零开始搭建k8s集群（亲测可用）51CTO博客_如何搭建k8s集群](https://blog.51cto.com/u_15287666/5269619)  