                            nginx应用    
-------------------------------------------------------------------------------------
# 一 nginx安装
---
* 1.1 安装基础环境包
<pre>
yum install -y pcre pcre-devel #支持伪静态功能rewrite 
yum install -y openssl openssl-devel 
rpm -qa pcre pcre-devel
pcre-7.8-7.el6.x86_64
pcre-devel-7.8-7.el6.x86_64
</pre> 

* 1.2 下载并且安装nginx服务
<pre>
wget -q http://nginx.org/download/nginx-1.6.3.tar.gz #下载nginx软件
1.2 编译安装
useradd nginx -s /sbin/nologin -M #添加用户
tar xf nginx-1.6.3.tar.gz         #减压配置文件
cd nginx-1.6.3                    #进入安装目录
./configure  --user=nginx --group=nginx --prefix=/application/nginx-1.6.3/ --with-http_stub_status_module  --with-http_ssl_module  #编译安装参数
make
make install
ln -s /application/nginx-1.6.3 /application/nginx  #创建软链接
/application/nginx/sbin/nginx -V #  查看编译参数
</pre>

* 1.3 最小化配置文件
<pre>
egrep -v "#|^$" nginx.conf.default > nginx.conf #过滤#号和空号

## 二 虚拟主机
---
* 2.1 基于域名的虚拟主机  
<pre>
cat nginx.conf
worker_processes  1; #服务员
events {
    worker_connections  1024; #每个服务员可以处理的并发连接
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / {
            root   html/www;
            index  index.html index.htm;
        }
    }
    server {
        listen       80;
        server_name  bbs.etiantian.org;
        location / {
            root   html/bbs;
            index  index.html index.htm;
        }
    }
    server {
        listen       80;
        server_name  blog.etiantian.org;
        location / {
            root   html/blog;
            index  index.html index.htm;
        }
    }
}
</pre>

* 2.1 基于端口的虚拟主机
<pre>
cat nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / {
            root   html/www;
            index  index.html index.htm;
        }
    }
   server {
        listen       81;
        server_name  bbs.etiantian.org;
        location / {
            root   html/bbs;
            index  index.html index.htm;
        }
    }
   server {
        listeen       82;
        server_name  blog.etiantian.org;
        location / {
            root   html/blog;
            index  index.html index.htm;
        }
    }
}  
</pre>

* 2.2 基于ip的虚拟主机   
<pre>
cat nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       172.16.1.8:80;
        server_name  172.16.1.8;
        location / {
            root   html/www;
            index  index.html index.htm;
        }
    }
    server {
        listen       10.0.0.8:80;
        server_name  10.0.0.8;
        location / {
            root   html/bbs;
            index  index.html index.htm;
        }
    }
    server {
        listen       10.0.0.4:80;
        server_name  10.0.0.4;
        location / {
            root   html/blog;
            index  index.html index.htm;
        }
    }
}
</pre>

* 2.3 nginx_status状态虚拟主机
<pre>
cat status.conf 
##status
server{
    listen  80;
    server_name  status.etiantian.org;
    location / {
      stub_status on;   #开启status模块
      access_log   off; #开启访问日志
    }
  }
</pre>
## 三 nginx访问日志切割
---
<pre>
cat /server/scripts/cut_nginx_log.sh 
#!/bin/sh
date=`date +%F -d "-1day"`
cd /application/nginx/logs &&\
mv www_access.log www_access_${date}.log
mv bbs_access.log bbs_access_${date}.log
mv blog_access.log blog_access_${date}.log
/application/nginx/sbin/nginx -s reload
crontab -l
00 00 * * *   /bin/sh /server/scripts/cut_nginx_log.sh >/dev/null 2>&1
</pre>
## 3.1 location 匹配顺序 #重点
<pre>
0=============================================
http://www.etiantian.org
http://www.etiantian.org/
        location = / {
            return 402;
        }
结尾=============================================
http://www.etiantian.org/index.html
        location / {
           return 401;
        }
3=============================================
http://www.etiantian.org/documents/document.html
        location /documents/ {
            return 403;
        }
1=============================================
http://www.etiantian.org/images/1.gif 
        location ^~ /images/ {
            return 404;
        }
2=============================================
http://www.etiantian.org/documents/1.jpg
         location ~* \.(gif|jpg|jpeg)$ {
            return 500;
        }
</pre>

