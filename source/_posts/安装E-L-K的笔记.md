---
title: 安装E.L.K的笔记
date: 2018-10-16 16:27:16
tags:
	-	ELK
	-	nginx
	-	springboot
categories:	工作笔记
---

# 关于搭建E.L.K的笔记

[TOC]

​	由于业务需要，需要的nginx以及系统日志进行汇总统计，并隔离开发人员对生产环境的访问。沿用比较成熟的E.L.K方案可以比较好的解决当前问题。本文仅当模拟搭建的时候进行的笔记记录，做为拾遗。安装环境为Centos7-X64，需要有外网访问权限以及yum，安装时请关闭防火墙以及IPV6网络服务。本文使用的E.L.K的大版本为6



## 安装nginx

​	采用阿里巴巴的分支tengine，具体安装请参考文档http://tengine.taobao.org/ 。

​	对nginx输出的日志格式进行定义，由于E.L.K的Grok比较繁琐，故采用JSON格式进行传输。对nginx.conf进行修改配置，格式参考如下：

```
        log_format json '{ "@timestamp": "$time_iso8601", '
         '"remote_addr": "$remote_addr", '
         '"remote_user": "$remote_user", '
         '"body_bytes_sent": "$body_bytes_sent", '
         '"request_time": "$request_time", '
         '"status": "$status", '
         '"request_uri": "$request_uri", '
         '"request_method": "$request_method", '
         '"http_referrer": "$http_referer", '
         '"http_x_forwarded_for": "$http_x_forwarded_for", '
         '"http_user_agent": "$http_user_agent"}';
 	access_log /data/logs/access.log  json;
    access_log  "pipe:rollback logs/access_log interval=1d baknum=7 maxsize=2G"  json;
```

配置完毕后可以对nginx进行reload，并尝试访问。如果正常应该返回类似如下日志：

```
114.114.114.114 - - [08/Oct/2018:21:47:04 -0400] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36" "-"
172.30.0.51 - - [08/Oct/2018:21:47:05 -0400] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36" "-"
```

## 安装JAVA8

​	由于E.L.K套件是运行在JAVA8上，所以需要安装相应依赖。

```
yum -y install java8
```

​	安装成功请使用

```
java -version
```

​	进行检测，如果显示正常的java版本则为安装成功

## Logstash的安装和配置

### 添加E.L.K的GPG-KEY

​	rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch (需要关闭服务器IPV6寻址服务)
	增加repo 官方文档提示为 

> Add the following in your /etc/yum.repos.d/ directory in a file with a .repo suffix, for example logstash.repo

```
	[logstash-6.x]
	name=Elastic repository for 6.x packages
	baseurl=https://artifacts.elastic.co/packages/6.x/yum
	gpgcheck=1
	gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
	enabled=1
	autorefresh=1
	type=rpm-md
```

​	安装logstash 

```
	sudo yum install logstash
```

### 对Nginx的日志进行解析配置

#### 1.备份原有默认配置

```
mkdir /opt/backups/logstash -p
mv /etc/logstash/logstash.yml /opt/backups/logstash/logstash.yml.BAK
```

#### 2.配置GeoIP

```
cd /etc/logstash/
wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz
gunzip GeoLite2-City.mmdb.gz
```

#### 3.安装插件

Logstash需要安装插件,跳转到lgostash/bin目录下 执行

```
 1)在线安装    ./logstash-plugin install logstash-codec-json_lines
 2)离线安装    https://rubygems.org/gems/logstash-codec-json_lines
 
```



#### 4.配置nginx的配置文件

在路径/etc/logstash/conf.d/下创建nginx.conf，下文中的#请在使用时删除，避免无法运行

```
input {
    file {#文件模式解析
        type = >"nginx_access" #定义类型
        path = >["/data/logs/access.log"] #文件路径
        start_position = >"beginning"#何时开始
        sincedb_path = >"/dev/null"#何处开始
    }

}
filter {
    if [type] == "nginx_access" {#如果为nginx_access类型
        json {
            source = >"message"
            remove_field = >"message"
        }
        mutate {
            convert = >["response", "integer"] 
            convert = >["bytes", "integer"] 
            convert = >["responsetime", "float"]
        }
        geoip {#GeoIP插件
            source = >"http_x_forwarded_for"
            target = >"geoip"
            add_tag = >["nginx-geoip"] 
            database = >"/etc/logstash/GeoLite2-City.mmdb"
        }
        date {
            match = >["timestamp", "dd/MMM/YYYY:HH:mm:ss Z"] 
            remove_field = >["timestamp"]
        }
        useragent {
            source = >"agent"
        }
    }
    output {#运往何处
        if [type] == "nginx_access" {
            elasticsearch {
                hosts = >["172.30.1.19:9200"] #elasticsearch服务器地址
                index = >"logstash-nginx-access-%{+YYYY.MM.dd}"#索引名称
                document_type = >"nginx_logs"#解析模板
            }
        }
        if [type] == "spring_boot" {
            elasticsearch {
                hosts = >["172.30.1.19:9200"] 
                index = >"springboot-%{+YYYY.MM.dd}"
                codec = >"json"
            }
        }
        stdout {
            codec = >rubydebug
        }
    }
```

## 安装elasticsearch

安装很简单不在此文具体表述，注意不要使用root账户去运行elasticsearch

```
yum -y install elasticsearch
```

启动执行

```
systemctl start elasticsearch.service
```

## 安装kibana

```
yum -y install kibana
```

启动

```
systemctl start kabana.service
```

## 安装elasticsearch Head 

略过，请Google，无难度

## SpringBoot日志输入到E.L.K

项目方面增加Maven依赖

```xml
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>5.0</version>
        </dependency>
```

如项目没有logback相关配置文件需要新建logback.xml或者修改已存在的logback配置信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration>
<configuration>
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>192.168.226.132:4560</destination><!-- logstash接收地址->
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>
  
    <root level="INFO">
        <appender-ref ref="LOGSTASH" />
        <appender-ref ref="CONSOLE" />
    </root>
 
```

增加解析Logstash的配置模板

```
input {
    tcp {
        port => 4560   
        codec => json_lines
		type => "springboot_xxxx"
    }
	
}
output{

  stdout { codec => rubydebug }
}


```

## 排错

E.L.K安装并不复杂，复杂是在启动配置，对于较多系统logstash配置往往会出错可以使用

```
cat /etc/logstash/conf.d/* > /tmp/total.conf
```

查看total.conf文件进行配置文件的定位