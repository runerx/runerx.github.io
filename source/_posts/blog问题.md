---
title: blog问题
tags:
  - blog
  - hexo
categories: 其他
date: 2022-8-17 17:15:15
---




一些博客配置问题

<!-- more -->



## github更新大小写问题

```
[shadowflow@ShadowOS hexo]$ cd .deploy_git/.git
[shadowflow@ShadowOS hexo]$ vim config
```

改为如下

```
ignorecase = false
```

然后重新部署，过一会儿就ok。
