title: Shell中的多command combine执行
date: 2014-02-18 17:33:03
categories: Shell
tags:
---
在Shell中使用“&&”和“||”连接多个command来用简单的语法表示多个command执行序列的关系是一种常见的做法。在Makefile的动作部分更是可以简单的通过这种combine的方式达到当一组命令序列中的某条命令出错则退出整个执行的逻辑并避免使用大量*if-else*的控制语句。  
一个有趣的问题是：当command不是一条简单的命令，而是一组命令甚至命令本身中包括if-else这样的控制结构的话，怎么来写这样的combined commands？考虑到不同的Shell，这个问题足够我这样的菜鸟喝一壶了......  
Why not googling? 不幸的是，网上没有这样的问题的比较清楚的解决方案，基本focus在简单命令的combine使用上，例子大同小异（当然，也许是我google的关键词的原因？）。
不管怎样，经过一些摸索，我通过初步跑通的两个例子，试图大致总结一下如何实现。
第一个例子是*Bash*的实现，如下:
```
   #!/bin/bash
    flag="YES"
    echo "1" && \
    (if  [[ $flag == "YES" ]];  then echo "2"; fi;) && \
    echo "3" && \
    (if [[ $flag == "YES" ]]; then echo "4"; fi;) && \
    echo "5"
```
其执行结果为：
```
1
2
3
4
5
```
另一个例子是*Csh*的实现，如下：
```
    #!/usr/bin/csh
    set flag="YES"
    echo "1" && \
    (if  ((  $flag == "YES" ))  then; echo "2"; endif) && \
    echo "3" && \
    (if  (( $flag == "YES" )) then; echo "4"; endif) && \
    echo "5"
```
同样实现了相同的执行结果：
```
1
2
3
4
5
```
注意不同shell中*if-else*控制结构中“*;*”的位置，这里很容易成一个坑。另外，*csh*不同于*bash*，如果变量没有定义的话，在判断时引用变量会报错退出。所以针对不同shell的时候，一定要小心它们的语法区别。  
通过这样的实现，我们可以使用不同的flag来控制一个command序列，使这个序列作为一个整体序列来工作，这不仅意味着只有前面的命令成功时，后续的命令才会执行。而且当我们知道怎么在其中加入条件控制之后，我们可以根据特定条件（flag或其他条件）来“**短路**”掉或“**插入**”某些命令以适应不同的应用场景。这在Makefile中非常有用。因为我们可能会针对不同的软硬件版本有不同的编译方式。用这样的combined commands, 我们可以自由插拔一些操作而又能方便地保证某个task编译时某一步出错时整个task的编译会报错退出。

