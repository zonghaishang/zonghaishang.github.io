---
title: 理解arthas原理
subtitle: arthas属于线上问题诊断工具，主要解决如下几个场景问题,这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？..
cover: /images/arthas.png
author: 
  nick: 诣极
  link: https://github.com/zonghaishang
tags:
- arthas
categories:
- arthas
date: 2018-10-26 00:02:19
---


## 背景

本章不会讲解arthas全部细节，会围绕arthas功能介绍和核心原理进行展开。

通过本章的分析，主要涵盖以下几个内容：

- arthas功能介绍
- 动手调试arthas
- arthas原理解析

`arthas`属于线上问题诊断工具，主要解决如下几个场景问题：

- 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
- 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
- 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
- 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
- 是否有一个全局视角来查看系统的运行状况？
- 有什么办法可以监控到JVM的实时运行状态？

说明: `arthas`功能介绍参考自开源官网，关键原理点是自己挖代码分析，不当之处欢迎拍砖。

### arthas常用命令

#### watch实时监控

观察方法 test.arthas.TestWatch#doGet 执行的入参，仅当方法抛出异常时才输出。

```shell
$ watch test.arthas.TestWatch doGet {params[0], throwExp} -e
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 65 ms.
ts=2018-09-18 10:26:28;result=@ArrayList[
    @RequestFacade[org.apache.catalina.connector.RequestFacade@79f922b2],
    @NullPointerException[java.lang.NullPointerException],
]
```

### trace跟踪方法调用耗时

观察方法执行的时哪个方法调用比较慢:

<img src="https://zonghaishang.github.io/images/trace.png" class="pretty">

### stack跟踪方法调用堆栈

查看方法 test.arthas.TestStack#doGet 的调用堆栈：

```java
$ stack test.arthas.TestStack doGet
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 286 ms.
ts=2018-09-18 10:11:45;thread_name=http-bio-8080-exec-10;id=d9;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.ParallelWebappClassLoader@25131501
    @test.arthas.TestStack.doGet()
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:624)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:731)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:303)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:241)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:241)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
        at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:220)
        at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:110)
        ...
        at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:169)
        at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:103)
        at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:116)
        at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:451)
        at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1121)
        at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:637)
        at org.apache.tomcat.util.net.JIoEndpoint$SocketProcessor.run(JIoEndpoint.java:316)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
```

### sc查看类加载信息

查找JVM中已经加载的类：

```java
$ sc -d org.springframework.web.context.support.XmlWebApplicationContext
 class-info        org.springframework.web.context.support.XmlWebApplicationContext
 code-source       /Users/xxx/work/test/WEB-INF/lib/spring-web-3.2.11.RELEASE.jar
 name              org.springframework.web.context.support.XmlWebApplicationContext
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       XmlWebApplicationContext
 modifier          public
 annotation
 interfaces
 super-class       +-org.springframework.web.context.support.AbstractRefreshableWebApplicationContext
                     +-org.springframework.context.support.AbstractRefreshableConfigApplicationContext
                       +-org.springframework.context.support.AbstractRefreshableApplicationContext
                         +-org.springframework.context.support.AbstractApplicationContext
                           +-org.springframework.core.io.DefaultResourceLoader
                             +-java.lang.Object
 class-loader      +-org.apache.catalina.loader.ParallelWebappClassLoader
                     +-java.net.URLClassLoader@6108b2d7
                       +-sun.misc.Launcher$AppClassLoader@18b4aac2
                         +-sun.misc.Launcher$ExtClassLoader@1ddf84b8
 classLoaderHash   25131501
```

### thread查看哪些线程比较占cpu？他们到底在做什么？

```java
$ thread -n 3
"as-command-execute-daemon" Id=29 cpuUsage=75% RUNNABLE
    at sun.management.ThreadImpl.dumpThreads0(Native Method)
    at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:440)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand$1.action(ThreadCommand.java:58)
    at com.taobao.arthas.core.command.handler.AbstractCommandHandler.execute(AbstractCommandHandler.java:238)
    at com.taobao.arthas.core.command.handler.DefaultCommandHandler.handleCommand(DefaultCommandHandler.java:67)
    at com.taobao.arthas.core.server.ArthasServer$4.run(ArthasServer.java:276)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)

    Number of locked synchronizers = 1
    - java.util.concurrent.ThreadPoolExecutor$Worker@6cd0b6f8

"as-session-expire-daemon" Id=25 cpuUsage=24% TIMED_WAITING
    at java.lang.Thread.sleep(Native Method)
    at com.taobao.arthas.core.server.DefaultSessionManager$2.run(DefaultSessionManager.java:85)

"Reference Handler" Id=2 cpuUsage=0% WAITING on java.lang.ref.Reference$Lock@69ba0f27
    at java.lang.Object.wait(Native Method)
    -  waiting on java.lang.ref.Reference$Lock@69ba0f27
    at java.lang.Object.wait(Object.java:503)
    at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)
```


### 动手调试arthas

参考[开源arthas文档](https://alibaba.github.io/arthas/quick-start.html)编写一个`Demo`:

```java
/**
 * @author yiji@apache.org
 */

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class Demo {
    static class Counter {
        private static AtomicInteger count = new AtomicInteger(0);

        public static void increment() {
            count.incrementAndGet();
        }

        public static int value() {
            return count.get();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        while (true) {
            Counter.increment();
            debug();
            System.out.println("counter: " + Counter.value());
            TimeUnit.SECONDS.sleep(1);
        }
    }

    public static void debug(){
        System.out.println("debug");
    }
}
```

然后编译Demo代码， `javac Demo.java`，然后执行下面指令运行调试模式：

```java
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,address=8000 Demo
```

然后在`intellij idea`中开一个远程调试挂到`Demo`程序：

<img src="https://zonghaishang.github.io/images/1540484931736.png" class="pretty">


接下来准备arthas可执行安装包：

```java
 arthas git:(master) ✗ ./as-package.sh
```

执行完上面脚本时，会在用户目录创建.arthas目录，里面包含可执行agent代码。

最后可在`./bin`里面执行`as.sh`，会自动列出进程的`pid`, 手动输出对应的序号，然后在`com.taobao.arthas.agent.AgentBootstrap#main`中打断点即可。

```java
$ ./as.sh
Arthas script version: 3.0.2
Found existing java process, please choose one and hit RETURN.
* [1]: 95428 
  [2]: 22647 org.jetbrains.jps.cmdline.Launcher
  [3]: 21736
  [4]: 13560 Demo
```

### arthas原理分析

