---
title: 使用kubeadm部署kubernetes集群
date: 2019-05-20 14:51:30
tags: kubeadm kubernetes, centos7 linux 
---


# 1.准备
### 1.1 系统配置
#### 在安装之前，需要先做如下准备。两台CentOS 7.6主机如下：
```
cat /etc/hosts
192.168.32.87 node1
192.168.32.88 node2
```
如果各个主机启用了防火墙，需要开放Kubernetes各个组件所需要的端口，可以查看Installing kubeadm中的”Check required ports”一节。 这里简单起见在各节点禁用防火墙：

```
systemctl stop firewalld
systemctl disable firewalld
```
禁用SELINUX：
```
setenforce 0
vi /etc/selinux/config
SELINUX=disabled
```

创建/etc/sysctl.d/k8s.conf文件，添加如下内容：
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```
执行命令使修改生效。
```
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```


### 1.2 kube-proxy开启ipvs的前置条件
由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：
```
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
```
在所有的Kubernetes节点node1和node2上执行以下脚本:

```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4 
 
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```
```
# keruse 'nf_conntrack' instead of 'nf_conntrack_ipv4' for linux kernel >= 4.19
```
上面脚本创建了的/etc/sysconfig/modules/ipvs.modules文件，保证在节点重启后能自动加载所需模块。 使用lsmod | grep -e ip_vs -e nf_conntrack_ipv4命令查看是否已经正确加载所需的内核模块。

接下来还需要确保各个节点上已经安装了ipset软件包yum install ipset。 为了便于查看ipvs的代理规则，最好安装一下管理工具ipvsadm yum install ipvsadm。

如果以上前提条件如果不满足，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式。


### 1.3 安装Docker
Kubernetes从1.6开始使用CRI(Container Runtime Interface)容器运行时接口。默认的容器运行时仍然是Docker，使用的是kubelet中内置dockershim CRI实现。

内网状态下直接执行：
```
yum install docker
```

安装docker的yum源:
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
查看最新的Docker版本：

yum list docker-ce.x86_64  --showduplicates |sort -r
docker-ce.x86_64            3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.3.ce-1.el7                    docker-ce-stable
docker-ce.x86_64            17.03.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.0.ce-1.el7.centos             docker-ce-stable
```


Kubernetes 1.12已经针对Docker的1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06等版本做了验证，需要注意Kubernetes 1.12最低支持的Docker版本是1.11.1。Kubernetes 1.13对Docker的版本依赖方面没有变化。 我们这里在各节点安装docker的18.06.1版本。



```
yum makecache fast

yum install -y --setopt=obsoletes=0 \
  docker-ce-18.06.1.ce-3.el7

systemctl start docker
systemctl enable docker
```
确认一下iptables filter表中FOWARD链的默认策略(pllicy)为ACCEPT。
```
iptables -nvL
Chain INPUT (policy ACCEPT 263 packets, 19209 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
```
   

> Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信。但这里通过安装docker 1806，发现默认策略又改回了ACCEPT，这个不知道是从哪个版本改回的，因为我们线上版本使用的1706还是需要手动调整这个策略的。
>