* 3.1 nginx 301跳转
<pre>
server {
        listen       80;
        server_name  etiantian.org;
        rewrite ^/(.*) http://www.oldboyedu.com/$1 permanent;
    }
</pre>

* 3.2 nginx302跳转
<pre>
server {
        listen       80;
        server_name  etiantian.org;
        rewrite ^/(.*) http://www.oldboyedu.com/$1 redirect;
    }
#301 永久跳转
#302 临时跳转
#理解：302跳转是基于域名跳转，别名是本地 
</pre>
[301和302的概念理解](http://oldboy.blog.51cto.com/2561410/1774260)

* 3.3 nginx域名跳转 

> www.etiantian.org/bbs 跳转 http://bbs.etiantian.org/   
>> rewrite ^(.*)/bbs/ http://bbs.etiantian.org/ break;
<pre>
-------------------------------------------------------------
    server {
        listen       80;
        server_name  etiantian.org;
        rewrite ^(.*)/bbs/ http://bbs.etiantian.org/ break;
    }
</pre>

* 3.4 网站加密
<pre>
cat bbs.conf 
   server {
        listen       80;
        server_name  bbs.etiantian.org;
        location / {
            root   html/bbs;
            index  index.html index.htm;
            auth_basic "You Know,fro ";
            auth_basic_user_file /application/nginx/conf/kibana.auth;
         } 
        location /status{
        stub_status on;
        access_log   off;
        }
        access_log logs/bbs_access.log main;
    }
-----------------------------------------------------------------------
auth_basic "You Know,fro ";  #提示信息
auth_basic_user_file /application/nginx/conf/kibana.auth;  #密码文件位置
-----------------------------------------------------------------------
给网站设置密码：
htpasswd -bc /application/nginx/conf/htpasswd oldboy 123456
yum install httpd-tools #安装htpasswd命令
</pre>
* 3.5 自定义status模块
<pre>
  server {
        listen       80;
        server_name  bbs.etiantian.org;
        location / {
            root   html/bbs;
            index  index.html index.htm;
         }
        location /status{
        stub_status on;
        access_log   off;
        }
        access_log logs/bbs_access.log main;
    }
------------------------------------------------
Active connections: 4 
server accepts handled requests
 123 123 128 
Reading: 0 Writing: 1 Waiting: 3
------------------------------------------------

nginx status详解
active connections – 活跃的连接数量
server accepts handled requests — 总共处理了123个连接 , 成功创建123次握手, 总共处理了128个请求
reading — 读取客户端的连接数.
writing — 响应数据到客户端的数量
waiting — 开启 keep-alive 的情况下,这个值等于 active – (reading+writing), 意思就是 Nginx 已经处理完正在等候下一次请求指令的驻留连接.
</pre>
* 3.6 添加访问日志  
<pre>
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / {
            root   html/www;
            index  index.html index.htm;
        }
       access_log logs/www_access.log main; #访问日志
    }
</pre>

* 3.6 自定义多个项目
<pre>
cat nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" ' #日志格式
                      '$status $body_bytes_sent "$http_referer" '    #日志格式
                      '"$http_user_agent" "$http_x_forwarded_for"';  #日志格式
    include       extar/*.conf;  #在conf目录下创建extar目录用于自定义方便管理
}

tree extar/
extar/
|-- bbs.conf
|-- blog.conf
|-- status.conf
`-- www.conf
</pre>
> 常用检查    
/application/nginx/sbin/nginx -t  #检查语法   
/application/nginx/sbin/nginx  -v #查看nginx版本   
/application/nginx/sbin/nginx  -V #查看nginx安装了哪些模块   
反向代理或负载均衡服务   #nginx可以做什么
前端业务数据缓存服务     #nginx可以做什么
web服务器              #nginx可以做什么
php                   #并发连接为300-1000   
java                  #并发连接为300-1500 数据库 并发连接为300-1500  
>说明：  
[nginx官方网站](http://nginx.org/)    
[http状态码说明](http://oldboy.blog.51cto.com/2561410/716294)  
[nginx模块说明](http://wenku.baidu.com/link?url=siGC4qCTshveWMmein2UrFQ6vx7ofT-_Nw--ySUQYTO0wV3MyfrUYzIhvMUdFeiKAKJOZL83KG3XwWa32Tce182bFcVwuyd-hcF1YPGa_da)      
  





