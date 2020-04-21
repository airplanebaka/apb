# 第一章 相关环境设置

## 安装Java开发环境

### 下载JAVA

进入官网下载地址

* [https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* 下载最新版本的jdk-xxxx-linux-x64.tar.gz安装包

### 安装Java

将下载的安装包拷贝到想要安装的目录

```text
// 解压安装包
tar -xzvf jdk-8u241-linux-x64.tar.gz

// 修改环境变量
vim ~/.bashrc

// 在环境变量文件最末添加
export JAVA_HOME=/home/soft/jdk1.8.0_241
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH:$HOME/bin

// 保存退出后生效设置
source ~/.bashrc

// 查看是否安装成功
java -version
```



## 安装SBT工具

进入官网下载

* [https://www.scala-sbt.org/download.html](https://www.scala-sbt.org/download.html)
* 下载最新sbt-x.x.x.tgz安装包

将下载的安装包移动到安装java包的目录

```text
// 解压安装包
tar zxvf sbt-x.x.x.tgz

// 在安装目录创建sbt文件
vim sbt

// 在sbt中添加如下代码
#!/bin/bash
BT_OPTS="-Xms2048M -Xmx4096M -Xss1M -XX:+CMSClassUnloadingEnabled -XX:MaxPermSize=256M"
java $SBT_OPTS -jar /home/soft/sbt/bin/sbt-launch.jar "$@"
*注意更改/home/soft/为你的安装目录*
 
// 修改sbt文件权限
chmod u+x sbt

// 配置sbt环境变量
vim /etc/profile
*在末尾添加*
export SBT_HOME=/home/soft/sbt
export PATH=${SBT_HOME}/bin:$PATH
*注意更改/home/soft/为你的安装目录*

// 生效配置
source /etc/profile

// 检测是否安装成功
sbt -version

```



## 安装Chisel开发软件















