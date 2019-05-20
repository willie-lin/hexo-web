---
title: 在kubernetes上部署Calico网络插件
date: 2019-05-20 16:46:21
tags: docker centos7 k8s calico
---


### 1.创建单主机Kubernetes集群
```
参考【使用kubeadm安装Kubernetes】
```
1. 作为具有sudo权限的普通用户，请在安装了kubeadm的主机上打开终端。

2. 使用以下命令初始化主服务器。
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
注意：如果您的网络中已在使用192.168.0.0/16，则必须选择不同的pod网络CIDR，在上述命令中替换192.168.0.0/16以及下面应用的任何清单中。

3. 执行以下命令配置kubectl（也返回 kubeadm init）。
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
````
4. 使用以下命令安装Calico。
```
kubectl apply -f \
https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
注意：您还可以 在新选项卡中查看YAML。

```
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
deployment.extensions/calico-kube-controllers created
serviceaccount/calico-kube-controllers created

```
5. 使用以下命令确认所有窗格正在运行。
```
watch kubectl get pods --all-namespaces
```
等到每个吊舱具有STATUS的Running。

```
NAMESPACE     NAME                                       READY   STATUS             RESTARTS   AGE
default       curl-66959f6557-wrctx                      1/1     Running            0          34m
kube-system   calico-kube-controllers-644fcf8fbf-8w7m5   1/1     Running            0          15m
kube-system   calico-node-c7fkl                          1/1     Running            0          15m
kube-system   calico-node-fkvv7                          1/1     Running            0          15m
kube-system   coredns-86c58d9df4-k5gtz                   1/1     Running            0          86m
kube-system   coredns-86c58d9df4-sgnk6                   1/1     Running            0          86m
kube-system   etcd-k8s-master                            1/1     Running            1          85m
kube-system   kube-apiserver-k8s-master                  1/1     Running            1          85m
kube-system   kube-controller-manager-k8s-master         1/1     Running            1          85m
kube-system   kube-flannel-ds-amd64-gngrk                0/1     CrashLoopBackOff   17         44m
kube-system   kube-flannel-ds-amd64-hgftp                0/1     CrashLoopBackOff   13         44m
kube-system   kube-proxy-78sj6                           1/1     Running            1          86m
kube-system   kube-proxy-xwbxv                           1/1     Running            2          86m
kube-system   kube-scheduler-k8s-master                  1/1     Running            2          85m

```

6. 按CTRL + C退出watch。

7. 删除主服务器上的污点，以便您可以在其上安排pod。
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
它应该返回以下内容。
```
node/<your-hostname> untainted
```

8. 使用以下命令确认您现在在群集中有一个节点。
```
kubectl get nodes -o wide
```
它应该返回类似下面的内容。

```
[root@k8s-master ~]# kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-master   Ready    master   92m   v1.13.4   10.166.0.4    <none>        CentOS Linux 7 (Core)   3.10.0-957.5.1.el7.x86_64   docker://18.9.3
k8s-node     Ready    <none>   92m   v1.13.4   10.166.0.5    <none>        CentOS Linux 7 (Core)   3.10.0-957.5.1.el7.x86_64   docker://18.6.1
```