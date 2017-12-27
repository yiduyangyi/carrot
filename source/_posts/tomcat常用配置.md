---
title: tomcat与jvm常用配置
tags: ["java", "配置与优化"]
categories: ["java", "配置与优化"]
icon: fa-handshake-o
---

# JVM常用配置
[jdk1.8完整配置](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html "官方文档")

    -XX:MaxHeapSize=size
    -XX:MaxHeapFreeRatio=percent
    -XX:MaxMetaspaceSize=size
    -XX:MaxNewSize=size
    -XX:NewRatio=ratio


# Tomcat常用配置

## Executor配置
历史版本中，tomcat每个Connector会有一个独立的thread pool。但通过<Executor/>元素，则可以为不同Connector，甚至不同组件，配置共享thread pool。

    <Executor
         name="tomcatThreadPool"
         namePrefix="catalina-exec-"
         maxThreads="500"
         minSpareThreads="30"
         maxIdleTime="60000"
         prestartminSpareThreads = "true"
         maxQueueSize = "100"
    />

* maxThreads：最大并发线程数。默认200，一般设置500~800，根据硬件配置调整  
* minSpareThreads：tomcat初始化时创建的线程数   
* maxIdleTime：如果当前线程大于初始化线程，空闲线程存活的时间，单位毫秒   
* prestartminSpareThreads：在Tomcat初始化的时候就初始化minSpareThreads的参数值，如果不等于true，minSpareThreads的值就没啥效果了    
* maxQueueSize：最大的等待队列数，超过则拒绝请求    

## Connector设置

    <Connector
     executor="tomcatThreadPool"
     port="8080"
     protocol="org.apache.coyote.http11.Http11Nio2Protocol"
     connectionTimeout="60000"
     maxConnections="10000"
     redirectPort="8443"
     enableLookups="false"
     acceptCount="100"
     maxPostSize="10485760"
     maxHttpHeaderSize="8192"
     compression="on"
     disableUploadTimeout="true"
     compressionMinSize="2048"
     acceptorThreadCount="2"
     compressableMimeType="text/html,text/plain,text/css,application/javascript,application/json,application/x-font-ttf,application/x-font-otf,image/svg+xml,image/jpeg,image/png,image/gif,audio/mpeg,video/mp4"
     URIEncoding="utf-8"
     processorCache="20000"
     tcpNoDelay="true"
     server="Server Version 11.0"
     />

* connectionTimeout：Connector接受一个连接后等待的时间(milliseconds)，默认值是60000。
* maxConnections：这个值表示最多可以有多少个socket连接到tomcat上  
* enableLookups：禁用DNS查询  
* acceptCount：当tomcat起动的线程数达到最大时，接受排队的请求个数，默认值为100。  
* maxPostSize：设置由容器解析的URL参数的最大长度，-1(小于0)为禁用这个属性，默认为2097152(2M) 请注意， FailedRequestFilter过滤器可以用来拒绝达到了极限值的请求。   
* maxHttpHeaderSize：http请求头信息的最大程度，超过此长度的部分不予处理。一般8K。   
* compression：是否启用GZIP压缩 on为启用（文本数据压缩） off为不启用， force 压缩所有数据    
* disableUploadTimeout：这个标志允许servlet容器使用一个不同的,通常长在数据上传连接超时。 如果不指定,这个属性被设置为true,表示禁用该时间超时。    
* compressionMinSize：当超过最小数据大小才进行压缩    
* acceptorThreadCount：用于接受连接的线程数量。增加这个值在多CPU的机器上,尽管你永远不会真正需要超过2。 也有很多非维持连接,您可能希望增加这个值。默认值是1。compressableMimeType：配置想压缩的数据类型
* URIEncoding：网站一般采用UTF-8作为默认编码。   
* processorCache：协议处理器缓存的处理器对象来提高性能。 该设置决定多少这些对象的缓存。-1意味着无限的,默认是200。 如果不使用Servlet3.0异步处理,默认是使用一样的maxThreads设置。 如果使用Servlet 3.0异步处理,默认是使用大maxThreads和预期的并发请求的最大数量(同步和异步)。  
* tcpNoDelay：如果设置为true,TCP_NO_DELAY选项将被设置在服务器套接字,而在大多数情况下提高性能。这是默认设置为true。     
* server：隐藏Tomcat版本信息，首先隐藏HTTP头中的版本信息

# 打开GC日志

    -Xloggc:/home/work/logs/gc-`date +%F_%H-%M-%S`.log -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:+PrintGCCause-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=5M


## tomcat日志配置
tomcat catalina日志切割后，不会清除catalina.out文件的内容。

- directory - The directory where to create the log file. If the path is not absolute, it is relative to the current working directory of the application. The Apache Tomcat configuration files usually specify an absolute path for this property, ${catalina.base}/logs Default value: logs
- rotatable - If true, the log file will be rotated on the first write past midnight and the filename will be {prefix}{date}{suffix}, where date is yyyy-MM-dd. If false, the file will not be rotated and the filename will be {prefix}{suffix}. Default value: true
- prefix - The leading part of the log file name. Default value: juli.
- suffix - The trailing part of the log file name. Default value: .log
- bufferSize - Configures buffering. The value of 0 uses system default buffering (typically an 8K buffer will be used). A value of <0 forces a writer flush upon each log write. A value >0 uses a BufferedOutputStream with the defined value but note that the system default buffering will also be applied. Default value: -1
- encoding - Character set used by the log file. Default value: empty string, which means to use the system default character set.
- level - The level threshold for this Handler. See the java.util.logging.Level class for the possible levels. Default value: ALL
- filter - The java.util.logging.Filter implementation class name for this Handler. Default value: unset
- formatter - The java.util.logging.Formatter implementation class name for this Handler. Default value: java.util.logging.SimpleFormatter
- maxDays - The maximum number of days to keep the log files. If the specified value is <=0 then the log files will be kept on the file system forever, otherwise they will be kept the specified maximum days. Default value: -1.


    1catalina.org.apache.juli.AsyncFileHandler.level = FINE
    1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
    1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
    1catalina.org.apache.juli.AsyncFileHandler.rotatable = true
    1catalina.org.apache.juli.AsyncFileHandler.maxDays  = 7

    2localhost.org.apache.juli.AsyncFileHandler.level = FINE
    2localhost.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
    2localhost.org.apache.juli.AsyncFileHandler.prefix = localhost.

    3manager.org.apache.juli.AsyncFileHandler.level = FINE
    3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
    3manager.org.apache.juli.AsyncFileHandler.prefix = manager.

    4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
    4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
    4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
