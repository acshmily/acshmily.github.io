---
title: docker-jenkins-sonarqube
date: 2019-10-10 09:15:24
tags:
	- DevOPS
	- jenkins
	- sonarqube
categories: 工作笔记
---

# jenkins联动sonarQube

## 版本信息

- Centos 7 （安装epel-release）
- Docker CE:Lastest TLS
- Aliyun Docker Repository
- Python Pip
- Docker-Compose or K8S

## Jenkins in Docker

```shell
docker run -u root -d -p 8080:8080 -p 50000:50000 -v jenkins-data:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock -v /usr/java/maven:/usr/local/maven -e JAVA_OPTS=-Duser.timezone=Asia/Shanghai --name jenkins jenkins/jenkins:lts
```

​	对外暴露 8080为Web 访问端口 50000为API调用端口

​	挂载宿主机容器卷 **jenkis-data** 对应容的的**/var/jenkins_home** 路径

​	挂载 **/var/run/docker.sock** 对应容器的 **/var/run/docker.sock** 

​	挂载**/usr/java/maven** 对应容器的 **/usr/local/maven** 

​	设置容器内JAVA额外配置参数时区为**上海** 

​	命名该容器为 jenkins

​	该镜像更多用法可以访问hub.docker.com 访问 jenkins:lts 进行查看

## SonarQube in Docker

​	由于SonarQube的数据库推荐为**Postgres**，按照官方建议编写docker-compose.yml

```yml
version: "2"

services:
  sonarqube:
    image: sonarqube
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://db:5432/sonar
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
      - sonarqube_temp:/opt/sonarqube/temp

  db:
    image: postgres
    networks:
      - sonarnet
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data

networks:
  sonarnet:
    driver: bridge

volumes:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_bundled-plugins:
  postgresql:
  postgresql_data:
```

**Note**

	- 你也可以将SonarQube的持久化数据库更换为MYSQL，但是注意最新版本的SonarQube不再支持MYSQL
	- 更多的配置可以访问 hub.docker.com 搜索 sonarqube 进行查看

```shell
# docker-compose.yml 当前路径下
docker-compose up -d # 后台运行
docker ps # 查看进程
```

## Jenkins 插件安装

**NOTE** 由于国内访问jenkins官方源速度不够理想，建议更换为国内镜像，例如 清华镜像源

