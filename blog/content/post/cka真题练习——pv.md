---
title: cka真题练习——pv
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——pv
## 创建一个hostPath类型的pv
```
Create a persistent volume with name app-config,of capacity 4Gi and access mode ReadWriteMany.The type of volume is hostPath and its location is /srv/app-config
```
### 答案
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
  storageClassName: manual
  hostPath:
    path: /srv/app-config
```

## 列出所有pv，按name排序
```
Set configuration context $kubectl config use-context k8s. List all PVs sorted by name, saving the full kubectl output to /opt/KUCC0010/my_volumes. Use kubectl own functionally for sorting the output, and do not manipulate it any further. 
```
### 答案
sort-by使用的字段定位符，可通过get -o yaml查看
```yaml
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"labels":{"type":"local"},"name":"app-config"},"spec":{"accessModes":["ReadWriteMany"],"capacity":{"storage":"4Mi"},"hostPath":{"path":"/srv/app-config"}}}
    creationTimestamp: "2021-12-04T09:03:48Z"
    finalizers:
    - kubernetes.io/pv-protection
    labels:
      type: local
    name: app-config
    resourceVersion: "297845"
    uid: d990646f-598c-45f4-92a5-b639c56371c9
  spec:
    accessModes:
    - ReadWriteMany
    capacity:
      storage: 4Mi
    hostPath:
      path: /srv/app-config
      type: ""
    persistentVolumeReclaimPolicy: Retain
    volumeMode: Filesystem
  status:
    phase: Available
```
若需要定位到name，那么是metadata.name；如需要定位到storage，那么是spec.capacity.storage

```
[wangzijie.byte ~]$ kubectl get -A pv --sort-by=metadata.name
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS      REASON   AGE
app-config                                 4Gi        RWX            Retain           Available                       csi-hostpath-sc            16h
pvc-2ed57e1b-7c5e-45a0-bfd3-a2c95a51bdbe   10Mi       RWO            Delete           Bound       default/pv-volume   csi-hostpath-sc            16h
```

```
apiVersion: v1
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"app-config"},"spec":{"accessModes":["ReadWriteMany"],"capacity":{"storage":"4Gi"},"hostPath":{"path":"/srv/app-config"},"storageClassName":"csi-hostpath-sc","volumeMode":"Filesystem"}}
    creationTimestamp: "2021-11-20T13:32:51Z"
    finalizers:
    - kubernetes.io/pv-protection
    name: app-config
    resourceVersion: "2763889"
    uid: 5d6476e4-39ed-45f7-a36e-2d660b9d80a8
```
- 若需要按storage排序：
```
[wangzijie.byte ~]$ kubectl get -A pv --sort-by=spec.capacity.storage
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS      REASON   AGE
pvc-2ed57e1b-7c5e-45a0-bfd3-a2c95a51bdbe   10Mi       RWO            Delete           Bound       default/pv-volume   csi-hostpath-sc            16h
app-config                                 4Gi        RWX            Retain           Available                       csi-hostpath-sc            16h
```
