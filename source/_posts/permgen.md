title: PermGen Memory Leak
date: 2014-02-21 15:46:05
categories: Java
tags:
---
PermGen memory leak是Java开发者时不时会遇到的一种比较恼人的内存问题，伴随的典型的症状就是**java.lang.OutOfMemoryError: PermGen space**这个错误。  
PermGen就是**Permanent Generation**这么个JVM里的内存区，里面存放的主要内容是一堆堆的Java Class定义。当您的Java应用跑起来的时候，相关的所有Class信息就会装载到这个内存区里。这个内存区的大小可以通过启动JVM的时候通过参数配置，通常不会很大，默认的是82M。当遇到刚才的那条OOME的时候，有两种可能：  
1. 您的application确实比较牛，用到了很多类，PermGen装不下了。这种情况的解决方法很简单，就是提高这个内存区的大小。  
2. 另外一种可能就是一刀入魂的内存泄露了。为啥会泄露，下面这张图对于理解这个问题很关键！（很赞的一张图，来自[这里](https://plumbr.eu/blog/what-is-a-permgen-leak)）  
![Class和ClassLoader引用关系](/image/class-classloader-refs.png)  
简单地说，每一个对象都会持有一个它的Class的引用，这个Class又会引用到装载这个Class的ClassLoader，最关键的是**每个ClassLoader又持有其所有装载的Class的引用**。那么问题就在于，当某一个Class因为某种原因没有被垃圾收集的话，它的ClassLoader装载的所有的Class也就不会被收集 —— 这么看ClassLoader真是负责任啊！

Tomcat因为其ClassLoader的机制和Webapp装载卸载的典型性就成为这个问题的温床。一个Webapp卸载后，按理说其对应的Class都应该没有用了从而被收集掉。可是一旦有某个Class仍然不幸保持引用，那么恭喜你，还有一堆从同一个ClassLoader装载的类也还僵尸般存在着，仍然霸占着有限的PermGen空间。  

一个很好的例子可以参看[Leaking Drivers](https://plumbr.eu/blog/what-is-a-permgen-leak)，里面详细描述了JDBC规范下DriverManager持有一个具体Driver的引用，使得该引用在应用卸载时不能释放，从而导致的PermGen memory leak的细节。 

如果搜索PermGen Memory Leak的解决方法，网上的方案五花八门，包括Overflow Stack上被狂点赞的那些帖子。但是真心都不是真正解决问题的思路。 

相比之下，[这篇帖子](https://plumbr.eu/blog/busting-permgen-myths)很坦诚也很中肯。对于这样的问题，我觉得负责任的回答应该是“**分析你的code，找到你的code中ClassLoader和Class的不恰当的关系。只有真正找到root cause，才能真正解决它**“。听起来像是废话，但是跟从网上那些仅从表面入手的建议，我的尝试证明了这句话的正确。  

好消息是Tomcat一直在努力从它的角度避免这些可能的不恰当关系的发生，他们令人尊敬的结果可以在[这里](http://wiki.apache.org/tomcat/MemoryLeakProtection)找到。 

如果您想查看您JVM中的PermGen的实时数据，JDK中的这个命令可以帮助您
```
<jdk path>/bin/jstat -gcpermcapacity <pid>
```