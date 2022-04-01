---
title: Log4j 1.x版本高并发引发BLOCKED死锁
date: 2021-05-13 19:25:00
img: https://cdn.jsdelivr.net/gh/joshphe/blogImage@main/img/beach.png  #设置本地图片
summary: Log4j 1.x版本高并发BLOCKED死锁
tags:
  - log4j
  - 高并发
  - 死锁	
  - BLOCKED
---

## Log4j 1.x版本高并发情况下引发BLOCKED死锁的问题

应用系统使用Apache log4j-1.2.16版本的jar包，系统在某一天运行过程中服务宕机，通过jstack命令可以发现大量死锁：

```java
"http-processor732" #79018 daemon prio=5 os_prio=0 tid=0x000000001b26c000 nid=0x2c64 waiting for monitor entry [0x0000000059c3d000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at org.apache.log4j.Category.callAppenders(Category.java:204)
	- waiting to lock <0x00000007209cfa80> (a org.apache.log4j.Logger)
	at com.cisco.ccbu.infra.serviceability.logging.CCBULogger.callAppendersDirect(CCBULogger.java:210)
	at com.cisco.ccbu.infra.serviceability.logging.CCBULogger.forcedCCBUTrace(CCBULogger.java:167)
	at com.cisco.ccbu.infra.serviceability.ServiceabilityManager.trace(ServiceabilityManager.java:2360)
	at com.cisco.ccbu.infra.serviceability.ServiceabilityManager.trace(ServiceabilityManager.java:2461)
	at com.cisco.ccbu.infra.serviceability.ServiceabilityManager.trace(ServiceabilityManager.java:2446)
	at com.audium.core.serviceability.SvcMgrUtil.trace(SvcMgrUtil.java:110)
	at com.audium.server.controller.Controller.newCall(Controller.java:1697)
	at com.audium.server.controller.Controller.doPost(Controller.java:1002)
	at com.audium.server.controller.Controller.doGet(Controller.java:546)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:494)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:412)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:754)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1385)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	- locked <0x00000007292e6cc0> (a org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
```

查看引发死锁线程如下：

```java
"http-processor161" #78384 daemon prio=5 os_prio=0 tid=0x0000000041dad000 nid=0x3bf8 runnable [0x000000005cc3c000]
   java.lang.Thread.State: RUNNABLE
	at sun.util.locale.provider.LocaleProviderAdapter.isSupportedLocale(LocaleProviderAdapter.java:299)
	at sun.util.resources.LocaleData$LocaleDataResourceBundleControl.getCandidateLocales(LocaleData.java:218)
	at java.util.ResourceBundle.getBundleImpl(ResourceBundle.java:1365)
	at java.util.ResourceBundle.getBundle(ResourceBundle.java:899)
	at sun.util.resources.LocaleData$1.run(LocaleData.java:167)
	at sun.util.resources.LocaleData$1.run(LocaleData.java:163)
	at java.security.AccessController.doPrivileged(Native Method)
	at sun.util.resources.LocaleData.getBundle(LocaleData.java:163)
	at sun.util.resources.LocaleData.getDateFormatData(LocaleData.java:127)
	at java.text.DateFormatSymbols.initializeData(DateFormatSymbols.java:710)
	at java.text.DateFormatSymbols.<init>(DateFormatSymbols.java:145)
	at sun.util.locale.provider.DateFormatSymbolsProviderImpl.getInstance(DateFormatSymbolsProviderImpl.java:85)
	at java.text.DateFormatSymbols.getProviderInstance(DateFormatSymbols.java:364)
	at java.text.DateFormatSymbols.getInstance(DateFormatSymbols.java:340)
	at java.util.Calendar.getDisplayName(Calendar.java:2110)
	at java.text.SimpleDateFormat.subFormat(SimpleDateFormat.java:1125)
	at java.text.SimpleDateFormat.format(SimpleDateFormat.java:966)
	at java.text.SimpleDateFormat.format(SimpleDateFormat.java:936)
	at java.text.DateFormat.format(DateFormat.java:345)
	at org.apache.log4j.helpers.PatternParser$DatePatternConverter.convert(PatternParser.java:443)
	at org.apache.log4j.helpers.PatternConverter.format(PatternConverter.java:65)
	at org.apache.log4j.PatternLayout.format(PatternLayout.java:506)
	at com.cisco.ccbu.infra.serviceability.logging.CCBUSyslogAppender.append(CCBUSyslogAppender.java:247)
	at org.apache.log4j.AppenderSkeleton.doAppend(AppenderSkeleton.java:251)
	- locked <0x00000007212a16d8> (a com.cisco.ccbu.infra.serviceability.logging.CCBUSyslogAppender)
	at org.apache.log4j.helpers.AppenderAttachableImpl.appendLoopOnAppenders(AppenderAttachableImpl.java:66)
	at org.apache.log4j.Category.callAppenders(Category.java:206)
	- locked <0x00000007209cfa80> (a org.apache.log4j.Logger)
	at com.cisco.ccbu.infra.serviceability.logging.CCBULogger.callAppendersDirect(CCBULogger.java:210)
	at com.cisco.ccbu.infra.serviceability.logging.CCBULogger.forcedCCBULog(CCBULogger.java:153)
	at com.cisco.ccbu.infra.serviceability.ServiceabilityManager.log(ServiceabilityManager.java:2222)
	at com.cisco.ccbu.infra.serviceability.ServiceabilityManager.info(ServiceabilityManager.java:2706)
	at com.audium.server.controller.Controller.doPost(Controller.java:750)
	at com.audium.server.controller.Controller.doGet(Controller.java:546)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:494)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:412)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:754)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1385)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	- locked <0x0000000728e76c80> (a org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
```

