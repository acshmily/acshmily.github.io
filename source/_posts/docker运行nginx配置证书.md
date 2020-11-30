---
title: docker运行nginx配置证书
date: 2020-11-30 21:18:32
tags:
	- nginx
	- docker
	- ssl
---

## 拉取nginx镜像

`docker pull nginx`

## 初始化运行

`docker run -p 443:443 -p 80:80 --name nginx \
-v /data/nginx/html:/usr/share/nginx/html \
-v /data/nginx/logs:/var/log/nginx \
-v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /data/nginx/conf/cert:/etc/nginx/cert \
-v /etc/localtime:/etc/localtime \
-d nginx`

分别映射nginx的html、log、config

预期会启动失败，因为/data/nginx/conf/nginx.conf : /etc/nginx/nginx.conf docker会默认mount一个文件夹，实际上是个文件

`rm -rf /data/nginx/config/nginx.conf`

创建正确的nginx配置

```
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    gzip  on;
	

	  server {
		listen	443;   #监听端口，ssl默认443端口。如果需要配置多个端口，可以继续添加server，用不同的端口就行
    #证书文件名称
    ssl_certificate #.crt; 
    #私钥文件名称
    ssl_certificate_key #.key; 
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
    
    server_name  ;   #服务器域名，需要和申请的证书匹配
		
		location / {
			root  /usr/share/nginx/html;  #网站根目录，和容器创建时指定的位置一致
			index index.html index.htm;
		}
	}
	# 301默认跳转到443
  server {
    listen 80;
    #填写绑定证书的域名
    server_name www.jq1024.com jq1024.com;
    #把http的域名请求转成https
    return 301 https://$host$request_uri;
  }
}
```

## 重新运行

`docker restart nginx`

done

