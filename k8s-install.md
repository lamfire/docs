# 一.安装前的环境准备工作

 

主机名称修改，目前我的3台主机名称分别是：

```
k8s-master :  192.168.1.200
k8s-node-1:   192.168.1.201
k8s-node-2:   192.168.1.202
```

 安装必需软件

```
apt install -y ipset ipvsadm chrony ca-certificates curl  software-properties-common apt-transport-https containerd locales
```

可以使用下面的方式更改名称：

```
hostnamectl set-hostname  k8s-master
```





设置各节点网络的别名

```
vim /etc/hosts
192.168.1.200  k8s-master
192.168.1.201 k8s-node-1
192.168.1.202 k8s-node-2
```

 

关闭防火墙【没有uwf服务的不用操作】

```
systemctl stop  uwf && systemctl disable uwf
```



 

关闭swap

```
swapoff -a
sed -i  '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
```



调大ulimit

```
cat >> /etc/security/limits.conf <<EOF
# End of file
* soft nproc 1000000
* hard nproc 1000000
* soft nofile 1000000
* hard nofile 1000000
root soft nproc 1000000
root hard nproc 1000000
root soft nofile 1000000
root hard nofile 1000000
EOF

ulimit -SHn 1000000

```

 

将桥接的IPv4流量传递到iptables的链

```
vim /etc/sysctl.conf

fs.file-max = 1000000
fs.inotify.max_user_instances = 8192
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.route.gc_timeout = 100
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 32768
net.core.netdev_budget = 5000
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_max_orphans = 32768
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.conf.all.route_localnet = 1

sysctl -p
```

 

如果出现缺少文件的现象

```
sysctl: cannot stat  /proc/sys/net/bridge/bridge-nf-call-iptables: 
```

则确认是否驱动加载完成,驱动加载:

```
modprobe br_netfilter
```

 

可以通过以下设置来实现自动加载模块

```
vim /etc/modules

modprobe -- nvmet-tcp
modprobe -- br_netfilter
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe --  nf_conntrack

```

现在，每次启动时，这些模块都会被自动加载。

 

 

 

设置时区及同步

```
rm -rf /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
sudo systemctl restart  chrony
sudo systemctl status chrony
chronyc sources
```



 

#  二.免密钥登录


k8s 要求 管理节点可以直接免密登录工作节点 的原因是：在集群搭建完成后，管理节点的 kubelet 需要登陆工作节点进行操作。

 

最后再来配置一下免秘钥登录，配置过程很简单，想让机器 master 访问机器 node，就把机器 master 的公钥放到机器 node  的~/.ssh/authorized_keys 文件里就行了。

 

首先我们在marster上生成一个密钥，输入下述命令后一路回车即可：

 

```
sudo ssh-keygen
```




然后登录node节点，并依次输入下述两条命令将其复制并写入到node的authorized_keys中：

 

```
sudo scp [root@192.168.1.200:~/.ssh/id_rsa.pub](mailto:root@192.168.1.200:~/.ssh/id_rsa.pub) /home sudo cat /home/id_rsa.pub >>  ~/.ssh/authorized_keys 
```

 

然后再次使用ssh node登录就可以发现直接连接上而不需要密码了。

 

# 三. 安装 k8s环境


每个节点都执行安装基础环境，并增加配置k8s阿里云源

 

\# 安装基础环境

```
curl -s  https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
echo 'deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main' >>  /etc/apt/sources.list.d/kubernetes.list 
apt update -y
```

 

配置containerd
注意文件是没有的，需要手动创建
最终把这个文件里面的sandbox_image改成下面的形式

 

```
mkdir /etc/containerd
containerd config default >  /etc/containerd/config.toml

 

[plugins."io.containerd.grpc.v1.cri"]
sandbox_image  = "registry.aliyuncs.com/google_containers/pause:3.8"

 

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup  = true

 

systemctl restart containerd
```

 

# 四. 配置master节点

 

 \#查看kubeadm kubelet kubectl有哪些版本，以及版本安装时的具体名称

```
apt-cache madison kubeadm kubelet kubectl 
kubeadm kubelet  kubectlapt -y install kubelet=1.27.6-00 kubeadm=1.27.6-00  kubectl=1.27.6-00
apt-mark hold kubelet kubeadm kubectl
systemctl start  kubelet 
systemctl enable kubelet
```

 

