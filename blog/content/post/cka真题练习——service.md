---
title: cka真题练习——service
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——service
```
Reconfigure the existing deployment front-end and add a port specification named http exposing port 80/tcp of the existing container nginx.
Create a new service named front-end-svc exposing the container port http.
Configure the new service to also expose the individual pods via a NodePort on the nodes on which they are scheduled
```

## 答案
### 给deploy的nginx container开一个80端口，起名为http
```
[wangzijie.byte ~]$ kubectl edit deploy front-end
...
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: front-end
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: front-end
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports: // 新增
        - containerPort: 80 // 新增
          name: http // 新增
          protocol: TCP // 新增
```
### 创建一个service front-end-svc，暴露刚刚创建的http ports
```
[wangzijie.byte ~]$ kubectl expose deploy front-end --name=front-end-svc --port=80 --target-port=80
service/front-end-svc exposed
```
### 配置这个新建的svc，将其以NodePort的形式暴露出来
```
[wangzijie.byte ~]$ kubectl edit svc front-end-svc
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-11-20T11:23:52Z"
  labels:
    app: front-end
  name: front-end-svc
  namespace: default
  resourceVersion: "2755680"
  uid: d9116169-9bc7-4534-a28a-1102ca665cd8
spec:
  clusterIP: 10.106.220.174
  clusterIPs:
  - 10.106.220.174
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30552
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: front-end
  sessionAffinity: None
  type: NodePort // 修改为NodePort
status:
  loadBalancer: {}
```