==at org.apache.log4j.Category.callAppenders(Category.java:206)==

​	==locked <0x00000007209cfa80> (a org.apache.log4j.Logger)==

通过日志可以大致看出问题是出在org.apache.log4j.Category.callAppenders方法中：

```java
public void callAppenders(LoggingEvent event)
  {
    int writes = 0;

    for (Category c = this; c != null; c = c.parent)
    {
      synchronized (c) {
        if (c.aai != null) {
          writes += c.aai.appendLoopOnAppenders(event);
        }
        if (!c.additive) {
          break;
        }
      }
    }

    if (writes == 0)
      this.repository.emitNoAppenderWarning(this);
  }
```

代码中可以看到该方法中使用了synchronized，synchronized是一个重量级锁，它会降低程序性能，被synchronized关键字声明的对象和代码块被视为临界区，当某个线程调用该对象的synchronized方法或者代码块时，便获取了该对象的锁，其他线程若要访问则需要等待，该方法中使用了appendLoopOnAppenders(event)方法：

```java
  public int appendLoopOnAppenders(LoggingEvent event)
  {
    int size = 0;

    if (this.appenderList != null) {
      size = this.appenderList.size();
      for (int i = 0; i < size; i++) {
        Appender appender = (Appender)this.appenderList.elementAt(i);
        appender.doAppend(event);
      }
    }
    return size;
  }
```

在该方法中，调用了doAppend(event)方法来追加日志输出，AppenderSkeleton类实现了Appender这个接口：

```java
  public synchronized void doAppend(LoggingEvent event)
  {
    if (this.closed) {
      LogLog.error("Attempted to append to closed appender named [" + this.name + "].");
      return;
    }

    if (!isAsSevereAsThreshold(event.getLevel())) {
      return;
    }

    Filter f = this.headFilter;

    while (f != null) {
      switch (f.decide(event)) { case -1:
        return;
      case 1:
        break;
      case 0:
        f = f.getNext();
      }
    }
```

doAppend方法上也加了synchronized锁，因此在有线程持有该方法的锁时，其他线程在调用该方法时需等待锁被释放后才能进行日志输出，因此在高并发的情况下很容易产生死锁的情况。

此时，大致找到死锁产生的原因后，考虑到应用系统属于多年来稳定运行的系统，升级log4j 2.x需要考虑到的因素比较多，临时解决这个问题则需要从日志并发量角度考虑：

1. ##### 尽量输出有效日志，减少无用日志的输出。

2. ##### 调整日志输出级别，一般开启info级别，必要情况下再提高日志输出级别。

3. ##### 条件允许的情况下升级log4j版本。

在此之前另一个系统碰到类似的log4j导致的性能问题，log4j配置如下：

```xml
	<appender class="com.primeton.ext.common.logging.AppRollingFileAppender" name="ROLLING_FILE_BS">

		<!--<param name="Threshold" value="INFO"/>-->

		<param name="Encoding" value="UTF-8"/>

		<param name="File" value="logs/bs.log"/>

		<param name="Append" value="true"/>

		<param name="MaxFileSize" value="10MB"/>

		<param name="MaxBackupIndex" value="200"/>

		<layout class="org.apache.log4j.PatternLayout">

			<param name="ConversionPattern" value="[%d{yyyy-MM-dd HH:mm:ss,SSS}][%p][%C:%L] %m%n"/>

		</layout>

	</appender>

	<appender name="ROLLING_FILE_BS_SYN" class="org.apache.log4j.AsyncAppender">

		<param name="BufferSize" value="8192"/>

		<param name="LocationInfo" value="true"/>

		<appender-ref ref="ROLLING_FILE_BS"/>

	</appender>
```

