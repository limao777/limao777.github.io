---
title: linux压缩和解压缩相关技术
date: 2017-10-24 16:33:42
category: 服务器
tags:
- 文件
- shell
- linux
---

# linux下常见的压缩格式
* .gz --gzip
* .bz2 -bzip2

# 常用压缩命令
## tar
### 打包
参数：
* c：创建压缩文件
* x：释放压缩文件
* v：打印提示信息
* f：指定压缩包名字
* z：使用gzip压缩文件
* j：使用bzip2的方式压缩文件

### 压缩
* tar 参数 压缩包名字 原材料
```bash
tar zcvf aaa.tar.gz ./data    //打包并压缩./data下的文件
tar zcvf aaa.tar.gz abc def ./data    //打包并压缩./data下的文件以及abc、def文件(或目录)
```

### 解压缩
* tar 参数 压缩包名字 -C 解压目录
```bash
tar zxvf aaa.tar.gz
tar zxvf aaa.tar.gz -C /tmp/aaa
```

## rar
### rar需要安装
```bash
sudo apt-get install rar
```

### 压缩
rar a 压缩包名(不用指定后缀) 压缩内容

### 解压缩
rar x 压缩包名 解压目录

## zip/unzip
### 压缩
zip 参数 压缩包名 原材料
如果有目录，加上参数r
### 解压缩
unzip 压缩包名 -d 解压目录
