---
title: k8s-pod资源限制
categories: 云原生安全
tags: 
- 云原生安全
- k8s
---



```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-deomo
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: "500m"
        memory: "2024Mi
      limits:
        cpu: "1000m"
        memory: "2048Mi"
```

资源单位

1个CPU = 1个超线程 = 1000毫核

K，M，G，T，P，E 以1000为换算标准

Ki，Mi，Gi，Ti，Pi，Ei 以1024为换算标准



如果只设置requests/limits都会自动设置相等。
