---
title: Tomcat源码分析之环境搭建
date: 2016-07-04 19:33:47
tags: [Tomcat,源码分析,环境搭建]
categories: Tomcat
---
### 准备工作
1、Tomcat7源码（官网下载即可）
2、jdk6环境（Tomcat7要求的编译环境是jdk6，所以如果之前安装jdk7的不用卸载，直接在安装一个jdk6就好了，修改一下JAVA_HOME即可。安装完之后记得检查java -version，如果还是java7版本，把c:\windows\system32目录下的javaw.exe、java.exe和javaws.exe三个文件删除就可以了。）
3、Ant 1.8.2 或者更高版本的构建工具。（直接官网下载解压，然后添加ANT_HOME,再在path中添加%ANT_HOME%\bin;，同样记得检查ant -version。）
4、eclipse工具
<!--more-->
### Ant 构建
cmd进入Tomcat source根目录。将build.properties.default文件重命名为build.properties，打开该文件找到base.path一项，可以修改value值，该路径下会下载tomcat依赖的包，可以自定义。然后在根目录，执行以下命令：
```
ant -p 
```
（如果出现找不到ant的命令，可能是权限不够，用管理员身份打开cmd。）
会出现一系列可能的选项，中间有一项ide-eclipse，说明可以将项目变成eclipse的项目。此时源码目录还没有.settings文件夹、.classpath和.project两个文件。接下来执行：
```
ant ide-eclipse
```
中间可能会遇到超时连接的问题，如下：
```
F:\tomcat-7.0-src\build.xml:2848: java.net.ConnectException: Connection timed out: connect
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.PlainSocketImpl.doConnect(PlainSocketImpl.java:351)
        at java.net.PlainSocketImpl.connectToAddress(PlainSocketImpl.java:213)
        at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:200)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:366)
        at java.net.Socket.connect(Socket.java:529)
        at com.sun.net.ssl.internal.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:570)
        ...
```
仔细向上查找问题，大概会发现有这么一行：
```
testexist:
     [echo] Testing  for C:\Users\恭帅/tomcat-build-libs/objenesis-1.2/objenesis-1.2.jar
downloadzip:
      [get] Getting: https://objenesis.googlecode.com/files/objenesis-1.2-bin.zip
      [get] To: C:\Users\${个人电脑的用户名}\tomcat-build-libs\download-471197819.zip
      [get] Error getting https://objenesis.googlecode.com/files/objenesis-1.2-bin.zip to C:\Users\${个人电脑的用户名}\tomcat-build-libs\download-471197819.zip
```
到这就知道为什么会出现超时连接了，解决办法：直接到网上下载objenesis-1.2.jar，不一定去googlecode.com，然后到“C:\Users\${个人电脑的用户名}\tomcat-build-libs\”文件夹下，新建objenesis-1.2文件夹，将objenesis-1.2.jar放在该文件夹下，重新执行：
```
ant ide-eclipse
```
出现BUILD SUCCESSFUL就好了，然后查看源码根目录会发现.settings文件夹、.classpath和.project两个文件，接下来是导入eclipse工具中了。
### 导入eclipse
1、打开eclipse可能会报错，因为我们删除过c:\windows\system32目录下面的三个文件，所以解决办法是，将path中的JAVA_HOME移到path最前面，就可以解决这个问题。
2、按照一般流程import>existing projects into workplace，就行了。
导入之后，会发现项目左上角出现红色的感叹号的问题，访问[这篇文章](http://my.oschina.net/u/2457218/blog/657410)，文末有详细的解决方案，在此不赘述了。
### 运行Tomcat
右键项目，Debug-Java Application-Bootstrap-start tomcat就行了。
----------------------更新---------------------------
第二次我在自己电脑上重新搭建了一下发现按照上述流程，导入eclipse之后一直启动不了，总是报下列错误：
```
...
...,Server instance is not configured.
```
我仔细看了output/build下面的文件夹，发现bin、conf、lib、log、webapps这些文件夹全部为空，Google了一下原因，项目通过
```
ant ide-eclipse
```
这个命令只是构建成了eclipse项目，但是没有编译，所以启动不成功，所以解决问题的办法就是在上述命令执行完之后，继续执行下条命令：
```
ant
```
-EOF-
