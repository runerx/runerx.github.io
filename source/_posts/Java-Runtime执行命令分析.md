---
title: Java Runtime执行命令分析
date: 2022-9-17 14:52:10
categories: Java安全
tags: 
- Java
- Java安全
---


前段时间测试的时候发现，通过白名单java进程使用Runtime系统调用进行命令执行可以绕过白名单进程监控。

<!-- more -->

## Runtime java代码

```Java
package main;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;


public class RuntimeTest {
    public static void main(String[] args) throws IOException {
        for(String s:args){
            System.out.println(s+"\t");
        }
        String command = args[0];

        Process process= Runtime.getRuntime().exec(command);
        InputStream fis = process.getInputStream();
        InputStreamReader isr = new InputStreamReader(fis);
        BufferedReader br = new BufferedReader(isr);
        String tmp = null;
        while ((tmp = br.readLine()) != null) {
            System.out.println(tmp);
        }
    }
}

```

将其打包成Jar。然后执行如下命令
```
java -jar runtime.jar "/tmp/fscan/fscan -h 192.168.1.1/16"
```

进程关系如下

```shell
[root@debian ~]# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
root      25002  21915  2 14:40 pts/13   00:00:00 java -jar runtime.jar /tmp/fscan/fscan -h 192.168.1.1/16
root      25012  25002 91 14:40 pts/13   00:00:05 /tmp/fscan/fscan -h 192.168.1.1/16
```

可以看见Runtime执行命令也是fork出一个子进程。