安装以下插件：

	- [Localization: Chinese (Simplified)](https://github.com/jenkinsci/localization-zh-cn-plugin)
	- [SonarQube Scanner for Jenkins](http://redirect.sonarsource.com/plugins/jenkins.html)
	- [Maven Integration plugin](https://wiki.jenkins.io/display/JENKINS/Maven+Project+Plugin)
	- [Email Ext Recipients Column Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Email+Ext+Recipients+Column+Plugin)
	- [Git plugin](https://github.com/jenkinsci/git-plugin)
	- [Git client plugin](https://github.com/jenkinsci/git-client-plugin)
	- [GIT server Plugin](https://wiki.jenkins.io/display/JENKINS/Git+Server+Plugin)

##SonarQube 插件安装

- 安装本地化语言插件

## SonarQube 配置

​	由于SCM轮询交由Jenkins处理，故关闭SCM轮询。

![image-20191010104017484](/Users/huanghongzhi/Library/Application Support/typora-user-images/image-20191010104017484.png)

## Jenkins 配置

### 配置 **SonarQube servers**

- Environment variables 设置为Enable
- SonarQube installations 
  - name：Jenkins 内现实该servers 的名称
  - Server URL：SonarQube 的地址
  - Server authentication token：默认不填

### 配置**Extended E-mail Notification**

- SMTP server 
- Default user E-mail suffix
- Default Content Type ： 选择为HTML
- Default Subject：标题
- Default Content：内容

**Default Subject**

```
【系统自动发送，请勿回复】$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!
```

**Default Content**

```html
<!DOCTYPE html>  
<html>  
<head>  
  <meta charset="UTF-8">  
  <title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>  
</head>  

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"  
offset="0">  
<table width="95%" cellpadding="0" cellspacing="0"  
style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">  
<tr>  
  <td><br />  
    <b><font color="#0B610B">构建信息</font></b>  
    <hr size="2" width="100%" align="center" />
  </td>  
</tr>  
<tr>  
  <td>  
    <ul>  
      <li>项目名称 ：${PROJECT_NAME}</li>  
      <li>构建状态 ：${BUILD_STATUS}</li>
      <li>构建编号 ：第${BUILD_NUMBER}次构建</li>  
      <li>触发原因 ：${CAUSE}</li>  
      <li>构建日志 ：<a href="${BUILD_URL}console">${BUILD_URL}console</a></li>  
      <li>构建URL ：<a href="${BUILD_URL}">${BUILD_URL}</a></li>  
      <li>工作目录 ：<a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>  
      <li>项目URL ：<a href="${PROJECT_URL}">${PROJECT_URL}</a></li>  
    </ul>  
  </td>  
</tr>  
<tr>  
  <td><b><font color="#0B610B">变更集</font></b>  
    <hr size="2" width="100%" align="center" />
  </td>  
</tr>  

<tr>  
  <td>${JELLY_SCRIPT,template="html"}<br/>  
    <hr size="2" width="100%" align="center" />
  </td>  
</tr>  

</table>  
</body>  
</html>  
```

## Jenkins 项目配置SonarQube

选择自动化风格项目进行项目构建

- 丢弃旧的构建
  - 策略：默认
  - 保持构建的天数 ：5
  - 保持构建的最大个数：2
- 源码管理：SVN或Git
- 构建触发器：SCM（策略H/15 * * * *）或Git Hook
- 构建环境
  - [x] Add timestamps to the Console Output 
  
  - [x] Prepare SonarQube Scanner environment
- 构建
  - **execute SonarQube Scanner**
    - Task to run：为空
    - JDK：默认
    - Analysis properties ：见下文
- 构建后操作
  - **Editable Email Notification**
    - [ ] Disable Extended Email Publisher 
    - Project From:为空
    - Project Recipient List：$DEFAULT_RECIPIENTS
    - Project Reply-To List：$DEFAULT_REPLYTO
    - Content Type：默认
    - Default Subject：$DEFAULT_SUBJECT
    - Default Content：$DEFAULT_CONTENT
    - Pre-send Script：$DEFAULT_PRESEND_SCRIPT
    - Post-send Script：$DEFAULT_POSTSEND_SCRIPT
    - Triggers
      - 配置 失败、成功的触发场景
      - Send To 新增 **Recipient List**
      - 添加收件人，多个用,分隔
      - 其他默认

成功的邮件内容特别配置

```html
<!DOCTYPE html>    
<html>    
<head>    
<meta charset="UTF-8">    
<title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>    
</head>    
    
<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"    
    offset="0">    
    <table width="95%" cellpadding="0" cellspacing="0"  style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">    
        <tr>    
<br />
            本邮件由系统自动发出，无需回复！<br/>            
            各位同事，大家好，以下为${PROJECT_NAME }项目构建信息</br> 
            <td><font color="#CC0000">构建结果 - ${BUILD_STATUS}</font></td>   
        </tr>    
        <tr>    
            <td><br />    
            <b><font color="#0B610B">构建信息</font></b>    
            <hr size="2" width="100%" align="center" /></td>    
        </tr>    
        <tr>    
            <td>    
                <ul>    
                    <li>项目名称 ： ${PROJECT_NAME}</li>    
                    <li>构建编号 ： 第${BUILD_NUMBER}次构建</li>    
                    <li>触发原因： ${CAUSE}</li>    
                    <li>构建状态： ${BUILD_STATUS}</li>    
                    <li>构建日志： <a href="${BUILD_URL}console">${BUILD_URL}console</a></li>    
                    <li>构建  URL ： <a href="${BUILD_URL}">${BUILD_URL}</a></li>    
                    <li>工作目录 ： <a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>    
                    <li>项目  URL ： <a href="${PROJECT_URL}">${PROJECT_URL}</a></li> 
                    <li>代码质量扫描结果  URL： <a href="${BUILD_LOG_REGEX, regex=".*ANALYSIS SUCCESSFUL, you can browse (.*)", showTruncatedLines=false, substText="$1"}   ">${BUILD_LOG_REGEX, regex=".*ANALYSIS SUCCESSFUL, you can browse (.*)", showTruncatedLines=false, substText="$1"}   </a></li>  
                </ul>    
 
<h4><font color="#0B610B">失败用例</font></h4>
<hr size="2" width="100%" />
$FAILED_TESTS<br/>
 
<h4><font color="#0B610B">最近提交(#$SVN_REVISION)</font></h4>
<hr size="2" width="100%" />
<ul>
${CHANGES_SINCE_LAST_SUCCESS, reverse=true, format="%c", changesFormat="<li>%d [%a] %m</li>"}
</ul>
详细提交: <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a><br/>
 
            </td>    
        </tr>    
    </table>    
</body>    
</html>
```

