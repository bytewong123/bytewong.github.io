---
title: cka真题练习——node调度
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——node调度
## 1. 将节点设置为不可用并重新调度所有运行在上面的pod
``` 
Set the node named ek8s-node-1 as unavailable and reschedule all the pods running on it
```

### 答案
```
kubectl cordon ek8s-node-1
kubectl drain ek8s-node-1 --force --ignore-daemonsets=true --delete-emptydir-data
```
## 2. 以NodeSelector方式调度pod
```
Schedule a pod as follows:
Name: nginx 
image: nginx
Node selector: disk=ssd
```
### 答案
- 获取pod模版
```
[wangzijie.byte ~]$ kubectl run nginx --image=nginx --dry-run=client -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
- 加上NodeSelector
```
nodeSelector:
    disk: ssd
status: {}
```
- 给node打标
```
apiVersion: v1
kind: Node
metadata:
  annotations:
    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
    node.alpha.kubernetes.io/ttl: "0"
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2021-10-24T10:27:34Z"
  labels:
    disk: ssd
  name: minikube-m02
```
- 检查是否打标成功
```
[wangzijie.byte ~]$ kubectl get node minikube-m02 --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
minikube-m02   Ready    <none>   27d   v1.22.2   disk=ssd
```
- 创建pod
```
[wangzijie.byte ~]$ kubectl apply -f pod.yaml
```
- 检查是否调度到minikube-m02节点
```
[wangzijie.byte ~]$ kubectl get pod -o wide | grep ^nginx
nginx                              1/1     Running   0          17m    172.17.0.5    minikube-m02   <none>           <none>
```
## 3. 检查有多少节点ready，不包括tainted的节点
```
Check to see how many nodes are ready(not including nodes tainted NoSchedule) and write the number to /opt/num.txt
```
### 答案
需要明确的是，如果一个节点被taint了，或者被cordon了，它的Taint会显示相应的信息
```
[wangzijie.byte ~]$ kubectl describe node | grep -i taint | grep NoSchedule
Taints:             gpu=no:NoSchedule
Taints:             node.kubernetes.io/unschedulable:NoSchedule
```
结果的第一行是手动taint的
结果的第二行是cordon的

- 先找出ready的节点
```
kubectl get nodes | grep -i ready | grep -v NotReady | wc -l
3
```
- 再找出不能被调度的节点
```
kubectl describe node | grep -i taint | grep NoSchedule | wc -l
2
```
- 输出结果
```
echo 1 > /opt/num.txt
```