# 2. 使用kubeadm部署Kubernetes
### 2.1 安装kubeadm和kubelet
 下面在各节点安装kubeadm和kubelet：
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
测试地址https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64是否可用，如果不可用需要科学上网。
```
curl https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
yum makecache fast
yum install -y kubelet kubeadm kubectl


Installed:
  kubeadm.x86_64 0:1.13.0-0                                    kubectl.x86_64 0:1.13.0-0                                                           kubelet.x86_64 0:1.13.0-0

Dependency Installed:
  cri-tools.x86_64 0:1.12.0-0                                  kubernetes-cni.x86_64 0:0.6.0-0                                                       socat.x86_64 0:1.7.3.2-2.el7
```
从安装结果可以看出还安装了cri-tools, kubernetes-cni, socat三个依赖：
官方从Kubernetes 1.9开始就将cni依赖升级到了0.6.0版本，在当前1.12中仍然是这个版本
socat是kubelet的依赖
cri-tools是CRI(Container Runtime Interface)容器运行时接口的命令行工具
运行kubelet –help可以看到原来kubelet的绝大多数命令行flag参数都被DEPRECATED了，如：
```
--address 0.0.0.0   The IP address for the Kubelet to serve on (set to 0.0.0.0 for all IPv4 interfaces and `::` for all IPv6 interfaces) (default 0.0.0.0) (DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.)
```
而官方推荐我们使用–config指定配置文件，并在配置文件中指定原来这些flag所配置的内容。具体内容可以查看这里Set Kubelet parameters via a config file。这也是Kubernetes为了支持动态Kubelet配置（Dynamic Kubelet Configuration）才这么做的，参考Reconfigure a Node’s Kubelet in a Live Cluster。

kubelet的配置文件必须是json或yaml格式，具体可查看这里。

Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。

关闭系统的Swap方法如下:
```
  swapoff -a
```
修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭。 swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：
```
  vm.swappiness=0
```
执行
```
sysctl -p /etc/sysctl.d/k8s.conf

使修改生效。
```

----

因为这里本次用于测试两台主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以这里修改kubelet的配置去掉这个限制。 之前的Kubernetes版本我们都是通过kubelet的启动参数–fail-swap-on=false去掉这个限制的。前面已经分析了Kubernetes不再推荐使用启动参数，而推荐使用配置文件。 所以这里我们改成配置文件配置的形式。


查看/etc/systemd/system/kubelet.service.d/10-kubeadm.conf，看到了下面的内容：

```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```
上面显示kubeadm部署的kubelet的配置文件–config=/var/lib/kubelet/config.yaml，实际去查看/var/lib/kubelet和这个config.yaml的配置文件都没有被创建。 可以猜想肯定是运行kubeadm初始化集群时会自动生成这个配置文件，而如果我们不关闭Swap的话，第一次初始化集群肯定会失败的。

所以还是老老实实的回到使用kubelet的启动参数–fail-swap-on=false去掉必须关闭Swap的限制。 修改/etc/sysconfig/kubelet，加入：
```
KUBELET_EXTRA_ARGS=--fail-swap-on=false
```
### 2.2 使用kubeadm init初始化集群
在各节点开机启动kubelet服务：
```
systemctl enable kubelet.service
```
接下来使用kubeadm初始化集群，选择node1作为Master Node，在node1上执行下面的命令：
```
kubeadm init \
  --kubernetes-version=v1.13.0 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.61.11
```
因为我们选择flannel作为Pod网络插件，所以上面的命令指定–pod-network-cidr=10.244.0.0/16。

执行时报了下面的错误：
```
[init] using Kubernetes version: v1.13.0
[preflight] running pre-flight checks
[preflight] Some fatal errors occurred:
        [ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```
有一个错误信息是running with swap on is not supported. Please disable swap。因为我们决定配置failSwapOn: false，所以重新添加–ignore-preflight-errors=Swap参数忽略这个错误，重新运行。

```
[root@k8s-master yyhmmwan]# kubeadm init \ --kubernetes-version=v1.13.4 \ --pod-network-cidr=10.244.0.0/16 \ --apiserver-a
dvertise-address=10.166.0.5 
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.166.0.4 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [10.166.0.4 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc ku
bernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.166.0.4]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/m
anifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 22.502906 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in t
he cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-master" as an anno
tation
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoS
chedule]
[bootstrap-token] Using token: 29h44u.gqmsznz07dl313l5
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term cer
tificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstra
p Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
You can now join any number of machines by running the following on each node
as root:
  kubeadm join 10.166.0.4:6443 
--token 29h44u.gqmsznz07dl313l5 
--discovery-token-ca-cert-hash 
sha256:ec7d11d9cdf220310f3aac7fbacceea41ddb7e86be9cf790c247c0c6802a8a7e
``` 

