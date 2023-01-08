---
title: cka真题练习——pod
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——pod
## 创建一个多容器的pod
```
Create a pod named multi-container with a single app container for each of the following images running in side
(there may be between 1 and 4 images specified):
nginx + redis + memcached + consul
```
## 答案
```
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
  - name: memcached
    image: memcached
  - name: consul
    image: consul
```