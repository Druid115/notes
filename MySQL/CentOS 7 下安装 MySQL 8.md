## CentOS7 下安装 MySQL 8

### 1. 添加 yum 源

MySQL 官网：https://dev.mysql.com/downloads/repo/yum/

下载之后通过 xftp 上传文件到服务器，然后执行 yum localinstall 命令：

~~~shell
yum localinstall mysql80-community-release-el7-3.noarch.rpm

# 执行完毕过后可以进入到 /etc/yum.repos.d 目录中查看文件，发现多了两个文件 mysql-community.repo mysql-community-source.repo

# 检查 MySQL 源是否安装成功
yum repolist enabled | grep "mysql.*-community.*"
~~~



### 2. 安装

~~~shell
yum install mysql-community-server

# 设置大小写不敏感
vi /etc/my.cnf
# 最后一行加上
lower_case_table_names=1
~~~



### 3. 启动

~~~shell
systemctl start mysqld.service
systemctl status mysqld.service
systemctl stop mysqld.service
# 开机自启动
systemctl enable mysqld
systemctl daemon-reload
~~~



### 4. 修改密码

~~~shell
# MySQL 默认创建了 root 用户的密码，这个密码打印在 MySQL 的日志文件 /var/log/mysqld.log 中，可以通过 temporary password 关键字来找出这个临时的密码
grep 'temporary password' /var/log/mysqld.log

mysql -u root -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';  
# 如果密码强度不够，会报 ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
set global validate_password_policy=0;
# 会报 ERROR 1193 (HY000): Unknown system variable 'validate_password_policy'，原因是 MySQL8.0 将其改成 validate_password.policy
set global validate_password.policy=0;

systemctl restart mysqld.service
~~~

%MYSQL_HOME%\bin

### 5. 开放远程连接

~~~shell
# MySQL 默认只对本机开放连接，需要对 mysql 表的 host 字段进行修改以支持其他主机连接，% 表示所有。
# 先连接数据库
use mysql;
update user set host = '%' where user = 'root';
# 更改完成之后刷新权限
flush privileges;

# 端口开放
firewall-cmd --zone=public --permanent --add-port=3306/tcp
firewall-cmd --reload

# 若使用这种
grant all privileges on *.* to 'root'@'%' identified by 'password' with grant option;
# 会报 ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'identified by `password`' at line 1 原因是此版的 MySQL 将创建账户和赋予权限分开了

# 正确写法（with grant option 这个选项表示该用户可以将自己拥有的权限授权给别人）
grant all privileges on *.* to 'root'@'%' with grant option;
~~~