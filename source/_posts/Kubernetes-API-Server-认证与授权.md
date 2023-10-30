---
title: Kubernetes API Server 认证与授权
date: 2022-01-04 16:11:53
categories: 云原生安全
tags:
- Kubernetes
---

> 凡所有相，皆是虚妄。若见诸相非相，即见如来。

了解Kubernetes API Server的认证与授权能让我们更清楚的了解Kubernetes的安全机制

<!-- more -->

## 1. Bootstrap Tokens认证

启动引导令牌是一种简单的持有者令牌（Bearer Token），这种令牌是在新建集群 或者在现有集群中添加新节点时使用的。 

启动引导令牌使用 `abcdef.0123456789abcdef` 的形式。 更加规范地说，它们必须符合正则表达式 `[a-z0-9]{6}\.[a-z0-9]{16}`。令牌的第一部分是 “Token ID”，它是一种公开信息，用于引用令牌并确保不会 泄露认证所使用的秘密信息。 第二部分是“令牌秘密（Token Secret）”，它应该被共享给受信的第三方。

启动引导令牌认证组件可以通过 API 服务器上的如下标志启用：

```
--enable-bootstrap-token-auth
```

我们可以在kube-apiserver.yaml文件中查看是否启用。

```yaml
[root@master manifests]# cat /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
......
spec:
  containers:
  - command:
	.......
    - --enable-bootstrap-token-auth=true
	.....
  
```

使用kubeadm查看Bearer Token

```bash
[root@master manifests]# kubeadm token create --print-join-command
kubeadm join 172.16.42.100:6443 --token j37jxe.gjv3g740s4gw045p --discovery-token-ca-cert-hash sha256:7e243bc69c1465069c54fd05d7c09c16d7150b09be2598b34f7fae9d60ebc68f
```

通过token访问api server(使用静态令牌访问需要添加Bearer)

```bash
curl -k --header "Authorization: Bearer fifuvj.jjqcsc3cmptk08aw" https://172.16.42.100:6443/api
```





## 2. 服务账号令牌

服务账号令牌是Kubernetes默认启用的用户认证机制，

通过如下命令查看token

```bash
[root@master manifests]# kubectl describe secrets
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
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjVlYkx2aWRndzBVbm42VHYzQmNXaXdjdkl4cGZyWDFFd3l0aFBJY3pxNEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tN3h0czQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjcyMDA5NmMzLWMwZWEtNGYwMy04ODljLTZkNDUwYWI2MmQyYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.sSEmxeJ2XIwb95t50ehRGvTUEOOQjjdTK5ula6AtgPV4z7m7YzpPKDSeQSauLWul272j9SLYyuiWR6rnoC4NvAgxgsAbapraJcyDjv0DadWUfbTcjLUSNGGaSQMuOyrYPA_ahK6UuFS_kAOiXL71gPLjE6xUcm8T4-aAgszUSG78OrMDmDao9ZoS2iD0TOVWNsxfUFw712meB4t2C5cj5DhfrDKdGLlJ4HRs3E0EYyPkX8NizaeSjifT-_AxFL7_KrL6GM4nMM8AEWoSn3v1N4CyQkvU5Bb2xu4hyZGX6OAWkcFNoMC4jEfeIgWbj6MC0elhuGYVznyr65GKtcjG
```

同样的方法访问api server

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



**创建一个 server account:**

