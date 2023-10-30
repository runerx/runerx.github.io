---
title: Java漏洞回显
categories: Java安全
tags: 
  - Java
  - Java安全
---





## Java反序列化漏洞回显









### defineClass

先说defineClass这个东西是因为下面的几种方式都是在其基础上进行改进。defineClass归属于ClassLoader类，其主要作用就是使用编译好的字节码就可以定义一个类。

