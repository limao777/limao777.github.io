---
title: linux bash find命令详解
date: 2017-10-24 15:55:00
category: 服务器
tags:
- linux
- shell
- 文件
---

## 按文件查找
find 查找的目录 -name "查找的文件名"
e.g. 
```bash
find . -name "hachi"
find . -name "*"    //查找当前目录下的所有文件
```

## 按文件类型查找
find 查找目录 -type 文件类型
* 普通文件：f
* 目录：d
* 符号链接：l
* 管道：p
* 套接字：s
* 字符设备：c
* 块设备：b
e.g. 
```bash
find ./ -type d    //列出目录
```

## 按文件大小查找
find 查找的目录 -size [+-]{单位}
单位：
k：小写
M：大写
e.g. 
```bash
find ./ -size +100k    //查找当前目录大于100k的文件，包含子目录
find ./ -size +100k -size -10M    //查找当前目录大于100k，小于10M的文件，包含子目录
```

## 按照日期查找
* 创建日期：-ctime -n/+n    (-n：n天以内，+n：n天以外)
* 修改日期：-mtime
* 访问时间：-atime
e.g. 
```bash
find ./ -ctime -1    //查找1天以内创建的文件
```

## 按深度查找
find 查找的目录 [-maxdepth -mindepth] 搜索深度
* maxdepth：n层以内的目录
* mindepth：至少n层以上的目录
e.g. 
```bash
find ./ -maxdepth 3 -name "hachi"    //查找3层目录以内的hachi文件
```

## 结合其他bash命令用法
```bash
find ./ -type d -exec ls -l {} \;    //查找当前目录并列出详细信息
find ./ -type d -ok ls -l {} \;    //查找当前目录并列出详细信息，每执行一个会询问是否继续
find ./ -type d | xargs ls -l    //查找当前目录并列出详细信息，这种效率比较高
```

## 根据文件内容查找 -grep
grep -r (有子目录) "查找的内容" 搜索的路径
e.g. 
```bash
grep -r "hachi" ./   //查找当前目录中带hachi字符串的文件
grep -r "hachi" ~ -n  //查找home目录中带hachi字符串的文件，并显示查出来的字符串在第多少行
```