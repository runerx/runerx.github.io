---
title: java类加载机制与反射
date: 2022-01-10 20:15:49
categories: Java安全
tags: 
- Java基础
---

<font style="color:Gray; float:left">莫道不消魂，帘卷西风，人比黄花瘦。</font><br>

<font style="color:Gray; float:right">——《醉花阴》</font>

<br>



Java类的加载、连接、初始化和反射。

<!-- more -->

## 1. 类的加载、连接和初始化

系统可能在第一次使用某个类时加载该类，也可能采用预加载机制来加载某个类。

### 1.1 JVM和类

当调用java命令运行某个Java程序时，该命令将会启动一个Java虚拟机进程，不管该Java程序有多么复杂，该程序启动了多少个线程，它们都处于该Java虚拟机进程里。同一个JVM的所有线程、所有变量都处于同一个进程里，它们都使用该JVM进程的内存区。当系统出现以下几种情况时，JVM进程将被终止。

- 程序运行到最后正常结束 
- 程序运行到使用System.exit()或Runtime.getRuntime().exit()代码处程序结束。
- 程序执行过程中遇到未捕获的异常或错误而结束。
- 程序所在平台强制结束了JVM进程。

当Java程序运行结束时，JVM进程结束，该进程在内存中的状态将会丢失。

我们编写测试Demo来证明java程序与JVM的关系。

**测试程序: A.java**

```java
package reflect;

public class A {
    // 定义该类的类变量
    public static int a = 6;
}

```

*类变量：独立于方法之外的变量，用 static 修饰。*

**程序1：ATest1**

接下来定义一个类用来创建A类的实例，并访问A对象的类变量a。

```java
package reflect;

public class ATest1 {
    public static void main(String[] args) {
        // 创建A类的实例
        A a = new A();
        // 让a实例的类变量a的值自加
        a.a ++;
        System.out.println(a.a);
    }
}

```

结果：

```
7
```

**程序2：ATest2**

新创建一个A类的实例，并访问A的类变量

```java
package reflect;

public class ATest2 {
    public static void main(String[] args) {
        // 创建A类的实例
        A b = new A();
        // 输出b实例的类变量a的值
        System.out.println(b.a);
    }
}

```

结果：

```
6
```

结果为6的原因就是创建的两个类是不同的JVM，程序1修改的值并不适用于程序2，程序1运行结束后在内存中的状态就消失了，与程序2新启的JVM没有任何关系



### 1.2 类的加载

当程序主动使用某个类时，如果该类还未被加载到内存中，则系统会通过<u>加载、连接、初始化</u>三个步骤来对该类进行初始化。如果没有意外，JVM将会连续完成这三个步骤，所以<u>有时也把这三个步骤统称为类加载或类初始化</u>。

<u>类加载指的是将类的class文件读入内存，并为之创建一个java.lang.Class对象</u>。也就是说，当程序中使用任何类时，系统都会为之创建一个java.lang.Class对象。这里可以看出类也是一种对象，就像《金刚经》里讲：凡所有相皆是虚妄。要用佛学来破除这些相，如果太专注于佛学本身也着相了，因为佛学本身也是相。

类的加载由类加载器完成，类加载器通常由JVM提供，这些类加载器也是所有程序运行的基础，<u>JVM提供的这些类加载器通常被称为系统类加载器</u>。除此之外，<u>开发者可以通过继承ClassLoader基类来创建自己的类加载器</u>。

通过使用不同的类加载器，可以从不同来源加载

- 从本地文件系统加载class文件
- 从JAR包加载class文件，JVM可以从JAR文件中直接加载该class文件。
- 通过网络加载class文件
- 把一个java源文件动态编译，并执行加载。

类加载器通常无须等到“首次使用”该类时才加载该类，java虚拟机规范允许系统预先加载某些类。



### 1.3 类的连接

当类被加载之后，系统为之生成一个对应的Class对象，接着将会进入连接阶段，连接阶段负责把类的二进制数据合并到JRE中。类连接又可分为如下三阶段。

1）验证：验证阶段用于检测被加载的类是否有正确的内部结构，并和其他类协调一致。

2）准备：类准备阶段则负责为类的类变量分配内存，并设置默认初始值。

3）解析：将类的二进制数据中的符号引用替换成直接引用



###  1.4 类的初始化

在类的初始化阶段，<u>虚拟机负责对类进行初始化，主要就是对类变量进行初始化</u>。在Java类中对类变量指定初始化值有两种方式：

1. 声明类变量时指定初始值
2. 使用静态初始化块为类变量指定初始化值。

代码片段

```java
public class Test
{
    //声明变量a时指定初始值
    static int a = 5;
    static int b; 
    static int c; //未指定，默认初始值为0
    static
    {
        //使用静态初始化块为变量b指定初始值
        b = 6;
    }
}
```

声明变量时指定初始值，静态初始化块都将被当成类的初始化语句，JVM会按这些语句在程序中的排列顺序依次执行它们，例如下面的类。

```java
package reflect;

public class Tes {
    static
    {
        //使用静态初始化块为变量b指定初始值
        b = 6;
        System.out.println("-------------");
    }
    //声明变量a时指定初始化值
    static int a = 5;
    static int b = 9; //*1
    static int c;

    public static void main(String[] args) {
        System.out.println(Tes.b); //9
    }
}

```

上面的代码先在静态初始化块中为b变量赋值，此时类变量的值为6；接着程序向下执行，执行到*1处这行代码也属于该类的初始化语句，所以程序再次为类变量赋值，也就是说，当类初始化结束后，该类变量b的值为9.

JVM初始化一个类包含如下几个步骤。

1. 假如这个类还没有被加载和连接，则程序先加载并连接该类。
2. 假如该类的直接父类还没有被初始化，则先初始化其直接父类。
3. 假如类中有初始化语句，则系统依次执行这些初始化语句。

当执行第二个步骤时，系统对直接父类的初始化步骤也遵循此步骤1~3；如果该直接父类又没有直接父类，则系统再次重复...依次类推。所以JVM最先初始化的总是java.lang.Object类。当程序主动使用任何一个类时，系统会保证该类以及所有父类（包括直接父类和间接父类）都会被初始化。



### 1.5 类的初始化时机

当java程序首次通过下面6种方式来使用某个类或者接口时，系统就会初始化该类或者接口。

- 创建类的实例。为某个类创建实例的方式包括：使用new操作符来创建实例，通过反射来创建实例，通过反序列化的方式创建实例
- 调用某个类的类方法（静态方法）。
- 访问某个类或接口的类变量，或为该类变量赋值。
- 使用反射方式强制创建某个类或接口对应的java.lang.Class对象。例如代码Class.forName("Person")，如果系统还未初始化Person类，则这行代码将会导致该Person类被初始化，并返回Person类对应的java.lang.Class对象。
- 初始化某个类的子类，当初始化某个类的子类时，该子类所有父类都会被初始化。
- 直接使用java命令来运行某个主类。当运行某个主类时，程序会先初始化该主类。

除此之外的特殊情况：

对于一个final型的变量，如果该类变量的值在编译时就可以确定下来，那么这个类变量相当于“宏变量”。java编译器会在编译时直接把这个类变量出现的地方替换成它的值，因此即使程序使用静态类变量，也不会导致该类的初始化。如果在编译时不能确定的final型变量该类还是会被初始化

## 2. 类加载器

类加载器负责将.class文件（可能在磁盘上，也可能在网络上）加载到内存中，并为之生成对应的java.lang.Class对象。

### 2.1 类加载机制

类加载器负责加载所有的类，系统为所有被载入内存中的类生成一个java.lang.Class实例。一旦一个类被载入JVM中，同一个类就不会被载入了。怎样才算是”同一个类”？在java中，<u>一个类用其全限定类名和其类加载器作为唯一标识</u>。例如在pg的包中有一个名为Person的类，被类加载器ClassLoader的实例kl负责加载。则该Person类对应的Class对象在JVM中表示为（Person、pg、kl）。

当JVM启动时，会形成由三个类加载器组成的初始类加载器层次结构。

- Bootstrap ClassLoader: 根类加载器
- Extension ClassLoader: 扩展类加载器
- System ClassLoader: 系统类加载器

Bootstrap ClassLoader被称为引导（也称为原始或根）类加载器，它负责加载Java的核心类。在Sum的JVM中，当执行java命令时，使用-Xbootclasspath或-D选项指定sun.boot.class.path系统属性值可以指定加载附加的类。

JVM的类加载机制主要有如下三种:

- 全盘负责。所谓全盘负责，就是当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显式使用另外一个类加载器来载入。
- 父类委托。所谓父类委托，则是先让parent类加载器试图加载该Class,只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。
- 缓存机制。缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存中搜寻该Class，只有当缓存区中不存在该Class对象时，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存中。这就是为什么修改了Class后，必须重启JVM，程序所做的修改才会生效的原因。

<u>类加载器之间的父子关系并不是类继承上的父子关系，这里的父子关系是类加载器实例之间的关系</u>

除了可以使用Java提供的类加载器之外，开发者也可以实现自己的类加载器，自定义的类加载器通过继承ClassLoader来实现，JVM中这4种类加载器的层次结构如下所示。

```
根类加载器<------扩展类加载器<-----系统类加载器<------用户类加载器
```

测试Demo

