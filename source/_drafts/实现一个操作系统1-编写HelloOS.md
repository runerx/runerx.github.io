---
title: 写一个操作系统
categories: 计算机底层
tags: 
- 计算机底层
- 操作系统
---

实现一个输出Hello的操作系统Demo

<!-- more -->

本文主要内容来自LMOS的极客时间专栏《操作系统实战45讲》

## 1. 程序的运行过程

### 1.1 程序编译过程 

- HelloWorld.c

  ```c
  #include "stdio.h"
  int main(int argc, char const *argv[])
  {
    printf("Hello World!\n");
    return 0;
  }
  ```

gcc HelloWorld.c -o HelloWorld 或者 gcc ./HelloWorld.c -o ./HelloWorld ，就可以编译这段代码。其实，GCC 只是完成编译工作的驱动程序，它会根据编译流程分别调用预处理程序、编译程序、汇编程序、链接程序来完成具体工作。

编译上述代码的过程

![](../images/pics/计算机底层/1.jpg) 

其实，我们也可以手动控制以上这个编译流程，从而留下中间文件方便研究：

- gcc HelloWorld.c -E -o HelloWorld.i 预处理：加入头文件，替换宏。
- gcc HelloWorld.c -S -c -o HelloWorld.s 编译：包含预处理，将 C 程序转换成汇编程序。
- gcc HelloWorld.c -c -o HelloWorld.o 汇编：包含预处理和编译，将汇编程序转换成可链接的二进制程序。
- gcc HelloWorld.c -o HelloWorld 链接：包含以上所有操作，将可链接的二进制程序和其它别的库链接在一起，形成可执行的程序文件。



### 1.2 程序装载执行

编译好的程序如何执行呢呢? 图灵提出的图灵机。

图灵机是一个抽象的模型，它是这样的：有一条无限长的纸带，纸带上有无限个小格子，小格子中写有相关的信息，纸带上有一个读头，读头能根据纸带小格子里的信息做相关的操作并能来回移动。

下面用图灵机执行一下“1+1=2”的计算。我们定义读头读到“+”之后，就依次移动读头两次并读取格子中的数据，最后读头计算把结果写入第二个数据的下一个格子里，整个过程如下图：

![](../images/pics/计算机底层/2.jpg) 

这个理想的模型是好，但是理想终归是理想，想要成为现实，我们得想其它办法。冯诺依曼提出，电子计算机使用二进制数制系统和储存程序，并按照程序顺序执行，他的电子计算机理论叫冯诺依曼体系结构。

根据冯诺依曼体系结构构成的计算机，必须具有如下功能：

- 把程序和数据装入到计算机中；
- 必须具有长期记住程序、数据的中间结果及最终运算结果；
- 完成各种算术、逻辑运算和数据传送等数据加工处理；
- 根据需要控制程序走向，并能根据指令控制机器的各部件协调操作；
- 能够按照要求将处理的数据结果显示给用户。

为了完成上述的功能，计算机必须具备五大基本组成部件：

- 装载数据和程序的输入设备；

- 记住程序和数据的存储器；

- 完成数据加工处理的运算器；

- 控制程序执行的控制器；

- 显示处理结果的输出设备。

根据冯诺依曼的理论，我们只要把图灵机的几个部件换成电子设备，就可以变成一个最小核心的电子计算机，如下图：

![](../images/pics/计算机底层/3.jpg) 

这次我们发现读头不再来回移动了，而是靠地址总线寻找对应的“纸带格子”。读取写入数据由数据总线完成，而动作的控制就是控制总线的职责了。



### 1.3 将 HelloWorld 程序装入原型计算机

我们可以通过 gcc -c -S HelloWorld 得到（只能得到其汇编代码，而不能得到二进制数据）。我们用 objdump -d HelloWorld ，其中有很多库代码（只需关注 main 函数相关的代码），如下图：

```
[root@debian /tmp]# objdump -d HelloWorld

HelloWorld：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
   f:	48 8d 3d 00 00 00 00 	lea    0x0(%rip),%rdi        # 16 <main+0x16>
  16:	e8 00 00 00 00       	callq  1b <main+0x1b>
  1b:	b8 00 00 00 00       	mov    $0x0,%eax
  20:	c9                   	leaveq
  21:	c3                   	retq
```

