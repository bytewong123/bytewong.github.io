---
title: cka真题练习——pvc
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——pvc
## 1. 题目如下
```
Create a new PersistenVolumeClaim:
Name: pv-volume
Class: csi-hostpath-sc
Capacity: 10Mi 
Create a new pod which mounts the PersistenVolumeClaim as a volume:
Name: web-server
image: nginx
Mount path: /usr/share/nginx/html
Configure the new pod to have ReadWriteOnce access on the volume.
Finally,using kubectl edit or kubectl patch expand the PersistentVolumeClaim to a capacity of 70Mi and record that change
```
## 答案
- 首先创建pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Mi
  storageClassName: csi-hostpath-sc
```

- 然后创建pod
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web-server
  name: web-server
spec:
  containers:
  - image: nginx
    name: web-server
    resources: {}
    volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: pvc
  restartPolicy: Always
  volumes:
    - name: pvc
      persistentVolumeClaim:
        claimName: pv-volume
status: {}
```

- 我们还需要自己创建一个pv，来实验这个pvc是否可以有效被创建
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
spec:
  capacity:
    storage: 4Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: csi-hostpath-sc
  hostPath:
    path: /srv/app-config
```
可以看到，这里我们使用了与pvc相同的storageCalssName，这样它们才能绑定在一起。如果pv的storageClassName显式设置为了””，那么只有pvc的storageClassName也显式设置为””，pvc才能与这个pv进行绑定。如果不声明storageClassName，那么会被自动绑定默认的storageClassName
- 由于这里我们是自定义了一个storageClass，因此这里还需手动创建一个storageClass才可以
```
[wangzijie.byte ~]$ cat sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: csi-hostpath-sc
provisioner: k8s.io/minikube-hostpath
```
- 最后我们创建这几个资源即可
```sh
[wangzijie.byte ~]$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
pv-volume   Bound    pvc-2ed57e1b-7c5e-45a0-bfd3-a2c95a51bdbe   10Mi       RWO            csi-hostpath-sc   12h

[wangzijie.byte ~]$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS      REASON   AGE
app-config                                 4Gi        RWX            Retain           Available                       csi-hostpath-sc            12h

[wangzijie.byte ~]$ kubectl get pod | grep web-server
web-server                         1/1     Running   0          11h
```
- 接下来我们需要改变pvc的容量要求
若我们直接使用edit指令，修改其requests的值，会发现修改失败
```
[wangzijie.byte ~]$ kubectl edit pvc pv-volume
error: persistentvolumeclaims "pv-volume" could not be patched: persistentvolumeclaims "pv-volume" is forbidden: only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize
You can run `kubectl replace -f /tmp/kubectl-edit-3924006334.yaml` to try this update again.
```
- 这是因为我们的StorageClass不支持修改size，因此需要为StorageClass加上以下属性，才允许pvc修改容量
```
[wangzijie.byte ~]$ kubectl get sc csi-hostpath-sc -o yaml
allowVolumeExpansion: true // 加上这一行后，即可修改容量
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"csi-hostpath-sc"},"provisioner":"k8s.io/minikube-hostpath"}
  creationTimestamp: "2021-11-20T13:42:25Z"
  name: csi-hostpath-sc
  resourceVersion: "2809884"
  uid: 3b82caaf-e1e9-416d-bee6-cbc3cd034419
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
```
- 最后修改pvc，注意需要加上—record来记录变化
```
kubectl edit pvc pv-volume --record=true
...
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 70Mi // 修改为70Mi即可
  storageClassName: csi-hostpath-sc
  volumeMode: Filesystem
  volumeName: pvc-2ed57e1b-7c5e-45a0-bfd3-a2c95a51bdbe
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Mi
  phase: Bound
```
