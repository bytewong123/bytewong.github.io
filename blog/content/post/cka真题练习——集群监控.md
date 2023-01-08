---
title: cka真题练习——集群监控
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——集群监控
## 查找cpu使用最多的pod
```
From the pod label name=overloaded-cpu, find pods running high cpu workloads and write the name of the pod
consuming most cpu to the file /opt/test1.txt (which already exists)
```

### 答案
通过top命令可以列出容器的cpu、memory使用情况。—sort-by可以对其进行排序。注意使用-A来列出所有namespace的情况
```
[wangzijie.byte ~]$ kubectl top pod -l name=overloaded-cpu --sort-by=cpu -A
NAMESPACE   NAME              CPU(cores)   MEMORY(bytes)
default     multi-container   10m          27Mi
default     big-corp-app      2m           1Mi
default     nginx             1m           0Mi
```

```
kubectl top pod -l name=overloaded-cpu --sort-by=cpu -A | head -2 | tail -1 | awk '{print $2}' > /opt/test1.txt
```