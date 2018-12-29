# cdh5x86_64
学习 CDH5.x 版 hadoop 

# 参考资料
  [官方文档](https://www.cloudera.com/documentation/enterprise/5-10-x/topics/cdh_ig_cdh5_cluster_deploy.html)

# 为什么要安装CDH版的Hadoop?
  第三方的发布版一般对hadoop做了集成\兼容\修复一些BUG,对安装过程做了简化处理,对众多的组件做了兼容测试,并提供web版的管理系统
  
# 安装前的准备
  查看要安装的版本支持的操作系统,数据库

# 安装

> JDK并配置环境变量 
注意安装目录 /usr/java/jdk-version, 否则会出现 not found java_home
```sh
#java
JAVA_HOME=/usr/java/jdk1.8.0_181
JRE_HOME=$JAVA_HOME/jre
PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/jt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

> 安装yum 库

```sh
yum --nogpgcheck localinstall cloudera-cdh-5-0.x86_64.rpm
rpm --import https://archive.cloudera.com/cdh5/redhat/7/x86_64/cdh/RPM-GPG-KEY-cloudera
yum install hadoop-yarn-resourcemanager
yum install hadoop-hdfs-namenode

# 部署NTP  Network Time Protocol 
yum install ntp
# 配置/etc/ntp.conf
server 0.cdh5
server 1.sunjx

chkconfig ntpd on

service ntpd start

ntpdate -u <your_ntp_server>

hwclock --systohc

```

> 配置
```sh
sudo cp -r /etc/hadoop/conf.empty /etc/hadoop/conf.my_cluster
sudo alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.my_cluster 50
sudo alternatives --set hadoop-conf /etc/hadoop/conf.my_cluster

```

sudo mkdir -p /data/1/dfs/nn /nfsmount/dfs/nn
sudo mkdir -p /data/1/dfs/dn
sudo chown -R hdfs:hdfs /data/1/dfs/nn /nfsmount/dfs/nn /data/1/dfs/dn
sudo chmod 700 /data/1/dfs/nn /nfsmount/dfs/nn

> core-site.xml
```sh

```

> 格式化hdfs
```sh
sudo -u hdfs hdfs namenode -format
```

> 启动
```sh
for x in `cd /etc/init.d ; ls hadoop-hdfs-*` ; do sudo service $x start ; done
```
