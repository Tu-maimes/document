---
title: CDH集群的搭建
tags: 作者:汪帅
grammar_cjkRuby: true
---


一，集群规划
【注意】：至少三台节点，主节点至少6G内存，另外两台至少4G内存，否则安装CDH5会因为运行环境不达标会报各种错误

|主机/服务     |   Server  |    Agent    |     zookeeper  |  namenode | secondarynamenode  |   datanode  |
|---|---|---|---|---|---|---|
|master01      |    是     |     是      |        是      |    是     |                    |     是      |
|master02      |           |     是      |        是      |           |         是         |     是      |
|slaver01      |           |     是      |        是      |           |                    |     是      |
|slaver02      |           |     是      |        是      |           |                    |     是      |
|slaver03      |           |     是      |        是      |           |                    |     是      |

二，linux系统环境准备
系统环境
实验环境: VMware虚拟机
操作系统：CentOS 6.8 x64 
Cloudera Manager：5.14.0
CDH: 5.14.0
CDH安装包地址：http://archive.cloudera.com/cdh5/parcels/5.14.0/  ，由于我们的操作系统为CentOS6.8，需要下载以下文件：
CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel
CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel.sha1
manifest.json    
Cloudera Manager下载地址：
http://archive.cloudera.com/cm5/cm/5/
安装说明
一 准备工作：系统环境搭建
以下操作均用root用户操作。
1. 网络配置设置hostname(所有节点)
#vi /etc/sysconfig/network修改hostname：
NETWORKING=yes
HOSTNAME=master01(每个节点设置自己的名字)
#service network restart   重启网络服务生效。
#reboot (整体46-50)

2.配置ip与hostname的映射关系
#vi /etc/hosts
192.168.1.46 master01
192.168.1.47 master02
192.168.1.48 slaver01
192.168.1.49 slaver02
192.168.1.50 slaver03
192.168.1.51 apiserver

注意：这里需要将每台机器的ip及主机名对应关系都写进去，本机(本地的Windows系统C:\Windows\System32\drivers\etc\hosts)的也要写进去，否则启动Agent的时候会提示hostname解析错误。

如没有 scp 则安装：
#yum -y install openssh-clients

3.打通SSH，设置ssh无密码登陆（所有节点）
在所有节点上执行
#ssh-keygen -t rsa   一路回车，生成无密码的密钥对。

将公钥添加到认证文件中：
#cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys文件

并设置authorized_keys的访问权限：
#chmod 600 ~/.ssh/authorized_keys
scp文件到所有datenode节点：
#scp ~/.ssh/authorized_keys root@master02:~/.ssh/
#scp ~/.ssh/authorized_keys root@slaver01:~/.ssh/
#scp ~/.ssh/authorized_keys root@slaver02:~/.ssh/
#scp ~/.ssh/authorized_keys root@slaver03:~/.ssh/

测试：在主节点上ssh master02，正常情况下，不需要密码就能直接登陆进去了。
4.yum源跟换和添加
  更换yum 源为163,请执行以下命令
		#cd /etc/yum.repos.d
		
		#yum install -y wget （下载资源命令）	
		#rename .repo .repo.bak *
		#wget http://mirrors.163.com/.help/CentOS6-Base-163.repo
		#yum clean all
		#yum makecache
		#yum install lrzsz

 设置文件打开数量和用户最大进程数
	
	查看文件打开数量        ulimit -a 
	查看用户最大进程数      ulimit -u
	vi /etc/security/limits.conf 
	增加以下内容：
	* soft nofile 65535
	* hard nofile 65535
	* soft nproc 32000
	* hard nproc 32000 		
		
