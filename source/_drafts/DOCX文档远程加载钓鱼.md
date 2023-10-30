---
title: DOCX文档远程加载钓鱼
categories: 红队技术
tags:
  - 社工
  - 红队
---



通过远程模板的方式，我们可以使得邮件中发送的docx文档不直接带宏，能规避静态检测，再通过混淆处理来规避邮件沙箱检测。

<!-- more -->



## 1. 宏制作

我们需要创建一个支持宏的Word模板（.dotm文件扩展名）。打开Word并勾选功能区上的Developer选项卡使其可见：















## 参考

- https://xz.aliyun.com/t/2496
- https://turn1tup.github.io/2021/10/20/word%E8%BF%9C%E7%A8%8B%E6%A8%A1%E6%9D%BF%E6%96%87%E6%A1%A3%E6%B7%B7%E6%B7%86/#
- https://turn1tup.github.io/2021/06/16/word%E5%AE%8F%E9%82%AE%E4%BB%B6%E9%92%93%E9%B1%BC-%E8%BF%9C%E7%A8%8B%E5%8A%A0%E8%BD%BD%E6%A8%A1%E6%9D%BF/
