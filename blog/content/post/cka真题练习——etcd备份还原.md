---
title: cka真题练习——etcd备份还原
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——etcd备份还原
```
First,create a snapshot of the existing etcd instance running at https://127.0.0.1:2379, SAVING THE SNAPSHOT TO /var/lib/backup/etcd-snapshot.db.Creating a snapshot of the given instance is expected to complete in seconds. if the operation seems to hang,something's likely wrong with your command.Use CTRL + C to cancel the operation and try again.
Next, restore an existing,previous snapshot located at /data/backup/etcd-snapshot-previous.db。The following TLS certificates/key are supplied for connecting to the server with etcdctl:
CA certificate:
/opt/xxxxx/ca.crt
Client certificate:
/opt/xxxxx/etcd-client.crt
Client key:
/opt/xxxxx/etcd-client.key
```
首先把当前运行的etcd快照保存到指定路径；然后从指定路径的快照恢复到etcd

## 1. 首先从docker中拷贝一个etcdctl客户端
```sh
controlplane $ docker ps -a | grep etcd
2996113909c7        303ce5db0e90             "etcd --advertise-cl…"   13 minutes ago      Up 13 minutes                                   k8s_etcd_etcd-controlplane_kube-system_f55b1127bacf8b74e06aeec3179004c2_1
87a845c131b4        303ce5db0e90             "etcd --advertise-cl…"   13 minutes ago      Exited (1) 13 minutes ago                       k8s_etcd_etcd-controlplane_kube-system_f55b1127bacf8b74e06aeec3179004c2_0
d80be976101b        k8s.gcr.io/pause:3.2     "/pause"                 13 minutes ago      Up 13 minutes                                   k8s_POD_etcd-controlplane_kube-system_f55b1127bacf8b74e06aeec3179004c2_0

controlplane $ docker cp 2996113909c7:/usr/local/bin/etcdctl /usr/bin
```

## 2. 务必将etcdctl的api版本指定为v3
若不指定，默认使用v2的etcdctl版本
```
controlplane $ export ETCDCTL_API=3
```

## 3. 生成快照
```sh
controlplane $ etcdctl --endpoints='https://127.0.0.1:2379' \
> --cacert='/etc/kubernetes/pki/etcd/ca.crt' \
> --cert='/etc/kubernetes/pki/apiserver-etcd-client.crt' \
> --key='/etc/kubernetes/pki/apiserver-etcd-client.key' \
> snapshot save /etcd.snapshot.db
Snapshot saved at /etcd.snapshot.db
```

## 3. 查看快照状态
```sh
> etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt --key=/etc/kubernetes/pki/apiserver-etcd-client.key snapshot status /etcd.snapshot.db -wtable
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 8e02c2c0 |     2051 |       2062 |     2.8 MB |
+----------+----------+------------+------------+
```

## 4. 恢复快照
```sh
etcdctl --endpoints='https://127.0.0.1:2379' \
 --cacert='/etc/kubernetes/pki/etcd/ca.crt' \
 --cert='/etc/kubernetes/pki/apiserver-etcd-client.crt' \
 --key='/etc/kubernetes/pki/apiserver-etcd-client.key' snapshot restore /etcd.snapshot.db
```