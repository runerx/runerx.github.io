---
title: Linux内核同步
tags:
  - Linux
categories: 计算机系统
---





## 1. 进程与进程之间的同步机制（信号链）

并发执行的单元对共享资源的同时访问会引发竞态问题。解决竞态问题的途径是保证对共享资源的互斥访问，所谓互斥访问是指一个执行单元在访问共享资源的时候，其他的执行单元被禁止访问。访问共享资源的代码区块叫做”临界区“，临界区需要某种内核同步方法来保护。



## 2. 内核同步方法（信号量semaphore）



```c
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/wait.h>
#include <linux/poll.h>
#include <linux/sched.h>
#include <linux/slab.h>

#define BUFFER_MAX    (64)
#define OK            (0)
#define ERROR         (-1)

struct cdev *gDev;
struct file_operations *gFile;
dev_t  devNum;
unsigned int subDevNum = 1;
int reg_major  =  232;
int reg_minor =   0;
char buffer[BUFFER_MAX];
struct semaphore sema;
int open_count = 0;

int hello_open(struct inode *p, struct file *f)
{
        down(&sema);
        if(open_count >= 1){
                up(&sema);
        printk(KERN_INFO "device is busy, hello_open fail");
                return -EBUSY;
        }
        open_count++;
        up(&sema);
    printk(KERN_INFO "hello_open ok");
    return 0;
}

int hello_close(struct inode *inode, struct file *filp)
{
        if(open_count != 1) {
                printk(KERN_INFO "something wrong, hello_close fail");
                return -EFAULT;
        }
    open_count--;
    printk(KERN_INFO "hello_close ok");
        return 0;
}

ssize_t hello_write(struct file *f, const char __user *u, size_t s, loff_t *l)
{
        printk(KERN_INFO "hello_write\n");
        int writelen = 0;
    writelen = BUFFER_MAX>s ? s : BUFFER_MAX;
    if (copy_from_user(buffer, u, writelen)) {
                return -EFAULT;
        }
        return writelen;
}
ssize_t hello_read(struct file *f, char __user *u, size_t s, loff_t *l)
{
        printk(KERN_INFO "hello_read\n");
    int readlen;
    readlen = BUFFER_MAX>s ? s : BUFFER_MAX;
    if (copy_to_user(u, buffer, readlen)) {
        return -EFAULT;
        }
    return readlen;
}
int hello_init(void)
{

    devNum = MKDEV(reg_major, reg_minor);
    if(OK == register_chrdev_region(devNum, subDevNum, "helloworld")){
        printk(KERN_INFO "register_chrdev_region ok \n");
    }else {
    printk(KERN_INFO "register_chrdev_region error n");
        return ERROR;
    }
    printk(KERN_INFO " hello driver init \n");
    gDev = kzalloc(sizeof(struct cdev), GFP_KERNEL);
    gFile = kzalloc(sizeof(struct file_operations), GFP_KERNEL);
    gFile->open = hello_open;
    gFile->release = hello_close;
    gFile->read = hello_read;
    gFile->write = hello_write;
    gFile->owner = THIS_MODULE;
    cdev_init(gDev, gFile);
    cdev_add(gDev, devNum, 1);

    sema_init(&sema, 1);
    return 0;
}

void __exit hello_exit(void)
{
        printk(KERN_INFO " hello driver exit \n");
    cdev_del(gDev);
    kfree(gFile);
    kfree(gDev);
    unregister_chrdev_region(devNum, subDevNum);
    return;
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");


```

不同于之前的案例，本次主要修改了hello_open函数

```c
int hello_open(struct inode *p, struct file *f)
{
        down(&sema);
        if(open_count >= 1){
                up(&sema);
        printk(KERN_INFO "device is busy, hello_open fail");
                return -EBUSY;
        }
        open_count++;
        up(&sema);
    printk(KERN_INFO "hello_open ok");
    return 0;
}
```





```
#include <linux/module.h>
 #include <linux/moduleparam.h>
 #include <linux/cdev.h>
 #include <linux/fs.h>
 #include <linux/wait.h>
 #include <linux/poll.h>
 #include <linux/sched.h>
 #include <linux/slab.h>

 #define BUFFER_MAX    (64)
 #define OK            (0)
 #define ERROR         (-1)

 struct cdev *gD
```















**互斥**

保证两个线程不能同时执行一段代码