先来简单介绍一下这三者： 

 

kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s  部署都将使用它来完成。
kubectl: k8s  的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载

 

下载完成后就要迎来重头戏了，初始化master节点，这一章节只需要在管理节点上配置即可，大致可以分为如下几步：

 

初始化 master 节点

 

运行下列命令，其中--apiserver-advertise-address参数的 ip  地址修改为自己的master主机地址，然后再执行。

 

```
kubeadm config images pull --image-repository  registry.aliyuncs.com/google_containers
kubeadm init  --image-repository=registry.aliyuncs.com/google_containers  --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16  --apiserver-advertise-address=192.168.1.61  
```



 


这里介绍一下一些常用参数的含义：

 

--apiserver-advertise-address: k8s 中的主要服务apiserver的部署地址，填自己的管理节点  ip
--image-repository: 拉取的 docker 镜像源，因为初始化的时候kubeadm会去拉 k8s  的很多组件来进行部署，所以需要指定国内镜像源，下不然会拉取不到镜像。
--pod-network-cidr: 这个是 k8s  采用的节点网络，因为我们将要使用flannel作为 k8s 的网络，所以这里填10.244.0.0/16就好
--kubernetes-version:  这个是用来指定你要部署的 k8s  版本的，一般不用填，不过如果初始化过程中出现了因为版本不对导致的安装错误的话，可以用这个参数手动指定。
--ignore-preflight-errors:  忽略初始化时遇到的错误，比如说我想忽略 cpu 数量不够 2  核引起的错误，就可以用--ignore-preflight-errors=CpuNum。错误名称在初始化错误时会给出来。
当你看到如下字样是，就说明初始化成功了，请把最后那行以kubeadm  join开头的命令复制下来，之后安装工作节点时要用到的，如果你不慎遗失了该命令，可以在master节点上使用kubeadm token create  --print-join-command命令来重新生成一条。

 

执行后发现错误（注意，所有节点都执行）
kubeadm  reset
如果在初始化过程中出现了任何Error导致初始化终止了，使用kubeadm reset重置之后再重新进行初始化。

 

成功后的提示

```
Your Kubernetes control-plane has initialized  successfully!
....
kubeadm join 192.168.1.222:6443 --token  jdp39b.vbex8e43ghaf43o8 \    
--discovery-token-ca-cert-hash  sha256:bc8938a25290033d19d04c57fc1bb4e3179584fe3d67b75f3486e374d9f51857

 
```

配置 kubectl 工具
按照提示执行（master上执行）

 

```
mkdir -p $HOME/.kube
sudo cp -i  /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g)  $HOME/.kube/config

 

export  KUBECONFIG=/etc/kubernetes/admin.conf

 

echo 'export KUBECONFIG=/etc/kubernetes/admin.conf'  >> .bashrc
```



 

执行完成后并不会刷新出什么信息，可以通过下面两条命令测试 kubectl是否可用：

 

查看已加入的节点

```
kubectl get nodes
```

 

查看集群状态

```
kubectl get cs
```

 

部署 flannel 网络
flannel是什么？它是一个专门为 k8s 设置的网络规划服务，可以让集群中的不同节点主机创建的  docker 容器都具有全集群唯一的虚拟IP地址。
在全部节点安装flannel

```
curl -L   https://github.com/flannel-io/flannel/releases/download/v0.20.1/flanneld-amd64 -o /usr/bin/flanneld
chmod +x /usr/bin/flanneld
```

 

在master安装

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

一般情况下可能会安装不成功，可以先把文件下载下来，然后更改镜像地址
把“image: docker.io”开头的内容，全部改成“image:  dockerproxy.com”开头即可

 

#### 置kube-proxy模式为IPVS模式

 

kube-proxy默认采用iptables作为代理，而iptables的性能有限，不适合生产环境，需要改为IPVS模式

查看模块是否已加载

```
lsmod | grep ip_vs
lsmod | grep  nf_conntrack
```

 

设置kube-proxy为IPVS模式

```
kubectl edit configmap  kube-proxy -n kube-system
mode: "ipvs"
```

 

重启kube-proxy组件并检查模式

```
kubectl rollout restart  daemonset kube-proxy -n kube-system 
```

 

检查kube-proxy运行模式,重启kube-proxy进程后容器Pods会销毁并重新创建

