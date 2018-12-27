---
title: 使用docker搭建LNMP
date: 2018-12-27 11:44:30
category: 服务器
tags:
- linux
- 架构
- PHP
- 容器化
---
docker是最近几年兴起的一套容器化环境，其隔离性比较好，趁着有空研究了一下PHP的docker化，虽然单机使用感觉和rpm包便利性差不多，但是在分布式系统上面结合k8s等还是有一定的优势。

先放一张整体结构图
{% asset_img all.jpg 总体结构图 %}


## 为什么nginx没放docker
正常情况下大型项目中容器化的都是应用程序，像mysql之类的也很少用docker包装，而nginx作为负载使用基本没有容器化的必要。
只不过有需求的话看过本文过后也可自行将nginx容器化。

## 资料收集
docker官网的帮助文档，dockerhub的说明书都是很好的资料来源，面对陌生的docker包通常阅读其说明书很有效！

## docker安装
docker安装在不通时代不同操作系统不一样，这里的操作系统为centos7，使用rpm包安装
* 从docker官网下载rpm安装包
* 在终端中执行yum install <安装包名字>，然后安装就结束了
* 如果要开机自动启动docker则需设置systemctl enable docker

## docker 网络创建
现在的docker不建议使用--link来建立连接，而使用新的方式
终端执行
```bash
## docker 创建私有网络
docker network create my-net
```

## docker memcached
可以打开 https://hub.docker.com/_/memcached 看说明书
终端执行
```bash
#拉取memcached镜像
docker pull memcached
#启动memcached，加入my-net网络，后台执行，映射11211端口，始终跟随docker启动的时候启动
docker run --name memcached --network my-net -d -p 11211:11211 --restart=always memcached
```

## docker mysql
可以打开 https://hub.docker.com/_/mysql 看说明书
为了现阶段的兼容性使用mysql5.7版本，其他版本自行变更即可，终端执行
```bash
#拉取mysql镜像
docker pull mysql:5.7
#启动mysql，加入my-net网络，后台执行，映射3306端口，始终跟随docker启动的时候启动，将root的password设置为123456；额外的，根据官网mysql说明书，文件储存于/var/lib/mysql，配置文件通常位于/etc/mysql/my.cnf，而docker中的root用户默认可以被外部网络访问
docker run --name mysql -e MYSQL_ROOT_PASSWORD=123456 --network my-net -d -p3306:3306 --restart=always mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

## docker PHP
docker化PHP是最麻烦的步骤，因为涉及到PHP的各种扩展库，甚至一些私有扩展库，这里我们在基于官方PHP包的情况下用dockerfile创建自己的私有包。
首先新建一个文件夹，在里面创建dockerfile文件，其中相关参数可以根据自身情况自行调整，如果使用TP，laravel，yaf甚至之前文章提到的自研PHP框架都能使用下面的dockerfile，
值得注意的是里面的tgz包都是从pecl下载到dockerfile同一级目录，如果版本号不同则要自行更改文件中的名字；当然，如果网络没问题可以直接使用类似“pecl install memcached-3.0.3 && docker-php-ext-enable memcached”替代
```text
FROM php:7.1-fpm

LABEL Name=php-fpm Version=0.0.1

RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev \
        libmemcached-dev \
        zlib1g-dev \
        libpq-dev \
        --no-install-recommends \
        && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install -j$(nproc) gd mcrypt gettext mysqli pdo_mysql pgsql pdo_pgsql

        COPY ./yaf-3.0.7.tgz /tmp/yaf-3.0.7.tgz
RUN cd /tmp \
        && mkdir yaf \
        && tar -zxvf yaf-3.0.7.tgz -C yaf --strip-components=1 \
#        && rm yaf-3.0.7.tgz \
        && docker-php-ext-configure /tmp/yaf --enable-yaf \
        && docker-php-ext-install /tmp/yaf

        COPY ./memcached-3.0.4.tgz /tmp/memcached-3.0.4.tgz
RUN cd /tmp \
        && mkdir memcached \
        && tar -zxvf memcached-3.0.4.tgz -C memcached --strip-components=1 \
#        && rm memcached-3.0.4.tgz \
        && docker-php-ext-configure /tmp/memcached --enable-memcached \
        && docker-php-ext-install /tmp/memcached