```bash
#创建账号
[root@master kubernetes]# kubectl create serviceaccount jenkins
serviceaccount/jenkins created

#查看账号信息
[root@master kubernetes]# kubectl get serviceaccounts jenkins -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-01-04T08:20:23Z"
  name: jenkins
  namespace: default
  resourceVersion: "198784"
  uid: f3a618fa-07b5-42ed-af2e-17d1330d8dcd
secrets:
- name: jenkins-token-c2fll


#查看该账号的token
[root@master kubernetes]# kubectl get secret jenkins-token-c2fll -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1USXlOekEyTWpRMU1Gb1hEVE14TVRJeU5UQTJNalExTUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS2xZCnJCTnF4S29FcGxIVVJFaHRGSkJENFRUNlRNamwyNFdrRGNObnhocnVwSVh3dXJJMzQxVi9ZN2hoRzBaSTFjeXAKcnFjVUNLMUlGQk1ZektLSnNrVE5nWG16WkZJUkFyNUtkSlI5RFQwWUt6MzY3dnUvYUsrU3ArUVRRQktQQ2ZORgpRd3VaZXkyYXAxRG5Ba1pQTWVZYlVzTHgwa1ljU2lBMitEbHNYdHdPRUtUSkNYVThKRHE1YXRPcVUxTXhPZVI0ClJqRFBLY0k3djBsV2xtdzFFSWlkTFA3aThFelhGcTVBUzA4R2gzZ2hSYyt2UHV6MWNhV29hOFBlVXBCWjZ0a0wKMU1aZlZvZVAvT1hmWVJDaURIY21LK05FSU0yWXFmVGI2NjlyODZyOFAzWnVTSko2a0pQdmd0YzdNZys2eWZiWgpzbGtrUzV4Q1YweHhwVkxLbTkwQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZFemhQNXFNdG52Q2tzUmR6ZkkyRVpleHkzTjRNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFCQmhqY1FFN0RFTFN6dkJFUDQ4c0pQVGNXUUVFT1dWT1lGUk9mUWNmNEk3VDhCY3BhRQowZXpGWi9HaXI4TXQwMFJFTHVGL3FpUElZR3BXOTYvMm5LK3RsQVdpRHBGOWhkTkRZdXlQWWpyd1V2TXl2eE5XCnViM2NaY1FmcGdab0FDWVZmUURJWXVMUGFDaDBNRmdzcnR4NmFDUWw5S3h4MnNIQ1lKcE9aZzRjOUtQNi9LSk0KTFhBT0tJeHpwbjg0cXR1N3ZLK0EwY2ZCd29VQUJwcGFqZlVHcVRocFlYLzhmSm51M3RoY01mMXN5NnVHeTdtMgp1bGNHWnpTdm9UME5ycGJ2RzR5ckZpcDRQYXdWRTgxYWI5VlhjN3pJUmJwM3B1YmlCalR1elJhZ1hQdUY4cmV0CkUyRVNXVGlESDZLcXVPU1UwNEt6dWJhbDNFaGVPQXhGekcySgotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklqVmxZa3gyYVdSbmR6QlZibTQyVkhZelFtTlhhWGRqZGtsNGNHWnlXREZGZDNsMGFGQkpZM3B4TkVraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbXBsYm10cGJuTXRkRzlyWlc0dFl6Sm1iR3dpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pYW1WdWEybHVjeUlzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJbVl6WVRZeE9HWmhMVEEzWWpVdE5ESmxaQzFoWmpKbExURTNaREV6TXpCa09HUmpaQ0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT21wbGJtdHBibk1pZlEuS3BWMlJ6M0RoWmRpQjJaNk9aWVVwVG9xOFJJc3VQWjFGbkp5RGpqSmluTjNkbHZxdl8tWUpob05wZm9vNkszUnRYNWRYczFkRnplNjZmYlJxbUJibHBGRVh0bXBtazNOM1hYZEFsdlhPWUpYRjU0b3JJTmliTncxUEZDcUZGY0k3ZFFEVWNpQkp2ZjJXUkhFZkZIb3JnUHNlOGFnbEpMdlpzMzdWWW1IZFJKWWQwMThhZW1wajhqZE5WdTNsWXJSQWoyMTNlQk1yemhxdHRyd2RNeVZFejBoVklBdE9MUE5QZ3pwRXFVSUp0Qm5rclVOb1BRNGFXUVlSWDM1eEdLTENGQUJNY3RmdEhHTE0zQlozby1yamx5X2tsdC1jcUdPcXdZSG0zc2ZzOXpYOG85Mm9BRkdDME9xeUQ1T2xLNmNoajNTbjd6UXRZTGhXaXI4R1dfNWhn
	......

#删除该账号
[root@master kubernetes]# kubectl delete serviceaccount jenkins
serviceaccount "jenkins" deleted
```



