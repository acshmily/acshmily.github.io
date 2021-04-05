---
title: kubernetes安装手记stage1
date: 2021-04-05 21:37:06
tags:
	-	kubernetes
	-	nginx
	-	springboot
categories:	kubernetes笔记
---

#kubernetes安装手记stage1    

​    由于只是研究学习用，刚好公司有资源可以申请，趁着有兴趣进行了搭建一下。

​	本次用到的硬件服务器约为9台，分别为：

| ip            | 配置         | 角色                  |
| ------------- | ------------ | --------------------- |
| 192.168.82.21 | 12U 64G 500G | master、node节点      |
| 192.168.82.22 | 12U 64G 500G | master、node节点      |
| 192.168.82.23 | 12U 64G 500G | master、node节点      |
| 192.168.82.24 | 12U 64G 500G | node节点              |
| 192.168.82.25 | 12U 64G 500G | node节点              |
| 192.168.82.26 | 4U 8G 500G   | harbor、yaml 配置节点 |
| 192.168.82.27 | 8U 16G 500G  | etcd 节点             |
| 192.168.82.28 | 8U 16G 500G  | etcd 节点             |
| 192.168.82.29 | 8U 16G 500G  | etcd 节点             |
| 192.168.82.30 | 4U 8G 500G   | keepalived节点、bind9 |
| 192.168.82.31 | 4U 8G 500G   | keepalived节点        |

## 部署bind9

`yum install bind9`

配置相关信息，主机域

```bash
cat >/var/named/host.com.zone <<'EOF'
$ORIGIN host.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
                2021033001 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.host.com.
$TTL 60 ; 1 minute
dns                A    192.168.83.30
HDSS7-21           A    192.168.82.21
HDSS7-22           A    192.168.82.22
HDSS7-23           A    192.168.82.23
HDSS7-24           A    192.168.82.24
HDSS7-25           A    192.168.82.25
HDSS7-26           A    192.168.82.26
HDSS7-27           A    192.168.82.27
HDSS7-28           A    192.168.82.28
HDSS7-29           A    192.168.82.29
HDSS7-30           A    192.168.82.30
HDSS7-31           A    192.168.82.31
EOF
```

配置业务域

```bash
cat >/var/named/bst.com.zone <<'EOF'
$ORIGIN bst.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.bst.com. dnsadmin.bst.com. (
                2021033001 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.zq.com.
$TTL 60 ; 1 minute
dns                A    192.168.82.30
harbor             A    192.168.82.26
EOF
```

所有节点都需要修改DNS配置为 192.168.82.30

## 安装Docker环境

` curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun `

每一台k8s节点都需要分别配置deamon.json，以182.21为例，bip为 172.7.21.1/24；同理 182.22 宿主机，则bip为 172.7.22.1/24

```bash
mkdir /etc/docker/

cat >/etc/docker/daemon.json <<EOF
{
    "graph": "/data/docker",
    "storage-driver": "overlay2",
    "insecure-registries": ["harbor.bst.com"],
    "bip": "172.7.21.1/24",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "live-restore": true
}
EOF
```

## 部署harbor

安装docker-compose

`yum install epel-release -y --nogpgcheck`

离线安装harbor方式，下载地址可参考https://github.com/goharbor/harbor/releases

```bash
tar xf harbor-offline-installer-v2.2.1.tgz -C /opt/
cd /opt/
mv harbor/ harbor-v2.2.1
ln -s /opt/harbor-v2.2.1/ /opt/harbor
```

修改部分配置文件,/opt/harbor/harbor.yml，以下仅为修改项目

```yaml
hostname: harbor.zq.com
http:
  port: 180 
  harbor_admin_password:Harbor12345
data_volume: /data/harbor
log:
  level: info
rotate_count: 50
rotate_size: 200M
location: /data/harbor/logs
```

创建必要的存储地址

`mkdir -p /data/harbor/logs`

安装运行harbor

```bash
cd /opt/harbor/
yum install docker-compose -y
sh /opt/harbor/install.sh
docker-compose ps
docker ps -a
```

安装NGINX反代harbor服务

```bash
yum install nginx -y
vim /etc/nginx/conf.d/harbor.zq.com.conf
## 添加配置yu
server {
    listen 80;
    server_name harbor.zq.com;

    client_max_body_size 1000m;

location / {
    proxy_pass http://127.0.0.1:180;
    }
}
```

## 配置证书环境

yaml服务器，下载安装cfssl、cfssljson

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/bin/cfssl-json
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/bin/cfssl-certinfo
chmod +x /usr/bin/cfssl*

