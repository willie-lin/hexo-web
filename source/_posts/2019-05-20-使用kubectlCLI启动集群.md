---
title: 使用kubectlCLI启动集群
date: 2019-05-20 23:20:06
tags: k8s kubectl 
---


###  使用kubectl CLI 启动集群
1. 可以使用kubectl CLI 与集群进行交互。这是用于管理Kubernetes和在群集上运行的应用程序的主要方法。
2. 可以通过以下方式发现群集及其运行状况的详细信息 
	```
	kubectl cluster-info
	```
3. 使用查看群集中的节点 
	```
	kubectl get nodes
	```
	如果节点标记为NotReady，则它仍在启动组件。

	此命令显示可用于托管应用程序的所有节点。现在我们只有一个节点，我们可以看到它的状态已准备好（它已准备好接受部署的应用程序）


### 第3步 - 部署容器

通过运行Kubernetes集群，现在可以部署容器。

1. 使用kubectl run它，它允许将容器部署到集群上 -
```
kubectl run first-deployment --image=katacoda/docker-http-server --port=80
```
2. 可以通过正在运行的Pod发现部署状态 
``` 
kubectl get pods
```
3. 容器运行后，可以根据需要通过不同的网络选项进行暴露。一种可能的解决方案是NodePort，它为容器提供动态端口。
```
kubectl expose deployment first-deployment --port=80 --type=NodePort
```

4. 以下命令查找已分配的端口并执行HTTP请求。
```
export PORT=$(kubectl get svc first-deployment -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')
echo "Accessing host01:$PORT"
curl host01:$PORT
```
结果是处理请求的容器。