## 3. X.509客户端

X.509客户端证书也可称为HTTPS证书认证，是基于CA根证书签名的双向数字证书认证方式，默认情况下Kubernetes开启此参数配置。

与X.509客户端证书相关的三个kube-apiserver启动参数：

1）client-ca-file: 指定CA根证书文件，内置CA公钥

2）tls-private-key-file：API Server私钥文件位置

3）tls-cert-file：API Server证书文件

```
[root@master manifests]# cat /etc/kubernetes/manifests/kube-apiserver.yaml
......
spec:
  containers:
  - command:
	........
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
	......
```

集群内部都是通过HTTPS证书互相访问，如果是集群外未配置证书的主机访问集群就会提示未授权访问。





## 4. OpenID Connect（OIDC）令牌

[OpenID Connect](https://openid.net/connect/) 是一种 OAuth2 认证方式， 被某些 OAuth2 提供者支持，例如 Azure 活动目录、Salesforce 和 Google。 协议对 OAuth2 的主要扩充体现在有一个附加字段会和访问令牌一起返回， 这一字段称作 [ID Token（ID 令牌）](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)。 ID 令牌是一种由服务器签名的 JSON Web 令牌（JWT），其中包含一些可预知的字段， 例如用户的邮箱地址，

要识别用户，身份认证组件使用 OAuth2 [令牌响应](https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse) 中的 `id_token`（而非 `access_token`）作为持有者令牌。 

![](../images/pic/k8s12.jpg)







## 5. Webhook 令牌身份认证 

Webhook 身份认证是一种用来验证持有者令牌的回调机制。

- `--authentication-token-webhook-config-file` 指向一个配置文件，其中描述 如何访问远程的 Webhook 服务。
- `--authentication-token-webhook-cache-ttl` 用来设定身份认证决定的缓存时间。 默认时长为 2 分钟。

下面是一个例子：

```yaml
# Kubernetes API 版本
apiVersion: v1
# API 对象类别
kind: Config
# clusters 指代远程服务
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # 用来验证远程服务的 CA
      server: https://authn.example.com/authenticate # 要查询的远程服务 URL。必须使用 'https'。

# users 指代 API 服务的 Webhook 配置
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # Webhook 插件要使用的证书
      client-key: /path/to/key.pem          # 与证书匹配的密钥

# kubeconfig 文件需要一个上下文（Context），此上下文用于本 API 服务器
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-sever
  name: webhook
```





## 6. 身份认证代理

API 服务器可以配置成从请求的头部字段值（如 `X-Remote-User`）中辩识用户。 这一设计是用来与某身份认证代理一起使用 API 服务器，代理负责设置请求的头部字段值。

- `--requestheader-username-headers` 必需字段，大小写不敏感。用来设置要获得用户身份所要检查的头部字段名称列表（有序）。第一个包含数值的字段会被用来提取用户名。
- `--requestheader-group-headers` 可选字段，在 Kubernetes 1.6 版本以后支持，大小写不敏感。 建议设置为 "X-Remote-Group"。用来指定一组头部字段名称列表，以供检查用户所属的组名称。 所找到的全部头部字段的取值都会被用作用户组名。
- `--requestheader-extra-headers-prefix` 可选字段，在 Kubernetes 1.6 版本以后支持，大小写不敏感。 建议设置为 "X-Remote-Extra-"。用来设置一个头部字段的前缀字符串，API 服务器会基于所给 前缀来查找与用户有关的一些额外信息。这些额外信息通常用于所配置的鉴权插件。 API 服务器会将与所给前缀匹配的头部字段过滤出来，去掉其前缀部分，将剩余部分 转换为小写字符串并在必要时执行[百分号解码](https://tools.ietf.org/html/rfc3986#section-2.1) 后，构造新的附加信息字段键名。原来的头部字段值直接作为附加信息字段的值。





## 7. API Server授权

Kubernetes包含四类授权模式：

- 节点(Node)授权
- 基于属性的访问控制（Attribute-Based Access Control, ABAC)
-  基于角色的访问控制（Role-Based Access Control, RBAC）
- 基于钩子（Webhook）方式的授权

目前使用RBAC机制较多。

RBAC策略包含以下核心概念。

1）Resource: 指Kubernetes中的资源，如Pod、Service等。

2）Role: 对Resource执行的操作，如对Pod执行create、update、delete等操作。

3）Entity: 代表一个应用程序，可以是一个用户、组或服务账户

4）Role Binding: 将Role绑定到Entity，表明在指定Resource上运行某Entity并执行一组操作。

就Role和Role Binding而言，Kubernetes定义了两种范围类型：

1）集群范围：Cluster Role和Cluster Role Binding

2）命名空间范围：Role和Role Binding



**常用的命令：**

1. 查看所有命名空间的Cluster Role:

   ```bash
   [root@master ~]# kubectl get clusterroles --all-namespaces
   NAME                                                                   CREATED AT
   admin                                                                  2021-12-27T06:26:04Z
   calico-kube-controllers                                                2021-12-27T06:30:23Z
   calico-node                                                            2021-12-27T06:30:24Z
   cluster-admin                                                          2021-12-27T06:26:04Z
   edit                                                                   2021-12-27T06:26:04Z
   kubeadm:get-nodes                                                      2021-12-27T06:26:11Z
   kubernetes-dashboard                                                   2022-01-04T04:32:05Z
   system:aggregate-to-admin                                              2021-12-27T06:26:04Z
   .....
   ```

   

2. 查看所有命名空间的Role：

   ```bash
   [root@master ~]# kubectl get roles --all-namespaces
   NAMESPACE              NAME                                             CREATED AT
   kube-public            kubeadm:bootstrap-signer-clusterinfo             2021-12-27T06:26:11Z
   kube-public            system:controller:bootstrap-signer               2021-12-27T06:26:04Z
   kube-system            extension-apiserver-authentication-reader        2021-12-27T06:26:04Z
   kube-system            kube-proxy                                       2021-12-27T06:26:12Z
   kube-system            kubeadm:kubelet-config-1.21                      2021-12-27T06:26:05Z
   kube-system            kubeadm:nodes-kubeadm-config                     2021-12-27T06:26:05Z
   kube-system            system::leader-locking-kube-controller-manager   2021-12-27T06:26:0
   ......
   ```

   

3. 查看预定义的Cluster Role对资源的访问权限

   ```bash
   [root@master ~]# kubectl describe clusterrole view
   Name:         view
   Labels:       kubernetes.io/bootstrapping=rbac-defaults
                 rbac.authorization.k8s.io/aggregate-to-edit=true
   Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
   PolicyRule:
     Resources                                    Non-Resource URLs  Resource Names  Verbs
     ---------                                    -----------------  --------------  -----
     bindings                                     []                 []              [get list watch]
     configmaps                                   []                 []              [get list watch]
     ......
   ```



**案例：创建一个用例来处理特殊的资源**

1）为应用服务程序建立服务账户资源：

```bash
[root@master ~]# kubectl create namespace coolapp
namespace/coolapp created
[root@master ~]# kubectl --namespace=coolapp create serviceaccount myappid
serviceaccount/myappid created
```

2）创建role，该Role只能在coolapp命名空间中查看和列出Pod。

```bash
[root@master ~]# kubectl --namespace=coolapp create role podview --verb=get --verb=list --resource=pods
role.rbac.authorization.k8s.io/podview created
[root@master ~]# kubectl --namespace=coolapp describe role/podview
Name:         podview
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list
```

3）创建Role Binding，将Role "podview"绑定至名为myappid的应用程序中。

```bash
[root@master ~]# kubectl --namespace=coolapp create rolebinding mypodviewer --role=podview --serviceaccount=coolapp:myappid
rolebinding.rbac.authorization.k8s.io/mypodviewer created
[root@master ~]# kubectl --namespace=coolapp describe rolebinding/mypodviewer
Name:         mypodviewer
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  podview
Subjects:
  Kind            Name     Namespace
  ----            ----     ---------
  ServiceAccount  myappid  coolapp
```

通过以上部分完成了RBAC的创建，下面我们可以通过Kubectl验证myappid服务账户是否拥有对Pod资源的list权限。

```bash
[root@master ~]# kubectl --namespace=coolapp auth can-i --as=system:serviceaccount:coolapp:myappid list pods
yes
```

再验证myappid服务账户是否拥有对Service资源的list权限：

```bash
[root@master ~]# kubectl --namespace=coolapp auth can-i --as=system:serviceaccount:coolapp:myappid list services
no
```



## 8. 准入控制器

当用户请求通过了API Server认证和授权后，便进入了准入控制器环节，相比于前面的API Server 的认证与授权机制，准入控制器是更为细粒度的资源控制机制，其支持Kubernetes的许多高级功能，如Pod安全策略（Pod Security Policy）、安全上下文（Security Context）、服务账户（Service Account）等。

准入控制器主要分为两个阶段，第一个阶段运行变更准入控制器，第二个阶段运行验证准入控制器。变更准入控制器和验证准入控制器都是通过Webhook的方式进行实现

### 8.1 Pod安全策略

Pod安全策略是集群级别的资源，主要在Pod的创建和更新阶段提供细粒度的权限控制，其在Kubernetes中被定义为一个准入控制器，集群管理员可通过kube-apiserver配置文件中指定"--enable-admission-plugins=NodeRestriction, PodSecurityPolicy"来完成启动。

Pod安全策略资源定义了一组Pod运行时必须遵守的条件及相关字段的默认值，只有Pod满足这些条件才会被Kubernetes接受。此外，Pod安全策略定义完成后，需要使用RBAC对其授权才能正常使用。

**案例演示：**

给api server添加psp权限

```
vim /etc/kubernetes/manifests/kube-apiserver.yaml 
```

![](../images/pic/k8s13.jpg) 

为运行此示例，配置一个名字空间和一个服务账号。我们将用这个服务账号来 模拟一个非管理员账号的用户。

```
kubectl create namespace psp-example
kubectl create serviceaccount -n psp-example fake-user
kubectl create rolebinding -n psp-example fake-editor --clusterrole=edit --serviceaccount=psp-example:fake-user
```

创建两个别名，以更清晰地展示我们所使用的用户账号，同时减少一些键盘输入：

```
alias kubectl-admin='kubectl -n psp-example'
alias kubectl-user='kubectl --as=system:serviceaccount:psp-example:fake-user -n psp-example'
```

example-psp.yaml:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

使用 kubectl 执行创建操作：

```
kubectl-admin create -f example-psp.yaml
```

现在，作为一个非特权用户，尝试创建一个简单的 Pod：

```
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pause
spec:
  containers:
    - name: pause
      image: k8s.gcr.io/pause
EOF
```

结果：

```
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: PodSecurityPolicy: unable to admit pod: []
```

 尽管 PodSecurityPolicy 被创建，Pod 的服务账号或者 `fake-user` 用户都没有使用该策略的权限。



























## 9. 参考

https://kubernetes.io/

《云原生攻防实践与体系构建》
