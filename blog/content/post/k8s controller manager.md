---
title: k8s controller manager
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# k8s controller manager
#k8s
controller manager其实是一系列controller的集合，在k8s项目的pkg/controller可以看到：
```sh
$ cd kubernetes/pkg/controller/
$ ls -d */              
deployment/             job/                    podautoscaler/          
cloud/                  disruption/             namespace/              
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
...
```
这个目录下面的每一个控制器，都以独有的方式负责某种编排功能。

controller都遵循 Kubernetes 项目中的一个通用编排模式，即：控制循环（control loop）。
比如，现在有一种待编排的对象 X，它有一个对应的控制器。那么可以用一段 Go 语言风格的伪代码，描述这个控制循环：
```go

for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

- 在具体实现中，实际状态往往来自于 Kubernetes 集群本身。比如，kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息，这些都是常见的实际状态的来源。
- 而期望状态，一般来自于用户提交的 YAML 文件。比如，Deployment 对象中 Replicas 字段的值。很明显，这些信息往往都保存在 Etcd 中。

以 Deployment 为例，简单描述一下它对控制器模型的实现：
1. Deployment 控制器从 Etcd 中获取到所有携带了“app: nginx”标签的 Pod，然后统计它们的数量，这就是实际状态；
2. Deployment 对象的 Replicas 字段的值就是期望状态；
3. Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod

这个操作，通常被叫作调谐（Reconcile）。而调谐的最终结果往往都是对被控制对象的某种写操作。比如，增加 Pod，删除已有的 Pod，或者更新 Pod 的某个字段。这也是 Kubernetes 项目“面向 API 对象编程”的一个直观体现。

> Kubernetes 使用的这个“控制器模式”，跟我们平常所说的“事件驱动”，有什么区别和联系。
> 1. 主动与被动区别，事件是主动通知，控制循环是被动发现
> 2. 事件往往是一次性的，如果操作失败比较难处理，但是控制器是循环一直在尝试的，更符合kubernetes申明式API，最终达到与申明一致
