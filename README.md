# 简介
FastDFS是一个开源的分布式文件系统，[官方介绍](https://www.oschina.net/p/fastdfs)有详细的介绍，不多赘述。本文主要是FastDFS的搭建及采坑指南。

<!-- more -->

# Step By Step Guide

## 系统

- 阿里云ECS Ubuntu 16.04

## 编译环境

按需安装，这里是针对新的ubuntu系统

```bash
$ apt-get install git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel wget vim
```

## 磁盘目录

|说明|位置|
|-|-|
|所有安装包|/usr/local/src|
|数据存储位置|/data/dfs/|

```bash
$ mkdir /data/dfs #创建数据存储目录（对于阿里云ECS，最好建立在数据盘上，是用来存放文件的）
$ cd /usr/local/src #切换到安装目录准备下载安装包
```

## 安装libfatscommon

```bash
$ wget https://github.com/happyfish100/libfastcommon/archive/master.zip
$ unzip master.zip
$ cd libfastcommon-1.0.39/
$ ./make.sh && ./make.sh install #编译安装
```

## 安装FastDFS

```bash
$ cd ../ #返回上一级目录
$ wget https://github.com/happyfish100/fastdfs/archive/master.zip
$ unzip master.zip
$ cd fastdfs-master/
$ ./make.sh && ./make.sh install #编译安装
#配置文件准备
$ cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
$ cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
$ cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf #客户端文件，测试用
$ cp /usr/local/src/fastdfs-master/conf/http.conf /etc/fdfs/ #供nginx访问使用
$ cp /etc/nginx/mime.types /etc/fdfs/ #供nginx访问使用
```

## 安装fastdfs-nginx-module

官网的文档，是针对没有安装过Nginx的机器，重新编译了一遍Nginx，把module直接编译进Nginx了。但是针对已经安装Nginx的服务器来说，显然是不好的。

根据[Nginx官方文档-编译第三方动态模块](https://www.nginx.com/blog/compiling-dynamic-modules-nginx-plus/)，编译了`fastdfs-nginx-module`，以供已存在的Nginx使用。



### 准备`fastdfs-nginx-module`源码包

```bash
$ cd ../ #返回上一级目录
$ wget https://github.com/happyfish100/fastdfs-nginx-module/archive/master.zip
$ unzip master.zip
```

### 获取对应版本的Nginx源码包

```bash
$ nginx -v # 确认服务器的Nginx版本
nginx version: nginx/1.14.2
$ wget http://nginx.org/download/nginx-1.14.2.tar.gz
$ tar -xzvf nginx-1.14.2.tar.gz
```

### 编译动态模块

```bash
$ cd nginx-1.14.2/
$ ./configure --with-compat --add-dynamic-module=/usr/local/src/fastdfs-nginx-module-master/src
$ make modules
```

### 将模块库（.so文件）复制到/etc/nginx/modules

```bash
$ cp ngx_http_fastdfs_module.so /etc/nginx/modules/
```

### 加载并使用模块

Tips: 要将模块加载到Nginx,在nginx.conf文件开头添加`load_module`命令

```bash
$ vim /etc/nginx/nginx.conf
# 添加如下命令
load_module modules/ngx_http_fastdfs_module.so;
# 保存退出
```

### 添加FastDFS配置使模块生效

```bash
$ vim /etc/nginx/conf.d/fastdfs.conf
# 添加如下配置
server {
    listen 8888;    ## 该端口为storage.conf中的http.server_port相同
    server_name {your_domain};
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

## 单机部署

这里只描述下单机环境的部署方式，集群在官方文档有，没有实际使用过。

### tracker配置

```bash
$ vim /etc/fdfs/tracker.conf
# 建议修改以下内容
bind_addr={你的内网IP}
base_path=/data/dfs # 建议修改为数据盘位置
# 可选修改
port=22122  # tracker服务器端口
```

### storage配置

```bash
$ vim /etc/fdfs/storage.conf
# 建议修改
base_path=/data/dfs  # 数据和日志文件存储根目录（建议修改为数据盘位置）
store_path0=/data/dfs  # 第一个存储目录（建议修改为数据盘位置）
tracker_server={tracker.bind_addr}:{tracker.port}  # tracker服务器IP和端口
http.server_port=8888  # http访问文件的端口（默认8888,看情况修改,和nginx中保持一致）
# 可选修改
port=23000  # storage服务端口（默认23000,一般不修改）
```

### client测试

```bash
$ vim /etc/fdfs/client.conf
# 建议修改
base_path=/data/dfs
tracker_server={tracker.bind_addr}:{tracker.port}  # tracker服务器IP和端口
# 保存后测试
$ fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.14.2.tar.gz
group1/M00/00/00/CgoKvlyUmi-AMVKDAA9-WL9wzEw.tar.gz # 下载时通过该ID下载
# 返回ID表示成功 如：group1/M00/00/00/xx.tar.gz
```

### 配置nginx访问

```bash
vim /etc/fdfs/mod_fastdfs.conf
# 建议修改
tracker_server={tracker.bind_addr}:{tracker.port}  # tracker服务器IP和端口
url_have_group_name=true
store_path0=/data/dfs
# 修改完保存
$ nginx -s reload
ngx_http_fastdfs_set pid=8364 # 看见这条消息说明nginx模块启动成功了
$ lsof -i:8888 # 查看Nginx下载端口是否正常启动
nginx   31061  root   10u  IPv4 20389985      0t0  TCP *:8888 (LISTEN)
```

### 测试下载

在浏览器输入

```
http://{IP}:8888/group1/M00/00/00/CgoKvlyUmi-AMVKDAA9-WL9wzEw.tar.gz?filename=nginx-1.14.2.tar.gz //刚才上传返回的ID
```

弹出下载文件框，说明部署成功！

![](/FastDFS-单机部署指南/20190322043758235.png)

## 相关命令

### 防火墙

```bash
$ sudo ufw enable|disable
```

### tracker

```bash
$ /etc/init.d/fdfs_trackerd start # 启动tracker服务
$ /etc/init.d/fdfs_trackerd restart # 重启动tracker服务
$ /etc/init.d/fdfs_trackerd stop # 停止tracker服务
$ update-rc.d fdfs_trackerd enable # 自启动tracker服务
```

### storage

```bash
$ /etc/init.d/fdfs_storaged start # 启动storage服务
$ /etc/init.d/fdfs_storaged restart # 重动storage服务
$ /etc/init.d/fdfs_storaged stop # 停止动storage服务
$ update-rc.d fdfs_storaged enable # 自启动storage服务
```

### nginx

```bash
$ service nginx start # 启动nginx
$ nginx -s reload # 重启nginx
$ nginx -s stop # 停止nginx
```

## 问题

### 执行nginx -s reload 后，访问502

```bash
# 查看nginx日志
$ vim /var/log/nginx/error.log
```

如果发现错误日志：`include file "http.conf" not exists, line: "#include http.conf"`，fastdfs nginx模块缺少配置文件，执行以下命令补全配置文件即可。

```bash
$ cp /usr/local/src/fastdfs-master/conf/http.conf /etc/fdfs/ #供nginx访问使用
$ cp /etc/nginx/mime.types /etc/fdfs/ #供nginx访问使用
```











