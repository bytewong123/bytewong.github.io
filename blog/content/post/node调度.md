---
title: node调度
date: 2023-01-08T18:06:57+08:00
draft: false
categories: [""]
tags: [""]
---

# node调度 
## 将节点设置为不可用并重新调度所有运行在上面的pod
``` 
Set the node named ek8s-node-1 as unavailable and reschedule all the pods running on it
```

## **Node节点禁止调度（平滑维护）方式- cordon，drain，delete**
cordon、drain和delete三个命令都会使node停止被调度，后期创建的pod不会继续被调度到该节点上，但操作的暴力程度却不一样。
 但是若指定pod调度到该节点，仍然可以被调度到该节点上

## **1. cordon 停止调度**（不可调度，临时从K8S集群隔离）
* 影响最小，只会将node标识为SchedulingDisabled不可调度状态。 

* 之后K8S再创建的pod资源，不会被调度到该节点。 

* 旧有的pod不会受到影响，仍正常对外提供服务。 

* 禁止调度命令”kubectl cordon node_name”。 

* 恢复调度命令”kubectl uncordon node_name”。（恢复到K8S集群中，变回可调度状态） 


## **2. drain 驱逐节点**（先不可调度，然后排干）
* 首先，驱逐Node上的pod资源到其他节点重新创建。 

* 接着，将节点调为SchedulingDisabled不可调度状态。 

* 禁止调度命令”kubectl drain node_name —force —ignore-daemonsets —delete-local-data” 

* 恢复调度命令”kubectl uncordon node_name”。（恢复到K8S集群中，变回可调度状态） 

* drain方式是安全驱逐pod，会等到pod容器应用程序优雅停止后再删除该pod。 

* drain驱逐流程：先在Node节点删除pod，然后再在其他Node节点创建该pod。所以为了确保drain驱逐pod过程中不中断服务（即做到”无感知”地平滑驱逐），必须保证要驱逐的pod副本数大于1，并且采用了”反亲和策略”将这些pod调度到不同的Node节点上了！也就是说，在”多个pod副本+反亲和策略”的场景下，drain驱逐过程对容器服务是没有影响的。 

 
需要注意：
* 对节点执行维护操作之前（例如：内核升级，硬件维护等），您可以使用 kubectl drain 安全驱逐节点上面所有的 pod。 

* drain安全驱逐方式将会允许 pod 里面的容器遵循指定的 PodDisruptionBudgets 执行优雅中止。也就是说，drain安全驱逐可以做到：优雅地终止pod里的容器进程。 

* kubectl drain 返回成功表明所有的 pod （除了排除的那些）已经被安全驱逐（遵循期望优雅的中止期，并且没有违反任何应用程序级别的中断预算）。 

* 然后，通过对物理机断电或者在云平台上删除节点所在的虚拟机，都能安全的将节点移除。 

 
一般线上K8S的PDB（PodDisruptionBudgets）配置的也是符合Pod驱逐的理想情况的，即maxUnavailable设置为0，maxSurge设置为1
　
默认情况下，kubectl drain 会忽略那些不能杀死的系统类型的 pod。drain命令中需要添加三个参数：—force、—ignore-daemonsets、—delete-local-data
* —force 当一些pod不是经 ReplicationController, ReplicaSet, Job, DaemonSet 或者 StatefulSet 管理的时候就需要用--force来强制执行 (例如:kube-proxy) 

* —ignore-daemonsets 无视DaemonSet管理下的Pod。即—ignore-daemonsets往往需要指定的,这是因为deamonset会忽略unschedulable标签(使用kubectl drain时会自动给节点打上不可调度标签),因此deamonset控制器控制的pod被删除后可能马上又在此节点上启动起来,这样就会成为死循环.因此这里忽略daemonset。 

* —delete-local-data 如果有mount local volumn的pod，会强制杀掉该pod。 

 
drain禁止调度的操作步骤：
1. 确定要排空的节点的名称
```
kubectl get nodes  
```
2. 查看pod
```
kubectl get pod
``` 
3. 命令node节点开始释放所有pod，并且不接收新的pod进程
```
kubectl drain [node-name] —force —ignore-daemonsets —delete-local-data 
```
4. 此时可以对该node节点进行平滑维护，后续需要恢复到k8s集群中：
```
kubectl uncordon [node-name]
```
 
## **3. delete 删除节点**
* 首先，驱逐Node节点上的pod资源到其他节点重新创建。 
* 驱逐流程：先在Node节点删除pod，然后再在其他Node节点上创建这些pod。 
* node节点删除，master失去对其控制，该节点从k8s集群摘除。 
* delete是一种暴力删除node的方式。在驱逐pod时是强制干掉容器进程，做不到优雅终止Pod。相比较而言，显然drain更安全。 
 
恢复调度（即重新加入到K8S集群中）
* delete删除后，后续如果需重新加入K8S集群。则需要重启node节点的kubelet服务，重启后，基于node的自注册功能，该节点才能重新加入到K8S集群，并且恢复使用（即恢复可调度的身份）。 

* 另外：如果kubelet服务重启后，node节点系统时间跟其他节点不一致，则导致该节点证书会失效！kubelet注册后，还需要手动approve签发TLS认证操作了。 


 
## **4. Node节点平滑维护**
**通常情况下，如果要对K8S集群中的一台Node节点进行平滑维护，如升级或调整配置。正确的操作：**
* **cordon临时从K8S集群隔离出来，标识为SchedulingDisabled不可调度状态。** 

* **drain排干该节点上的pod资源到其他node节点上。** 

* **对该节点展开平滑维护操作，如升级或调整配置。** 

* **uncordon恢复，重新回到K8S集群，变回可调度状态。** 

 
同时注意：为了确保drain驱逐pod的时候，容器应用服务不中断，必须满足：
* 要驱逐的pod副本数量必须大于1 

* 要配置”反亲和策略”，确保被驱逐的pod被调度到不同的Node节点上 

* deployment采用滚动更新，设置maxUnavailable为0，maxSurge为1 