```
systemctl daemon-reload && systemctl restart kubelet  && systemctl restart containerd  
```

 

\#确认所有的Pod都在正确的网络上运行 

```
[root@k8s-master](mailto:root@k8s-master):~# kubectl get pods  -A -o wide
NAMESPACE   NAME                 READY   STATUS  RESTARTS     AGE  IP       NODE     NOMINATED NODE   READINESS GATES
kube-flannel  kube-flannel-ds-6snmr        1/1    Running  0        32m  192.168.1.223  k8s-node-1   <none>      <none>
kube-flannel   kube-flannel-ds-mktvm        1/1   Running  0        43m   192.168.1.222  k8s-master  <none>       <none>
kube-flannel  kube-flannel-ds-ptdj8        1/1    Running  0        32m  192.168.1.224  k8s-node-2   <none>      <none>
kube-system   coredns-66f779496c-lrlkc       1/1   Running  0        48m   10.244.0.2   k8s-master  <none>       <none>
kube-system  coredns-66f779496c-vxlfv       1/1    Running  0        48m  10.244.0.3   k8s-master   <none>      <none>
kube-system   etcd-k8s-master           1/1   Running  0        48m   192.168.1.222  k8s-master  <none>       <none>
kube-system  kube-apiserver-k8s-master      1/1    Running  0        48m  192.168.1.222  k8s-master   <none>      <none>
kube-system   kube-controller-manager-k8s-master  1/1   Running  0        48m   192.168.1.222  k8s-master  <none>       <none>
kube-system  kube-proxy-8jnwd           1/1    Running  7 (51s ago)   32m  192.168.1.223  k8s-node-1   <none>      <none>
kube-system   kube-proxy-gx42h           1/1   Running  15 (4m24s ago)  48m   192.168.1.222  k8s-master  <none>       <none>
kube-system  kube-proxy-z9lz2           1/1    Running  5 (49s ago)   32m  192.168.1.224  k8s-node-2   <none>      <none>
kube-system   kube-scheduler-k8s-master      1/1   Running  0        48m   192.168.1.222  k8s-master  <none>       <none>
```

​              

 

至此，k8s 管理节点部署完成。

 

# 五. 配置slave节点并加入网络

```
apt -y install kubelet=1.27.6-00 kubeadm=1.27.6-00  
apt-mark hold kubelet kubeadm 
systemctl start kubelet 
systemctl  enable kubelet
```

 

配置文件处理（仅在node上进行）
复制master节点的 /etc/kubernetes/admin.conf 到 node节点  $HOME/.kube/config

```
mkdir -p $HOME/.kube
sudo scp  [root@192.168.1.200:/etc/kubernetes/admin.conf](mailto:root@192.168.1.200:/etc/kubernetes/admin.conf) $HOME/.kube/config

sudo chown $(id -u):$(id -g)  $HOME/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >>  $HOME/.bashrc
source ~/.bashrc
```


执行从步骤 4 中保存的命令即可完成加入，注意，这条命令每个人的都不一样，不要直接复制执行：

 

```
kubeadm join 192.168.1.200:6443 --token  jdp39b.vbex8e43ghaf43o8 --discovery-token-ca-cert-hash  sha256:bc8938a25290033d19d04c57fc1bb4e3179584fe3d67b75f3486e374d9f51857
```



 

待控制台中输出以下内容后即为加入成功：

```
This node has joined the  cluster:
\* Certificate signing request was sent to apiserver and a response  was received.* 
The Kubelet was informed of the new secure connection  details.
Run 'kubectl get nodes' on the master to see this node join the  cluster.
```

 

 

 

随后登录master查看已加入节点状态，可以看到node-1已加入，并且状态均为就绪。至此，k8s 搭建完成：

 

```
[root@devsvr](mailto:root@devsvr):~# kubectl get node -o  wide
NAME     STATUS  ROLES      AGE   VERSION  INTERNAL-IP    EXTERNAL-IP  OS-IMAGE    KERNEL-VERSION    CONTAINER-RUNTIME
devsvr    Ready  control-plane  27m   v1.28.2   192.168.1.222  <none>    Ubuntu 23.10  6.5.0-27-generic   containerd://1.7.2
k8s-node-1  Ready  <none>     3m51s   v1.28.2  192.168.1.223  <none>    Ubuntu 23.10  6.5.0-27-generic   containerd://1.7.2
k8s-node-2  Ready  <none>     3m24s   v1.28.2  192.168.1.224  <none>    Ubuntu 23.10  6.5.0-27-generic   containerd://1.7.2
```



 