上述结果分成四列：第一列为地址；第二列为十六进制，表示真正装入机器中的代码数据；第三列是对应的汇编代码；第四列是相关代码的注释。这是 x86_64 体系的代码，由此可以看出 x86 CPU 是变长指令集。

![](../images/pics/计算机底层/4.jpg) 

现代电子计算机正是通过内存中的信息（指令和数据）做出相应的操作，并通过内存地址的变化，达到程序读取数据，控制程序流程（顺序、跳转对应该图灵机的读头来回移动）的功能。

这和图灵机的核心思想相比，没有根本性的变化。只要配合一些 I/O 设备，让用户输入并显示计算结果给用户，就是一台现代意义的电子计算机。



## 2. 基于硬件的Hello OS

### 2.1 PC 机的引导流程

写操作系统要用汇编和 C 语言，尽管这个 Hello OS 很小，但也要用到两种编程语言。其实，现有的商业操作系统都是用这两种语言开发出来的。我们也不打算从 PC 的引导程序开始写起，原因是目前我们的知识储备还不够，所以先借用一下 GRUB 引导程序，只要我们的 PC 机上安装了 Ubuntu Linux 操作系统，GRUB 就已经存在了。这会大大降低我们开始的难度，也不至于打消你的热情。

Hello OS引导流程：

![](../images/pics/计算机底层/5.jpg) 

简单解释一下，PC 机 BIOS 固件是固化在 PC 机主板上的 ROM 芯片中的，掉电也能保存，PC 机上电后的第一条指令就是 BIOS 固件中的，它负责检测和初始化 CPU、内存及主板平台，然后加载引导设备（大概率是硬盘）中的第一个扇区数据，到 0x7c00 地址开始的内存空间，再接着跳转到 0x7c00 处执行指令，在我们这里的情况下就是 GRUB 引导程序。

更先进的UEFI BIOS则不同，这里就不深入其中了，你可以通过链接自行了解。

#### 1.4.2 Hello OS 引导汇编代码

明白了 PC 机的启动流程，下面只剩下我们的 Hello OS 了，我们马上就去写好它。我们先来写一段汇编代码。这里我要特别说明一个问题：为什么不能直接用 C？C 作为通用的高级语言，不能直接操作特定的硬件，而且 C 语言的函数调用、函数传参，都需要用栈。栈简单来说就是一块内存空间，其中数据满足后进先出的特性，它由 CPU 特定的栈寄存器指向，所以我们要先用汇编代码处理好这些 C 语言的工作环境。

```

;彭东 @ 2021.01.09
MBT_HDR_FLAGS EQU 0x00010003
MBT_HDR_MAGIC EQU 0x1BADB002 ;多引导协议头魔数
MBT_HDR2_MAGIC EQU 0xe85250d6 ;第二版多引导协议头魔数
global _start ;导出_start符号
extern main ;导入外部的main函数符号
[section .start.text] ;定义.start.text代码节
[bits 32] ;汇编成32位代码
_start:
jmp _entry
ALIGN 8
mbt_hdr:
dd MBT_HDR_MAGIC
dd MBT_HDR_FLAGS
dd -(MBT_HDR_MAGIC+MBT_HDR_FLAGS)
dd mbt_hdr
dd _start
dd 0
dd 0
dd _entry
;以上是GRUB所需要的头
ALIGN 8
mbt2_hdr:
DD MBT_HDR2_MAGIC
DD 0
DD mbt2_hdr_end - mbt2_hdr
DD -(MBT_HDR2_MAGIC + 0 + (mbt2_hdr_end - mbt2_hdr))
DW 2, 0
DD 24
DD mbt2_hdr
DD _start
DD 0
DD 0
DW 3, 0
DD 12
DD _entry
DD 0
DW 0, 0
DD 8
mbt2_hdr_end:
;以上是GRUB2所需要的头
;包含两个头是为了同时兼容GRUB、GRUB2
ALIGN 8
_entry:
;关中断
cli
;关不可屏蔽中断
in al, 0x70
or al, 0x80
out 0x70,al
;重新加载GDT
lgdt [GDT_PTR]
jmp dword 0x8 :_32bits_mode
_32bits_mode:
;下面初始化C语言可能会用到的寄存器
mov ax, 0x10
mov ds, ax
mov ss, ax
mov es, ax
mov fs, ax
mov gs, ax
xor eax,eax
xor ebx,ebx
xor ecx,ecx
xor edx,edx
xor edi,edi
xor esi,esi
xor ebp,ebp
xor esp,esp
;初始化栈，C语言需要栈才能工作
mov esp,0x9000
;调用C语言函数main
call main
;让CPU停止执行指令
halt_step:
halt
jmp halt_step
GDT_START:
knull_dsc: dq 0
kcode_dsc: dq 0x00cf9e000000ffff
kdata_dsc: dq 0x00cf92000000ffff
k16cd_dsc: dq 0x00009e000000ffff
k16da_dsc: dq 0x000092000000ffff
GDT_END:
GDT_PTR:
GDTLEN dw GDT_END-GDT_START-1
GDTBASE dd GDT_START
```