```java
package reflect;

import java.io.IOException;
import java.net.URL;
import java.util.Enumeration;

public class ClassLoaderPropTest {
    public static void main(String[] args) throws IOException {
        //获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println("系统类加载器：" + systemClassLoader);
        /*
        获取系统类加载器的加载路径——通常通常由CLASSPATH环境变量指定，如果操作系统没有指定CLASSPATH环境变量，则默认以当前路径作为系统类
        加载加载器的加载路径。
         */
        Enumeration<URL> eml = systemClassLoader.getResources("");
        while(eml.hasMoreElements()) {
            System.out.println(eml.nextElement());
        }

        //获取系统类加载器的父亲加载器，得到扩展类加载器
        ClassLoader extensionLoader = systemClassLoader.getParent();
        System.out.println("扩展类加载器：" + extensionLoader);
        System.out.println("扩展类加载器的加载路径: " + System.getProperty("java.ext.dirs"));
        System.out.println("扩展类加载器的parent: " + extensionLoader.getParent());

    }
}

```

运行结果：

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=54220:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/tools.jar:/Users/shadowflow/code/java/test/target/classes:/Users/shadowflow/.m2/repository/javassist/javassist/3.12.1.GA/javassist-3.12.1.GA.jar reflect.ClassLoaderPropTest
系统类加载器：sun.misc.Launcher$AppClassLoader@18b4aac2
file:/Users/shadowflow/code/java/test/target/classes/
扩展类加载器：sun.misc.Launcher$ExtClassLoader@610455d6
扩展类加载器的加载路径: /Users/shadowflow/Library/Java/Extensions:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java
扩展类加载器的parent: null
 
Process finished with exit code 0
```

从运行结果可以看出，系统类加载器的加载路径是程序运行的当前路径，扩展类加载器在jdk的/lib目录。扩展类的父类也就是根类加载器是null。这是因为根类加载器并没有继承ClassLoader抽象类，所有扩展类加载器的`getParent()`方法返回null。但实际上，扩展类加载器的父类加载器是根类加载器，只是根类加载器并不是Java实现的。

类加载器加载Class大致要经过如下8个步骤。

1. 检测此Class是否载入过（即在缓冲区中是否有此Class），如果有则直接进入第8步，否则接着执行第2步
2. 如果父类加载器不存在（如果没有父类加载器，则要么parent一定是根类加载器，要么本身就是根类加载器），则跳到第4步执行；如果父类加载器存在，则接着第三步。
3. 请求使用父类加载器去载入目标类，如果成功载入则跳到第8步，否则接着执行第5步。
4. 请求使用根类加载器来载入目标类，如果成功载入则跳到第8步，否则跳到7步。
5. 当前类加载器尝试寻找Class文件（从与此ClassLoader相关的类路径寻找），如果找到则执行第6步，如果找不到则跳到第7步。
6. 从文件中载入Class，成功载入后跳到第8步
7. 抛出ClassNotFoundException异常
8. 返回对应的java.lang.Class对象。

其中5、6步允许重写ClassLoader的`findClass()`方法类实现自己的载入策略，甚至重写`loadClass()`方法类实现自己的载入过程。

### 2.2 创建并使用自定义的类加载器

JVM中除根类加载器之外的所有类加载器都是ClassLoader子类的实例，开发者可以通过扩展ClassLoader的子类。并重写该ClassLoader所包含的方法来实现自定义类加载器。ClassLoader中包含大量的protected方法——这些方法都可以被子类重写。

ClassLoader类有如下两个关键方法：

1. `loadClass(String name, boolean resolve)`：该方法为ClassLoader的入口点，根据指定名称来加载类，系统就是调用ClassLoader的该方法来获取指定类对应的Class对象。

2. findClass(String name)`: 根据指定名称来查找类。

如果需要实现自定义的ClassLoader，则可以通过重写以上两个方法来实现，<font color="Red">通常推荐重写`findClass()`方法</font>而不是`loadClass()`方法。`loadClass()`方法的执行步骤如下。

1. 用`findLoadedClass(String)`来检查是否已经加载类，如果已经加载则直接返回
2. 在父类加载器上调用`loadClass()`方法。如果父类加载器为null，则使用根类加载器来加载。
3. 调用`findClass(String)`方法查找类。

从上面的步骤可以看出，重写`findClass()`方法可以避免覆盖默认类加载器的父类委托、缓冲机制两种策略；如果重写`loadClass()`方法则实现逻辑更为复杂。

在ClassLoader里还有一个核心方法：Class defineClass(String name, byte[]b，int off, intlen)，该方法负责将指定类的字节码文件读入字节数组byte[] b内，并把它转化为Class对象，该字节码文件可以来源于文件、网络等。

`defineClass()`方法管理JVM的许多复杂的实现，它负责将字节码分析成运行时数据结构，并校验有效性等。但是不用重写该方法，实际上该方法是final的，没办法重写。

ClassLoader还有一些普通方法。

- `findSystemClass(String name)`: 从本地文件系统装入文件。它在本地文件系统中寻找类文件，如果存在，就使用`defineClass()`方法将原始字节转换为Class对象，以将该文件转换成类。
- `static getSystemClassLoader()`: 这是一个静态方法，用于返回系统类加载器
- `getParent()`: 获取类加载器的父类加载器。
- resolveClass(Class<?> c): 链接指定的类。类加载器可以使用此方法来链接类c。
- `findLoadedClass(String name)`: 如果此Java虚拟机已经加载了名为name的类，则直接返回该类对应的Class实例，否则返回Null。该方法是Java类加载缓存机制的体现。

下面的程序开发了一个自定义的ClassLoader，该ClassLoader通过重写`findClass()`来实现自定义的类加载机制。这个ClassLoader可以在加载类之前先编译该类源文件，从而实现运行Java之前先编译程序的目标，这样即可通过该ClassLoader直接运行Java源文件。

```java
package reflect.loader;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.lang.reflect.Method;


public class CompileClassLoader extends ClassLoader {
    // 读取一个文件的内容
    private byte[] getBytes(String filename)
            throws IOException {
        File file = new File(filename);
        long len = file.length();
        byte[] raw = new byte[(int) len];
        try (
                FileInputStream fin = new FileInputStream(file)) {
            // 一次读取Class文件的全部二进制数据
            int r = fin.read(raw);
            if (r != len)
                throw new IOException("无法读取全部文件："
                        + r + " != " + len);
            return raw;
        }
    }

    // 定义编译指定Java文件的方法
    private boolean compile(String javaFile)
            throws IOException {
        System.out.println("CompileClassLoader:正在编译 "
                + javaFile + "...");
        // 调用系统的javac命令
        Process p = Runtime.getRuntime().exec("javac " + javaFile);
        try {
            // 其他线程都等待这个线程完成
            p.waitFor();
        } catch (InterruptedException ie) {
            System.out.println(ie);
        }
        // 获取javac线程的退出值
        int ret = p.exitValue();
        // 返回编译是否成功
        return ret == 0;
    }

    // 重写ClassLoader的findClass方法
    protected Class<?> findClass(String name)
            throws ClassNotFoundException {
        Class clazz = null;
        // 将包路径中的点（.）替换成斜线（/）
        String fileStub = name.replace(".", "/");
        String javaFilename = fileStub + ".java";
        String classFilename = fileStub + ".class";
        File javaFile = new File(javaFilename);
        File classFile = new File(classFilename);
        // 当指定Java源文件存在，且Class文件不存在，或者Java源文件
        // 的修改时间比Class文件的修改时间更晚时，重新编译
        if (javaFile.exists() && (!classFile.exists()
                || javaFile.lastModified() > classFile.lastModified())) {
            try {
                // 如果编译失败，或者该Class文件不存在
                if (!compile(javaFilename) || !classFile.exists()) {
                    throw new ClassNotFoundException(
                            "ClassNotFoundExcetpion:" + javaFilename);
                }
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
        // 如果Class文件存在，系统负责将该文件转换成Class对象
        if (classFile.exists()) {
            try {
                // 将Class文件的二进制数据读入数组
                byte[] raw = getBytes(classFilename);
                // 调用ClassLoader的defineClass方法将二进制数据转换成Class对象
                clazz = defineClass(name, raw, 0, raw.length);
            } catch (IOException ie) {
                ie.printStackTrace();
            }
        }
        // 如果clazz为null，表明加载失败，则抛出异常
        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }
        return clazz;
    }

    // 定义一个主方法
    public static void main(String[] args) throws Exception {
        // 如果运行该程序时没有参数，即没有目标类
        if (args.length < 1) {
            System.out.println("缺少目标类，请按如下格式运行Java源文件：");
            System.out.println("java CompileClassLoader ClassName");
        }
        // 第一个参数是需要运行的类
        String progClass = args[0];
        System.out.println(progClass); //reflect.loader.Hello
        // 剩下的参数将作为运行目标类时的参数
        // 将这些参数复制到一个新数组中
        String[] progArgs = new String[args.length - 1];
        System.arraycopy(args, 1, progArgs
                , 0, progArgs.length);
        CompileClassLoader ccl = new CompileClassLoader();
        // 加载需要运行的类
        Class<?> clazz = ccl.loadClass(progClass);
        // 获取需要运行的类的主方法
        Method main = clazz.getMethod("main", (new String[0]).getClass());
        Object argsArray[] = {progArgs};
        main.invoke(null, argsArray);
    }
}

```

上面的程序重写了`findClass()`方法，通过重写该方法就可以实现自定义的类加载机制。`findClass()`方法中先检查需要加载的Class文件是否存在。如果不存在则先编译文件，在调用ClassLoader的`defineClass()`方法来加载这个Class文件，并生成相应的Class对象。

接下来可以随意提供一个简单的主类，该类无须编译就可以使用上面的CompileClasss Loader来运行它。

编写如下测试类：

```java
package reflect.loader;

public class Hello {
    public static void main(String[] args) {
        for (String arg : args) {
            System.out.println("参数：" + arg);
        }
    }
}

```

首先对CompileClassLoader进行编译：

