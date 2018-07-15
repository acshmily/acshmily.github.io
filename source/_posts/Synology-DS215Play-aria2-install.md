---
title: 关于群晖DS215安装aria2
date: 2018-05-12 20:44:04
tags:
	-	群晖
	-	Synology
	-	ds215
	-	aria2
categories:	安装手册
---
群晖其实默认自带了下载工具,如果没有特殊需求的用户其实无需额外再处理安装Aria2，首先我们需要了解**[Aria2][1]**到底是什么东西,我们可以在Github上找到这个项目.项目介绍是这样写的:

> aria2 is a utility for downloading files. The supported protocols are HTTP(S), FTP, SFTP, BitTorrent, and Metalink. aria2 can download a file from multiple sources/protocols and tries to utilize your maximum download bandwidth. It supports downloading a file from HTTP(S)/FTP/SFTP and BitTorrent at the same time, while the data downloaded from HTTP(S)/FTP/SFTP is uploaded to the BitTorrent swarm. Using Metalink's chunk checksums, aria2 automatically validates chunks of data while downloading a file like BitTorrent.

翻译成中文大概是这样的意思:

> aria2是一个下载文件的工具,它支持的协议可以是Http(s),Ftp,SFTP,BT以及Metalink.aria2会尝试基于多并发协议以达到充分你的下载带宽的目的.若需要下载的文件已经被上传至BT流则aria2可以同时基于HTTP(S)/FTP/SFTP以及BT协议下载,基于Metalink的校验机制,aria2会实现基于BT下载一样的自动块校验.

—— 介绍完毕,如果你觉得这个工具是你所需要的,可以继续往下看.同时你需要了解关于群晖系统的一些背景知识

> * 群晖NAS自带的DSM是基于Unix高度定制化的系统
> * 对于比较新款的型号可以直接在套件中心安装Docker,可以使用Docker直接安装aria2的Docker镜像
> * aira2可以自行进行使用乌班图交叉编译<比较折腾>,同时需要匹配对应的Bootstrap
> * 本文目的是让你快速使用上aria2工具,故不会从技术实现代码上说明

—— 我推荐使用群晖第三方安装库源的安装方式:



1. 打开群晖的套件中心添加第三方源,我推荐http://www.cphub.net/
2. 而后可以在套件中心搜索**[IPKGui][2]**,如无法搜到结果可以直接访问cphub的网页版进入download板块找到该组建下载后直接在群晖上手动安装,IPKGui目前在cphub的资源地址为https://www.cphub.net/?p=ipkgui
3. 安装完毕后建议先重启群晖的NAS系统,后在程序目录找到IPKGui这个程序打开搜索aria2即可直接完成安装
4. 若第三步安装失败,则需要在群晖设置里打开SSH权限,需要telnet进行cli方式安装文章下方会提供参考 
5. 群晖需要建立一个共享文件夹,例如aria2download,同时记录好这个文件夹在群晖的path路径
6. 确认步骤设置完毕则可以直接在telnet上执行启动aria2常驻
7. 验证,结束.




### 安装命令参考

> 仅供参考,假定你已经安装了ssh或者telnet工具,群晖亦开启了ssh/telnet服务.并且上面的安装步骤 1 2 3 安装无异常,下方代码段凡涉及到**#**开头的行为说明,无需在终端中输入

#获取root权限
sudo -i 
#切换目录到 opt/bin 下
cd /opt/bin
#更新ipkg 软件版本信息
ipkg update
#安装aira2
ipkg install aria2
#启动,注意 --dir=后面为你创建的共享文件夹的路径,请对应使用正确的路径
aria2c --enable-rpc --rpc-listen-all=true --rpc-allow-origin-all --rpc-secret=bilibiliganbei --max-connection-per-server=16 --split=16 --min-split-size=1M --dir=/volume1/aria2download -D 

------

## 浏览器上调用aria2推荐
可以使用[Ci-Aria2BDY][3]里面有具体说明,github项目地址为:https://github.com/CiChui/Ci-Aria2BDY/


[1]: https://github.com/aria2/aria2
[2]: https://www.dd-wrt.com/wiki/index.php/Ipkg
[3]: https://github.com/CiChui/Ci-Aria2BDY/