5.安装Oracle的Jdk（所有节点）所有jdk安装在各个节点下的同一个目录级别下
CentOS，自带OpenJdk，不过运行CDH5需要使用Oracle的Jdk，本次安装使用的是jdk8。
卸载自带的OpenJdk，使用
#rpm -qa | grep java
查询java相关的包，使用
#rpm -e --nodeps 包名卸载
去Oracle的官网下载jdk的tar.gz安装包
上传安装包到/opt/eco_cluster/software
使用解压指令解压到指定的文件夹
#tar -zxvf jdk-8u152-linux-x64.tar.gz -C /opt/eco_cluster/jdk
配置环境变量 
#vi /etc/profile 
在最后添加
export JAVA_HOME=/opt/eco_cluster/jdk/jdk1.8.0_152
export PATH=$JAVA_HOME/bin:$PATH
使用
#scp -r  /opt/eco_cluster/jdk/jdk1.8.0_152 root@master02:/opt/eco_cluster/jdk
#scp -r  /opt/eco_cluster/jdk/jdk1.8.0_152 root@slaver01:/opt/eco_cluster/jdk
#scp -r  /opt/eco_cluster/jdk/jdk1.8.0_152 root@slaver02:/opt/eco_cluster/jdk
#scp -r  /opt/eco_cluster/jdk/jdk1.8.0_152 root@slaver03:/opt/eco_cluster/jdk
配置所有节点的环境变量
source /etc/profile 让配置的profile生效
java -version 查看是否安装成功

（#ln -s /opt/eco_cluster/jdk/jdk1.8.0_152 /usr/java/jdk1.8）

6 执行以下命令扩大jdk的使用范围
（root用户）
/*暂时不做*/ #echo "JAVA_HOME=/opt/eco_cluster/jdk/jdk1.8.0_152" >> /etc/environment