```bash
[shadowflow@ShadowOS java]% javac reflect/loader/CompileClassLoader.java 
```

然后不编译Hello直接运行Hello

```bash
[shadowflow@ShadowOS java]% java reflect.loader.CompileClassLoader reflect.loader.Hello shadotest
reflect.loader.Hello
CompileClassLoader:正在编译 reflect/loader/Hello.java...
参数：shadotest

```

本示例程序提供的类加载功能比较简单，仅仅提供了在运行之前先编译Java源文件的功能，实际上，使用自定义的类加载器，可以实现如下常见功能：

- 执行代码前自动验证数字签名
- 根据用户提供的密码解密代码，从而可以实现代码混淆器来避免反编译*.class文件。
- 根据用户需求来动态的加载类。
- 根据应用需求来把其他数据以字节码的形式加载到应用中



### 2.3 URL ClassLoader类

Java为ClassLoader提供了一个URLClassLoader实现类，该类也是系统类加载器和扩展类加载器的父类（此处的父类，就是指类与类之间的继承关系）。URLClassLoader功能比较强大，它既可以从本地文件系统获取二进制文件来加载类，也可以从远程主机获取二进制文件来加载类。

在应用程序中可以直接使用URLClassLoader加载类，URLClassLoader类提供了如下两个构造器。

- `URLClassLoader(URL[] urls)`: 使用默认的父类加载器创建一个ClassLoader对象，该对象将从urls所指定的系列路径来查询并加载类。
- `URLClassLoader(URL[] urls, ClassLoader parent)`: 使用指定的父类加载器创建一个ClassLoader对象，其他功能与前一个构造器相同。

一旦得到了URLCLassLoader对象之后，就可以调用该对象的`loadClass()`方法来加载指定类。下面程序示范了如何从文件系统中加载MySQL驱动，并使用该驱动来获取数据库连接。通过这种方式来获取数据库连接，可以无须将MySQL驱动添加到CLASSPATH环境变量中。

```java
package reflect.loader;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.sql.Connection;
import java.sql.Driver;
import java.util.Properties;

public class URLClassLoaderTest {
    private static Connection conn;
    //定义一个获取数据库连接的方法
    public static Connection getConn(String url, String user, String pass) throws Exception {
        if (conn == null) {
            //1）创建一个URL数组
            URL[] urls = {new URL("file:mysql-connector-java-5.1.30-bin.jar")};
            //2）以默认的ClassLoader作为父ClassLoader，创建URLClassLoader
            URLClassLoader myClassLoader = new URLClassLoader(urls);
            //3）加载MySQL的JDBC驱动，并创建默认实例
            Driver driver = (Driver) myClassLoader.loadClass("com.mysql.jdbc.Dreive").getConstructor().newInstance();
            //创建一个设置JDBC连接属性的Properties对象
            Properties props = new Properties();
            //至少需要为该对象传入user和password两个属性
            props.setProperty("user", user);
            props.setProperty("password", pass);
            //调用Driver对象的connect方法来获取数据库连接
            conn = driver.connect(url, props);
        }
        return conn;
    }

    public static void main(String[] args) throws Exception {
        System.out.println(getConn("jdbc:mysql://localhost:3306/mysql", "root", "root"));
    }
}

```

上面的代码创建了一个URLClassLoader对象，该对象使用默认的父类加载器，该类加载器的类加载路径是当前路径下的mysql-connector-java-5.1.30-bin.jar文件，将MySQL驱动复制到该路径下，这样保证该ClassLoader可以正常加载到com.mysql.jdbc.Driver类。第3）行代码使用使用ClassLoader的loadClass()加载指定类，并调用Class对象的`newInstance()`方法创建了一个该类的默认实例——也就得到了`com.mysql.jdbc.Driver`类的对象，当然该对象的实现类实现了java.sql.Driver接口，所以程序将其强制类型转换为Driver。这里通过Driver而不是DriverManager来获取数据库连接。

正如前面看到的，创建URLClassLoader时传入的一个URL数组参数，该ClassLoader就可以从这系列URL指定的资源中加载指定类，这里的URL可以以file:为前缀，表明从本地文件系统加载；可以以http:为前缀，表明从互联网通过HTTP访问来加载；也可以是ftp:为前缀，表明从互联网通过FTP访问来加载......功能非常强的。

## 3. 通过反射查看类信息

Java程序中的许多对象在运行时都会出现两种类型：编译时类型和运行时类型，例如代码：`Person p = new Student()`;这行代码将会生成一个p变量，该变量的编译时类型为Person，运行时类型为Student。除此之外还有更极端的情形，程序在运行时接收到外部传入的一个对象，该对象的编译时类型时Object，但程序又需要调用该对象运行时类型的方法。

为了解决这些问题，程序需要在运行时发现对象和类的真实信息。解决该问题有以下两种方法。

- 第一种做法是假设在编译时和运行时都完全知道类型的具体信息，在这种情况下，可以先使用instanceof运算符进行判断，再利用强制类型转换将其转换成其运行时类型的变量即可。
- 第二种做法是编译时根本无法预知该对象和类可能属于哪些类，程序只依靠运行时信息来发现该对象和类的真实信息，这就必须使用反射。



### 3.1 获得Class对象

前面已经介绍过了，每个类被加载之后，系统就会为该类生成一个对应的Class对象，通过该Class对象就可以就可以访问到JVM中的这个类。在Java程序中获得Class对象通常有如下三种方式：

1. 使用Class类的`forName(String clzzName)`静态方法。该方法需要传入字符串参数，该字符串参数的值是某个类的全限定类名（必须添加完整包名）。

2. 调用某个类的class属性来获取该类对应的Class对象。例如，Person.class将会返回Person类对应的Class对象。

3. 调用某个对象的`getClass()`方法。该方法是java.lang.Object类中的一个方法，所以所有的Java对象都可以调用该方法，该方法将会返回该对象所属类对应的Class对象。

对于第一种方式和第二种方式都是直接根据类来取得该类的Class对象，相比之下，第二种方式有如下两种优势：

1. 代码更安全。程序在编译阶段就可以检查需要访问的Class对象是否存在。
2. 程序性能更好。因为这种方式无须调用方法，所以性能更好

也就是说，大部分时候都应该使用第二种方式来获取指定类的Class对象。但如果程序只能获得一个字符串，例如"java.lang.String"，若需要获取该字符串对应的Class对象                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    ，则只能使用第一种方式，使用Class的`forName(String clazzName)`。

一旦获得某个类对应的Class对象之后，程序就可以调用Class对象的方法来获得该对象和该类的真实信息。



### 3.2 从Class中获取信息

Class类提供了大量的实例方法来获取该Class对象所对应类的详细信息，Class类大致包含如下方法，下面每个方法都可能包括多个重载的版本，可以查阅API文档来获取。

**获取构造器：**