<param name="LocationInfo" value="true"/>

该配置作用是输出java文件名称和行号，默认为false。

在jar包中定位LocationInfo类的代码如下：

```java
  public LocationInfo(Throwable t, String fqnOfCallingClass)
  {
    if ((t == null) || (fqnOfCallingClass == null))
      return;
    if (getLineNumberMethod != null)
      try {
        Object[] noArgs = null;
        Object[] elements = (Object[])(Object[])getStackTraceMethod.invoke(t, noArgs);
        String prevClass = "?";
        for (int i = elements.length - 1; i >= 0; i--) {
          String thisClass = (String)getClassNameMethod.invoke(elements[i], noArgs);
          if (fqnOfCallingClass.equals(thisClass)) {
            int caller = i + 1;
            if (caller < elements.length) {
              this.className = prevClass;
              this.methodName = ((String)getMethodNameMethod.invoke(elements[caller], noArgs));
              this.fileName = ((String)getFileNameMethod.invoke(elements[caller], noArgs));
              if (this.fileName == null) {
                this.fileName = "?";
              }
              int line = ((Integer)getLineNumberMethod.invoke(elements[caller], noArgs)).intValue();
              if (line < 0)
                this.lineNumber = "?";
              else {
                this.lineNumber = String.valueOf(line);
              }
              StringBuffer buf = new StringBuffer();
              buf.append(this.className);
              buf.append(".");
              buf.append(this.methodName);
              buf.append("(");
              buf.append(this.fileName);
              buf.append(":");
              buf.append(this.lineNumber);
              buf.append(")");
              this.fullInfo = buf.toString();
            }
            return;
          }
          prevClass = thisClass;
        }
        return;
      } catch (IllegalAccessException ex) {
        LogLog.debug("LocationInfo failed using JDK 1.4 methods", ex);
      } catch (InvocationTargetException ex) {
        if (((ex.getTargetException() instanceof InterruptedException)) || ((ex.getTargetException() instanceof InterruptedIOException)))
        {
          Thread.currentThread().interrupt();
        }
        LogLog.debug("LocationInfo failed using JDK 1.4 methods", ex);
      } catch (RuntimeException ex) {
        LogLog.debug("LocationInfo failed using JDK 1.4 methods", ex);
      }
    String s;
    synchronized (sw) {
      t.printStackTrace(pw);
      s = sw.toString();
      sw.getBuffer().setLength(0);
    }

    int ibegin = s.lastIndexOf(fqnOfCallingClass);
    if (ibegin == -1) {
      return;
    }

    if ((ibegin + fqnOfCallingClass.length() < s.length()) && (s.charAt(ibegin + fqnOfCallingClass.length()) != '.'))
    {
      int i = s.lastIndexOf(fqnOfCallingClass + ".");
      if (i != -1) {
        ibegin = i;
      }

    }

    ibegin = s.indexOf(Layout.LINE_SEP, ibegin);
    if (ibegin == -1)
      return;
    ibegin += Layout.LINE_SEP_LEN;

    int iend = s.indexOf(Layout.LINE_SEP, ibegin);
    if (iend == -1) {
      return;
    }

    if (!inVisualAge)
    {
      ibegin = s.lastIndexOf("at ", iend);
      if (ibegin == -1) {
        return;
      }
      ibegin += 3;
    }

    this.fullInfo = s.substring(ibegin, iend);
  }
```
可以看到有一块代码也加了synchronized锁，再通过打印堆栈来获取行号信息，因此评估该方法在高并发的情况下也会造成性能问题

```java
String s;
synchronized (sw) {
  t.printStackTrace(pw);
  s = sw.toString();
  sw.getBuffer().setLength(0);
}
```
因此在ConversionPattern配置中将%L去掉，先后测试的数据如下：

![](https://cdn.jsdelivr.net/gh/joshphe/blogImage@main/img/picture1.png)

![](https://cdn.jsdelivr.net/gh/joshphe/blogImage@main/img/picture2.png)

测试相对比较简单，数据不具有绝对的参考性，但可以大致看出locationinfo的参数配置在一定情况下也会带来性能问题。