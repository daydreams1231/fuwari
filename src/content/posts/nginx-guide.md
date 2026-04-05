---
title: Nginx学习
published: 2026-03-07
description: 'Debian13 Nginx学习记录'
image: ''
tags: [Linux, HTTP, Nginx, Web]
category: '随笔'
draft: false 
lang: ''
---

# 安装
```shell
# 从apt源安装
sudo apt install nginx

# 从源码安装
## 下载源码
wget https://nginx.org/download/nginx-1.29.5.tar.gz
## 解压
tar xvf nginx-1.29.5.tar.gz && cd nginx-1.29.5
## 配置编译选项 (最简形式即 --prefix=/usr/bin)
./configure --help

make && make install
```

## 配置文件
主配置文件为 `/etc/nginx/nginx.conf`, 一般不需要特别去动这个 <br>
该配置文件会自动加载 `/etc/nginx/conf.d/*.conf` 和 `/etc/nginx/sites-enabled/*` 下的子配置文件 <br>
一般来说一个网站就用一个配置文件, 这样方便管理 <br>

```text
# 使用哪个用户来运行nginx, 主配置文件一般会指定为 www-data
user www-data;
# nginx进程数, auto即可
worker_processes  auto;
# nginx进程CPU亲和性设置, 也保持auto即可
worker_cpu_affinity auto;
# nginx run pid
pid /var/run/nginx.pid;
# 错误日志
error_log /var/log/nginx/error.log warn;

events {
    # 轮询方法选择
    use epoll;
    # 对于单个nginx进程, 其可接受的最大连接数, CPU单核性能越强, 这个值就可相应的越高
    worker_connections 1024;
}

http {
    # 开启高效传输模式
    sendfile on;
    # 减少网络报文段的数量
    tcp_nopush on;

    types_hash_max_size 2048;
    server_tokens off;
   # Nginx访问日志存放位置
    access_log  /var/log/nginx/access.log  main;
    gzip on;

    # http请求保持时间, 单位为秒, 超过这个时间的空闲连接会被关闭
    keepalive_timeout 60;

    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    
    # server块需要单独讲, 这里省略了
    # http块里可有多个server块
    server {}
}
```

### server块
```
server {
    # 监听端口
    listen 80;
    # 匹配Http请求的域名, 如果nginx只有一个域名, 这里可填 _ 或者该域名
    # 如果有不同的域名都需要80/443端口, 则必须配置该选项, 让nginx能区分谁是谁
    server_name _;

    # 匹配uri请求
    location    /   {
        # 网站根目录
    	root /usr/share/nginx/html;
        # 网站默认页面 (即访问 http://域名/ 时会显示的页面, 等效于 http://域名/index.html)
    	index index.html index.htm;
    }
    
    error_page 500 502 503 504 /50x.html;  # 默认50x对应的访问页面
    error_page 400 404 error.html;   # 同上
    }
```
location 指令后面：
  - = 精确匹配路径，用于不含正则表达式的 uri 前，如果匹配成功，不再进行后续的查找；
  - ^~ 用于不含正则表达式的 uri 前，表示如果该符号后面的字符是最佳匹配，采用该规则，不再进行后续的查找；
  - ~ 表示用该符号后面的正则去匹配路径，区分大小写；
  - ~* 表示用该符号后面的正则去匹配路径，不区分大小写。跟 ~ 优先级都比较低，如有多个location的正则能匹配的话，则使用正则表达式最长的那个；

如果 uri 包含正则表达式，则必须要有 ~ 或 ~* 标志. <br>
```
Nginx 有一些常用的全局变量，你可以在配置的任何位置使用它们，如下表：
全局变量名	功能
$host	请求信息中的 Host，如果请求中没有 Host 行，则等于设置的服务器名，不包含端口
$request_method	客户端请求类型，如 GET、POST
$remote_addr	客户端的 IP 地址
$args	请求中的参数
$arg_PARAMETER	GET 请求中变量名 PARAMETER 参数的值，例如：$http_user_agent(Uaer-Agent 值), $http_referer...
$content_length	请求头中的 Content-length 字段
$http_user_agent	客户端agent信息
$http_cookie	客户端cookie信息
$remote_addr	客户端的IP地址
$remote_port	客户端的端口
$http_user_agent	客户端agent信息
$server_protocol	请求使用的协议，如 HTTP/1.0、HTTP/1.1
$server_addr	服务器地址
$server_name	服务器名称
$server_port	服务器的端口号
$scheme	HTTP 方法（如http，https）

还有更多的内置预定义变量，可以直接搜索关键字「nginx内置预定义变量」可以看到一堆博客写这个，这些变量都可以在配置文件中直接使用。
```