以上的汇编代码分为 4 个部分：

1. 代码 1~40 行，用汇编定义的 GRUB 的多引导协议头，其实就是一定格式的数据，我们的 Hello OS 是用 GRUB 引导的，当然要遵循 GRUB 的多引导协议标准，让 GRUB 能识别我们的 Hello OS。之所以有两个引导头，是为了兼容 GRUB1 和 GRUB2。

2. 代码 44~52 行，关掉中断，设定 CPU 的工作模式。你现在可能不懂，没事儿，后面 CPU 相关的课程我们会专门再研究它。
3. 代码 54~73 行，初始化 CPU 的寄存器和 C 语言的运行环境。
4. 代码 78~87 行，GDT_START 开始的，是 CPU 工作模式所需要的数据，同样，后面讲 CPU 时会专门介绍。



#### 1.4.2 Hello OS 的主函数

到这，不知道你有没有发现一个问题? 上面的汇编代码调用了 main 函数，而在其代码中并没有看到其函数体，而是从外部引入了一个符号。

那是因为这个函数是用 C 语言写的在（/lesson02/HelloOS/main.c）中，最终它们分别由 nasm 和 GCC 编译成可链接模块，由 LD 链接器链接在一起，形成可执行的程序文件：

```c

//彭东 @ 2021.01.09
#include "vgastr.h"
void main()
{
  printf("Hello OS!");
  return;
} 
```

以上这段代码，你应该很熟悉了吧？不过这不是应用程序的 main 函数，而是 Hello OS 的 main 函数。其中的 printf 也不是应用程序库中的那个 printf 了，而是需要我们自己实现了。你可以先停下歇歇，再去实现 printf 函数。

#### 1.4.3 控制计算机屏幕

接着我们再看下显卡，这和我们接下来要写的代码有直接关联。

计算机屏幕显示往往是显卡的输出，显卡有很多形式：集成在主板的叫集显，做在 CPU 芯片内的叫核显，独立存在通过 PCIE 接口连接的叫独显，性能依次上升，价格也是。

独显的高性能是游戏玩家们所钟爱的，3D 图形显示往往要涉及顶点处理、多边形的生成和变换、纹理、着色、打光、栅格化等。而这些任务的计算量超级大，所以独显往往有自己的 RAM、多达几百个运算核心的处理器。因此独显不仅仅是可以显示图像，而且可以执行大规模并行计算，比如“挖矿”。

我们要在屏幕上显示字符，就要编程操作显卡。

其实无论我们 PC 上是什么显卡，它们都支持一种叫 VESA 的标准，这种标准下有两种工作模式：字符模式和图形模式。显卡们为了兼容这种标准，不得不自己提供一种叫 VGABIOS 的固件程序。

下面，我们来看看显卡的字符模式的工作细节。

它把屏幕分成 24 行，每行 80 个字符，把这（24*80）个位置映射到以 0xb8000 地址开始的内存中，每两个字节对应一个字符，其中一个字节是字符的 ASCII 码，另一个字节为字符的颜色值。如下图所示：

![](../images/pics/计算机底层/6.jpg)

明白了显卡的字符模式的工作细节，下面我们开始写代码。

这里先提个醒：C 语言字符串是以 0 结尾的，其字符编码通常是 utf8，而 utf8 编码对 ASCII 字符是兼容的，即英文字符的 ASCII 编码和 utf8 编码是相等的（关于utf8编码你可以自行了解）。

```

//彭东 @ 2021.01.09
void _strwrite(char* string)
{
  char* p_strdst = (char*)(0xb8000);//指向显存的开始地址
  while (*string)
  {
    *p_strdst = *string++;
    p_strdst += 2;
  }
  return;
}

void printf(char* fmt, ...)
{
  _strwrite(fmt);
  return;
}
```

