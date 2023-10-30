---
title: Java ASM
tags:
  - Java安全
  - Java字节码
categories: Java安全
date: 2022-01-27 09:14:46
---

<font style="color:Gray; float:left">若菩萨有我相、人相、众生相、寿者相，即非菩萨。</font><br>

<font style="color:Gray; float:right">——《金刚般若波罗蜜经》</font>

<br>

ASM入门

<!-- more -->



## 1. 概念

[ASM](http://asm.ow2.org/) 是一个 Java 字节码操控框架。它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。Java class 被存储在严格格式定义的 .class 文件里，这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。ASM 从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。

简单的说，ASM可以读取解析`class`文件内容，并提供接口让你可以对`class`文件字节码内容进行CRUD操作。

ASM一般用于动态代理，我们知道JDK通过反射也实现了动态代理，但这种方式的缺点就是性能低，通过ASM可以在字节码层面进行原有类修改。



## 2. 使用

参考[ASM用户手册](https://asm.ow2.io/asm4-guide.pdf)，用户手册我们关注怎么用就行了。

ASM provides three core components based on the ClassVisitor API to generate and transform classes: •

- The ClassReader class parses a compiled class given as a byte array, and calls the corresponding visitXxx methods on the ClassVisitor instance passed as argument to its accept method. It can be seen as an event producer. 
- The ClassWriter class is a subclass of the ClassVisitor abstract class that builds compiled classes directly in binary form. It produces as output a byte array containing the compiled class, which can be retrieved with the toByteArray method. It can be seen as an event consumer. •
- The ClassVisitor class delegates all the method calls it receives to another ClassVisitor instance. It can be seen as an event filter. The next sections show with concrete examples how these components can be used to generate and transform classes.

 ClassVisitor 接口提供了三个核心组件。

- **ClassReader**

  读取编译过的class文件，对其解析

- **ClassWriter**

  直接生成一个class

- **ClassVisitor**

  ASM API用于生成和修改class文件都是基于ClassVisitor，通过重写ClassVisitor的方法来实现字节码的读取、修改与生成。



### 2.1 ClassReader

解析指定类的class文件

这里我们直接复制文档中的代码

```java
package ASM;

import jdk.internal.org.objectweb.asm.*;

import java.io.IOException;

import static jdk.internal.org.objectweb.asm.Opcodes.ASM4;

public class ClassPrinter extends ClassVisitor {
    public ClassPrinter() {
        super(ASM4);
    }

    public void visit(int version, int access, String name,
                      String signature, String superName, String[] interfaces) {
        System.out.println(name + " extends " + superName + " {");
    }

    public void visitSource(String source, String debug) {
    }

    public void visitOuterClass(String owner, String name, String desc) {
    }

    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
        return null;
    }

    public void visitAttribute(Attribute attr) {
    }

    public void visitInnerClass(String name, String outerName, String innerName, int access) {
    }

    public FieldVisitor visitField(int access, String name, String desc,
                                   String signature, Object value) {
        System.out.println(" " + desc + " " + name);
        return null;
    }

    public MethodVisitor visitMethod(int access, String name,
                                     String desc, String signature, String[] exceptions) {
        System.out.println(" " + name + desc);
        return null;
    }

    public void visitEnd() {
        System.out.println("}");
    }

    public static void main(String[] args) throws IOException {
        ClassPrinter cp = new ClassPrinter();
        ClassReader cr = new ClassReader("java.lang.Runnable");
        cr.accept(cp, 0);
    }
}

```

运行结果

```
java/lang/Runnable extends java/lang/Object {
 run()V
}
```

上面的程序重写了visit， visitField方法，visitMethod等方法，运行结果成功获取到了Runnable类信息。

读一下自己写的类

```java
 public static void main(String[] args) throws IOException {
        ClassPrinter cp = new ClassPrinter();
        ClassReader cr = new ClassReader("ASM.HelloWorld");
        cr.accept(cp, 0);
    }
```

```
ASM/HelloWorld extends java/lang/Object {
 <init>()V
 main([Ljava/lang/String;)V
}

```



### 2.2 ClassWriter

直接生成类的字节码。

我们需要生成一个接口

```java
package pkg;
public interface Comparable extends Mesurable {
	int LESS = -1;
	int EQUAL = 0;
	int GREATER = 1;
	int compareTo(Object o);
}
```



还是使用文档中的代码

```java
package ASM;


import jdk.internal.org.objectweb.asm.ClassWriter;
import static jdk.internal.org.objectweb.asm.Opcodes.*;



public class ClassWriteTest {
    public static void main(String[] args) {
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_5, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE,
                "pkg/Comparable", null, "java/lang/Object",
                null);
        cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS", "I",
                null, new Integer(-1)).visitEnd();
        cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL", "I",
                null, new Integer(0)).visitEnd();
        cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER", "I",
                null, new Integer(1)).visitEnd();
        cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "compareTo",
                "(Ljava/lang/Object;)I", null, null).visitEnd();
        cw.visitEnd();
        byte[] b = cw.toByteArray();
        
        //通过classloader读取字节码
        MyClassLoader myClassLoader = new MyClassLoader();
        Class c = myClassLoader.defineClass("pkg.Comparable", b);
        System.out.println(c.getMethods()[0].getName());
    }

}

```

改动一下，删除掉了`new String[] { "pkg/Mesurable" }`这是继承类，这里用不上。

通过重写ClassLoader的defineClass方法将生成的字节码读取以便访问

```java
package ASM;


public class MyClassLoader extends ClassLoader {
    public Class defineClass(String name, byte[] b) {
        return defineClass(name, b, 0, b.length);
    }
}

```

当然上面也可以用`FindClass()`来读取class

ClassWriteTest的执行结果如下

```
compareTo
```





### 2.3 Transforming classes

先写一个Run类和一个Test类

```java
package ASM;

public class Run {
    public void run() {
        System.out.println("程序运行起来了");
    }
}

```

```java
package ASM;

public class Test {
   public static void test() {
       System.out.println("先测试......");
   }
}

```

我们要实现的功能就是通过ASM插入JVM指令，在调用Run类前先执行`test()`方法，并生成class文件

```java
package ASM;

import jdk.internal.org.objectweb.asm.ClassReader;
import jdk.internal.org.objectweb.asm.ClassVisitor;
import jdk.internal.org.objectweb.asm.ClassWriter;
import jdk.internal.org.objectweb.asm.MethodVisitor;
import static jdk.internal.org.objectweb.asm.Opcodes.ASM4;
import static jdk.internal.org.objectweb.asm.Opcodes.INVOKESTATIC;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;

public class ClassTransferTest {
    public static void main(String[] args) throws Exception {
        ClassReader cr = new ClassReader(ClassPrinter.class.getClassLoader().getResourceAsStream("ASM/Run.class"));

        ClassWriter cw = new ClassWriter(0);
        ClassVisitor cv = new ClassVisitor(ASM4, cw) {
            @Override
            public MethodVisitor visitMethod(int i, String s, String s1, String s2, String[] strings) {
                MethodVisitor mv = super.visitMethod(i, s, s1, s2, strings);
                return new MethodVisitor(ASM4, mv) {
                    @Override
                    public void visitCode() {
                        visitMethodInsn(INVOKESTATIC, "ASM/Test", "test", "()V", false);
                        super.visitCode();
                    }
                };
            }
        };

        cr.accept(cv,0);
        byte[] b2 = cw.toByteArray();
        MyClassLoader cl = new MyClassLoader();
        cl.loadClass("ASM.Test");
        Class c2 = cl.defineClass("ASM.Run", b2);
        c2.getConstructor().newInstance();

        String path = (String)System.getProperties().get("user.dir");
        File f = new File(path + "/ASM/");
        f.mkdirs();
        System.out.println("生成的class文件地址：" + path + "/ASM/Run_0.class");
        FileOutputStream fos = new FileOutputStream(new File(path + "/ASM/Run_0.class"));
        fos.write(b2);
        fos.flush();

    }
}

```

生成的class文件为

```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package ASM;

public class Run {
    public Run() {
        Test.test();
        super();
    }

    public void run() {
        Test.test();
        System.out.println("程序运行起来了");
    }
}

```

成功通过`visitMethodInsn(INVOKESTATIC, "ASM/Test", "test", "()V", false);`指令将`test()`插入了run的类并生成了字节码。



## 总结

参考文档，我们通过ASM框架对class文件进行了读取、修改与生成。





