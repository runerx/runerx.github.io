---
title: Sysdig and Falco now powered by eBPF 翻译
tags:
---

[原文地址](https://sysdig.com/blog/sysdig-and-falco-now-powered-by-ebpf/)

过去的日子里，我们将一项linux内核的核心技术——eBPF应用到了Sysdig的agent中。现在，在Sysdig的发行版中，eBPF是一个可选的技术选项。今天，我们分享我们整合eBPF内部工作的更多细节。为了祝贺这个激动人心的技术，我们发布了一些列文章。在这篇博客里，我们以更高的视角审视eBPF，接下来我们将介绍基于eBPF的Sysdig的实现。





## 关于eBPF的介绍

什么是eBPF? Extended Berkeley Packet Filter (eBPF) 是一项现代的，强大的技术，广泛应用于linux内核相关的应用，包括网络和追踪。核心就是，eBPF允许用户给内核注入几乎通用的代码。这些代码将会在某些时间点运行，通常是某些内核事件，这些事件包括但不限于网络和追踪。

我们这里不会花太多时间描述eBPF技术本身，该技术目前已经有很多文档，下面有一些比较好的入门材料

- [A Thorough Introduction to eBPF](https://lwn.net/Articles/740157/)
- [BPF and XDP Reference Guide](https://cilium.readthedocs.io/en/latest/bpf/)
- [Linux Extended BPF (eBPF) Tracing Tools](http://www.brendangregg.com/ebpf.html)



### 使用eBPF的目的

在上面已经提到，使用eBPF的最终目的是让用户的代码在内核执行。 这听起来跟传统的内核扩展很相似[loadable kernel modules](https://en.wikipedia.org/wiki/Loadable_kernel_module)。可加载的内核模块是通过insmod/modprobe命令将编译后的c代码加载到运行的内核中，这些代码通常用户hook内核子系统，当时间发生的时候就会自动调用代码。这对于追踪函数和一些新硬件设备非常有用。事实上，真相并如此。eBPF最大的优势是，不像内核模块，它将只运行被认为是完全安全的代码。这意味着，它绝不会引起内核崩溃或者内核不稳定。这是eBPF最大的卖点。



### 编写eBPF程序

为了实现所说的安全保证，eBPF本质上是一个进程虚拟机。这个虚拟机运行以用户的名义运行安全程序。eBPF给用户暴露一个虚拟处理器，