下面四个方法用于获取Class对应类所包含的构造器

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Constructor<T> getConstructor(Class<?>...parameter Types)`  | 返回此`Class`对象对应类的、带指定形参列表的`public`构造器。  |
| `Constructor<?>[] getConstructors()`                         | 返回此 `Class`对象对应类的所有`public`构造器。               |
| `Constructor<T> getDeclaredConstructor( Class<?>...parameterTypes)` | 返回此 `Class`对象对应类的带指定形参列表的构造器,与构造器的访问权限无关。 |
| `Constructor<?>[] getDeclaredConstructors()`                 | 返回此 `Class`对象对应类的所有构造器,与构造器的访问权限无关。 |

**获取方法：**

下面4个方法用于获取`Class`对应类所包含的方法。

| 方法                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `Method getMethod(String name, Classs<?> ... parameterTypes)` | 返回此`Class`对象对应类的、带指定形参列表的 `public`方法。   |
| `Method[] getMethods()`                                      | 返回此`Class`对象所表示的类的所有 `public`方法。             |
| `Method getDeclaredMethod(String name, Class<?> ... parameter Types)` | 返回此 `Class`对象对应类的、带指定形参列表的方法,与方法的访问权限无关。 |
| `Method[] getDeclaredMethods()`                              | 返回此`Class`对象对应类的全部方法,与方法的访问权限无关       |

**获取成员变量：**

如下4个方法用于访问`Class`对应类所包含的成员变量。

| 方法                                  | 描述                                                         |
| :------------------------------------ | :----------------------------------------------------------- |
| `Field getField(String name)`         | 返回此 `Class`对象对应类的、指定名称的 `public`成员变量。    |
| `Field[] getFields()`                 | 返回此`Class`对象对应类的所有 `public`成员变量。             |
| `Field getDeclaredField(String name)` | 返回此 `Class`对象对应类的、指定名称的成员变量,与成员变量的访问权限无关。 |
| `Field[] getDeclaredFields()`         | 返回此 `Class`对象对应类的全部成员变量,与成员变量的访问权限无关 |

**获取Annotation：**

如下几个方法用于访问`Class`对应类上所包含的`Annotation`

| 方法                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `<A extends Annotation> A getAnnotation(Class<A> annotationClass` | 尝试获取该 `Class`象对应类上存在的、指定类型的 `Annotation`;如果该类型的注解不存在,则返回`null`。 |
| `Annotation[] getAnnotations()`                              | 返回修饰该 `Class`对象对应类上存在的所有 `Annotation`        |
| `<A extends annotation> A getDeclaredAnnotation(Class<A> annotationClass)` | 这是`Java 8`新增的方法,该方法尝试获取直接修饰该`Class`对象对应类的、指定类型的`Annotation`;如果该类型的注解不存在,则返回`null`。 |
| `Annotation[] getDeclaredAnnotations()`                      | 返回直接修饰该`Class`对应类的所有 `Annotation`               |
| `<A extends Annotation> A getAnnotationsByType(Class<A> annotationClass)` | 该方法的功能与前面介绍的`getAnnotation()`方法基本相似。但由于`Java 8`增加了重复注解功能,因此需要使用该方法获取修饰该类的、指定类型的多个`Annotation` |

**获取包含的内部类：**

如下方法用于访问该`Class`对象对应类包含的内部类。

| 方法                    | 描述                                        |
| :---------------------- | :------------------------------------------ |
| `Class<?> getClasses()` | 返回该`Class`对象对应类里包含的全部内部类。 |

 **获取所在的外部类**

如下方法用于访问该`Class`对象对应类所在的外部类。

| 方法                           | 描述                                |
| :----------------------------- | :---------------------------------- |
| `Class<?> getDeclaringClass()` | 返回该`Class`对象对应类所在的外部类 |

**访问该Class对象对应类所实现的接口**

如下方法用于访问该`Class`对象对应类所实现的接口

| 方法                          | 描述                                      |
| :---------------------------- | :---------------------------------------- |
| `Classs<?>[] getInterfaces()` | 返回该`Class`对象对应类所实现的全部接口。 |

**获取所继承的父类**

如下几个方法用于访问该`Class`对象对应类所继承的父类。

| 方法                               | 描述                                         |
| :--------------------------------- | :------------------------------------------- |
| `Class<? super T> getSuperClass()` | 返回该`Class`对象对应类的超类的`Class`对象。 |

**获取修饰符 所在包 类名等基本信息**

如下方法用于获取`Class`对象对应类的修饰符、所在包、类名等基本信息。

| 方法                     | 描述                                                         |
| :----------------------- | :----------------------------------------------------------- |
| `int getModifiers()`     | 返回此类或接口的所有修饰符。修饰符由 `public`、 `protected`、 `private`、`final`、`static`、 `abstract`等对应的常量组成,返回的整数应使用 `Modifier`工具类的方法来解码,才可以获取真实的修饰符。 |
| `Package getPackage()`   | 获取此类的包。                                               |
| `String getName()`       | 以字符串形式返回此`Class`象所表示的类的名称。                |
| `String getSimpleName()` | 以字符串形式返回此`Class`对象所表示的类的简称。              |

**判断该类是否为接口 枚举 注解类型**

除此之外, `Class`对象还可调用如下几个判断方法来判断该类是否为接口、枚举、注解类型等。

| 方法                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `boolean isAnnotation()`                                     | 返回此`Class`对象是否表示一个注解类型(由`@interface`定义)    |
| `boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)` | 判断此`Class`对象是否使用了`Annotation`修饰                  |
| `boolean isAnonymousClass()`                                 | 返回此`Class`对象是否是一个匿名类。                          |
| `boolean isArray()`                                          | 返回此`Class`对象是否表示一个数组类。                        |
| `boolean isEnum()`                                           | 返回此`Class`象是否表示一个枚举(由`enum`关键字定义)。        |
| `boolean isInterface()`                                      | 返回此 `Class`对象是否表示一个接口(使用 `interface`定义)。   |
| `boolean isInstance(Object object)`                          | 判断`object`是否是此`Class`对象的实例,该方法可以完全代替`instanceof`操作符。 |



上面的多个`getMethod()`方法和`getConstructor()`方法中，都需要传入多个类型为Class<?>的参数，用于获取指定的方法或指定的构造器。关于这个参数的作用，假设某个类包含如下三个info方法签名。

- `public void info()`
- `public void info(String str)`
- `public void info(String str, Integer num)`

这三个同名方法属于重载，它们的方法名相同，但参数列表不同。在Java语言中要确定一个方法光有方法名是不行的，如果仅仅只是指定Info方法——实际上可以是上面三个方法中的任意一个！如果需要确定一个方法，则应该由方法名和形参列表来确定，但形参没有任何实际意义，所以只能由形参类型来确定。例如想指定第二个info方法，则必须指定方法名为info，形参类别为String.class——因此在程序中获取该方法使用如下代码：

```
//前一个参数指定方法名，后面的个数可变的Class参数指定形参类型列表。
clazz.getMethod("info", String.class)
```

如果需要获取第三个info方法，则使用如下代码：

```
//前一个参数指定方法名，后面的个数可变的Class参数指定形参类型列表
clazz.getMethod("info", String.class, Integer.class)
```

获取构造器时无须传入构造器名——同一个类的所有构造器的名字都是相同的，所以要确定一个构造器只要指定形参列表即可。

下面程序示范了如何通过该Class对象来获取对应类的详细信息。

```java
package reflect;

import java.util.*;
import java.lang.reflect.*;
import java.lang.annotation.*;

// 定义可重复注解
@Repeatable(Annos.class)
@interface Anno {}
@Retention(value=RetentionPolicy.RUNTIME)
@interface Annos {
    Anno[] value();
}
// 使用4个注解修饰该类
@SuppressWarnings(value="unchecked")
@Deprecated
// 使用重复注解修饰该类
@Anno
@Anno
public class ClassTest
{
    // 为该类定义一个私有的构造器
    private ClassTest() {
    }

    // 定义一个有参数的构造器
    public ClassTest(String name) {
        System.out.println("执行有参数的构造器");
    }

    // 定义一个无参数的info方法
    public void info() {
        System.out.println("执行无参数的info方法");
    }

    // 定义一个有参数的info方法
    public void info(String str) {
        System.out.println("执行有参数的info方法"
                + "，其str参数值：" + str);
    }

    // 定义一个测试用的内部类
    class Inner{
    }



    public static void main(String[] args) throws Exception {
        // 下面代码可以获取ClassTest对应的Class
        // 获取该Class对象所对应类的全部构造器
        Class<ClassTest> clazz = ClassTest.class;
        Constructor[] ctors = clazz.getDeclaredConstructors();
        System.out.println("ClassTest的全部构造器如下：");
        for (Constructor c : ctors) {
            System.out.println(c);
        }

        // 获取该Class对象所对应类的全部public构造器
        Constructor[] publicCtors = clazz.getConstructors();
        System.out.println("ClassTest的全部public构造器如下：");
        for (Constructor c : publicCtors) {
            System.out.println(c);
        }

        // 获取该Class对象所对应类的全部public方法
        Method[] mtds = clazz.getMethods();
        System.out.println("ClassTest的全部public方法如下：");
        for (Method md : mtds) {
            System.out.println(md);
        }

        // 获取该Class对象所对应类的指定方法
        System.out.println("ClassTest里带一个字符串参数的info()方法为：" + clazz.getMethod("info" , String.class));

        // 获取该Class对象所对应类的上的全部注解
        Annotation[] anns = clazz.getAnnotations();
        System.out.println("ClassTest的全部Annotation如下：");
        for (Annotation an : anns) {
            System.out.println(an);
        }
        System.out.println("该Class元素上的@SuppressWarnings注解为：" + Arrays.toString(clazz.getAnnotationsByType(SuppressWarnings.class)));
        System.out.println("该Class元素上的@Anno注解为：" + Arrays.toString(clazz.getAnnotationsByType(Anno.class)));

        // 获取该Class对象所对应类的全部内部类
        Class<?>[] inners = clazz.getDeclaredClasses();
        System.out.println("ClassTest的全部内部类如下：");
        for (Class c : inners) {
            System.out.println(c);
        }

        // 使用Class.forName方法加载ClassTest的Inner内部类
        // 通过getDeclaringClass()访问该类所在的外部类
        Class inClazz = Class.forName("reflect.ClassTest$Inner");
        System.out.println("inClazz对应类的外部类为：" + inClazz.getDeclaringClass());
        System.out.println("ClassTest的包为：" + clazz.getPackage());
        System.out.println("ClassTest的父类为：" + clazz.getSuperclass());
    }
}
```

运行结果

```java
/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=62107:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/tools.jar:/Users/shadowflow/code/java/test/target/classes:/Users/shadowflow/.m2/repository/javassist/javassist/3.12.1.GA/javassist-3.12.1.GA.jar reflect.ClassTest
ClassTest的全部构造器如下：
private reflect.ClassTest()
public reflect.ClassTest(java.lang.String)
ClassTest的全部public构造器如下：
public reflect.ClassTest(java.lang.String)
ClassTest的全部public方法如下：
public void reflect.ClassTest.info()
public void reflect.ClassTest.info(java.lang.String)
public static void reflect.ClassTest.main(java.lang.String[]) throws java.lang.Exception
public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
public final void java.lang.Object.wait() throws java.lang.InterruptedException
public boolean java.lang.Object.equals(java.lang.Object)
public java.lang.String java.lang.Object.toString()
public native int java.lang.Object.hashCode()
public final native java.lang.Class java.lang.Object.getClass()
public final native void java.lang.Object.notify()
public final native void java.lang.Object.notifyAll()
ClassTest里带一个字符串参数的info()方法为：public void reflect.ClassTest.info(java.lang.String)
ClassTest的全部Annotation如下：
@java.lang.Deprecated()
@reflect.Annos(value=[@reflect.Anno(), @reflect.Anno()])
该Class元素上的@SuppressWarnings注解为：[]
该Class元素上的@Anno注解为：[@reflect.Anno(), @reflect.Anno()]
ClassTest的全部内部类如下：
class reflect.ClassTest$Inner
inClazz对应类的外部类为：class reflect.ClassTest
ClassTest的包为：package reflect
ClassTest的父类为：class java.lang.Object

Process finished with exit code 0

