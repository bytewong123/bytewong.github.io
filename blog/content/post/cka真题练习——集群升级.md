---
title: cka真题练习——集群升级
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——集群升级
```
Given an existing Kubernetes cluster running version 1.18.8,upgrade all of the Kubernetes control plane and node components on the master node only to version 1.19.0.
You are also expected to upgrade kubelet and kubectl on the master node.
Be sure to drain the master node before upgrading it and uncordon it after the upgrade.do not upgrade the worker nodes,etcd,the container manager,the CNI plugin,the DNS service or any other addons.
```

升级master节点的版本、kubelet的版本、kubectl的版本，并且不要升级etcd；

```sh
controlplane $ kubectl get node
NAME           STATUS   ROLES    AGE     VERSION
controlplane   Ready    master   4m11s   v1.18.0
node01         Ready    <none>   3m35s   v1.18.0
```

## 1. 使master节点暂停调度
```sh
controlplane $ kubectl drain controlplane --ignore-daemonsets
node/controlplane already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-flannel-ds-amd64-2wnc7, kube-system/kube-proxy-f4q2q
evicting pod kube-system/coredns-66bff467f8-nw82p
evicting pod kube-system/coredns-66bff467f8-sjx8r
pod/coredns-66bff467f8-sjx8r evicted
pod/coredns-66bff467f8-nw82p evicted
node/controlplane evicted
controlplane $ kubectl get node
NAME           STATUS                     ROLES    AGE     VERSION
controlplane   Ready,SchedulingDisabled   master   5m24s   v1.18.0
node01         Ready                      <none>   4m48s   v1.18.0
```

## 2. 登录到master节点，并切换到root，配置补全
```sh
controlplane $ ssh controlplane
Warning: Permanently added 'controlplane' (ECDSA) to the list of known hosts.
controlplane $ sudo -i
controlplane $ source <(kubectl completion bash)
```

## 3. 检查本地是否拥有1.19.0版本的kubeadm
```sh
controlplane $ apt-cache show kubeadm | grep 1.19.0
Version: 1.19.0-00
Filename: pool/kubeadm_1.19.0-00_amd64_8165879db2293a6f6d6c795c66bd8b8949079c303ae73a1baa9ad93af97787a2.deb
```

> 注意，若没有该版本的kubeadm，那么需要去添加相应的源，方法可见：
> [Installing kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)，搜索kubeadm install即可

## 4. 安装kubeadm
```sh
controlplane $ apt-get install kubeadm=1.19.0-00
```
注意，这里的version需要指定为apt-cache show结果中Version后的值，即1.19.0-00

## 5. 升级master节点的版本
```sh
controlplane $ kubeadm upgrade apply 1.19.0 --etcd-upgrade=false
```
注意，由于我们不需要升级etcd，因此需要指定—etcd-upgrade=false，不指定默认为true

## 6. 升级kubelet
```sh
controlplane $ apt-get install kubelet=1.19.0-00
```

## 7. 升级kubectl
```sh
controlplane $ apt-get install kubectl=1.19.0-00
```

## 8. 检查版本
```sh
controlplane $ kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.0", GitCommit:"e19964183377d0ec2052d1f1fa930c4d7575bd50", GitTreeState:"clean", BuildDate:"2020-08-26T14:30:33Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.0", GitCommit:"e19964183377d0ec2052d1f1fa930c4d7575bd50", GitTreeState:"clean", BuildDate:"2020-08-26T14:23:04Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}
```
客户端、服务端版本都已经为1.19.0了，成功

```sh
controlplane $ kubectl get node
NAME           STATUS                     ROLES    AGE   VERSION
controlplane   Ready,SchedulingDisabled   master   18m   v1.19.0
node01         Ready                      <none>   17m   v1.18.0
```
master节点的版本也已经为1.19.0了

## 9. 恢复master节点的调度
```sh
controlplane $ kubectl uncordon controlplane
node/controlplane uncordoned
controlplane $ kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
controlplane   Ready    master   19m   v1.19.0
node01         Ready    <none>   19m   v1.18.0
```
