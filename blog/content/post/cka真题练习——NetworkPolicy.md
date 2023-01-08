---
title: cka真题练习——NetworkPolicy
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——NetworkPolicy
## 在某个ns内部，网络通信只允许访问pod的9200端口
```
Create a new NetworkPolicy named allow-port-from-namespace that allows Pods in the existing namespace my-app to connect to port 9200 of other pods in the same namespace
```

# networkpolicy的配置解读
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

## spec.podSelector
这个配置指的是当前namespace所有需要被筛选的pod集合A；
例如，当配置的policyTypes为ingress时，即筛选所有进入到pod集合A的流量；
当配置的policyTypes为egress时，即筛选所有从pod集合A流出的流量
podSelector可以使用label来筛选，填{}代表筛选当前namespace（由metadata.namespace决定）的所有集合

## spec.policyTypes
这个配置即需要的策略，可以有Ingress、Egress。Ingress代表进入当前pod集合的流量，Egress代表离开当前pod集合的流量。pod集合由spec.podSelector决定。

## spec.ingress.from
有以下几种维度进行筛选
### namespaceSelector
只约束指定的namespace下的所有pod可以进入spec.podSelector的pod集合；
### podSelector
只约束指定的pod可以进入spec.podSelector的pod集合；若只配podSelector，不配namespaceSelector，则代表只有当前namespace下选定的pod可以访问当前namespace的spec.podSelector的pod集合
### namespaceSelector、podSelector
约束namespaceSelector筛选的namespace下的podSelector筛选的pod可以进入/离开spec.podSelector的pod集合

# 示例
## 只允许当前namespace的pod访问当前namespace的所有pod的80端口
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
```

- spec.podSelector为{}，代表筛选的集合为my-app namespace下的所有pod
- ingress.from.podSelector为{}，代表哪些pod可以访问spec.podSelector筛选出来的pod，并且没配namespaceSelector，所以即当前namespace下所有的pod。

## 只允许指定namespace的pod访问当前namespace的所有pod的80端口
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: default
    ports:
    - protocol: TCP
      port: 80
```
- spec.podSelector为{}，代表筛选的集合为my-app namespace下的所有pod
- ingress.from.namespaceSelector筛选了default namespace，代表哪些pod可以访问spec.podSelector筛选出来的pod，并且没配podSelector，所以即namespaceSelector筛选出来的namespace下所有的pod。
- 如何写namespaceSelector，可以通过以下命令查看namespace有哪些标签
```sh
[wangzijie.byte k8s]$ kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   56m   kubernetes.io/metadata.name=default
kube-node-lease   Active   56m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   56m   kubernetes.io/metadata.name=kube-public
kube-system       Active   56m   kubernetes.io/metadata.name=kube-system
my-app            Active   51m   kubernetes.io/metadata.name=my-app
```
根据以上结果写namespaceSelector即可
