---
title: 用C语言创建一个守护进程并使用定时器
date: 2017-11-14 15:10:58
category: C语言
tags:
- linux
- C语言
---

## 守护进程的特点
* 后台服务进程
* 独立于控制终端
* 周期性执行某任务
* 不受用户登陆注销影响
* 一般采用以d结尾的名字（服务）

## 进程组
进程组组长一般是组里面第一个进程
进程组的ID==进程组组长的ID

## 会话 - 多个进程组
### 创建一个会话注意事项
不能是进程组长
创建会话的进程成为新进程组的组长
有些linux版本需要root权限执行次操作（ubuntu不需要）
创建出的新会话会丢弃原有的控制终端
一般步骤：先fork，父进程死，子进程执行创建会话操作
### 获取进程所属的会话ID
pid_t getsid(pid_t pid);
### 创建一个会话
pid_t setsid(void);

##创建守护进程模型
fork子进程，父进程退出
子进程创建新会话
改变当前工作目录chdir
重设文件掩码
关闭文件描述符
执行核心工作

## 代码如下
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <signal.h>
#include <sys/time.h>
#include <time.h>
#include <fcntl.h>


void dowork(int no){
    time_t curtime;
    //get current sys time and write it to file
    time(&curtime);
    //format time
    char *pt = ctime(&curtime);
    int fd = open("/tmp/aaaaattt.txt", O_CREAT | O_WRONLY | O_APPEND, 0644);
    write(fd, pt, strlen(pt)+1);
    close(fd);

}


int main(int argc, const char* argv[]){
    pid_t pid = fork();
    if(pid>0){
        exit(1);
    }
    else if(pid==0){
        setsid();
        //改变进程工作目录
        chdir("/tmp");
        //配置文件掩码
        umask(0);
        //关闭文件描述符
        close(STDIN_FILENO);
        close(STDOUT_FILENO);
        close(STDERR_FILENO);
        //执行核心操作，注册信号捕捉

        struct sigaction act;
        act.sa_handler = 0;
        act.sa_handler = dowork;

        sigemptyset(&act.sa_mask);
        sigaction(SIGALRM, &act, NULL);

        //创建定时器
        struct itimerval val;

        //首次触发时间
        val.it_value.tv_usec = 0;
        val.it_value.tv_sec = 2;
        //循环周期
        val.it_interval.tv_usec = 0;
        val.it_interval.tv_sec = 1;
        setitimer(ITIMER_REAL, &val, NULL);
        while(1);
    }
    return 0;
}
```
