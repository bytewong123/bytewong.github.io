---
title: cka真题练习——deploy
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——deploy
```
scale the deployment testpod to 4 pods
```

### 答案
```
kubectl scale deploy testpod --replicas=4
```