---
title: gcc相关概念
date: 2017-10-25 16:03:21
category: 服务器
tags:
- linux
- 编译技术
---

## gcc工作流程
### 预处理 --E
* 宏替换
* 头文件展开
* 注释去掉
* XXX.c -> xxx.i

### 编译 --S
xxx.i -> xxx.s
汇编文件

### 汇编 -c
xxx.s -> xxx.o
二进制文件

链接
xxx.o -> xxx（可执行）

## gcc常用参数
### 显示版本
-v/--version

### 编译的时候指定头文件路径
-I {文件路径}

### 指定编译出来的文件名字
-o {文件名}

### 汇编文件生成二进制文件
-c：最终得到一个.o文件

### 使用gdb调试
-g

### 编译的时候指定一个宏
-D     //主要是测试程序的时候用
e.g.
```bash
gcc hello.c -D DEBUG -o hello
//"-D DEBUG"相当于C代码中加上了如下一行：
#define DEBUG
```

### 编译时添加警告信息，即有警告也会在编译时提示
-wall

### 优化代码
-On：n是优化级别：1，2，3

## 静态库的制作和使用
### 命名规则：
libxxx.a(通常只有xxx自定义)

### 制作步骤
* 原材料：源代码
* 将.c文件生成.o：gcc abc.c def.c -c
* 将.o文件打包：使用ar，e.g. ar rcs libtest.a *.o(将当前目录所有.o打包为mytest.a，之后可以用nm libtest.a查看打包后包含的方法)
e.g.
```c
head.h文件：
#ifndef __HEAD_H_
#define __HEAD_H_
int add (int a, int b);
#endif


abc.c文件：
#include "head.h"
int add (int a, int b){
    int res = a + b;
    return res;
}


main.c文件：
#include <stdio.h>
#include "head.h"
int main(void){
    int sum = add(1,2);
    printf("%d\n",sum);
    return 1;
}
```

### 静态库使用
使用-L参数指定静态库路径
使用-l指定静态路名字
e.g.
```bash
gcc main.c -L ./lib -l test -o main
```
## 动态库的制作和使用(ldd {文件名}可以查看应用程序依赖库的路径)
### 命名规则
libxxx.so

### 制作步骤
* 将源文件生成.o
* gcc a.c b.c -fpic(fPIC)

### 打包
gcc shared a.o b.o -0 ;ibxxx.so

###库的使用
* 头文件a.h
* 动态库 libtest.so
参考函数声明编程测试程序main.c
gcc main.c -I ./ -L ./ -l test -o app

### 动态库加载失败的问题
对于elf格式的程序（使用file app查看），是由ld-linux.so*来完成（用ldd app查看），先后搜索elf文件的DT-RPATH段 - 环境变量LD_LIBRARY_PATH - /etc/ld.so.cache文件列表 - /lib，/usr/lib目录找到库文件后将其载入内存
所以如果要用我们自己的库，则需用以下方法：
* 拷贝自己制作的共享库到/lib或/usr/lib
* 临时设置LD_LIBRARY_PATH
```bash
export LD_LIBRARY_PATH=库路经:$LD_LIBRARY_PATH
```
* 永久设置
用户级别：~/.bashrc加上上述命令，并使用source ~/.bashrc重启终端
系统级别：
1./etc/profile加上上述命令，并使用source /etc/profile重启终端
2./etc/ld.so.cache文件列表，找到一个配置文件/etc/ld.so.conf把动态库绝对路i经添加到文件中，再执行sudo ldconfig -v
