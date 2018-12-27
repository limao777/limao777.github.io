---
title: makefile相关概念
date: 2017-10-30 15:37:46
category: 服务器
tags:
- linux
- shell
- 编译技术
---

# make
## 一个概念：
gcc -> 编译器
make -> linux自带的构建器，构建的规则在makefile中

## makefile文件的命名
* makefile、Makefile两者均可

## makefile中的规则
如：gcc a.c b.c c.c -o app
分为三个部分：目标、依赖、命令
（makefile文件中）使用目标：依赖（换行）（tab缩进）命令：

### 第一个版本的编写
```text
app:a.c b.c c.c
{TAB}gcc a.c b.c c.c -o app
```
然后执行make命令即可

### 第二个版本的编写，优化编译
原理：
* 检测依赖是否存在：向下搜索下边的规则，如果有规则是用来生成查找依赖的，执行规则中的命令
* 依赖存在：判断是否需要更新，原则：目标时间>依赖时间，反之，则更新
```text
app:a.o b.o c.o
{TAB}gcc a.o b.o c.o -o app

a.o:a.c
{TAB}gcc a.c -c

b.o:b.c
{TAB}gcc b.c -c

c.o:c.c
{TAB}gcc c.c -c
```

### 第三个版本
自定义变量(通常用小写):
obj=a.o b.o c.o
obj=10
变量的取值：aa=$(obj)
makefile自带的变量：大写
* CPPFLAGS
* CC
自动变量：
* $@：规则中的目标
* $<：规则中的第一个依赖
* $^：规则中所有的依赖
自动变量只能在规则中的命令中使用
```text
obj = a.o b.o c.o
target = app
$(target):$(obj)
{TAB}gcc $(obj) -o $(target)

%.o:%.c
{TAB}gcc -c $< -o $@
```
模式匹配：
%.o:%.c
第一次：a.o没有%就匹配了a，变成了gcc -c a.c -o a.o
第二次和第三次类似

### 第四个版本，解决可移植性比较差的问题
先介绍几个函数：
* 查找指定目录下指定类型的文件：wildcard    #e.g. src = $(wildcard ./*.c)    //空格后面加上参数
* 匹配替换：patsubst    #e.g. obj = $(patsubst %.c,%.o,$(src))    //匹配用百分号，将src中的.c替换为.o
```text
src = $(wildcard ./*.c)
obj = $(patsubst %.c,%.o,$(src))
target = app
$(target):$(obj)
{TAB}gcc $(obj) -o $(target)    #或gcc $^ -o $@

%.o:%.c
{TAB}gcc -c $< -o $@
```

### 第五个版本，带自动清理项目
即让make生成不是终极目标的目标
```text
src = $(wildcard ./*.c)
obj = $(patsubst %.c,%.o,$(src))
target = app
$(target):$(obj)
{TAB}gcc $(obj) -o $(target)    #或gcc $^ -o $@

%.o:%.c
{TAB}gcc -c $< -o $@

.PHONY:CLEAN    #声明clean为为目标，使clean不受文件爱你更新检查的干扰
clean:
{TAB}-rm $(obj) $(target) -f    #"-"表示如果执行失败继续执行后面的，-f是rm的参数，强制执行不报错
```
执行make clean即可
