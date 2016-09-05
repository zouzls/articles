---
title: Docker安装与启动
date: 2016-07-27 13:43:33
tags: [docker,windows10,安装,启动]
categories: Docker
---
### 背景
第一次听到docker这个词是15年7月份，老板让项目组老大看看docker相关技术，据说是可以提高公司项目部署的效率，不过遗憾的是的当时已经准备快离职了，所以没能早一点一睹docker容器的庐山真面目。现在看来docker的热度也不是一年之前能比的了，不过也激发了我对技术的好奇心，决定挖这样一个坑，来激励自己去研究一下docker的内部原理和实际应用。本篇文章记录安装过程中出现的一些问题和解决过程，重点在于启动过程中的几个错误。
注：不同的系统和硬件环境下安装文件和过程不太一样，所以:
1. 采用Docker Toolbox方式安装的请绕行。
2. 以下适用于Docker for Windows Beta的安装，详情请看下面「安装-环境要求」。

<!--more-->
### 安装
#### 环境要求（非常重要！）
- 系统环境
64bit Windows 10 Pro, Enterprise and Education (1511 November update, Build 10586 or later)。在cortana输入框键入winver，然后回车enter即可查看系统版本。
- 内存大小
最好是>4G内存,后面为docker分配的内存至少是1G，太小的内存可能导致启动不成功。
- Hyper-V开启
如果不支持Hyper-V虚拟化，就只能用 Docker Toolbox方式安装了。查看是否支持Hyper-V方式如下，打开「任务管理器-性能」，默认是禁用的（如何开启见下方「启动-关于虚拟化未开启错误」），如下：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/3bf33a87e950352a23e067375043fbf2b2118b08.jpg)

#### 下载&安装
进入[页面](https://docs.docker.com/docker-for-windows/)，点击Get Docker for Windows，下载完InstallDocker.msi文件直接安装即可，直到finish（launch Docker默认打勾），安装过程很快。
### 启动
#### 关于「虚拟化未开启」错误
安装完之后docker会尝试第一次启动，如果电脑Hyper-V之前未开启，docker会提示开启Hyper-V并且电脑随即进入重启，我在重启的时候忘记进入BIOS设置了，结果重启完之后，（因为docker安装完是设置开机自启动的）docker启动的时候就出现下面错误了。
**解决方式**：再次重启，并且过程中进入BIOS，不同的电脑进入BIOS方式不同，Lenovo是按住F2，照下图操作即可：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/192152313325897.jpg)

#### 关于「Unable ...Exit 1」错误
当你觉得修改完BIOS，觉得已经万事大吉的时候，再重新Start docker的时候，可能又会如下错误，我在docker（1.12.0） build 5579的版本中遇到，错误码如下：
```
Unable to write to the database. Exit 1
```
**解决方式**：打开网络与共享中心，可以看到多了一个关于docker虚拟机的vEthernet（DockerNAT）连接，点击进入，配置虚拟机的DNS网关并且禁用IPV6，如图：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20160727152117.png)
配置完之后，再启动错误码就会消失了。

#### 关于「Not enough memory」错误
对于内存较低的可能出现启动时内存不够而导致失败的情况，因为docker默认是配置2G内存。
**解决方式**：右键docker图标-setting-Advanced，如图：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20160727153322.png)可以尝试不断减少Memory，直到启动OK，Docker正确启动后，左下角显示running。
### Demo演示
下面尝试run两个container，第一个是hello-world，第二个是webserver。在执行docker run containerName命令之后，Docker会先在本地检查对应是否存在该镜像，如果不存在再到Docker Hub上检查，如果有则立即下载到宿主机。
#### hello-world
在cmd或者PowerShell中输入：docker run hello-world，却不料出现Network timed out while trying to connect to...如下：
```
bash.exe"-3.1$ docker run hello-world
Unable to find image 'hello-world:latest' locally
Pulling repository docker.io/library/hello-world
c:\Program Files\Docker\Docker\Resources\bin\docker.exe: Network timed out while trying to connect to https://index.docker.io/v1/repositories/library/hello-world/images
. You may want to check your internet connection or if you are behind a proxy..
```
**解决方式**：通过Google从docker-machine restart default这条命令得到提示，如果直接执行该命令报错不成功，手动exit docker之后，重新start，再docker run hello-world，问题解决！
```
bash.exe"-3.1$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world

c04b14da8d14: Pull complete
Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
```
#### webserver nginx
运行一个docker化的webserver，输入：docker run -d -p 80:80 --name webserver nginx，如下：
```
bash.exe"-3.1$ docker run -d -p 80:80 --name webserver nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx

51f5c6a04d83: Pull complete
a3ed95caeb02: Pull complete
51d229e136d0: Pull complete
bcd41daec8cc: Pull complete
Digest: sha256:0fe6413f3e30fcc5920bc8fa769280975b10b1c26721de956e1428b9e2f29d04
Status: Downloaded newer image for nginx:latest
592533495a5f766ce87c4460fe277460db9c5eed2469c87625a0d1ba2633dc43
```
继续执行：docker ps，可以看到各个容器的详情，不过目前只有一个nginx，如下：
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                         NAMES
592533495a5f        nginx               "nginx -g 'daemon off"   2 minutes ago       Up 2 minutes        0.0.0.0:80->80/tcp, 443/tcp   webserver
```
可以看到nginx的IP和端口是：0.0.0.0:80，浏览器访问localhost（或者docker/），就会看到如下页面：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20160727201002.png)

### Hyper-V管理器
windows自带有Hyper-V管理器，进入「开始菜单-windows管理工具」，可以看到新建的 Hyper-V管理器，打开之后可以看到WIN-4FQM25FK460虚拟机，以及虚拟机中正在运行的MobyLinuxVM实例，如下：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20160727190746.png)

### 参考资料
[Windows安装官方文档](https://docs.docker.com/docker-for-windows/)
[关于「Unable to write to the database. Exit 1」错误](https://forums.docker.com/t/docker-1-12-for-windows-10-is-not-working/16647/2)


-EOF-