#        COPY ./imagick-3.4.3.tgz /tmp/imagick-3.4.3.tgz
# RUN cd /tmp \
#        && mkdir imagick \
#       && tar -zxvf imagick-3.4.3.tgz -C imagick --strip-components=1 \
#        && rm imagick-3.4.3.tgz \
#       && docker-php-ext-configure /tmp/imagick --enable-imagick \
#        && docker-php-ext-install /tmp/imagick

        COPY ./redis-4.2.0.tgz /tmp/redis-4.2.0.tgz
RUN cd /tmp \
        && mkdir redis \
        && tar -zxvf redis-4.2.0.tgz -C redis --strip-components=1 \
#        && rm redis-4.2.0.tgz \
        && docker-php-ext-configure /tmp/redis --enable-redis \
        && docker-php-ext-install /tmp/redis

        COPY ./swoole-4.2.9.tgz /tmp/swoole-4.2.9.tgz
RUN cd /tmp \
        && mkdir swoole \
        && tar -zxvf swoole-4.2.9.tgz -C swoole --strip-components=1 \
#        && rm swoole-4.2.9.tgz \
        && docker-php-ext-configure /tmp/swoole --enable-swoole \
        && docker-php-ext-install /tmp/swoole

        COPY ./hachi0.0.3.tgz /tmp/hachi0.0.3.tgz
RUN cd /tmp \
        && mkdir hachi \
        && tar -zxvf hachi0.0.3.tgz -C hachi --strip-components=1 \
#        && rm hachi0.0.3.tgz \
        && docker-php-ext-configure /tmp/hachi --enable-hachi \
        && docker-php-ext-install /tmp/hachi

# 如果没有自己的php.ini则下列两行应被RUN mv $PHP_INI_DIR/php.ini.production $PHP_INI_DIR/php.ini替换
        COPY ./php.ini /tmp/php.ini
RUN mv /tmp/php.ini $PHP_INI_DIR/php.ini

RUN apt-get autoremove \
        && apt-get clean \
        && apt-get autoclean \
        && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

接下来在终端中执行
```bash
docker build -t php-fpm:v0.0.1 ./
#如果构建自己的dockerhub私有包则是
docker build -t <自己名字>/php-fpm:v0.0.1 ./
```

然后启动docker PHP
```bash
#启动php-fpm，加入my-net网络，后台执行，映射9000端口，始终跟随docker启动；
#重点注意：挂载/data/www/default到docker中的/data/www/default
docker run --name php-fpm --network my-net -d -p 9000:9000 --restart=always -v /data/www/default:/data/www/default php-fpm:v0.0.1
```

对应的nginx配置：
```text 
location ~ \.php$ {
        root           /data/www/default;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /data/www/default/$fastcgi_script_name;
        include        fastcgi_params;
    }
```

最后再说明几点：
* 如果要使用docker内部的my-net网络则再host处填写docker的name即可，比如php连接数据库，那么数据库的host就是mysql，连接memcached那么host就是memcached（看docker run写的--name）
* 如果要以cgi方式运行php且兼容fpm则docker run应改为：
```bash
docker run --name php --network my-net -d -p 9000:9000 --restart=always -v /data/www/default:/data/www/default php-fpm:v0.0.1 /usr/local/bin/php <你的php文件>
#或者精简的完全cgi模式
docker run --name php --network my-net -d php-fpm:v0.0.1 /usr/local/bin/php <你的php文件>
```

测试文件：
```php
<?php
//phpinfo();

$cc = doSql('root','select');
var_dump($cc);
function doSql($keyword, $method = 'count')
    {
        if ($method == 'count') {
            $sql = "select count(0) from user where User LIKE ?";
        } else {
            $sql = "select * from user where User LIKE ?";
        }

            $pdo = new PDO("mysql:host=mysql;port=3306;dbname=mysql", "root", "123456", array(
                PDO::ATTR_PERSISTENT => true
            ));
            $pdo->exec("set names utf8");

        $dopdo = $pdo->prepare($sql);
        $dopdo->execute(array(
            '%' . $keyword . '%'
        ));
        $res = $dopdo->fetch();

        if ($method == 'count') {
            return $res;
        } else {
            return $res;
        }
    }


$m = new Memcached;
$servers = array(
    array('memcached', 11211, 33),
    array('memcached', 11211, 67)
);
$m->addServers($servers);

var_dump($m);

$m->set('a','b',1);
var_dump($m->get('a'));

sleep(1); 

var_dump($m->get('a'));
```
