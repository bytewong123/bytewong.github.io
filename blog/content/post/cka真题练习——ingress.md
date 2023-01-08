---
title: cka真题练习——ingress
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# cka真题练习——ingress
```
Create a new nginx Ingress resource as follows:
Name: ping
Namespace: ing-internal  Exposing service hello on path /hello using service port 80 
The availability of service hello can be checked using the following command,which should return hello: curl -kl <INTERNAL_IP>/hello
```
## 答案
### 创建ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ping
  namespace: ing-internal
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 80

```