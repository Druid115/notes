## Oracle 笔记（一）— CentOS7 下安装 Oracle 11g

使用 xftp 远程连接服务器进行图形化安装，而非静默安装，需要安装 xManager 或者其他图形化界面工具，否则图形化安装界面无法弹出。

### 1. 关闭服务

```shell
# 关闭防火墙
systemctl stop firewalld.service
# 禁止使用防火墙
systemctl disable firewalld.service

# 关闭SELinux
vi /etc/selinux/config
```



### 2. 修改 IP

~~~shell
vi /etc/hosts

# 可通过 hostname 查看
192.168.10.74 主机名
~~~



### 3. 安装相关依赖

~~~shell
# 依赖源一
yum -y install binutils-2.*
yum -y install compat-libcap1-*
yum -y install compat-libstdc++*
yum -y install compat-libstdc++-*
yum -y install gcc-4*
yum -y install gcc-c++-4*
yum -y install glibc-2*
yum -y install glibc-devel-2*
yum -y install ksh
yum -y install libgcc-4.*
yum -y install libstdc++-4.*
yum -y install libstdc++-devel-4*
yum -y install libaio-0.*
yum -y install libaio-devel-0.*
yum -y install make-3.*
yum -y install sysstat-9.*
yum -y install unixODBC-*

# 依赖源二
yum -y install binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel expat gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers libaio libaio-devel libgcc libstdc++ libstdc++-devel make pdksh sysstat unixODBC unixODBC-devel

# 检查依赖是否安装完整
rpm -q binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel expat gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers libaio libaio-devel libgcc libstdc++ libstdc++-devel make pdksh sysstat unixODBC unixODBC-devel | grep "not installed"

# 使用第二种源的时候，需要手动安装 pdksh
wget -O /tmp/pdksh-5.2.14-37.el5_8.1.x86_64.rpm http://vault.centos.org/5.11/os/x86_64/CentOS/pdksh-5.2.14-37.el5_8.1.x86_64.rpm
cd /tmp/
rpm -ivh pdksh-5.2.14-37.el5_8.1.x86_64.rpm

# 若出现 pdksh conflicts with ksh-20120801-1***
rpm -e ksh-20120801-1***
rpm -Uvh pdksh-5.2.14-36.el5.x86_64.rpm
~~~

这里提供了两种依赖源，最好都安装。后续可能会出现缺少 pdksh 这个依赖包的错误，原因是 OUI（Oracle Universal Installer）进行预检查的时候会使用命令：

~~~shell
/bin/rpm -q --qf %{version} redhat-release
~~~

来确定 Linux 的版本，但是 redhat-release 已经被 redhat-release-server 包所取代，所以安装软件会无法识别Linux 的版本。这时 OUI 会默认的使用 Linux4 的前置条件来检查现有的操作系统情况。除了安装 pdksh 这个依赖包外，还有一种解决办法：修改 <unzip path>/database/stage/cvu/cv/admin 目录下的 cvu_config文件，将其中的 CV_ASSUME_DISTID=OEL4 改为 CV_ASSUME_DISTID=OEL6 保存后重新 runInstaller。



### 4. 配置参数

~~~shell
vi /etc/sysctl.conf

fs.aio-max-nr = 1048576
fs.file-max = 6815744
### 48G memory
#kernel.shmall = 11324620
#kernel.shmmax = 46385646800
### 32G memory
#kernel.shmall = 7549748
#kernel.shmmax = 30923764532
### 16G memory
kernel.shmall = 3774873
kernel.shmmax = 15461882265
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586

# 使内核参数生效
sysctl -p
~~~

fs.aio-max-nr：同时可以拥有的的异步 IO 请求数目，1048576 等于 1024*1024 也就是 1024K 个。

fs.file-max：系统中可以同时打开的文件数目。

kernel.shmall：控制共享内存总大小，以页为单位。一个32位的 Linux 系统，8G 的内存，可以设置 kernel.shmall = 2097152，即为：2097152*4k/1024/1024 = 8G。就是说可用共享内存一共 8G，这里的 4K 是 32 位操作系统一页的大小，即 4096 字节。

kernel.shmmax：定义单个共享内存段的最大值。

kernel.shmmni：共享内存段的最大数量。

kernel.sem：以 kernel.sem = 250 32000 100 128为例：

​						250 是参数 semmsl 的值，表示一个信号量集合中能够包含的信号量最大数目。

​       				 32000 是参数 semmns 的值，表示系统内可允许的信号量最大数目。

​      			      100 是参数 semopm 的值，表示单个 semopm() 调用在一个信号量集合上可以执行的操作数量。

​       			     128 是参数 semmni 的值，表示系统信号量集合总数。

net.ipv4.ip_local_port_range：表示应用程序可使用的 IPv4 端口范围。

net.core.rmem_default：表示套接字接收缓冲区大小的缺省值。

net.core.rmem_max：表示套接字接收缓冲区大小的最大值。

net.core.wmem_default：表示套接字发送缓冲区大小的缺省值。

