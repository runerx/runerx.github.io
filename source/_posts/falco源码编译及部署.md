---
title: falco源码编译及部署
tags:
  - 云原生安全
categories: 云原生安全
date: 2022-05-30 09:39:15
---


源码编译falco以及falco内核模块，使其在debian或者alpine基础镜像中运行，alpine打包出来的镜像低于30M。官方的falco镜像差不多1G大小。

<!-- more -->

## 1. 编译安装



安装kernel module driver

```
#centos
yum -y install kernel-devel-$(uname -r)

#Debian
apt install linux-headers-$(uname -r)
```

运行构建镜像(这里使用宿主机的代理，方便下载国外的工具)

```
docker run --name falcobuild -itd -v /lib/modules:/lib/modules --network=host falcosecurity/falco-builder bash
#进入容器开启代理
docker exec -it falcobuild bash

export ALL_PROXY='socks5://172.16.42.1:1086'
export all_proxy=http://172.16.42.1:1087


```

主机上下载

```
git clone https://github.com/falcosecurity/falco.git
#复制到容器
docker cp falco falcobuild:/root/
```

容器里编译

```
mkdir build && cd build

#debian/centos容器
cmake -DUSE_BUNDLED_DEPS=true -DCREATE_TEST_TARGETS=OFF ../
#alpine容器
cmake -DUSE_BUNDLED_DEPS=true -DCREATE_TEST_TARGETS=OFF -DMUSL_OPTIMIZED_BUILD=On ../

make sinsp
make driver
make falco
make falco_engine
```

将容器里编译好的复制出来运行

```
docker cp falcobuild:/root/falco/build/driver /falco/alpine/falco/build/
docker cp falcobuild:/root/falco/build/userspace /falco/alpine/falco/build/


#rmmod falco
insmod driver/falco.ko
./userspace/falco/falco -c ../falco.yaml -r ../rules/falco_rules.yaml
```



参考：

- https://falco.org/docs/getting-started/source/

- https://github.com/falcosecurity/libs







## 2. docker构建

### 2.1 使用Debian

```
docker run --name myfalco -itd --privileged -v /boot:/boot:ro -v /var/run/docker.sock:/var/run/docker.sock -v /lib/modules:/lib/modules debian:buster
```



```
docker exec -it falco /bin/bash
mkdir -p /agent/falco/driver
mkdir -p /agent/falco/rules

#将编译好的复制到容器
docker cp /root/falco/build/userspace/falco/falco falco:/agent/falco
docker cp /root/falco/build/driver/falco.ko falco:/agent/falco/driver
docker cp /root/falco/falco.yaml falco:/agent/falco
docker cp /root/falco/rules/falco_rules.yaml falco:/agent/falco/rules
docker cp /root/falco/rules/falco_rules.local.yaml falco:/agent/falco/rules

##安装一些必要的软件
sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
sed -i 's|security.debian.org/debian-security|mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
apt update
apt --fix-broken install
apt install vim
apt install busybox
apt install kmod

#将webserver设为false
vim /agent/falco/falco.yaml

#加载内核模块
insmod /agent/falco/driver/falco.ko

#运行
/agent/falco/falco -c /agent/falco/falco.yaml -r /agent/falco/rules/
```

保存镜像

```
docker commit falco myfalco:v0.21
```

运行

```
docker run --rm --name myfalco --privileged --rm -it -v /boot:/boot:ro -v /var/run/docker.sock:/var/run/docker.sock -v /lib/modules:/lib/modules myfalco:v0.21

/agent/falco/falco -c /agent/falco/falco.yaml -r /agent/falco/rules
```



### 2.2 使用Alpine

```shell
docker run --name myfalco -itd --privileged -v /boot:/boot:ro -v /var/run/docker.sock:/var/run/docker.sock -v /lib/modules:/lib/modules alpine:latest

docker exec -it myfalco sh

#替换镜像源加速
sed -i "s/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g" /etc/apk/repositories

#安装必要工具
apk update
apk add --no-cache busybox
apk add --no-cache kmod
apk add --no-cache logger



#创建falco目录
mkdir -p /agent/falco/driver
mkdir -p /agent/falco/rules

#将宿主机falco文件移动到容器
docker cp /falco/alpine/falco/build/userspace/falco/falco myfalco:/agent/falco
docker cp /falco/alpine/falco/build/driver/falco.ko myfalco:/agent/falco/driver
docker cp /falco/alpine/falco/falco.yaml myfalco:/agent/falco
docker cp /falco/alpine/falco/rules/falco_rules.yaml myfalco:/agent/falco/rules
docker cp /falco/alpine/falco/rules/falco_rules.local.yaml myfalco:/agent/falco/rules

insmod /agent/falco/driver/falco.ko

/agent/falco/falco -c /agent/falco/falco.yaml -r /agent/falco/rules/

#测试抓取capabilites权限
docker run --name test --rm -it --cap-add=SYS_ADMIN --security-opt apparmor=unconfined --rm alpine sh


#commit
docker commit myfalco myfalco:v0.31

#通过alpine打包的只有29.6M,通过debian打包却要300多M
[root@node1 build]# docker images | grep myfalco
myfalco                       v0.31     07a980694db7   39 seconds ago   29.6MB

```





