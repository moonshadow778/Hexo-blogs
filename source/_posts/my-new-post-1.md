title: Perl的进程控制(一)
date: 2014-03-21 16:06:16
categories: Perl
tags:
---
先简单总结一下，Perl里调用shell命令的时候，会用到进程控制。经常用到的有以下几个方式: 
1. 通过system() 调用shell命令，Perl是父进程，Shell本身是子进程，实际执行的命令本身是孙子进程。  
例子： 
```
system "ls -l \$HOME"; 
```
  注意双引号内变量内插需要转义符。 
2. 通过exec()调用shell命令。1.与2.的区别在于system会等待shell返回，而exec直接结束Perl进程，要执行的命令在Shell中执行完毕后退出。  
例子： 
```
exec "date";
```
3. 反引号··用于捕获命令输出，不能乱用。通常返回的内容带回车符，依照需要可能要使用chomp。Perl解释反引号里的值的方式类似于system的单参数形式，在解释器中会以双引号字符串形式展开，这意味着反斜线转义与变量内插都会正常处理。  
例子： 
```
my $now = `date`;
print "The time is now $now";
```
4. 以上都是同步处理子进程。 使用open连接至文件句柄的方式可以启动并发运行的子进程。注意管道符号的位置，它表示文件句柄是被管道到命令的标准输出还是标准输入。  
例子： 
```
my $child_pid = open ( my $CMD, "-|", @cmd_array ) or die "Failed to create a child process";
```
  在上面的例子里， 一个文件句柄$CMD被打开并通过open被连接，@cmd_array表示的一个命令（包括其options）被异步地执行，其输出则管道至$CMD。 
5. 再高级一点儿的话，Perl也可以使用类似系统调用的fork()。和Unix下的fork()系统调用是一样一样的。  
例子： 
```
my $cld_pid = fork();# fork出一个子进程
if (！defined $cld_pid) {
    die "Failed to fork a process.\n";
}
unless ($cld_pid) {
    # 子进程
    print "child!\n";
    sleep 3;
} else {
    # 父进程
	print "parent!\n";
	waitpid($cld_pid, 0);
}
# 父进程子进程都会执行到的地方
print "common.\n";
```
此例子的输出是：
```
parent
child!
common.
common.
```
  注意父进程在打印第11行后会等待子进程结束，然后才执行15行。  
实际上，system 做的事情，就是fork一个子进程处理命令，在fork出的子进程内调用exec执行命令（对应替换我们上面例子里的第8行），然后父进程wait在子进程的pid上。因为是exec调用，所以子进程就结束了， 等候在其上的父进程也就结束退出。—— 这是Unix系统标准的系统调用方式。  

