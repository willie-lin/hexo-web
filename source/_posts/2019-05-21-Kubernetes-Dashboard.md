---
title: Kubernetes-Dashboard
date: 2019-05-21 00:03:26
tags: k8s dashboard
---

## 下载官方提供的 Dashboard 组件部署的 yaml 文件
```language
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

## 修改 yaml 文件中的镜像
k8s.gcr.io 修改为 registry.cn-hangzhou.aliyuncs.com/google_containers，后续所有 yaml 文件中，只要涉及到 image 的，都需要做同样的修改，因为国内 k8s.gcr.io 这个地址被墙了。

## 修改 yaml 文件中的 Dashboard Service，暴露服务使外部能够访问
```language
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31111
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

## 启动 Dashboard
```language
kubectl apply -f kubernetes-dashboard.yaml
```

## 访问 Dashboard

地址： https://<Your-IP>:31111/
注意：必须是 https

## 创建能够访问 Dashboard 的用户
新建文件 account.yaml ，内容如下：

```language
# Create Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
# Create ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

作者：GoGooGooo
链接：https://www.jianshu.com/p/31bee0cecaf2
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。
```

## 获取登录 Dashboard 的令牌 （Token）
```language
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```


# 参考
[k8s 安装](https://www.jianshu.com/p/31bee0cecaf2)