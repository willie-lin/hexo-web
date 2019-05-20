---
title: network-policy-分析
date: 2019-05-20 17:03:34
tags: k8s calico networkpolicy ingress engress
---


## 一个例子

```
kind: NetworkPolicy （资源类型）
apiVersion: networking.k8s.io/v1 （api版本号）
metadata:
  name: allow-same-namespace  （这条策略的名字）
  namespace: default  （对应到的名字空间）
spec:
  podSelector:    （下发的范围选择器）
    matchLabels:   （匹配具有如下Lable的） 
      color: blue
  ingress:          （入站策略）
  - from:            （允许哪些src访问)
    - podSelector:   （又是选择器）
        color: red
    to:              （目的端口）
      ports:
      - port: 80

```
这条就是，允许red访问blue的80端口

## 还可以定义允许别的名字空间的选择器
```language
  ingress:
  - from:
    - podSelector:
        color: red
      namespaceSelector: 
        shape: square
```

## 出站策略
```language
  egress:
  - to:
    - podSelector: 
        color: red
      ports:
      - port: 80
```

## 配置ip地址模式
```language
  egress:
  - to:
    - ipBlock:
        cidr: 172.18.0.0/24
```

## deny all
```language
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
  types:
  - Ingress
  - Egress
```

以下配置成功了
```language
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
查看具体描述
```
kubectl describe networkpolicy
```


## 参考
[https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
[https://docs.projectcalico.org/master/security/kubernetes-network-policy#create-deny-all-default-ingress-and-egress-network-policy](https://docs.projectcalico.org/master/security/kubernetes-network-policy#create-deny-all-default-ingress-and-egress-network-policy)
