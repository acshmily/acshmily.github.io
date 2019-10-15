---
title: centos-ulimit-modify
date: 2019-10-15 16:08:11
tags:
	- linux
	- centos
categories: 工作笔记
---

# 调整centos7的ulimit相关配置

​	针对大并发以及分布式系统，系统默认的句柄数、最大线程数等需要进行修改，由于centos7下的sshd大版本为**8**的调整策略和以往不同，需要修改三处。

```shell
vi /etc/security/limits.conf
* soft nofile 40960
* hard nofile 81920
* soft nproc 40960
* hard nproc 81920

```

```shell
vi /etc/pam.d/sshd
#%PAM-1.0
auth       include      system-auth
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    required     pam_loginuid.so
session    required     pam_limits.so
```

```shell
/etc/ssh/sshd_config  UsePAM  yes
```



退出登录后重新登录，进行验证

```shell
ulimit -n
```

