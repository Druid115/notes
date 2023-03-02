## CentOS7 下安装 JDK

### 1. 安装 JDK

下载地址为 http://www.oracle.com/technetwork/java/javase/downloads/index.html

~~~shell
mkdir /usr/local/java

# 进入jdk源码包所在目录,将下载到压缩包拷贝到java文件夹中
cp jdk-8u221-linux-x64.tar.gz /usr/local/java

cd /usr/local/java
tar -zxvf jdk-8u221-linux-x64.tar.gz
~~~



### 2. 配置环境变量

~~~shell
vi /etc/profile

# Java environment
export JAVA_HOME=/usr/local/java/jdk1.8.0_221
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

# 让环境变量生效
source /etc/profile

java -version
~~~