代码很简单，printf 函数直接调用了 _strwrite 函数，而 _strwrite 函数正是将字符串里每个字符依次定入到 0xb8000 地址开始的显存中，而 p_strdst 每次加 2，这也是为了跳过字符的颜色信息的空间。

到这，Hello OS 相关的代码就写好了，下面就是编译和安装了。你可别以为这个事情就简单了，下面请跟着我去看一看。

#### 1.4.4 编译和安装 Hello OS

Hello OS 的代码都已经写好，这时就要进入安装测试环节了。在安装之前，我们要进行系统编译，即把每个代码模块编译最后链接成可执行的二进制文件。

你可能觉得我在小题大做，编译不就是输入几条命令吗，这么简单的工作也值得一说？

确实，对于我们 Hello OS 的编译工作来说特别简单，因为总共才三个代码文件，最多四条命令就可以完成。

但是以后我们 Hello OS 的文件数量会爆炸式增长，一个成熟的商业操作系统更是多达几万个代码模块文件，几千万行的代码量，是这世间最复杂的软件工程之一。所以需要一个牛逼的工具来控制这个巨大的编译过程。

#### 1.4.5 make 工具

make 历史悠久，小巧方便，也是很多成熟操作系统编译所使用的构建工具。

在软件开发中，make 是一个工具程序，它读取一个叫“makefile”的文件，也是一种文本文件，这个文件中写好了构建软件的规则，它能根据这些规则自动化构建软件。

makefile 文件中规则是这样的：首先有一个或者多个构建目标称为“target”；目标后面紧跟着用于构建该目标所需要的文件，目标下面是构建该目标所需要的命令及参数。

与此同时，它也检查文件的依赖关系，如果需要的话，它会调用一些外部软件来完成任务。

第一次构建目标后，下一次执行 make 时，它会根据该目标所依赖的文件是否更新决定是否编译该目标，如果所依赖的文件没有更新且该目标又存在，那么它便不会构建该目标。这种特性非常有利于编译程序源代码。

任何一个 Linux 发行版中都默认自带这个 make 程序，所以不需要额外的安装工作，我们直接使用即可。

为了让你进一步了解 make 的使用，接下来我们一起看一个有关 makefile 的例子：

```
CC = gcc #定义一个宏CC 等于gcc
CFLAGS = -c #定义一个宏 CFLAGS 等于-c
OBJS_FILE = file.o file1.o file2.o file3.o file4.o #定义一个宏
.PHONY : all everything #定义两个伪目标all、everything
all:everything #伪目标all依赖于伪目标everything
everything :$(OBJS_FILE) #伪目标everything依赖于OBJS_FILE，而OBJS_FILE是宏会被#替换成file.o file1.o file2.o file3.o file4.o
%.o : %.c
	$(CC) $(CFLAGS) -o $@ $<
```

我来解释一下这个例子：

make 规定“#”后面为注释，make 处理 makefile 时会自动丢弃。

makefile 中可以定义宏，方法是在一个字符串后跟一个“=”或者“:=”符号，引用宏时要用“\$(宏名)”，宏最终会在宏出现的地方替换成相应的字符串，例如：\$(CC) 会被替换成 gcc，$( OBJS_FILE) 会被替换成 file.o file1.o file2.o file3.o file4.o。

.PHONY 在 makefile 中表示定义伪目标。所谓伪目标，就是它不代表一个真正的文件名，在执行 make 时可以指定这个目标来执行其所在规则定义的命令。但是伪目标可以依赖于另一个伪目标或者文件，例如：all 依赖于 everything，everything 最终依赖于 file.c file1.c file2.c file3.c file4.c。

虽然我们会发现，everything 下面并没有相关的执行命令，但是下面有个通用规则：“%.o : %.c”。其中的“%”表示通配符，表示所有以“.o”结尾的文件依赖于所有以“.c”结尾的文件。

例如：file.c、file1.c、file2.c、file3.c、file4.c，通过这个通用规则会自动转换为依赖关系：file.o: file.c、file1.o: file1.c、file2.o: file2.c、file3.o: file3.c、file4.o: file4.c。

然后，针对这些依赖关系，分别会执行：`$(CC) $(CFLAGS) -o $@ $< `命令，当然最终会转换为：`gcc –c –o xxxx.o xxxx.c`，这里的“xxxx”表示一个具体的文件名。

#### 1.4.6 编译

![](../images/pics/计算机底层/7.jpg) 