优化linux内存的使用
设置将 /proc/sys/vm/swappiness 设置为 0 (修改swap空间的swappiness，降低对硬盘的缓存)
(root用户）输入：
#echo "vm.swappiness=0"  >> /etc/sysctl.conf ）


7.关闭防火墙和SELinux
注意： 需要在所有的节点上执行，因为涉及到的端口太多了，临时关闭防火墙是为了安装起来更方便，安装完毕后可以根据需要设置防火墙策略，保证集群安全。
关闭防火墙：
#service iptables stop （临时关闭）  
#chkconfig iptables off （重启后生效）
#service iptables status (查看防火墙)
关闭SELINUX（实际安装过程中发现没有关闭也是可以的，不知道会不会有问题，还需进一步进行验证）:
#setenforce 0 （临时生效）  
修改 /etc/selinux/config 下的 SELINUX=disabled （重启后永久生效）



8.Mysql Bundle 安装

在192.168.1.46上

#groupadd mysql
#useradd -r -g mysql -s /bin/false mysql
#rpm -qa | grep mysql
mysql-libs-5.1.73-7.el6.x86_64

#rpm -e –-nodeps mysql-libs-5.1.73-7.el6.x86_64

如没有perl，则安装
#yum -y install openssh-clients
#yum -y install perl
#yum -y install numactl
#yum -y install libxslt-devel

或者一次性安装linux开发基础工具
#yum  groupinstall Development tools



#cd /opt/software
#tar -xvf mysql-5.7.21-1.el6.x86_64.rpm-bundle.tar -C /opt/mysql
#cd /opt/mysql
mysql-community-libs-5.7.21-1.el6.x86_64.rpm
mysql-community-devel-5.7.21-1.el6.x86_64.rpm
mysql-community-server-5.7.21-1.el6.x86_64.rpm
mysql-community-test-5.7.21-1.el6.x86_64.rpm
mysql-community-embedded-5.7.21-1.el6.x86_64.rpm
mysql-community-client-5.7.21-1.el6.x86_64.rpm
mysql-community-libs-compat-5.7.21-1.el6.x86_64.rpm
mysql-community-common-5.7.21-1.el6.x86_64.rpm
mysql-community-embedded-devel-5.7.21-1.el6.x86_64.rpm

#rpm -ivh mysql-community-common-5.7.21-1.el6.x86_64.rpm
#rpm -ivh mysql-community-libs-5.7.21-1.el6.x86_64.rpm
#rpm -ivh mysql-community-libs-compat-5.7.21-1.el6.x86_64.rpm
#rpm -ivh mysql-community-client-5.7.21-1.el6.x86_64.rpm
#rpm -ivh mysql-community-server-5.7.21-1.el6.x86_64.rpm
#rpm -ivh mysql-community-devel-5.7.21-1.el6.x86_64.rpm

#service mysqld start

由于MySQL5.7.4之前的版本中默认是没有密码的，登录后直接回车就可以进入数据库，进而进行设置密码等操作。
其后版本对密码等安全相关操作进行了一些改变，在安装过程中，会在安装日志中生成一个随机密码

#grep 'temporary password' /var/log/mysqld.log

2018-03-14T06:39:48.307922Z 1 [Note] A temporary password is generated for root@localhost: qlMiQbT_q5CJ

#mysql -uroot -p

注: 如果没有修改密码，系统将会提示错误：在"ALTER USER"操作前不会执行其他操作；
并且确保修改之后的密码的安全程度能够满足MySQL的要求。推荐数字、字母和特殊符号混用。


mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Ecosystem@46';

另：如果想要修改密码的话，可以执行以下语句：
mysql> update mysql.user set authentication_string=password('Ecosystem@46') where user='root';

注: 从MySQL5.7.6开始，mysql.user表中存储密码的字段不再是password，而是authentication_string。

#/usr/bin/mysql_secure_installation --安装完mysql后可执行自带的安全设置脚本(此次未执行)

使其它机器可以root登录测试使用：
mysql> grant all privileges on *.* to 'root'@'%' identified by 'Ecosystem@46' with grant option;

mysql> FLUSH PRIVILEGES;

#vi /etc/my.cnf    修改配置等

然后在集群其它节点安装mysql客户端及相应依赖包。


9.集群时间同步（主节点）

	9.1 rpm -qa |grep ntpd 》》没有安装则需要安装此服务 》》yum install ntp
	 
	9.2 vi /etc/ntp.conf
		去掉这个注释，将地址改成网段地址
		restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
		
		注释掉这几个
		#server 0.centos.pool.ntp.org iburst
		#server 1.centos.pool.ntp.org iburst
		#server 2.centos.pool.ntp.org iburst
		#server 3.centos.pool.ntp.org iburst
	
		添加一下内容
        server 0.asia.pool.ntp.org
        server 3.asia.pool.ntp.org
        restrict 0.asia.pool.ntp.org nomodify notrap noquery
        restrict 3.asia.pool.ntp.org nomodify notrap noquery
		server 127.127.1.0
		fudge  127.127.1.0  stratum 10
	
	9.3 vi /etc/sysconfig/ntpd  加入下面一句话，用于配置boot时间和系统时间同步
		SYNC_HWCLOCK=yes
	
   
	9.4 启动ntpd服务器	
        ntpdate -u 3.asia.pool.ntp.org
		sudo service ntpd start
		sudo service ntpd status
		chkconfig ntpd on
		chkconfig --list |grep ntpd
		
		
	9.5 客户机配置（从节点）
		使用 root用户操作
        去掉这个注释，将地址改成网段地址
		restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
        
        
        注释掉这几个
		# server 0.centos.pool.ntp.org iburst
        # server 1.centos.pool.ntp.org iburst
        # server 2.centos.pool.ntp.org iburst
        # server 3.centos.pool.ntp.org iburst
        
        
        
        添加一下内容
        server 192.168.1.46
        restrict 192.168.1.46 nomodify notrap noquery
        server  127.127.1.0     # local clock
        fudge   127.127.1.0 stratum 10
        
    9.6 启动ntpd客服端	
        ntpdate -u 192.168.1.46
		sudo service ntpd start
		sudo service ntpd status
		chkconfig ntpd on
		chkconfig --list |grep ntpd
        
10. 安装Cloudera Manager Server 和Agent

主节点解压安装cloudera manager的目录默认位置在/opt下，解压：

#tar -zxvf cloudera-manager-el6-cm5.14.0_x86_64.tar.gz -C /opt

解压数据库驱动包拷贝到 /opt/cm-5.14.0/share/cmf/lib/

#cp mysql-connector-java-5.1.45/mysql-connector-java-5.1.45-bin.jar /opt/cm-5.14.0/share/cmf/lib/


在mysql配置文件密码安全级别
#vi /etc/my.cnf 
validate-password=OFF (设置不使用安全插件)

Agent配置
修改/opt/cm-5.14.0/etc/cloudera-scm-agent/config.ini中的server_host为主节点的主机名(master01)。
同步Agent到所有节点
#scp -r /opt/cm-5.14.0/ root@master02:/opt/

在所有节点创建cloudera-scm用户
#useradd --system --home=/opt/cm-5.14.0/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm

在主节点【master01】初始化CM5的数据库
#/opt/cm-5.14.0/share/cmf/schema/scm_prepare_database.sh mysql cm -hlocalhost -uroot -pEcosystem@46 --scm-host localhost scm scm scm

准备Parcels，用以安装CDH5
将CHD5相关的Parcel包放到主节点的/opt/cloudera/parcel-repo/目录中（parcel-repo需要手动创建）。
相关的文件如下：
CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel
CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel.sha1
manifest.json
最后将CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel.sha1，重命名为CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel.sha，这点必须注意，
否则，系统会重新下载CDH-5.14.0-1.cdh5.14.0.p0.24-el6.parcel文件。

启动CDH Web安装界面服务
通过
#/opt/cm-5.14.0/etc/init.d/cloudera-scm-server start启动服务端。
通过
#/opt/cm-5.14.0/etc/init.d/cloudera-scm-agent start启动Agent服务(启动所有节点的agent)。

我们启动的其实是个service脚本，需要停止服务将以上的start参数改为stop就可以了，重启是restart。


CDH5的安装配置
Cloudera Manager Server和Agent都启动以后，就可以进行CDH5的安装配置了。
这时可以通过浏览器访问主节点的7180端口测试一下了（由于CM Server的启动需要花点时间，这里可能要等待一会才能访问），
默认的用户名和密码均为admin：
登录界面后选择免费版进行安装



五，Clouder Manager Server和agent 安装过程异常情况处理 

1　异常1
已启用“透明大页面”，它可能会导致重大的性能问题。
版本为“CentOS release 6.3 (Final)”且版本为“2.6.32-279.el6.x86_64”的 Kernel 已将 enabled 设置为“[always] never”，
并将 defrag 设置为“[always] never”。请运行“echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag”以禁用此设置，
然后将同一命令添加到一个 init 脚本中，如 /etc/rc.local，这样当系统重启时就会设置它 
（每个节点）
解决方案
vi /etc/rc.local
添加一下内容
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled


2 异常2
Bigtop 的原因： 由于CDH不会使用系统默认JAVA_HOME环境变量，而是使用Bigtop进行管理，故我们需要安装Bigtop的规则在指定的位置安装jdk。
/opt/cm-5.14.0/lib64/cmf/service/common/cloudera-config.sh
其中可以看到：
local JAVA8_HOME_CANDIDATES=(
'/usr/java/jdk1.8'
'/usr/java/jre1.8'
'/usr/lib/jvm/j2sdk1.8-oracle'
'/usr/lib/jvm/j2sdk1.8-oracle/jre'
'/usr/lib/jvm/java-8-oracle'
)

于是，建立一个已经有的JAVA_HOME  链接到 /usr/java/jdk1.8 就好了！
建立连接指令
#ln -s /opt/eco_cluster/jdk/jdk1.8.0_152 /usr/java/jdk1.8

原博客地址https://www.cnblogs.com/FlyAway2013/p/7503524.html


3.Oozie找不到驱动程序
把mysql的驱动包拷贝到各服务器相应目录
/opt/cloudera/parcels/CDH/lib/oozie/libext/与/opt/cloudera/parcels/CDH/lib/oozie/lib

#scp mysql-connector-java-5.1.45-bin.jar root@master02:/opt/cloudera/parcels/CDH/lib/oozie/libext/
#scp mysql-connector-java-5.1.45-bin.jar root@master02:/opt/cloudera/parcels/CDH/lib/oozie/lib

注意:重装搭建部署集群时需要先执行
#rm -rf /dfs
在去mysql数据库中把oozie数据库删除再新建一个oozie

4.将mysql驱动拷贝到集群各服务器hive相应目录中
#scp mysql-connector-java-5.1.45-bin.jar root@master02:/opt/cloudera/parcels/CDH/lib/hive/lib


hue界面的用户名root密码为ecosystem

5.修改HUE中的配置时区为Asia/Shanghai



6.解决使用第三方json类以及自带的正则类入库hive中无法使用where条件语句

所有节点：
在hive-site.xml中添加配置

#vi /opt/cloudera/parcels/CDH/lib/hive/conf/hive-site.xml
<property>
    <name>hive.aux.jars.path</name>
    <value>
       file:///opt/cloudera/parcels/CDH/lib/hive/auxlib/json-serde-1.3.8-jar-with-dependencies.jar,file:///opt/cloudera/parcels/CDH/lib/hive/auxlib/json-udf-1.3.8-jar-with-dependencies.jar,file:///opt/cloudera/parcels/CDH/lib/hive/auxlib/hive-contrib-1.1.0-cdh5.14.0.jar
    </value>
  </property>

把添加的配置文件中的jar文件拷贝到/opt/cloudera/parcels/CDH/lib/hive/auxlib

cp /opt/cloudera/parcels/CDH/lib/hive/lib/hive-contrib-1.1.0-cdh5.14.0.jar /opt/cloudera/parcels/CDH/lib/hive/auxlib
重启后生效

7.hue中报如下问题：
Could not start SASL: Error in sasl_client_start (-4) SASL(-4): no mechanism available: No worthy mechs found

安装如下支持库
#yum install cyrus-sasl-plain  cyrus-sasl-devel  cyrus-sasl-gssapi

或中hive-site.xml中添加配置
<property>  
        <name>hive.server2.authentication</name>  
        <value>NOSASL</value> 
</property>


8.运行Spark2的程序写HBase出现找不到HBase的相关类,解决方案(所有节点)
    将HBase的以下类拷贝到/opt/cloudera/parcels/SPARK2/lib/spark2/jars
    
     guava-12.0.1.jar
     hbase-client-1.2.0-cdh5.14.0.jar
     hbase-common-1.2.0-cdh5.14.0.jar
     hbase-procedure-1.2.0-cdh5.14.0.jar
     hbase-server-1.2.0-cdh5.14.0.jar
     hbase-protocol-1.2.0-cdh5.14.0.jar
     hbase-hadoop2-compat-1.2.0-cdh5.14.0.jar
     hbase-hadoop-compat-1.2.0-cdh5.14.0.jar
     hbase-annotations-1.2.0-cdh5.14.0.jar
     hbase-prefix-tree-1.2.0-cdh5.14.0.jar
     hbase-spark-1.2.0-cdh5.14.0.jar
     protobuf-java-2.5.0.jar
     htrace-core-3.2.0-incubating.jar
     
     
9.运行spark的程序找不到 com.mysql.jdbc.Driver
    把数据库连接驱动拷贝到/user/oozie/share/lib/lib_20180608171440/spark/目录下
    更新Oozie的share-lib
    #oozie admin -oozie http://192.168.1.46:11000/oozie -sharelibupdate
    
    
    
    
10.安装Apache解决HUE的负载均衡不能启动
    yum install httpd
    yum install mod_ssl