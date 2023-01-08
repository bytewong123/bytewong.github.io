---
title: cka真题练习——日志
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——日志
## 从名为bar的pod中监听到包含file-not-found的日志，并将其输出到/opt/test/bar目录下
```
Monitor the logs of pod bar and:
Exract log lines corresponding to error file-not-found
Write them to /opt/test/bar
```
### 答案
```
kubectl logs bar | grep 'file-not-found' > /opt/test/bar
```

