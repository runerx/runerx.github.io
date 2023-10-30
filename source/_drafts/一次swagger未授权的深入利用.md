---
title: 一次swagger未授权的深入利用
tags:
  - CTF
  - Web安全
categories: Web安全
---







访问页面如下

![](../images/pics/打靶/37.jpg)



搜索栏里的地址替换为互联网地址，成功访问

![](../images/pics/打靶/38.jpg)





### 列目录接口

通过....//绕过
```
http://ip:port/api/v1/files?dirname=....//
```



### 文件下载接口

```
http://ip:port/api/v1/file?filename=....//main
```

下载了main程序。



下载下来是go程序，





















































