```

统一证书路径放在/opt/certs下，创建CA根证书、etcd服务端证书

```bash
cat >/opt/certs/ca-csr.json <<EOF
{
    "CN": "Besttone",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "C": "CN",
        "ST": "shanghai",
        "L": "shanghai",
        "O": "lianjie",
        "OU": "ops"
        }
    ],
    "ca": {
    	"expiry": "175200h"
    }
}
EOF
cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
cat >/opt/certs/ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "175200h"
        },
        "profiles": {
            "server": {
            "expiry": "175200h",
            "usages": [
            "signing",
            "key encipherment",
            "server auth"
            ]
        },
        "client": {
            "expiry": "175200h",
            "usages": [
            "signing",
            "key encipherment",
            "client auth"
            ]
        },
        "peer": {
            "expiry": "175200h",
            "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
            ]
        }
        }
    }
}
EOF
cat >/opt/certs/etcd-peer-csr.json <<EOF
{
    "CN": "k8s-etcd",
    "hosts": [
        "192.168.82.27",
        "192.168.82.28",
        "192.168.82.29",
        "192.168.82.30",
        "192.168.82.31"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
    {
        "C": "CN",
        "ST": "shanghai",
        "L": "shanghai",
        "O": "lianjie",
        "OU": "ops"
    }
    ]
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json |cfssl-json -bare etcd-peer


```

## 配置etcd集群

下载etcd执行文件解压到/opt/etcd下，并下载证书

```bash
useradd -s /sbin/nologin -M etcd
mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server
cd /opt/etcd/certs/
wget http://yaml.bst.com/certs/ca.pem
wget http://yaml.bst.com/certs/etcd-peer-key.pem
wget http://yaml.bst.com/certs/etcd-peer.pem
chmod 600 etcd-peer-key.pem
chown -R etcd.etcd /opt/etcd
chown -R etcd.etcd /opt/etcd-v3.2.32/
chown -R etcd.etcd /data/logs/etcd-server/
chown -R etcd.etcd /data/etcd
```

配置sh文件，每一台etcd节点配置文件稍有有不同，关注name、listen-peer-urls、initial-advertise-peer-urls

```bash
cat >/opt/etcd/etcd-server-startup.sh <<'EOF'
#!/bin/sh
./etcd \
--name etcd-server-82-29 \
--data-dir /data/etcd/etcd-server \
--listen-peer-urls https://192.168.82.29:2380 \
--listen-client-urls https://192.168.82.29:2379,http://127.0.0.1:2379 \
--quota-backend-bytes 8000000000 \
--initial-advertise-peer-urls https://192.168.82.29:2380 \
--advertise-client-urls https://192.168.82.29:2379,http://127.0.0.1:2379 \
--initial-cluster etcd-server-82-27=https://192.168.82.27:2380,etcd-server-82-28=https://192.168.82.28:2380,etcd-server-82-29=https://192.168.82.29:2380 \
--ca-file ./certs/ca.pem \
--cert-file ./certs/etcd-peer.pem \
--key-file ./certs/etcd-peer-key.pem \
--client-cert-auth \
--trusted-ca-file ./certs/ca.pem \
--peer-ca-file ./certs/ca.pem \
--peer-cert-file ./certs/etcd-peer.pem \
--peer-key-file ./certs/etcd-peer-key.pem \
--peer-client-cert-auth \
--peer-trusted-ca-file ./certs/ca.pem \
--log-output stdout
EOF
chmod +x /opt/etcd/etcd-server-startup.sh
```

配置supervisor管理文件，并启动

```bash
cat >/etc/supervisord.d/etcd-server.ini <<EOF
[program:etcd-server] ; 显示的程序名,类型my.cnf,可以有多个
command=sh /opt/etcd/etcd-server-startup.sh
numprocs=1 ; 启动进程数 (def 1)
directory=/opt/etcd ; 启动命令前切换的目录 (def no cwd)
autostart=true ; 是否自启 (default: true)
autorestart=true ; 是否自动重启 (default: true)
startsecs=30 ; 服务运行多久判断为成功(def. 1)
startretries=3 ; 启动重试次数 (default 3)
exitcodes=0,2 ; 退出状态码 (default 0,2)
stopsignal=QUIT ; 退出信号 (default TERM)
stopwaitsecs=10 ; 退出延迟时间 (default 10)
user=etcd ; 运行用户
redirect_stderr=true ; 是否重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log
stdout_logfile_maxbytes=64MB ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4 ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
## 启动服务，检查etcd集群是否正常
supervisorctl update
supervisorctl status
netstat -lntup|grep etcd
/opt/etcd/etcdctl cluster-health
/opt/etcd/etcdctl member list
```



