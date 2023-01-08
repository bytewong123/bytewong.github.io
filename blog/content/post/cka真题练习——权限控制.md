---
title: cka真题练习——权限控制
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——权限控制
## 创建ClusterRole绑定到service account
```
You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace. 
Create a new ClusterRole named deployment-clusterrole, which only allows to create the following resource types:
Deployment
statefulset
Daemonset 
Create a new ServiceAccount named cicd-token in the existing namespace app-team1.
bind the new ClusterRole deployment-clusterrole to the new Service Account cicd-token, limited to the namespace app-team1.
```

## 解析
我们需要创建一个ClusterRole，只拥有deploy、statefulset、daemonset的create权限，并且将其绑定到一个指定的sa以及ns下

## 答案
### 创建ClusterRole
```sh
[wangzijie.byte ~]$ kubectl create clusterrole deployment-clusterrole --verb=create --resource=deployment,daemonset,statefulset
clusterrole.rbac.authorization.k8s.io/deployment-clusterrole created
```

### 创建namespace
```sh
[wangzijie.byte ~]$ kubectl create ns app-team1
namespace/app-team1 created
```

### 创建sa
```sh
[wangzijie.byte ~]$ kubectl create sa cicd-token -n app-team1
serviceaccount/cicd-token created
```

### 创建rolebinding
注意，clusterrole是作用在全局的，rolebinding是作用在某一个命名空间的，因此创建时**必须指定创建在具体的命名空间**。rolebinding可以绑定一个具体的role，但这个role必须与其在同一个命名空间；rolebinding也可以绑定clusterrole，这个clusterrole需要作用于rolebinding的命名空间，即rolebinding被创建的命名空间。因此这里需要注意将rolebinding创建在app-team1的命名空间下。

```sh
[wangzijie.byte ~]$ kubectl create rolebinding deployment-create --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token -n app-team1
rolebinding.rbac.authorization.k8s.io/deployment-create created
```

## 验证是否成功
### 获取cicd-token的secret
```
[wangzijie.byte ~]$ kubectl get sa cicd-token -n app-team1 -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-11-20T04:13:25Z"
  name: cicd-token
  namespace: app-team1
  resourceVersion: "2728259"
  uid: d133da61-453a-433b-9e17-af28e3cf3af5
secrets:
- name: cicd-token-token-q6lxx
```
### 获取token
```
[wangzijie.byte ~]$ kubectl get secret  cicd-token-token-q6lxx -n app-team1 -o yaml
apiVersion: v1
data:
  ca.crt: XXX
  namespace: YXBwLXRlYW0x
  token: XXX
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: cicd-token
    kubernetes.io/service-account.uid: d133da61-453a-433b-9e17-af28e3cf3af5
  creationTimestamp: "2021-11-20T04:13:25Z"
  name: cicd-token-token-q6lxx
  namespace: app-team1
  resourceVersion: "2728258"
  uid: 7c969430-32e2-40fa-9b9a-16a2bcca7e92
type: kubernetes.io/service-account-token
```
### 创建一个user
```
[wangzijie.byte ~]$ token=`echo XXX | base64 -d`
[wangzijie.byte ~]$ kubectl config set-credentials --token=$token
```

### 创建一个context，绑定刚刚创建的user
```
kubectl config set-context cka --user=cka
```
### 使用当前的context
```
[wangzijie.byte ~]$ kubectl config use-context cka
Switched to context "cka".
```
### 检查context
```
[wangzijie.byte ~]$ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         cka        minikube   cka
          minikube   minikube   minikube   default
          node1      node1      node1      default
```
切换成功

### 尝试操作
```
[wangzijie.byte ~]$ kubectl run pod nginx --image=nginx
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:app-team1:cicd-token" cannot create resource "pods" in API group "" in the namespace "default"

[wangzijie.byte ~]$ kubectl create deployment nginx --image=nginx
error: failed to create deployment: deployments.apps is forbidden: User "system:serviceaccount:app-team1:cicd-token" cannot create resource "deployments" in API group "apps" in the namespace "default"

[wangzijie.byte ~]$ kubectl get pod -n app-team1
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:app-team1:cicd-token" cannot list resource "pods" in API group "" in the namespace "app-team1"

[wangzijie.byte ~]$ kubectl create deployment nginx --image=nginx -n app-team1
deployment.apps/nginx created
```

## 总结
- rolebinding要作用在某个ns，需要在目标ns下创建rolebinding
- rolebinding可以绑定同一ns下的role，也可以绑定全局的clusterrole
- 熟悉kubectl context的操作
