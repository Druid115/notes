## CentOS7 下安装 SFTP

### 1. 环境检查

~~~shell
# 查看 openssl 的版本,版本必须大于 4.8p1
ssh -V
~~~



### 2. 创建用户和组

~~~shell
# 创建 sftp 组
groupadd sftp

# 创建用户 -s 禁止用户 ssh 登陆 -G 加入用户组
useradd -G sftp -s /sbin/nologin 【username】

# 创建密码
passwd 【username】
~~~



### 3. 修改 sshd 配置文件

~~~shell
vi /etc/ssh/sshd_config

# 下面这行注释掉
#Subsystem sftp /usr/libexec/openssh/sftp-server
# 最后加入
Subsystem sftp internal-sftp
# 以下需要添加在文件的最下方，否则 root 用户无法登录
Match Group sftp
X11Forwarding no
AllowTcpForwarding no
# 只能访问默认的用户目录（自己的目录），例如 /home/test；
ChrootDirectory %h
ForceCommand internal-sftp
~~~



### 4. 配置 SFTP 账号访问路径

~~~shell
chown root:sftp /home/【username】
chgrp -R sftp /home/【username】
chmod -R 755 /home/【username】
# 设置用户可以上传的目录,改目录下允许用户上传删除修改文件及文件夹
mkdir /home/【username】/upload
chown -R 【username】:sftp /home/【username】/upload
chmod -R 755 /home/【username】/upload
~~~



### 5. 重启 sshd 服务

~~~shell
systemctl restart sshd
~~~