```

由此可见Class提供的功能非常丰富，值得指出的是，虽然定义ClassTest类时使用了@SuppressWarning注解，但程序运行时无法分析出该类里包含的该注解，这是因为@SuppressWarnings使用了@Retention(value=SOURCE)修饰，这表明@SuppressWarnings只能保存在源码级别上，而通过ClassTest.class获取该类运行时Class对象，所以程序无法访问到@SuppressWarnings注解。

通过Class对象可以得到大量的Method、Constructor、Field等对象，这些对象分别代表该类所包括的方法、构造器和成员变量等，程序还可以通过这些对象来执行实际的功能，例如调用方法、创建实例。



### 3.3 Java 8 新增的方法参数反射

Java 8 在java.lang.reflect包下新增了一个Executable抽象基类，该对象代表可执行的类成员，该类派生了Constructor、Method两个子类。

Executable基类提供了大量方法来获取修饰该方法或构造器的注解信息；还提供了`isVarArgs()`方法用于判断该方法或构造器是否包含数量可变的形参，以及通过`getModifiers`()`方法来获取该方法或构造器的修饰符。除此之外，Executable提供了如下两个方法来**获取该方法或参数的形参个数及形参名**。

| 方法                          | 描述                           |
| :---------------------------- | :----------------------------- |
| `int getParameterCount()`     | 获取该构造器或方法的形参个数   |
| `Parameter[] getParameters()` | 获取该构造器或方法的所有形参。 |

**获取参数信息：**

上面第二个方法返回了一个`Parameter`数组, `Parameter`也是`Java 8`新增的`API`,每个`Parameter`对象代表方法或构造器的一个参数。 `Parameter`也提供了大量方法来获取声明该参数的泛型信息,还提供了如下常用方法来获取参数信息。

| 方法                          | 描述                                                      |
| :---------------------------- | :-------------------------------------------------------- |
| `getModifiers()`              | 获取修饰该形参的修饰符。                                  |
| `String getName()`            | 获取形参名                                                |
| `Type getParameterizedType()` | 获取带泛型的形参类型。                                    |
| `Cass<?> getType()`           | 获取形参类型                                              |
| `boolean isNamePresent()`     | 该方法返回该类的`class`文件中是否包含了方法的形参名信息。 |
| `boolean isVarArgs()`         | 该方法用于判断该参数是否为个数可变的形参。                |

需要指出的是，使用javac命令编译源文件时，默认生成的class文件并不包含方法的形参名信息，因此调用`isNamePresent()`方法将会返回false，调用`getName()`方法也不能得到该参数的形参名。如果希望javac命令编译Java源文件时可以保留形参信息，则需要为该命令指定`-parameters`选项

下面程序示范Java8的方法参数反射功能。

```java
package reflect.loader;


import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.List;

class Test {
    public void replace(String str, List<String> list){}
}

public class MethodParameterTest {
    public static void main(String[] args) throws NoSuchMethodException {
        //获取Test的calss对象
        Class<Test> clazz = Test.class;
        //获取Test类的带两个参数的replace()方法
        Method replace = clazz.getMethod("replace", String.class, List.class);
        //获取指定方法的参数个数
        System.out.println("replace方法参数个数：" + replace.getParameterCount());
        //获取replace的所有参数信息
        Parameter[] parameters = replace.getParameters();
        int index = 1;
        for (Parameter p : parameters) {
            if (p.isNamePresent()) {
                System.out.println("---第" + index++ + "个参数信息---");
                System.out.println("参数名：" + p.getName());
                System.out.println("形参类型：" + p.getType());
                System.out.println("泛型类型：" + p.getParameterizedType());
            }
        }
    }
}

```

上面的程序定义了一个包含简单的Test类，该类中包含一个`replace(String str, List<String>list)`方法，程序中获取了该方法，接下来分别打印该方法的形参名，形参类型和泛型信息。

直接运行的结果为：

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=63973:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/tools.jar:/Users/shadowflow/code/java/test/target/classes:/Users/shadowflow/.m2/repository/javassist/javassist/3.12.1.GA/javassist-3.12.1.GA.jar reflect.loader.MethodParameterTest
replace方法参数个数：2
```

原因是打印的值位于`p.isNamePresent()`条件为true的执行体内，也就是只有当该类的class文件中包含形参名信息时，程序才会执行条件体内的打印代码。因此需要使用如下命令来编译该程序：

```
[shadowflow@ShadowOS loader]% javac -parameters -d . ./MethodParameterTest.java
```

上面的-parameters选项用于控制javac命令保留方法形参名信息。

运行：

```
[shadowflow@ShadowOS loader]% java reflect.loader.MethodParameterTest                              
replace方法参数个数：2
---第1个参数信息---
参数名：str
形参类型：class java.lang.String
泛型类型：class java.lang.String
---第2个参数信息---
参数名：list
形参类型：interface java.util.List
泛型类型：java.util.List<java.lang.String>
[shadowflow@ShadowOS loader]% 
```



## 4. 使用反射生成并操作对象

Class对象可以获得该类里的方法（由Method对象表示）、构造器（由Constructor对象表示）、成员变量（由Field对象表示），这三个类都位于java.lang.reflect包下，并实现了java.lang.reflect.Member接口。程序可以通过Method对象来执行对应的方法，通过Constructor对象来调用对应的构造器创建实例，通过Field对象直接访问并修改对象的成员变量值。

###  4.1 创建对象

通过反射来生成对象需要先使用Class对象获取指定的Constructor对象，再调用Constructor对象的`newInstance()`方法来创建该Class对象对应的实例。通过这种方式可以选择使用指定的构造器来创建实例。

在很多Java EE框架中都需要根据配置文件信息来创建Java对象，从配置文件读取的只是某个类的字符串类名，程序需要根据该字符串来创建对应的实例，就必须使用反射。

下面的程序实现了一个简单的对象池，该对象池会根据配置文件读取key-value对，然后创建这些对象，并将这些对象放入一个HashMap中

ObjectPoolFactory.java

```java
import java.io.FileInputStream;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

public class ObjectPoolFactory {
    // 定义一个对象池，前面的对象名，后面是实际对象
    private Map<String, Object> objectPool = new HashMap<>();

    // 定义一个创建对象的方法
    private Object createObject(String clazzName) throws Exception{
        final Class<?> clazz = Class.forName(clazzName);
        return clazz.getConstructor().newInstance();
    }

    // 该方法根据指定文件来初始化对象池
    public void initPool(String fileName) {
        try(FileInputStream fis = new FileInputStream(fileName)){
            Properties props = new Properties();
            props.load(fis);
            for(String name : props.stringPropertyNames()){
                objectPool.put(name, createObject(props.getProperty(name)));
            }
        } catch (Exception ex) {
            System.out.println("读取" + fileName + "异常");
        }
    }
    public Object getObject(String name) {
        return objectPool.get(name);
    }

    public static void main(String[] args) {
        final ObjectPoolFactory pf = new ObjectPoolFactory();
        pf.initPool("obj.txt");
        System.out.println(pf.getObject("a"));
        System.out.println(pf.getObject("b"));
    }
}
```

上面的程序中`createObject()`方法里`Class.forName`根据字符串获取对应的class对象，然后用`newInstance()`方法即可创建一个Java对象。`initPool()`会读取属性文件，对属性文件中每个key-value对创建一个Java对象，其中value是该Java对象的实现类，而key是该Java对象放入对象池中的名字。为该程序提供如下属性配置文件。

obj.txt

```
a=java.util.Date
b=javax.swing.JFrame
```

ObjectPoolFactory.java不写package，使用命令运行

```
[shadowflow@ShadowOS reflect]% javac ObjectPoolFactory.java                     
[shadowflow@ShadowOS reflect]% java ObjectPoolFactory                 
Mon Jan 17 10:13:03 CST 2022
javax.swing.JFrame[frame0,0,23,0x0,invalid,hidden,layout=java.awt.BorderLayout,title=,resizable,normal,defaultCloseOperation=HIDE_ON_CLOSE,rootPane=javax.swing.JRootPane[,0,0,0x0,invalid,layout=javax.swing.JRootPane$RootLayout,alignmentX=0.0,alignmentY=0.0,border=,flags=16777673,maximumSize=,minimumSize=,preferredSize=],rootPaneCheckingEnabled=true]

```

当main函数执行到(1)方法的时候会输出时间。这表明对象池中已经有一个名为a的对象，该对象是一个java.util.Date对象。执行到(2)处的时候，将看到输出一个JFrame对象。

如果不想利用默认构造器来创建Java对象，而想利用指定的构造器来创建Java对象，则需要利用Constructor对象，每个Constructor对象对应一个构造器。为了利用指定的构造器来创建Java对象，需要如下三个步骤。

1. 获取该类的class对象
2. 利用Class对象的`getConstructor()`方法来获取指定的构造器。
3. 调用Constructor的`newInstance()`方法来创建Java对象

下面的程序利用反射来创建一个JFrame对象，而且使用指定的构造器。

```java
package reflect;

import java.lang.reflect.Constructor;

public class CreateJFrame {
    public static void main(String[] args) throws Exception {
        //获取JFrame对应的Class对象
        Class<?> jframeClazz = Class.forName("javax.swing.JFrame");
        //获取JFrame中带一个字符串参数的构造器
        Constructor ctor = jframeClazz.getConstructor(String.class);
        //调用Constructor的newInstance方法创建对象
        Object obj = ctor.newInstance("测试窗口");
        //输出JFrame对象
        System.out.println(obj);
    }
}

