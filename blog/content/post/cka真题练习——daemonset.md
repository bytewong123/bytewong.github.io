---
title: cka真题练习——daemonset
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——daemonset
## 使用daemonset的方式，保证每个node都有一个nginx容器
```
Set configuration context $kubectl config use-context k8s. Ensure a single instance of Pod nginx is running on each node of the Kubernetes cluster where nginx also represents the image name which has to be used. Do no override any taints currently in place. Use Daemonset to complete this task and use ds.kusc00201 as Daemonset name. 
```

### 答案
- 首先创建daemonset，拷贝官方文档进行修改
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds.kusc00201
spec:
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

- 查看
可以看到三个节点都有该pod了
```
[wangzijie.byte ~]$ kubectl get pod -o wide | grep ds.kusc
ds.kusc00201-l52qx                 1/1     Running   0          4m23s   172.17.0.3    minikube-m03   <none>           <none>
ds.kusc00201-lq7vl                 1/1     Running   0          4m23s   172.17.0.31   minikube       <none>           <none>
ds.kusc00201-vgvdx                 1/1     Running   0          3m13s   172.17.0.2    minikube-m02   <none>           <none>
```

查看ds，三个都已创建成功
```
[wangzijie.byte ~]$ kubectl get ds
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds.kusc00201   3         3         3       3            3           <none>          5m20s
```