上面记录了完成的初始化输出的内容，根据输出的内容基本上可以看出手动初始化安装一个Kubernetes集群所需要的关键步骤。

其中有以下关键内容：
```
[kubelet-start] 生成kubelet的配置文件”/var/lib/kubelet/config.yaml”
[certificates]生成相关的各种证书
[kubeconfig]生成相关的kubeconfig文件
[bootstraptoken]生成token记录下来，后边使用kubeadm join往集群中添加节点时会用	
到下面的命令是配置常规用户如何使用kubectl访问集群：
 ```
```
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```
最后给出了将节点加入集群的命令
  kubeadm join 10.166.0.4:6443 
--token 29h44u.gqmsznz07dl313l5 
--discovery-token-ca-cert-hash 
sha256:ec7d11d9cdf220310f3aac7fbacceea41ddb7e86be9cf790c247c0c6802a8a7e
```
查看一下集群状态：
```
[root@k8s-master yyhmmwan]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
```

确认个组件都处于healthy状态。

集群初始化如果遇到问题，可以使用下面的命令进行清理：
```
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```

### 2.3 安装Pod Network
接下来安装flannel network add-on：
```
mkdir -p ~/k8s/
cd ~/k8s
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f  kube-flannel.yml

clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

如果Node有多个网卡的话，参考flannel issues 39701，目前需要在kube-flannel.yml中使用–iface参数指定集群主机内网网卡的名称，否则可能会出现dns无法解析。需要将kube-flannel.yml下载到本地，flanneld启动参数加上–iface=<iface-name>
```
......
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth1
......
```

使用kubectl get pod –all-namespaces -o wide确保所有的Pod都处于Running状态。


```
[root@k8s-master k8s]# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                 READY   STATUS              RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-86c58d9df4-k5gtz             0/1     ContainerCreating   0          45m    <none>       k8s-node     <none>           <none>
kube-system   coredns-86c58d9df4-sgnk6             0/1     ContainerCreating   0          45m    <none>       k8s-node     <none>           <none>
kube-system   etcd-k8s-master                      1/1     Running             1          44m    10.166.0.4   k8s-master   <none>           <none>
kube-system   kube-apiserver-k8s-master            1/1     Running             1          44m    10.166.0.4   k8s-master   <none>           <none>
kube-system   kube-controller-manager-k8s-master   1/1     Running             1          44m    10.166.0.4   k8s-master   <none>           <none>
kube-system   kube-flannel-ds-amd64-gngrk          0/1     CrashLoopBackOff    4          3m7s   10.166.0.5   k8s-node     <none>           <none>
kube-system   kube-flannel-ds-amd64-hgftp          0/1     CrashLoopBackOff    4          3m7s   10.166.0.4   k8s-master   <none>           <none>
kube-system   kube-proxy-78sj6                     1/1     Running             0          45m    10.166.0.5   k8s-node     <none>           <none>
kube-system   kube-proxy-xwbxv                     1/1     Running             2          45m    10.166.0.4   k8s-master   <none>           <none>
kube-system   kube-scheduler-k8s-master            1/1     Running             2          44m    10.166.0.4   k8s-master   <none>           <none>

```

### 2.4 master node参与工作负载
使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。这是因为当前的master节点node1被打上了node-role.kubernetes.io/master:NoSchedule的污点：
```
kubectl describe node node1 | grep Taint
Taints:             node-role.kubernetes.io/master:NoSchedule
```
因为这里搭建的是测试环境，去掉这个污点使node1参与工作负载：
```
kubectl taint nodes node1 node-role.kubernetes.io/master-
node "node1" untainted
```
### 2.5 测试DNS
```
kubectl run curl --image=radial/busyboxplus:curl -it
kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl create instead.
If you don't see a command prompt, try pressing enter.
[ root@curl-5cc7b478b6-r997p:/ ]$ 
```

进入后执行nslookup kubernetes.default确认解析正常:
```
nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```