```

上面的程序中获取了JFrame类的指定构造器，如果要唯一的确定某类的构造器，只要指定构造器的形参列表即可。在获取构造器的时候传入一个String类型，即表明想获取只有一个字符串的参数的构造器。

程序中使用指定构造器的`newInstance()`方法来创建一个Java对象，当调用Constructor对象的`newInstance()`方法时通常需要传入参数，因为调用Construcot的`newInstance()`方法实际上等于调用它对应的构造器，传给`newInstance()`方法的参数将作为对应构造器的参数。

对于上面的程序中已知java.swing.JFrame类的情形，通常没有必要使用反射来创建该对象，毕竟通过反射创建对象的性能要稍微低一点。实际上，只有当程序需要动态创建某个类的对象时才会考虑使用反射，通常在开发通用性比较广的框架、基础平台时可能会大量使用反射。



### 4.2 调用方法

当获得某个类的class对象后，就可以通过该Class对象的`getMethods()`方法或者`getMethod()`方法来获取全部方法或者指定方法——这两个方法的返回值是Method数组，或者Method对象。

每个Method对象对应一个方法，获得Method对象后，程序就可以通过该Method来调用它对应的方法。在Method里包含一个`invoke()`方法，该方法的签名如下

- Object invoke(Object obj, Object... args): 该方法的obj是执行该方法的主调，后面的args是执行该方法时传入该方法的实参。

下面的程序对前面的对象池工厂进行加强，允许在配置文件中增加配置对象的成员变量的值，对象池工厂会读取为该对象配置的成员变量值，并利用该对象对应的setter方法设置成员变量的值。



```java
package reflect;

import jdk.internal.util.xml.impl.Input;

import java.io.*;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;
import java.util.Properties;

public class ExtendedObjectPoolFactory {
    //定义一个对象池，前面是对象名，后面是实际对象
    private Map<String, Object> objectPool = new HashMap<>();
    private Properties config = new Properties();

    //从指定属性文件中初始化Properties对象
    public void init(String fileName) {
        try(FileInputStream fis = new FileInputStream(fileName)) {
            config.load(fis);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //定义一个创建对象的方法
    //该方法只传入一个字符串类名，程序可以根据该类名生成Java对象
    private Object createObject(String clazzName) throws Exception {
        //根据字符串来获取对应的Class对象
        Class<?> clazz = Class.forName(clazzName);
        //使用clazz对应类的默认构造器创建实例
        return clazz.getConstructor().newInstance();
    }

    //该方法根据指定文件来初始化对象池
    //它会根据配置文件来创建对象
    public void initPool() throws Exception {
        for (String name : config.stringPropertyNames()) {
            //每读取一个key-value对，如果key中不包含百分号（%）
            //这就表明是根据value来创建一个对象
            //调用createObject创建对象，并将对象添加到对象池中
            if (!name.contains("%")) {
                objectPool.put(name, createObject(config.getProperty(name)));
            }
        }
    }

    //该方法将会根据属性文件来调用指定对象的setter方法
    public void initProperty() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        for (String name : config.stringPropertyNames()) {
            //每取出一对key-value对，如果key中包含百分号
            //即可认为该key用于控制调用对象的setter方法设置值
            //百分号前半为对象名字，后半控制stter方法名
            if (name.contains("%")) {
                //将配置文件的key按%分割
                String[] objAndProp = name.split("%");
                //取出调用setter方法的参数值
                Object target = getObject(objAndProp[0]);
                //获取stter方法名:set + "首字母大写" + 剩下部分
                String mtdName = "set" + objAndProp[1].substring(0, 1).toUpperCase() + objAndProp[1].substring(1);
                //通过target的getClass()获取它的实现类所对应的Class对象
                Class<?> targetClass = target.getClass();
                //获取希望调用的stter方法
                Method mtd = targetClass.getMethod(mtdName, String.class);
                //通过Method的invoke方法执行stter方法
                //将config.getProperty(name)的值作为调用setter方法的参数
                mtd.invoke(target, config.getProperty(name));

            }
        }
    }

    public Object getObject(String name) {
        //从objectPool中取出指定name对应的对象
        return objectPool.get(name);
    }

    public static void main(String[] args) throws Exception {
        ExtendedObjectPoolFactory epf = new ExtendedObjectPoolFactory();

        epf.init("/Users/shadowflow/code/java/test/src/main/java/reflect/extObj.txt");
        epf.initPool();
        epf.initProperty();
        System.out.println(epf.getObject("a"));
    }


}

```

上面程序中`initProperty()`方法里`Method mtd = targetClass.getMethod(mtdName, String.class)`获取目标类包含一个String参数的stter方法，`mtd.invoke(target, config.getProperty(name))`通过调用Method的`invoke()`方法来执行该setter方法，该方法执行完后，就相当于执行了目标对象的setter方法。为上面程序配置文件

```
a=javax.swing.JFrame
b=javax.swing.JLabel
#set the title of a
a%title=Test Title
```

上面配置文件中的a%title行表明希望调用a对象的`setTitle()`方法，调用该方法的参数值为Test Title。编译、运行上面的ExtendedObjectPoolFactory.java程序，可以看到输出一个JFrame窗口，该窗口的标题为Test Title。

当通过Method的`invoke()`方法来调用对应的方法时，Java会要求程序必须有调用该方法的权限。如果程序确实需要调用某个对象的private方法，则可以先调用Method对象的如下方法。

> setAccessible(boolean flag): 将Method对象的accessible设置为指定的布尔值。值为true，指示该Method在使用是应该取消Java语言的访问权限检查；值为false，则指示该Method在使用是要实施Java语言的访问权限检查。

实际上，`setAccessible()`方法并不属于Method，而是属性它的父类AccessibleObject。因此Method、Constructor、Field都可调用该方法，从而实现通过反射来调用private方法、private构造器和private成员变量。也就是说，它们可以通过调用该方法来取消访问权限检查，通过反射即可访问private成员。



### 4.3 访问成员变量值

通过Class对象的`getFields()`或`getField()`方法可以获取该类所包括的全部成员变量或指定成员变量。Field提供了如下两组方法来读取或设置成员变量值。

- getXxx(Object obj): 获取obj对象的该成员变量的值。此处的Xxx对应8种基本类型，如果该成员变量的类型是引用类型，则取消get后面的Xxx。
- setXxx(Object obj, Xxx val): 将obj对象的该成员变量设置成val值。此处的Xxx对应8种基本类型，如果该成员变量的类型是引用类型，则取消set后面的Xxx。

使用这两个方法可以随意地访问对象的所有成员变量，包括private修饰的成员变量。

```java
package reflect;

import java.lang.reflect.Field;

public class FieldTest {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        //创建一个Person对象
        Person p = new Person();
        //获取Person类对应的Class对象
        Class<Person> personClazz = Person.class;
        //获取Person的名为name的成员变量
        //使用getDeclaredField()方法表明可获取各种访问控制符的成员变量
        Field nameField = personClazz.getDeclaredField("name");
        //设置通过反射访问该成员变量时取消访问权限检查
        nameField.setAccessible(true);
        //调用set()方法为p对象的name成员变量设置值
        nameField.set(p, "shadowtest");
        //获取Person类名为age的成员变量
        Field ageField = personClazz.getDeclaredField("age");
        //设置通过反射访问该成员变量时取消访问权限检查
        ageField.setAccessible(true);
        //调用setInt()方法为P对象的age成员变量设置值
        ageField.setInt(p, 30);
        System.out.println(p);
    }

}

class Person {
    private String name;
    private int age;

    public String toString(){
        return "Person[name:" + name + ", age:" + age + " ]";
    }
}

```

上面程序中先定义了一个Person类，该类里包含两个private成员变量，该类里包含两个private成员变量：name和age，在通常情况下，这两个成员变量只能在Person类里访问。但本程序FieldTest的`main()`方法中通过反射修改了Person对象的name、age两个成员变量的值。使用`getDeclaredField()`方法获取了名为name的成员变量，注意此处不是`getField()`方法，因为`getField()`方法只能获取public访问控制的成员变量，而`getDeclaredField()`方法则可以获取所有的成员变量。



### 4.4 操作数组

在java.lang.reflect包下还提供了一个Array类，Array对象可以代表所有的数组。程序可以通过使用Array来动态地创建数组，数组操作元素等。

Array提供了如下几类方法:

- static Object newInstance(Class<?> compnentType, int... length): 创建一个具有指定元素类型、指定维度的新数组。
- static xxx getXxx(Object array, int index): 返回array数组中第index个元素。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变为get(Object array, int index).
- `static void setXxx(Object array, int index, xxx val):`将array数组中第index个元素的值设为val。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变成set(Object array, int index, Object val).

下面的程序示范了如何使用Array来生成数组，为指定数组元素赋值，并获取指定数组元素的方式。

```java
package reflect;

import java.lang.reflect.Array;

public class ArrayTest1 {
    public static void main(String[] args) {
        //创建一个元素类型为String，长度为10的数组
        Object arr = Array.newInstance(String.class, 10);
        //依次为arr数组中index为5、6的元素赋值
        Array.set(arr, 5, "shadowtest");
        Array.set(arr, 6, "shadowtestx");
        //依次取出arr数组中index为5、6的元素的值
        Object book1 = Array.get(arr,5);
        Object book2 = Array.get(arr,6);
        //输出arr数组中index为5、6的元素的值
        System.out.println(book1);
        System.out.println(book2);

    }
}