### 2.4 使用no-driver镜像

```
docker run --name myfalco --rm -itd \
    --privileged \
    -v /var/run/docker.sock:/host/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /lib/modules:/lib/modules \
    -v /proc:/host/proc:ro \
    falcosecurity/falco-no-driver:latest


mkdir -p /agent/falco/driver
mkdir -p /agent/falco/rules

#将编译好的复制到容器
docker cp /root/falco/build/userspace/falco/falco myfalco:/agent/falco
docker cp /root/falco/build/driver/falco.ko myfalco:/agent/falco/driver
docker cp /root/falco/falco.yaml myfalco:/agent/falco
docker cp /root/falco/rules/falco_rules.yaml myfalco:/agent/falco/rules
docker cp /root/falco/rules/falco_rules.local.yaml myfalco:/agent/falco/rules

##安装一些必要的软件
apt update
apt --fix-broken install
apt install busybox
apt install vim
apt install kmod

```



```
docker run --name myfalcox --rm -itd \
    --privileged \
    -v /var/run/docker.sock:/host/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /lib/modules:/host/lib/modules \
    -v /proc:/host/proc:ro \
    myfalco:v2.3.01
```



### 2.8 完整的falco镜像

```
docker run --rm --name myfalco -itd \
    --privileged \
    -v /var/run/docker.sock:/host/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    -v /etc:/host/etc:ro \
    falcosecurity/falco:latest
    
mkdir -p /agent/falco/driver
mkdir -p /agent/falco/rules

#将编译好的复制到容器
docker cp /root/falco/build/userspace/falco/falco myfalco:/agent/falco
docker cp /root/falco/build/driver/falco.ko myfalco:/agent/falco/driver
docker cp /root/falco/falco.yaml myfalco:/agent/falco
docker cp /root/falco/rules/falco_rules.yaml myfalco:/agent/falco/rules
docker cp /root/falco/rules/falco_rules.local.yaml myfalco:/agent/falco/rules



##安装一些必要的软件
sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
sed -i 's|security.debian.org/debian-security|mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
apt update
apt --fix-broken install
apt install vim
apt install busybox
apt install kmod


docker
```









### 2.7 Dockerfile打包



- vi Dockerfile

  ```dockerfile
  FROM myfalco:v0.31
  
  COPY ./docker-entrypoint.sh /
  
  ENTRYPOINT ["/docker-entrypoint.sh"]
  
  CMD ["/agent/falco/falco", "-c", "/agent/falco/falco.yaml", "-r", "/agent/falco/rules" ]
  ```

  

- vi docker-entrypoint.sh

  ```shell
  #!/usr/bin/env sh
  a=`lsmod | cut -d' ' -f1 | grep falco`
  b="falco"
  
  if [ "$a" == "$b" ]
  then
    rmmod falco
    insmod /agent/falco/driver/falco.ko
    echo "falco.ko exists, falco.ko has reloaded!"
  else
    insmod /agent/falco/driver/falco.ko
    echo "falco.ko not exits, falco.ko has loaded!"
  fi
  exec "$@"
  ```

  chmod +x docker-entrypoint.sh

```
 docker build -t myfalco:v1.31 .
```

测试运行

```
docker run --rm --name myfalcox -it --privileged -v /boot:/boot:ro -v /var/run/docker.sock:/var/run/docker.sock:ro -v /lib/modules:/lib/modules:ro myfalco:v1.31
```



参考：

- https://falco.org/docs/getting-started/running/





## 3. k8s部署

部署到node1节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falco
  labels:
    app: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      nodeSelector:
        kubernetes.io/hostname: node1
      containers:
        - name: falco
          image: myfalco:v1.31
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true
# Uncomment the 3 lines below to enable eBPF support for Falco.
# This allows Falco to run on Google COS.
# Leave blank for the default probe location, or set to the path
# of a precompiled probe.
#          env:
#          - name: SYSDIG_BPF_PROBE
#            value: ""
          args: [ "/agent/falco/falco", "-c", "/agent/falco/falco.yaml", "-r", "/agent/falco/rules", "--cri", "/run/containerd/containerd.sock", "-K", "/var/run/secrets/kubernetes.io/serviceaccount/token"]
          volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-socket
              readOnly: true
            - mountPath: /run/containerd/containerd.sock
              name: containerd-socket
              readOnly: true
            - mountPath: /boot
              name: boot-fs
              readOnly: true
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
      volumes:
        - name: docker-socket
          hostPath:
            path: /var/run/docker.sock
        - name: containerd-socket
          hostPath:
            path: /run/containerd/containerd.sock
        - name: boot-fs
          hostPath:
            path: /boot
        - name: lib-modules
          hostPath:
            path: /lib/modules
```









