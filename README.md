# cdh5x86_64
  学习 CDH5.x 版 hadoop 

# 参考资料
  [官方文档](https://www.cloudera.com/documentation/enterprise/5-10-x/topics/cdh_ig_cdh5_cluster_deploy.html)

# 为什么要安装CDH版的Hadoop?
  第三方的发布版一般对hadoop做了集成\兼容\修复一些BUG,对安装过程做了简化处理,对众多的组件做了兼容测试,并提供web版的管理系统
  
# 安装前的准备
  查看要安装的版本支持的操作系统,数据库,jdk

# 安装过程

> 1.（所有节点）配置虚拟机 hostname \ ip \ hosts
```sh
# 编辑 /etc/hostname
master

# 编辑 /etc/hosts
192.168.207.130 master
192.168.207.131 slave01
192.168.207.132 slave02

# 编辑 /etc/sysconfig/network-scripts/ifcfg-ens33
BOOTPROTO="static"
IPADDR="192.168.207.130"
GATEWAY="192.168.207.2"
```

> 2.（所有节点）关闭selinux
```sh
# 临时关闭
setenforce 0

# 查看状态
getenforce

# /etc/selinux/config 禁用selinux,重启后有效
SELINUX=disabled
```

> 3.（所有节点）关闭防火墙
```sh
# 停止防火墙
service firewalld stop

# 关闭自启动
chkconfig firewalld off

```

> 4.（所有节点）配置时间同步服务 NTP 
```sh
# （所有节点）部署NTP  Network Time Protocol 
yum install ntp

# （所有节点）配置/etc/ntp.conf
server 192.168.207.130 profer

# 时间同步服务主机上设置本机为ntp服务器
restrict -6 ::1
restrict 192.168.207.0 mask 255.255.255.0 nomodify # 允许局域网客户端
fudge  127.127.1.0 stratum 10

# （所有节点）配置自启动ntp服务
chkconfig ntpd on

# （所有节点）启动ntp服务
service ntpd start
```

> 5.（所有节点）JDK并配置环境变量 
注意指定安装目录 /usr/java/jdk-version, 否则会出现 not found java_home
```sh
#java
JAVA_HOME=/usr/java/jdk1.8.0_181
JRE_HOME=$JAVA_HOME/jre
PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/jt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

> 6.（所有节点）安装所需依赖库
```sh
yum -y install  chkconfig  python  bind-utils  psmisc  libxslt  zlib  sqlite  cyrus-sasl-plain  cyrus-sasl-gssapi  fuse  fuse-libs  redhat-lsb
```

==以上步骤可以先完成一台master 虚拟机,然后复制出slave01，slave02 ，slave01、slave02修改 hostname、ip ，删除第4步中 “设置本机为ntp服务器”的配置==

> 7.（所有节点）ssh 免秘钥登录
```sh
# 生成秘钥凭证
ssh-keygen -t rsa
# 复制凭证到131机器上
ssh-copy-id 192.168.207.131
```

> 安装 mysql（master节点）
```sh

rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.56-2.el7.x86_64

wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
yum clean all
yum makecache
sudo yum install mysql-server
# 自启动mysql
sudo /sbin/chkconfig mysqld on

```

> 配置初始化mysql /etc/my.conf
```sh
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
# symbolic-links = 0

key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
#and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

# For MySQL version 5.1.8 or later. For older versions, reference MySQL documentation for configuration help.
binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
```

> 启动、停止mysql
```sh
# 启动
sudo systemctl start mysqld

```

> 设置mysql 密码
```sh
sudo /usr/bin/mysql_secure_installation
[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] y
New password:
Re-enter new password:
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
All done!

```

> 创建 Cloudera Manager Server 所需要的数据库
```sql

CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY '000000';

```
> 如果还需要其他组件,则创建对应的库和用户权限

|组件|数据库|用户
|---|---|---
|Cloudera Manager Server|scm|scm
|Activity Monitor|amon|amon
|Reports Manager|rman|rman
|Hue|hue|hue
|Hive Metastore Server|metastore|hive
|Sentry Server|sentry|sentry
|Cloudera Navigator Audit Server|nav|nav
|Cloudera Navigator Metadata Server|navms|navms
|Oozie|oozie|oozie

> 安装mysql驱动
Important: Do not use the yum install command to install the MySQL driver package, because it installs openJDK, and then uses the Linux alternatives command to set the system JDK to be openJDK.
注意 不要使用yum 方式安装mysql 驱动包，因为它会安装openjdk
```sh
mkdir /usr/share/java
sudo cp mysql-connector-java-5.1.47/mysql-connector-java-5.1.47-bin.jar /usr/share/java/mysql-connector-java-5.1.47-bin.jar
```

> 9.安装 Cloudera Manager Server&Agent
> 9.1方式一：联网方式安装
```sh
wget https://archive.cloudera.com/cm5/installer/5.15.0/cloudera-manager-installer.bin
chmod u+x cloudera-manager-installer.bin
sudo ./cloudera-manager-installer.bin
```

> 9.2 方式二：离线安装 将安装包copy到所有集群机器上
[https://archive.cloudera.com/cm5/repo-as-tarball/](https://archive.cloudera.com/cm5/repo-as-tarball/)
[https://archive.cloudera.com/cdh5/repo-as-tarball/](https://archive.cloudera.com/cdh5/repo-as-tarball/)

> 9.3 方式三：不能连Internet的集群方式,内部库方式,在master机器上安装内部库

> 9.3.1 安装web server
```sh
# 添加库 /etc/yum.repos.d/CentOS-Base.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

yum clean all 
yum makecache
yum install nginx 

```

> 9.3.2 创建内部库，解压tar包并移动到web服务目录下，移动文件并更改权限后，请访问http://web_server>/cm以验证是否看到文件索引。如果看不到任何内容，则Web服务器可能已配置为不显示索引。
```sh
tar xvfz cm5.14.0-centos7.tar.gz
mkdir -p /usr/share/nginx/html
sudo mv cm /usr/share/nginx/html
sudo chmod -R ugo+rX /usr/share/nginx/html/cm

# 设置nginx配置文件
```

> 9.3.3 修改客户端以使用内部库
```sh
# /etc/yum.repos.d/cloudera-repo.repo
[cloudera-repo]
name=cloudera-repo
baseurl=http://<web_server>/cm/5
enabled=1
gpgcheck=0

yum clean all 
yum makecache
```

> 9.3.4 使用内部库方式安装
```sh
chmod u+x cloudera-manager-installer.bin
sudo ./cloudera-manager-installer.bin --skip_repo_package=1
```