```

上面的程序中代码分别是通过Array创建数组，为数组元素设置值，访问数组元素的值，程序通过Array就可以动态创建并操作数组。



## 5. 使用反射生成JDK动态代理

在Java的java.lang.reflect包下提供了一个Proxy类和一个InvocationHandler接口，通过使用这个类和接口可以生成JDK动态代理类或动态代理对象。

### 5.1 使用Proxy和InvocationHandler创建动态代理

proxy提供了用于创建动态代理类和代理对象的静态方法，它也是所有动态代理类的父类。如果在程序中为一个或多个接口动态地生成实现类，就可以使用Proxy来创建动态代理类：如果需要为一个或多个接口动态的创建实例，也可以使用Proxy来创建动态代理实例。

Proxy提供了如下两个方法来创建动态代理类和动态代理实例。

- `static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces):`

  创建一个动态代理类所对应的Class对象，该代理类将实现interface所指定的多个接口。第一个ClassLoader参数指定生成动态代理类的类加载器。

- `static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h):`

  直接创建一个动态代理对象，该代理对象的实现类实现了interfaces指定的系列接口，执行代理对象的每个方法时都会被替换执行InvocationHandler对象的invoke方法。  

实际上，即使采用第一个方法生成动态代理类之后，如果程序需要通过该代理类来创建对象，依然需要传入一个InvocationHandler对象。也就是说，系统生成的每个代理对象都有一个与之关联的InvocationHandler对象。

计算机是很”蠢“的，当程序使用反射方式为指定接口生成系列动态代理对象时，这些动态代理对象的实现类实现了一个或多个接口。动态代理对象就需要实现一个或多个接口里定义的所有方法，但问题是：系统怎么知道如何实现这些方法？这个时候就轮到InvocationHandler对象登场了——当执行动态代理对象里的方法时，实际上会替换成调用InvocationHandler对象的invoke方法。

程序中可以采用先生成一个动态代理类，然后通过动态代理类来创建代理对象的方式生成一个动态代理对象。代码片段如下：

```
//创建一个InvocationHandler对象
InvocationHandler handler = new MyInvocationHandler(...);
//使用Proxy生成一个动态代理类proxyClass
Class proxyClass = Proxy.getProxyClass(Foo.class.getClassLoader(), new Class[] {Foo.class})
//获取proxyClass类中带一个InvocationHandler参数的构造器
constructor ctor = proxyClass.getConstructor(new Class[] {InvocationHandler.class});
//调用ctor的newInstance方法来创建动态实例
Foo f = (Foo)ctor.newInstance(new Object[]{handler});
```

上面的代码也可以简化成如下代码：

```
//创建一个InvocationHandler对象
InvocationHandler handler = new MyInvocationHandler(...);
//使用Proxy直接生成一个动态代理
Foo f = (Foo)Proxy.newProxyInstance(Foo.class.getClassLoader(), new Class[]{Foo.class}, handler);
```

下面程序示范了使用Proxy和InvocationHandler来生成动态代理对象。

```java
package reflect;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

interface Person {
    void walk();
    void sayHello(String name);
}

class MyInvokationHandler implements InvocationHandler {
    /*
    执行动态代理对象的所有方法时，都会被替换成执行如下的invoke方法
    其中：
        proxy: 代表动态代理对象
        method: 代表正在执行的方法
        args: 代表调用目标方法时传入的实参
     */
    public Object invoke(Object proxy, Method method, Object[] args) {
        System.out.println("-----正在执行的方法：" + method);
        if (args != null) {
            System.out.println("下面是执行该方法时传入的实参为：");
            for (Object val : args) {
                System.out.println(val);
            }
        } else {
            System.out.println("调用该方法没有实参！");
        }return null;
    }
}
public class ProxyTest {
    public static void main(String[] args) {
        //创建一个InvocationHandler对象
        InvocationHandler handler = new MyInvokationHandler();
        //使用指定的InvocationHandler来生成一个动态代理对象
        Person p = (Person) Proxy.newProxyInstance(Person.class.getClassLoader(), new Class[]{Person.class}, handler);
        //调用动态代理对象的walk()和sayHello()方法
        p.walk();
        p.sayHello("孙悟空");
    }
}

```

上面的程序首先提供了一个Person接口，该接口包含了`walk()`和`sayHello()`两个抽象方法，接着定义了一个简单的InvocationHandler实现类，定义该实现类时需要重写`invoke()`方法中的三个参数，解释如下

- proxy: 代表动态代理对象
- method: 代表正在执行的方法
- args: 代表调用目标方法时传入的实参。

上面程序中`InvocationHandler handler = new MyInvokationHandler();`创建了一个InvocationHandler对象，`Person p = (Person) Proxy.newProxyInstance(Person.class.getClassLoader(), new Class[]{Person.class}, handler);`根据InvocationHandler对象创建了一个动态代理对象。运行上面的程序

```
-----正在执行的方法：public abstract void reflect.Person.walk()
调用该方法没有实参！
-----正在执行的方法：public abstract void reflect.Person.sayHello(java.lang.String)
下面是执行该方法时传入的实参为：
孙悟空
```

运行结果可以看出，不管程序时执行代理对象的`walk()`方法，还是执行代理对象的`sayHello()`方法，实际上都是执行InvocationHandler对象的`invoke()`方法

看完上面的示例，可能有人会觉得这个程序没有太大的实用价值，难以理解Java动态代理的魅力。实际上，在普通编程过程中，确实无须实用动态代理，但在编写框架或底层基础代码时，动态代理的作用就非常大。



### 5.2 动态代理和AOP

根据前面介绍的Proxy和InvocationHandler，实在很难看出这种动态代理的优势。下面介绍一种更实用的动态代理机制。

开发实际应用的软件系统是，通常会存在相同代码段重复出现的情况，在这种情况下，对于许多刚开始从事软件开发的人而言，他们的做法是：选中那些代码，一路”复制“、”粘贴“，立即实现了系统功能，如果仅仅从软件功能上来看，他们确实已经完成了软件开发。

如果有100甚至1000个地方使用了相同的代码，如果要修改、维护这段代码将会变成噩梦。在这种情况下，大部分有经验的开发者都会将这段代码定义成一个方法，然后让另外代码直接调用即可。但是这又面对一个问题，如果有一百个地方都调用了一个方法，都硬编码了，同样不优雅，需要解耦，可以通过动态代理来避免硬编码。

由于JDK动态代理只能为接口创建动态代理，所以下面先提供一个Dog接口，该接口代码非常简单，仅仅在该接口里定义了两个方法

```java
package reflect;

public interface Dog {
    //info()方法申明
    void info();
    //run()方法声明
    void run();
}

```

上面接口里只是简单地定义了两个方法，并未提供方法实现。如果直接使用Proxy为该接口创建动态代理对象，则动态代理对象的所有方法的执行效果又将完全一样。实际情况通常是，软件系统会为该Dog接口提供一个或多个实现类，此处先提供一个简单的实现类：GunDog

```java
package reflect;

public class GunDog implements Dog{
    @Override
    public void info() {
        System.out.println("hunt dog");
    }

    @Override
    public void run() {
        System.out.println("run quick");
    }
}

```

上面代码没有丝毫的特别之处，该Dog的实现类仅仅为每个方法提供了一个简单实现。再看需要实现的功能：程序执行`info()`、`run()`方法时能调用某个通用方法，但又不想以硬编码方式调用该方法。下面提供一个DogUtil类，该类里包含两个通用方法。

```java
package reflect;

public class DogUtil {
    //第一个拦截器方法
    public void method1() {
        System.out.println("====模拟第一个通用方法===");
    }
    public void method2() {
        System.out.println("===模拟通用方法二==");
    }
}

```

借助于Proxy和InvocationHandler就可以实现——当程序调用`info()`方法和`run()`方法时，系统可以”自动“将`method1()`和`method2()`两个通用方法插入`info()`和`run()`方法中执行。

这个程序的关键在于下面的MyInvokationHandler类，该类是一个InvocationHandler实现类，该实现类的`invoke()`方法将会作为代理对象的方法实现。

```java
package reflect;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MyInvokationHandler implements InvocationHandler {
    //需要被代理的对象
    private Object target;
    public void setTarget(Object target) {
        this.target = target;
    }

    //执行动态代理对象的所有方法时，都会被替换成执行如下的invoke方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        DogUtil du = new DogUtil();
        //执行DogUtil对象中的method方法
        du.method1();
        //以target作为主调来执行method方法
        Object result = method.invoke(target, args);
        //执行DogUtil对象中的method2方法
        du.method2();
        return result;
    }
}

```

上面的程序实现了`invoke()`方法时包含一行关键代码`Object result = method.invoke(target, args);`，这行代码通过反射以target作为主调来执行method方法，这就是回调了target对象的原有方法。这行代码之前调用了DogUtil对象的`method1()`方法，之后调用了DogUtil对象的`method2()`方法。

下面再为程序提供了一个MyProxyFactory类，该对象专为指定的target生成动态代理实例。

```java
package reflect;

import java.lang.reflect.Proxy;

public class MyProxyFactory {
    //为指定的target生成动态代理对象
    public static Object getProxy(Object target) {
        //创建一个MyInvokationHandler对象
        MyInvokationHandler handler = new MyInvokationHandler();
        //为MyInvokationHandler设置target对象
        handler.setTarget(target);
        //创建并返回一个动态代理
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), handler);
    }
}

```

上面的动态代理工厂类提供了一个`getProxy()`方法，该方法为target对象生成一个动态代理对象，这个动态代理对象与target实现了相同的接口，所以具有相同的public方法——从这个意义上来看，动态代理对象可以当成target对象使用。当程序调用动态代理对象的指定方法时，实际上将变为执行MyInvocationHandler对象的`invoke()`方法。例如，调用动态代理对象的`info()`方法，其执行步骤如下：

1. 创建DogUtil实例
2. 执行DogUtil实例的`method1()`方法
3. 使用反射以target作为调用者执行`info()`方法
4. 执行DogUtil实例的`method2()`方法

看到上面的执行过程，读者应该已经发现：当使用动态代理对象来代替target对象时，代理对象的方法就实现了前面的要求——程序执行`info()`、`run()`、方法时自动”插入“`method1()`、`method2()`方法。<font color="red">如果有一百个地方在调用`info()`、`run()`方法时都需要调用`method1()`和`method2()`方法，我们就不用重复写调用方法的代码了，我们写了动态代理，那么就会在自动添加method方法，实现了功能的增强。其实这里我们可以发现，动态代理跟装饰器是有点类似的，装饰器的定义是：在不改变原有对象的基础上，给原始对象添加增强功能。都是功能增强，它们还是有区别的，装饰器是对之前的功能进行扩展，而动态代理添加的可能是完全无关的一个模块。</font>

下面提供一个主程序来测试中动态代理的效果。

```java
package reflect;