查看启动的pod服务

 

```
[root@devsvr](mailto:root@devsvr):~# kubectl get pods -A  -o wide

NAMESPACE   NAME               READY   STATUS       RESTARTS    AGE   IP       NODE      NOMINATED NODE  READINESS GATES
kube-flannel   kube-flannel-ds-8dtqc      0/1   Init:0/2      0        6m17s  192.168.1.223  k8s-node-1  <none>       <none>
kube-flannel  kube-flannel-ds-dht4q      0/1    Init:0/2      0       5m50s  192.168.1.224  k8s-node-2   <none>      <none>
kube-flannel   kube-flannel-ds-td96l      1/1   Running       0        25m   192.168.1.222  devsvr    <none>       <none>
kube-system  coredns-66f779496c-gsdvv     1/1    Running       0       30m   10.244.0.2   devsvr     <none>      <none>
kube-system   coredns-66f779496c-hkxxh     1/1   Running       0        30m   10.244.0.3   devsvr    <none>       <none>
kube-system  etcd-devsvr           1/1    Running       4       30m   192.168.1.222  devsvr     <none>      <none>
kube-system   kube-apiserver-devsvr      1/1   Running       0        30m   192.168.1.222  devsvr    <none>       <none>
kube-system  kube-controller-manager-devsvr  1/1    Running       0       30m   192.168.1.222  devsvr     <none>      <none>
kube-system   kube-proxy-m28ml         0/1   ContainerCreating  0        6m17s  192.168.1.223  k8s-node-1  <none>       <none>
kube-system  kube-proxy-t4lr8         0/1    CrashLoopBackOff  8 (4m5s ago)  30m   192.168.1.222  devsvr     <none>      <none>
kube-system   kube-proxy-vz8lf         0/1   ContainerCreating  0        5m50s  192.168.1.224  k8s-node-2  <none>       <none>
kube-system  kube-scheduler-devsvr      1/1    Running       0       30m   192.168.1.222  devsvr     <none>      <none>
```



 

 

# 六.重置节点

 

如果子节点需要重新加入到其他节点，还需要在kubeadm  reset后，执行下面命令，不然会出现奇奇怪怪问题，比如cni0没删除，或者创建pod起不来

 

```
ifconfig cni0 down  
ip link delete  cni0

systemctl stop kubelet
systemctl stop containerd

 

kubeadm reset

 

rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm  -rf /etc/cni/

 

systemctl restart kubelet
systemctl restart  containerd 
```



 


至此你应该已经搭建好了一个完整可用的双节点 k8s 集群了。

 

 

# 七.常见问题修复

 

如果你是使用virtualBox部署的虚拟机，并且虚拟机直接无法使用网卡1的 ip 地址互相访问的话（例如组建双网卡，网卡1为 NAT  地址转换用来上网，网卡2为Host-only，用于虚拟机之间访问）。就需要执行本节的内容来修改 k8s  的默认网卡。不然会出现一些命令无法使用的问题。如果你的默认网卡可以进行虚拟机之间的相互访问，则没有该问题。

 

## 1.修改 kubelet 默认地址


访问kubelet配置文件：

 

```
sudo vi  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

 

在最后一行ExecStart 之前 添加如下内容：

 

```
Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.1.223"
```

 

重启kubelet：

 

```
systemctl daemon-reload && systemctl restart kubelet
```

 

至此修改完成，更多信息详见 kubectl logs、exec、port-forward 执行失败问题解决 。

 

 

## 2.修改 flannel 的默认网卡

 

编辑flannel配置文件

 

```
sudo kubectl edit daemonset kube-proxy -n  kube-system
```

 

找到spec.template.spec.containers.args字段并添加--iface=网卡名，例如我的网卡是enp0s8：

 

```
--iface=enp0s8
```

 

:wq保存修改后输入以下内容删除所有 flannel，k8s 会自动重建：

 

```
kubectl delete pod -n kube-system -l  app=flannel
```

 

至此修改完成，更多内容请见 解决k8s无法通过svc访问其他节点pod的问题 。

 