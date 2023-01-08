---
title: cka真题练习——sidecar
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——sidecar
## 通过一个sidecar，收集主容器的日志
```
Without changing its existing containers,an existing pod needs to be integrated into kubernetes's built-in logging
architecture (e.g.kubectl logs).Adding a streaming sidecar container is a good and common way to accomplish
this requirement.
Add a busybox sidecar container to the existing pod legacy-app.The new sidecat container has to run the
following command: 
/bin/sh -c tail -n+1 /var/log/legacy-app.log
Use a volume mount named logs to make the file /var/log/legacy-app.log available to the sidecar container.
Don't modify the existing container.
Dont modify the path of the log file,both containers must access it at /var/log legacy-app.log
```

### 答案
- 首先，构造一个业务容器
```
apiVersion: v1
kind: Pod
metadata:
  name: big-corp-app
spec:
  containers:
  - name: big-corp-app
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      mkdir -p /var/log
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/big-corp-app.log;
        i=$((i+1));
        sleep 1;
      done
```

- 接着，我们需要想办法将其/var/log/big-corp-app.log，能由一个sidecar通过kubectl log的方式输出出来。
这个思路主要是让业务容器和sidecar容器都挂载一个相同的emptyDir，这样，两个容器挂载的目录就能共享了。
```
apiVersion: v1
kind: Pod
metadata:
  name: big-corp-app
spec:
  containers:
  - name: big-corp-app
    image: busybox
    volumeMounts:
    - mountPath: /var/log/
      name: logs
    args:
    - /bin/sh
    - -c
    - >
      mkdir -p /var/log
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/big-corp-app.log;
        i=$((i+1));
        sleep 1;
      done
  - image: busybox
    command: ['sh', '-c', 'tail -n+1 -f /app/big-corp-app.log']
    name: busybox
    resources: {}
    volumeMounts:
    - mountPath: /app/
      name: logs
  volumes:
  - name: logs
    emptyDir: {}
```

需要注意的是，sidecar容器挂载的目录不一定要是业务容器挂载的目录。业务容器挂载目录以后，挂载目录下的所有文件与sidecar容器挂载的目录下的所有文件共享
- 需要先删除容器，再新建这个容器