public class Test {
    public static void main(String[] args) {
        //创建一个原始的GunDog对象，作为target
        Dog target = new GunDog();
        //以指定的target来创建动态代理对象
        Dog dog = (Dog)MyProxyFactory.getProxy(target);
        dog.info();
        dog.run();
    }
}

```

上面程序中的dog对象实际上是动态代理对象，只是该动态代理对象也实现了Dog接口，所以也可以当成Dog对象使用。程序执行dog的`info()`和`run()`方法时，实际上会先执行DogUtil的`method1()`方法，再执行target对象的`info()`和`run()`方法，最后执行DogUtil的`method2()`方法。运行上面程程序会看到如下运行结果

```
====模拟第一个通用方法===
hunt dog
===模拟通用方法二==
====模拟第一个通用方法===
run quick
===模拟通用方法二==
```

从结果来看，不难发现采用动态代理可以非常灵活地实现解耦。通常而言，使用Proxy生成一个动态代理时，往往并不会凭空产生一个动态代理，这样没有太大的实际意义。通常都是为指定的目标对象生成动态代理。

这种动态代理在AOP（Aspect Orient Programming，面向切面编程）中被称为AOP代理，<font color="red">AOP代理可代替目标对象，AOP代理包含了目标对象的全部方法。但AOP代理中的方法与目标对象的方法存在差异：AOP代理里的方法可以在执行目标方法之前、之后插入一些通用处理。</font>



## 6. 反射和泛型

从JDK5以后，Java的Class类增加了泛型功能，从而允许使用泛型来限制Class类，例如，String.class的类型实际上是`Class<String>`。如果Class对应的类暂时未知，则使用Class<?>。通过在反射中使用泛型，可以避免使用反射生成的对象需要强制类型转换。

### 6.1 泛型和Class类

使用`Class<T>`泛型可以避免强制类型转换。例如，下面提供一个简单的对象工厂，该对象工厂可以根据指定类来提供该类的实例。

```java
package reflect;

public class CrazyitObjectFactory {
    public static Object getInstance(String clsName) throws Exception {
        //创建指定类的class对象
        Class cls = Class.forName(clsName);
        //返回该class对象创建的实例
        return cls.newInstance();
    }
}

```

上面的代码根据指定的字符串类型创建了一个新对象，但这个对象的类型是Object，因此当需要使用CrazyitObjectFactory的`getInstance()`方法来创建对象时，将会看到如下代码：

```
//获取实例后需要强制类型转换
Date d = (Date)Crazyit.getInstance("java.util.Date");
```

甚至出现如下代码：

```
JFrame f = (JFrame)Crazyit.getInstance("java.util.Date");
```

上面代码在编译时不会有任何问题，但运行时将抛出ClassCastException异常，因为程序试图将一个Date对象转换成JFrame对象。

如果将上面的CrazyitObjectFactory工厂类改写成使用泛型后的Class，就可以避免这种情况。

```java
package reflect;

import javax.swing.*;
import java.util.Date;

public class CrazyitObjectFactory2 {
    public static <T> T getInstance(Class<T> cls) throws Exception {
        return  cls.newInstance();
    }

    public static void main(String[] args) throws Exception {
        //获取实例后无须类型转换
        Date d = CrazyitObjectFactory2.getInstance(Date.class);
        JFrame f = CrazyitObjectFactory2.getInstance(JFrame.class);
    }
}
```

在上面程序的`getInstance()`方法中传入一个`Class<T>`参数，这是一个泛型化的Class对象，调用该Class对象的`newInstance()`方法将会返回一个T对象，如程序中粗体子代码所示。接下来当使用CrazyitObjectFactory2工厂类的`getInstance()`方法来产生对象时，无须使用强制类型转换，系统会执行更严格的检测，不会出现ClassCastException运行时异常。

前面介绍Array类来创建数组时，曾经看到如下代码：

```
//使用Array的newInstance方法来创建一个数组
Object arr = Array.newInstance(String.class, 10);
```

对于上面的代码其实使用并不是非常方便，因为`newInstance()`方法返回的确实是一个String[]数组，而不是简单的Object对象。如果需要将arr对象当成String[]数组使用，则必须使用强制类型转换——这是不安全的操作。

为了示范泛型的优势，可以对Array的`newInstance()`方法进行包装

```java
package reflect;

import java.lang.reflect.Array;

public class CrazyitArray {
    //对Array的newInstance方法进行包装
    @SuppressWarnings("unchecked")
    public static <T> T[] newInstance(Class<T> componentType, int length) {
        return (T[]) Array.newInstance(componentType, length); //(1)
    }

    public static void main(String[] args) {
        //使用CrazyitArray的newInstance()创建一维数组
        String[] arr = CrazyitArray.newInstance(String.class, 10);
        //使用CrazyitArray的newInstance()创建二维数组
        //在这种情况下，只要设置数组元素的类型时int[]即可
        int[][] intArr = CrazyitArray.newInstance(int[].class, 5);
        arr[5] = "shadowtest";
        //intArray是二维数组，初始化该数组的第二个数组元素
        //二维数组的元素必须是一维数组
        intArr[1] = new int[]{23, 12};
        System.out.println(arr[5]);
        System.out.println(intArr[1][1]);
    }
}

```

上面程序`newInstance()`方法对Array类提供的`newInstance()`方法进行了包装，将方法签名改成了`public static <T> T[] newInstance(Class<T> componentType, int length)`，这就保证程序通过该`newInstance()`方法创建数组时返回值就是数组对象，而不是Object对象，从而避免强制类型转换。

> 程序中会有一个unchecked编译告警，使用@SuppressWarnings("unchecked")来抑制这个告警信息

### 6.2 使用反射来获取泛型信息

通过指定类对应的Class对象，可以获得该类里包含的所有成员变量，不管该成员变量是使用private修饰，还是使用public修饰。获得了成员变量对应的Field对象后，就可以很容易地获得该成员变量的数据类型，即使用如下代码即可获得指定成员变量的类型。

```
//获取成员变量f的类型
Class<?> a = f.getType();
```

但这种方式只对普通类型的成员变量有效。如果该成员变量的类型是有泛型类型的类型，如`Map<String, Integer>`类型，则不能准确地得到该成员变量的泛型参数。

为了获得指定成员变量的泛型类型，应先使用如下方法来获取该成员变量的泛型类型。

```
//获得成员变量f的泛型类型
Type gType = f.getGenericType();
```

然后将Type对象强制类型转换为ParameterizedType类提供了如下两个方法。

- `getRawType():`返回没有泛型信息的原始类型
- `getActualTypeArguments():`返回泛型参数的类型

下面是一个获取泛型类型的完整程序。

```java
package reflect;

import java.lang.reflect.Field;
import java.lang.reflect.Parameter;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.Map;

public class GenericTest {
    private Map<String ,Integer> score;

    public static void main(String[] args) throws NoSuchFieldException {
        Class<GenericTest> clazz = GenericTest.class;
        Field f = clazz.getDeclaredField("score");
        //直接使用getType()取出类型只对普通类型的成员变量有效
        Class<?> a = f.getType();
        //下面将看到仅输出java.util.Map
        System.out.println("score的类型是：" +a);
        //获得成员变量f的泛型
        Type gType = f.getGenericType();
        //如果gType类型是ParameterizedType对象
        if(gType instanceof ParameterizedType) {
            //强制类型转换
            ParameterizedType pType = (ParameterizedType) gType;
            //获取原始类型
            Type rType = pType.getRawType();
            System.out.println("原始类型时：" + rType);
            //取得泛型类型的泛型参数
            Type[] tArgs = pType.getActualTypeArguments();
            System.out.println("泛型信息是：");
            for (int i = 0; i < tArgs.length; i++) {
                System.out.println(tArgs[i]);
            }

        }else {
            System.out.println("泛型获取出错");
        }

    }
}

```

运行结果如下

```
score的类型是：interface java.util.Map
原始类型时：interface java.util.Map
泛型信息是：
class java.lang.String
class java.lang.Integer
```

从上面的运行结果可以知道，`getType()`只能获取普通类型的成员变量数据类型；对于增加了泛型的成员变量，应该使用`getGenericType()`方法来获得其类型。

> Type也是Java.lang.reflect包下的一个接口，该接口代表所有类型的公共高级接口，Class是Type接口的实现类。Type包括原始类型、参数化类型、数组类型、类型变量和基本类型等。



## 7. 总结

本文介绍了Java反射的相关知识。从类的加载、初始化开始，深入介绍了Java类加载器的原理和机制。重点介绍了Class、Method、Field、Constructor、Type、ParameterizedType等类和接口的用法，包括动态代理创建Java实例和动态调用Java对象的方法。介绍的两个对象工厂示例就是Spring框架的核心。还介绍了Proxy和Invocation来创建JDK动态代理，并介绍了JDK动态代理和AOP之间的关系，这也是Java灵活性的重要方面。





## 8. 参考

- https://lanlan2017.github.io/JavaReadingNotes/712f16a6/
- 《疯狂Java讲义》

