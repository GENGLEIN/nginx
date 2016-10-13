# nginx负载均衡   
* 负载均衡软件 
* 1.1 nginx    # 中小企业
* 1.2 lvs      # LVS（tcp）+Nginx(http) 大型企业
* 1.3 haproxy  # HAPROXY 专业的负载软件
#安装 
<pre>
# yum install openssl openssl-devel pcre pcre-devel -y
# mkdir -p /home/oldboy/tools
# cd /home/oldboy/tools
# wget -q http://nginx.org/download/nginx-1.6.3.tar.gz
# ls -l nginx-1.6.3.tar.gz
# useradd nginx -s /sbin/nologin -M
# tar xf nginx-1.6.3.tar.gz
# cd nginx-1.6.3
# ./configure  --user=nginx --group=nginx --prefix=/application/nginx-1.6.3 --with-http_stub_status_module  --with-http_ssl_module
# make
# make install
# ln -s /application/nginx-1.6.3 /application/nginx
# /application/nginx/sbin/nginx
</pre>

## 实现nginx需要俩个模块 
* 2.1 ngx_http_upstream_module 
* 2.2 ngx_http_proxy_module
## 三 负载均衡  
<pre>
[root@lb01 conf]# cat nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    upstream backend {
    server 172.16.1.7:80  weight=1;
    server 172.16.1.8:80  weight=1;
}
    server {
        listen       80;
        server_name  blog.etiantian.org;
        location / {
            index  index.blog index.htm;
            proxy_pass http://backend;
            proxy_set_header Host  $host;
            roxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        }
    }
}




</pre>
相同域名不同业务的local简析
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
upstream upload_pools {
         server 10.0.0.8:80  weight=1;
    }
upstream default_pools {
         server 10.0.0.8:8080  weight=1;
   }
upstream static_pools {
         server 10.0.0.7:80  weight=1;
}
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / { 
        proxy_pass http://default_pools;
        include proxy.conf;
       }
         location /static/ { 
        proxy_pass http://static_pools;
        include proxy.conf;
      }
        location /upload/ { 
        proxy_pass http://upload_pools;
        include proxy.conf;
     }
    }
}
-----------
nginx健康检查：
wget https://codeload.github.com/yaoweibin/nginx_upstream_check_module/zip/master
unzip master
cd nginx-1.6.3
patch -p1 < ../nginx_upstream_check_module-master/check_1.5.12+.patch
pkill nginx
./configure --prefix=/application/nginx-1.6.3 --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --add-module=../nginx_upstream_check_module-master/
make
mv /application/nginx/sbin/nginx{,.ori}
cp ./objs/nginx /application/nginx/sbin/
/application/nginx/sbin/nginx -V
/application/nginx/sbin/nginx

[root@lb01 nginx-1.6.3]# cat /application/nginx/conf/nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
#upload server
upstream android_pools {
         server 10.0.0.8:80  weight=1;
      check interval=3000 rise=2 fall=5 timeout=1000 type=http;
    }
upstream pc_pools {
         server 10.0.0.8:8080  weight=1;
 check interval=3000 rise=2 fall=5 timeout=1000 type=http;
   }
#static
upstream iphone_pools {
         server 10.0.0.7:80  weight=1;
 check interval=3000 rise=2 fall=5 timeout=1000 type=http;
}
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / {
        if ($http_user_agent ~* "android")
          {
            proxy_pass http://android_pools;
          }
        if ($http_user_agent ~* "iphone")
          {
            proxy_pass http://iphone_pools;
            }
        proxy_pass http://pc_pools;
        include proxy.conf;
       }
        location /status {
            check_status;
            access_log off;
        }

    }
}
------------------
-------------不同设备访问不同站点目录------
pc:
http://192.168.16.41
android
http://192.168.16.41/ 
iphone
http://192.168.16.41/static/

worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
#upload server
upstream android_pools {
         server 10.0.0.8:80  weight=1;
    }
upstream pc_pools {
         server 10.0.0.8:8080  weight=1;
   }
#static
upstream iphone_pools {
         server 10.0.0.7:80  weight=1;
}
    server {
        listen       80;
        server_name  www.etiantian.org;
        location / {
        if ($http_user_agent ~* "android")
          {
            proxy_pass http://android_pools;
          }
        if ($http_user_agent ~* "iphone")
          {
            proxy_pass http://iphone_pools;
            }
        proxy_pass http://pc_pools;
        include proxy.conf;
       }
    }
}