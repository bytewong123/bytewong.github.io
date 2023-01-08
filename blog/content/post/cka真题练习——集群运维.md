---
title: cka真题练习——集群运维
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——集群运维
## 检查有问题的node并恢复
```
A kubernetes woker node,named wk8s-node-0 is in state Not Ready. Investigate why this is the case,and
perform any appropriate steps to bring the node to a Ready state,ensuring that any changes are made
permanent.
You can ssh to the failed node using: 
ssh wk8s-node-0
You can assume elevated privileges on the node with the following command:  
sudo -i
```

### 答案
- 模拟集群节点异常的情况
```
[wangzijie.byte ~]$ kubectl get node
NAME           STATUS                     ROLES                  AGE   VERSION
minikube       Ready                      control-plane,master   38d   v1.22.2
minikube-m02   Ready                      <none>                 27d   v1.22.2
minikube-m03   Ready,SchedulingDisabled   <none>                 13d   v1.22.2
```

- 登录到minikube-m02节点上，将kubelet停掉
```sh
[wangzijie.byte ~]$ minikube ssh -n minikube-m02
docker@minikube-m02:~$ ps -ef | grep kubelet
root         923       1  2 Oct24 ?        13:20:41 /var/lib/minikube/binaries/v1.22.2/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=docker --hostname-override=minikube-m02 --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.3
docker    685085  685069  0 05:34 pts/1    00:00:00 grep --color=auto kubelet

docker@minikube-m02:~$ sudo systemctl stop kubelet

docker@minikube-m02:~$ ps -ef | grep kubelet
docker    685641  685069  0 05:36 pts/1    00:00:00 grep --color=auto kubelet
```

- 再次查看node状态，m02节点已经变为了NotReady
```
[wangzijie.byte ~]$ kubectl get node
NAME           STATUS                     ROLES                  AGE   VERSION
minikube       Ready                      control-plane,master   38d   v1.22.2
minikube-m02   NotReady                   <none>                 27d   v1.22.2
minikube-m03   Ready,SchedulingDisabled   <none>                 13d   v1.22.2
```

- 恢复m02节点，登录上去启动kubelet即可
```sh
[wangzijie.byte ~]$ minikube ssh -n minikube-m02
Last login: Sun Nov 21 05:36:23 2021 from 192.168.49.1

docker@minikube-m02:~$ sudo systemctl start kubelet

docker@minikube-m02:~$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sun 2021-11-21 05:46:11 UTC; 9s ago
       Docs: http://kubernetes.io/docs/
   Main PID: 685746 (kubelet)
      Tasks: 18 (limit: 18915)
     Memory: 62.3M
     CGroup: /docker/6aabb3889d8f5e2f1c009daee173dbc3f482a3ec1739f6a959af8f954358a4b0/system.slice/kubelet.service
             └─685746 /var/lib/minikube/binaries/v1.22.2/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/
kubelet/config.yaml --container-runtime=docker --hostname-override=minikube-m02 --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.
3
docker@minikube-m02:~$ ps -ef | grep kubelet
root      685746       1  4 05:46 ?        00:00:01 /var/lib/minikube/binaries/v1.22.2/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=docker --hostname-override=minikube-m02 --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.3
docker    686225  685735  0 05:46 pts/1    00:00:00 grep --color=auto kubelet
```

再次查看node状态，m02已变为Ready
```
[wangzijie.byte ~]$ kubectl get node
NAME           STATUS                     ROLES                  AGE   VERSION
minikube       Ready                      control-plane,master   38d   v1.22.2
minikube-m02   Ready                      <none>                 27d   v1.22.2
minikube-m03   Ready,SchedulingDisabled   <none>                 13d   v1.22.2
```

设置开机自启
```
systemctl enable kubelet 
```