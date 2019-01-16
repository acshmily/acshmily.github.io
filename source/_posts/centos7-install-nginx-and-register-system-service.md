---
title: 在Centos7下手动安装NGINX并注册为服务
date: 2019-01-16 13:45:17
tags: nginx 不务正业 centos
---

# 在Centos7下手动安装NGINX并注册为服务

​	由于公司部分服务器操作系统参差不齐并且Nignx版本也不尽一致，存在安全以及配置各种问题，索性安排同事全部推倒重来，由于团队任务过重，我也就来凑热闹来帮忙了。

## 目标	

-  安装最新的Nginx

-  添加ngx_log_if模块

-  注册为centos的服务并开机自动重启

## 准备工作

1. 最新的Nginx源代码 

   [地址点我]: https://nginx.org/en/download.html	"Nginx下载官网"

2. 下载ngx_log_if模块,保存 ngx_http_aclog_bypass_module.c 以及config文件，保存为文件夹名为 ngx_log_if

   [ngx_log_if模块github项目]: https://github.com/cfsego/ngx_log_if	"ngx_log_if"

## 安装

​	本文以当前时间2019年01月16日最新NGINX版本号为1.15.8仅供参考

1. 上传 Nginx.tar.gz 以及ngx_log_if文件夹

2. 解下tar -zxvf Nginx.tar.gz

3. ```
   cd Nginx
   ```

4. ```
   useradd -s /sbin/nologin nginx -M  
   ```

5. ```
   ./configure --prefix=/usr/local/nginx  --user=nginx --group=nginx --with-http_stub_status_module  --with-http_ssl_module --add-module=/home/hbadmin/ngx_log_if
   ```

   注意：--prefix 代表即将编译的目标路径 --user 运行的用户 --group代表用户组 --with代表安装哪些模块 --add-moudle代表外部加载模块

6. ```
   make && make install
   ```

##配置	


   ```
   vim /etc/init.d/nginx
   ```

 参考配置，代码块内nginx=以及NGINX_CONF_FILE按照你的nginx配置进行修改

   ```
   #!/bin/sh
   #
   # nginx - this script starts and stops the nginx daemon
   #
   # chkconfig:   - 85 15
   # description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
   #               proxy and IMAP/POP3 proxy server
   #processname: nginx
   # config:      /etc/nginx/nginx.conf
   # config:      /etc/sysconfig/nginx
   #pidfile:     /usr/local/nginx/logs/nginx.pid
   
   # Source function library.
   . /etc/rc.d/init.d/functions
   
   # Source networking configuration.
   . /etc/sysconfig/network
   
   # Check that networking is up.
   [ "$NETWORKING" = "no" ] && exit 0
   
   nginx="/usr/local/nginx/sbin/nginx"
   prog=$(basename $nginx)
   
   NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"
   
   [ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
   
   lockfile=/var/lock/subsys/nginx
   
   make_dirs() {
      # make required directories
      user=`$nginx -V 2>&1 | grep "configure arguments:.*--user=" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
      if [ -n "$user" ]; then
         if [ -z "`grep $user /etc/passwd`" ]; then
            useradd -M -s /bin/nologin $user
         fi
         options=`$nginx -V 2>&1 | grep 'configure arguments:'`
         for opt in $options; do
             if [ `echo $opt | grep '.*-temp-path'` ]; then
                 value=`echo $opt | cut -d "=" -f 2`
                 if [ ! -d "$value" ]; then
                     # echo "creating" $value
                     mkdir -p $value && chown -R $user $value
                 fi
             fi
          done
       fi
   }
   
   start() {
       [ -x $nginx ] || exit 5
       [ -f $NGINX_CONF_FILE ] || exit 6
       make_dirs
       echo -n $"Starting $prog: "
       daemon $nginx -c $NGINX_CONF_FILE
       retval=$?
       echo
       [ $retval -eq 0 ] && touch $lockfile
       return $retval
   }
   
   stop() {
       echo -n $"Stopping $prog: "
       killproc $prog -QUIT
       retval=$?
       echo
       [ $retval -eq 0 ] && rm -f $lockfile
       return $retval
   }
   
   restart() {
       configtest || return $?
       stop
       sleep 1
       start
   }
   
   reload() {
       configtest || return $?
       echo -n $"Reloading $prog: "
       killproc $nginx -HUP
       RETVAL=$?
       echo
   }
   
   force_reload() {
       restart
   }
   
   configtest() {
     $nginx -t -c $NGINX_CONF_FILE
   }
   
   rh_status() {
       status $prog
   }
   
   rh_status_q() {
       rh_status >/dev/null 2>&1
   }
   
   case "$1" in
       start)
           rh_status_q && exit 0
           $1
           ;;
       stop)
           rh_status_q || exit 0
           $1
           ;;
       restart|configtest)
           $1
           ;;
       reload)
           rh_status_q || exit 7
           $1
           ;;
       force-reload)
           force_reload
           ;;
       status)
           rh_status
           ;;
       condrestart|try-restart)
           rh_status_q || exit 0
               ;;
       *)
           echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
           exit 2
   esac
   ```

修改权限

      ```
       chmod 755 /etc/init.d/nginx
      ```

 添加系统服务

	```
	 		vim /lib/systemd/system/nginx.service  
	```

修改为以下内容，部分配置变量需要结合实际进行修改

   ```
   
   [Unit]
   Description=nginx - high performance web server
   Documentation=http://nginx.org/en/docs/
   After=network.target remote-fs.target nss-lookup.target
   
   [Service]
   Type=forking
   PIDFile=/usr/local/nginx/logs/nginx.pid
   ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
   ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
   ExecReload=/bin/kill -s HUP $MAINPID
   ExecStop=/bin/kill -s QUIT $MAINPID
   PrivateTmp=true
   
   [Install]
   WantedBy=multi-user.target
   ```
相关命令

   ```
   #开机自启动
   systemctl enable nginx.service
   #停止开机自启动
   systemctl disable nginx.service
   #查询当前状态
   systemctl status nginx.service
   #启动服务
   systemctl start nginx.service
   #重新启动服务
   systemctl restart nginx.service
   #停止服务
   systemctl stop nginx.service
   #重新加载配置
   systemctl reload nginx.service
   #查看所有已启动的服务
   systemctl list-units --type=service
   ```

   ## 参考

   https://www.nginx.com/resources/wiki/start/topics/examples/systemd/