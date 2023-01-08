---
title: k8s架构体系
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# k8s架构体系
#k8s

## 容器的本质
容器的本质可以从静态视图、动态视图两方面来描述。
- 静态视图：一组联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs，这一部分被称为“容器镜像”
- 动态视图：一个由 Namespace+Cgroups 构成的隔离环境，这一部分被称为“容器运行时”
## k8s架构
### 控制节点（master）
#### kube-apiserver
如果需要与 Kubernetes 集群进行交互，就要通过 API。 [Kubernetes API](https://www.redhat.com/zh/topics/containers/what-is-the-kubernetes-API)  是 Kubernetes 控制平面的前端，用于处理内部和外部请求。API 服务器会确定请求是否有效，如果有效，则对其进行处理。您可以通过 REST 调用、kubectl 命令行界面或其他命令行工具（例如 kubeadm）来访问 API
#### kube-scheduler
集群是否状况良好？如果需要新的容器，要将它们放在哪里？这些是 Kubernetes 调度程序所要关注的问题。
调度程序会考虑容器集的资源需求（例如 CPU 或内存）以及集群的运行状况。随后，它会将容器集安排到适当的计算节点
#### kube-controller-manager
控制器的作用是从API Server获得**所需状态**。它检查要控制的节点的当前状态，确定是否与**所需状态**存在任何差异，并解决它们（如果有）。
#### etcd
配置数据以及有关集群状态的信息位于  [etcd](https://www.redhat.com/zh/topics/containers/what-is-etcd) （一个键值存储数据库）中
### 计算节点（node）
Kubernetes 集群中至少需要一个计算节点，但通常会有多个计算节点。容器集经过调度和编排后，就会在节点上运行。如果需要扩展集群的容量，那就要添加更多的节点。
#### kubelet
kubelet在集群中的每个节点上运行。它是Kubernetes内部的主要代理。它监视从API Server发送来的任务，执行任务，并报告给主节点。它还会监视Pod，如果Pod不能完全正常运行，则会向控制程序报告。然后，基于该信息，主服务器可以决定如何分配任务和资源以达到**所需状态**。
- kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。
- kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互。这个插件，是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件
- kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与 kubelet 进行交互的接口，分别是 CNI（Container Networking Interface）和 CSI（Container Storage Interface）
#### container runtime
容器运行时从容器镜像库中拉取镜像，然后启动和停止容器。容器运行时由第三方软件或插件（例如Docker）担当。
#### kube-proxy
kube-proxy确保每个节点都获得其IP地址，实现本地iptables和规则以处理路由和流量负载均衡。