net.core.wmem_max：表示套接字发送缓冲区大小的最大值。

一般来说，出于性能上的考虑，还需要配置用户的资源限制，以便改进 Oracle 用户的有关 nofile（可打开的文件描述符的最大数）和 nproc （单个用户可用的最大进程数量）。

~~~shell
vi /etc/security/limits.conf

# soft 是软限制，hard 是硬限制。用户可以超过 soft 设置的值，但一定不能超过 hard 的值 。
oracle              soft    nproc   2047
oracle              hard    nproc   16384
oracle              soft    nofile  1024
oracle              hard    nofile  65536
~~~

实现 /etc/security/limits.conf 中定义的各项限制。

~~~shell
vi /etc/pam.d/login

session required /lib64/security/pam_limits.so
session required pam_limits.so

vi /etc/profile

if [ $USER = "oracle" ]; then
   if [ $SHELL = "/bin/ksh" ]; then
       ulimit -p 16384
       ulimit -n 65536
    else
       ulimit -u 16384 -n 65536
   fi
fi

# 使/etc/profile文件生效
source /etc/profile
~~~

禁用 Transparent HugePages （启用 Transparent HugePages，可能会导致内存在运行时的延迟分配，Oracle 官方建议使用标准的 HugePages）

~~~shell
# 查看是否启用 如果显示 [always] 说明启用了
cat /sys/kernel/mm/transparent_hugepage/enabled

echo never > /sys/kernel/mm/transparent_hugepage/enabled
# 重新启动系统以使更改成为永久更改
~~~



### 5. 创建操作系统用户、组和安装目录

~~~shell
# Oracle inventory 组
groupadd oinstall
# OSDBA 组
groupadd dba
# OSOPER 组
groupadd oper
# Oracle 软件所有者
useradd -g oinstall -G dba oracle
# 修改 Oracle 用户密码
passwd oracle

mkdir -p /db/app/oracle/product/11.2.0
chown -R oracle:oinstall /db/app/oracle/product/11.2.0
~~~



### 6. 配置 oracle 用户的环境变量

~~~shell
vi /home/oracle/.bash_profile

umask 022
# export ORACLE_HOSTNAME=主机名
export ORACLE_BASE=/db/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0
export ORACLE_SID=ORCL
export PATH=.:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$ORACLE_HOME/jdk/bin:$PATH
export LC_ALL="en_US"
export LANG="en_US"
export NLS_LANG="AMERICAN_AMERICA.ZHS16GBK"
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"
~~~



### 7. 上传并解压 oracle 压缩包

~~~shell
# 通过 xftp 将压缩包上传至 /db 目录下
unzip -q linux.x64_11gR2_database_1of2.zip -d /db
unzip -q linux.x64_11gR2_database_2of2.zip -d /db
chown -R oracle:oinstall /db/
chmod -R 775 /db/
~~~



### 8. 安装 oracle 软件

启动 Xmanger - passive，选择第二个（选择第一个会出现弹出层的弹出层无法显示的问题）。

![](E:\学习资料\笔记\images\20190820183022.png)

~~~shell
su - oracle
cd /db/database
LANG=C

# DISPLAY 变量是用来设置将图形显示到何处，该 IP 为本地主机的 IP
export DISPLAY=192.168.10.51:1.0
xhost +
# access control disabled, clients can connect from any host 则为成功

./runInstaller
~~~

安装过程中的几个问题：

1. 选择”仅安装数据库软件”，默认安装单实例，默认企业版安装，设置目录为”/db/app/oracle/oraInventory”。
2. 若提示无权限创建文件夹，则赋予文件夹777最高权限。
3. 安装过程中跳出的窗口，需要手动以 root 用户执行脚本。

~~~shell
sh /db/app/oracle/inventory/orainstRoot.sh
sh /db/app/oracle/product/11.2.0/root.sh
~~~

4. 安装前检查环境要求时，出现警告，可以直接忽略。



### 9. 配置 oracle 监听

~~~shell
# 这三步如果已经设置，可以省略
su - oracle
LANG=C
export DISPLAY=192.168.10.51:1.0
xhost +

# 查看监听的状态
lsnrctl status
# 停止和启动监听
lsnrctl stop
lsnrctl start
~~~

弹出窗口后，基本默认选择就行。



### 10. 创建数据库

~~~shell
 dbca
~~~

需要注意的几点：

1. 此处的 SID 必须和 oracle 用户的环境变量 ORACLE_SID 设置相同。

![](E:\学习资料\笔记\images\20190820195202.png)

2. 不配置 OEM（Oracle Enterprise Manager 企业管理器，是通过一组 Oracle 程序，为管理分布式环境提供管理服务），耗费系统资源。
3. 存储类型选择：文件系统，选择第三个使用 OMF。
4. Size 选项：Linux 进程数改成200。
5. 字符集选项：设置当地字符集，要和环境变量 NLS_LANG 一致。
6. 设置数据库允许的最大文件数为2048。