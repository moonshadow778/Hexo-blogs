title: Perl的进程控制(二)
date: 2014-03-21 17:14:56
categories: Perl
tags:
---
在实际的编程中，启动一个子进程，然后等待其结束，然后父进程接着干别的，这是很理想的情况。经常地，我们会需要一个Timeout的机制，在子进程运行的时间超出了我们的期望，明显情况不对的时候能够及时地终止它。  
好消息是Perl提供了类似于Unix的信号机制，我们可以在某个信号上注册一个对应该信号的处理逻辑，比如收到ALARM信号的时候就强制退出（杀死）子进程。而父进程通过调用alarm函数使得在指定的时间会有一个SIGALRM信号发送过来，这样就能够通过(控制发送ALARM信号+ALARM信号处理函数）实现一个定时的机制。 
当然，父进程还是通过waitpid以等待子进程的正常结束，ALARM信号只是用于处理Timeout的情况。所以waitpid之后，要调用“alarm 0”来取消之前的Timeout定时器。  
一个典型的例子：
```
#!/usr/bin/perl

use strict;
use POSIX ":sys_wait_h";

my $timeOut = 5;
my $cmd = 'ping 10.10.10.10';
my $rv = 0;

my $child_pid = open ( my $CMD, "-|", $cmd ) // return -1;

if ( $child_pid ) {
  # parent process
  eval {
    # Handle the SIGALRM and kill the child process(es)
    local $SIG{ALRM} = sub {
    kill 'SIGTERM', $child_pid;
    sleep 1; # Give the process time to shut down nicely, then murder it.
    kill 'SIGKILL', $child_pid;
    die "timeout\n";
    };

    #-- Start Timer --------------------------------------------------------------#
    alarm $timeOut if $timeOut;
    # Wait for child processes. SIGALRM should take care of any hangs.
    waitpid($child_pid, 0);    # $? is set by waitpid when the process exits.
    $rv = $?;
    alarm 0 if $timeOut;
    # -- Time's up! --------------------------------------------------------------#
  };

  if ( $@ ) {
    #Exception handle
    print "Error: $@\n";
    $rv = -1;
  }

  if ($rv != 0) {
    print "Failed to issue the command:" . "$rv" . "\n";
  } else {
    print "Succeed to issue the command:" . "$rv" . "\n";
  }
}
close $CMD;
```
上面的代码通过一个ping命令来模拟一个运行超时的命令。第10行的open函数用于异步启动一个子进程执行该命令。16~21行的代码实现了针对SIGALRM信号的处理，24行启动了Alarm机制的timer，28行在非timeout的情况（子进程正常退出的情况）下取消timer。 
注意eval经常被用于这种情况去捕获一个代码block中可能的异常。
