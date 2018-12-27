---
title: 线程相关概念以及pthread
date: 2017-11-16 14:02:07
category: C语言
tags:
- linux
- C语言
---

## 几个知识点
创建线程之后，地址空间没有变化
进程退出成了线程-主线程
主线程和子线程有各自独立的pcb（子线程的pcb是从主线程拷贝来的）
在linux下，线程就是轻量级进程（内核看pcb）

### 主线程和子线程共享的数据
.text(代码段)
.bss（未初始化的变量）
.data（已初始化的变量）
堆
动态库家再去
环境变量
命令行参数
-- 线程间通信可以依靠全局变量、堆

### 主线程和子线程不共享的数据
栈(栈区被线程平均分配)

### （补充）进程间共享的资源
代码
文件描述符
内存映射区 -- mmap

### 查看指定线程的LWP号
ps -Lf {pid}
注意线程号和进程ID是有区别的，线程号给内核看

## 线程相关函数
### 创建线程
函数原型
```C
//如果成功返回0，失败返回错误号
//perror()不能使用该函数打印错误信息，它只是个数字
itn pthread_create(
pthread_t *thread,    //线程ID=无符号长整型
const pthread_attr_t *attr,    //线程属性，通常为NULL
void *(*start_routine) (void *),    //线程处理函数
void *arg    //线程处理函数参数
);
```
参数
* thread：创出参数，线程创建成功以后，会被设置一个合适的值
* attr：默认传NULL
* start_routine：子线程的处理函数
* arg：回调函数的参数
主线程先退出，子线程会被强制结束

### 单个线程退出
函数原型
```C
void pthread_exit(void *retval);
```

### 阻塞等待线程退出，获取线程退出状态
函数原型
```C
int pthread_join(pthread_t thread, void **retval);
```

### 线程分离
函数原型
```C
int pthread_detach(pthread_t thread);
```
调用该函数后不需要pthread_join()，子线程会自动回收自己的PCB

### 杀死（取消）线程
函数原型
```C
int pthread_cancel(pthread_t thread);
```
在要杀死的子线程对应的处理函数的内部必须做过一次系统调用


简单的e.g.
```C
//gcc编译时参数加上 -lpthrad
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <pthread.h>


void* myfunc(void* arg){
        int num = *(int*)arg;
        printf("%dth child thread id:%ld\n", num, pthread_self());
        return NULL;
}

int main(int argc, const char* argv[]){
        //创建一个子线程
        int i = 0;
        pthread_t pthid[5];
        for(i=0;i<5;++i){
                pthread_create(&pthid[i], NULL, myfunc, (void*)&i);    //此处用的引用传递，会导致多线程的i有可能取到同一个
        }
        printf("parent thread id:%ld\n", pthread_self());
        for(i=0;i<5;i++){
                printf("i=%d\n",i);
        }
        sleep(2);
        return 0;
}
```
