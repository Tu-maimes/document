---
title: Ambari集群的安装
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---


## Centos7.0 安装详解

 -  [参考链接](https://www.cnblogs.com/wcwen1990/p/7630545.html)
 -  [参考链接](https://blog.csdn.net/liuxiangke0210/article/details/80534614)

本文基于vmware workstations进行CentOS7安装过程展示，关于vmware workstations安装配置本人这里不再介绍，基本过程相当于windows下安装个软件而已。

 1. 打开vmware workstations，文件->新建虚拟机，出现如下界面，选择“自定义（高级）”选项，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384179863.png)

 2. 此步骤默认，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384205707.png)

 3. 在出现下面界面，选中“稍后安装操作系统”选项，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384249008.png)

 4. 在出现如下界面，客户机操作系统选择“linux”，版本选择“CentOS 64位”，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384336509.png)

 5. 出现如下界面，输入自定义虚拟机名称，虚拟机名称最好能做到望文生义，这里是“CentOS7_CDH_bd06”，指定虚拟机位置，这里是“D:\Virtual Machines\CentOS7_CDH_bd06”，然后下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384396912.png)

 6. 出现下面界面，选择处理器数量和每个处理器核心数量，这里分别是2和4，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384431507.png)

 7. 出现如下界面，指定虚拟机占用内存大小，这里是2048M，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384461898.png)

 8. 出现如下界面，选择网络连接类型，这里选择“使用桥接网络”，各位安装虚拟机过程根据需要自行选择，安装向导中已经针对各种模式进行了比较规范的说明，这里补充说明如下：

1）使用桥接网络：虚拟机ip与本机在同一网段，本机与虚拟机可以通过ip互通，本机联网状态下虚拟机即可联网，同时虚拟机与本网段内其他主机可以互通，这种模式常用于服务器环境架构中。

2）使用网络地址转换（NAT）：虚拟机可以联网，与本机互通，与本机网段内其他主机不通。

3）使用仅主机模式网络：虚拟机不能联网，与本机互通，与本机网段内其他主机不通。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384671383.png)

 9. 默认，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384703010.png)
 10. 默认、下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384728647.png)

 11. 默认，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384766205.png)

 12. 出现下面界面，输入虚拟机磁盘大小，默认20g一般不够使用，建议设置略大一些，这里设置虚拟机磁盘大小为80G，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384820915.png)

 13. 默认，下一步继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384840599.png)

 14. 默认、点击“完成”结束虚拟机创建：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543384865741.png)

 15. 退出安装向导后，我们可以在虚拟机管理界面左侧栏看到刚刚创建的虚拟机，右侧栏可以看到虚拟机详细配置信息。
 16. 上图界面中点击“编辑虚拟机设置”选项，出现如下界面：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385004649.png)

 17. 上图中需要指定“CD/DVD(IDE)”安装镜像，移除“USB控制器”、“声卡”和“打印机”，然后点击确定，按照上述设置后界面如下图所示：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385050686.png)

 18. 点击开启虚拟机进入CentOS7操作系统安装过程。
 19. 虚拟机控制台出现界面，选择Install CentOS liunx 7，点击回车键继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385110314.png)

 20. 根据提示点击回车键继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385145125.png)

 21. 如下界面默认选择English，点击Continue继续：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385185877.png)

 22. CentOS7安装配置主要界面如下图所示，根据界面展示，这里对以下3个部分配置进行说明：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385210362.png)

Localization和software部分不需要进行任何设置，其中需要注意的是sofrware selection选项，这里本次采用默认值（即最小化安装，这种安装的linux系统不包含图形界面）安装，至于其他组件，待后期使用通过yum安装即可。（建议最小化安装里面的软件全选）

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385279057.png)

如上图，system部分需要必须规划配置的是图中红色部分选项，即磁盘分区规划，另外可以在安装过程中修改network & host name选项中修改主机名（默认主机名为localhost.localdomain）。具体配置过程如下：

点击“installation destination”，进入如下界面，选中80g硬盘，下来滚动条到最后，选中“i will configure partitioning”，即自定义磁盘分区，最后点击左上角done进行磁盘分区规划：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385299964.png)

 23. CentOS7划分磁盘即在下图界面进行，这里先说明一下前期规划：

./boot：1024M，标准分区格式创建。
swap：4096M，标准分区格式创建。
/：剩余所有空间，采用lvm卷组格式创建。
规划后界面如下，点击done完成分区规划，在弹出对话框中点击“accept changs”：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385348782.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385379701.png)

 24. 完成磁盘规划后，点击下图红框部分，修改操作系统主机名，这里修改为db06（如第二图所示），然后点击done完成主机名配置，返回主配置界面：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385410539.png)

 25. 在下图中，其实从第24步配置开始我们就可以发现右下角“begin installtion”按钮已经从原本的灰色变成蓝色，这说明已经可以进行操作系统安装工作了，点击“begin installtion”进行操作系统安装过程。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385471834.png)

 26. 在下图用户设置中需要做的仅是修改root用户密码，点击“root password”，设置密码，如果密码安全度不高，比如我这里的密码为“root1234”，那么可能需要点击2次确定才可以。当root密码设置成功再次返回安装界面时我们可以发现之前user setting界面红色警告消失了，对比下面图1和图3：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385512426.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385529826.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385549130.png)

 27. 在下图，操作系统安装已经完成，点击reboot重启操作系统。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385568082.png)

 28. 使用root用户登录（即root），修改IP地址（vi /etc/sysconfig/network-scripts/ifcfg-ens32）：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385604584.png)

按字符键“i”进入编辑模式，修改/etc/sysconfig/network-scripts/ifcfg-ens32文件内容如下：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385636061.png)

按“esc”键后，输入:wq回车，完成配置文件编辑。

输入：service network restart命令重启网卡，生效刚刚修改ip地址，`ping www.baidu.com` 测试网络连通性。

![ ](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543385672431.png)
 
 
好了，至此，CentOS7操作系统安装成功了。


## 安装Ambari集群

### 基础环境的搭建

以下操作均用root用户操作。

#### 网络配置设置hostname(所有节点)

`vi /etc/sysconfig/network`修改hostname：
NETWORKING=yes
HOSTNAME=master(每个节点设置自己的名字)
`service network restart`   重启网络服务生效。
`reboot`重启所有机器

#### 配置ip与hostname的映射关系

``` nginx
vi /etc/hosts


192.168.3.100 master
192.168.3.101 slaver01
192.168.3.102 slaver02
```

==注意==：这里需要将每台机器的ip及主机名对应关系都写进去，本机(本地的Windows系统C:\Windows\System32\drivers\etc\hosts)的也要写进去，否则启动Agent的时候会提示hostname解析错误。

#### 打通SSH，设置ssh无密码登陆（所有节点）

在所有节点上执行 `ssh-keygen -t rsa`   一路回车，生成无密码的密钥对。

将公钥添加到认证文件中：`cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`文件

并设置authorized_keys的访问权限：
`chmod 600 ~/.ssh/authorized_keys`
scp文件到所有datenode节点：
`scp ~/.ssh/authorized_keys root@slaver01:~/.ssh/`
`scp ~/.ssh/authorized_keys root@slaver02:~/.ssh/`

测试：在主节点上ssh slaver02，正常情况下，不需要密码就能直接登陆进去了。

#### 设置文件打开数量和用户最大进程数

	
	查看文件打开数量        `ulimit -a` 
	查看用户最大进程数      `ulimit -u`
	

``` stata
vi /etc/security/limits.conf 

	增加以下内容：
	* soft nofile 65535
	* hard nofile 65535
	* soft nproc 32000
	* hard nproc 32000 
```

####  关闭每台机器的防火墙

``` vala
#停止firewall
systemctl stop firewalld.service  
#禁止firewall开机启动
systemctl disable firewalld.service  
#查看默认防火墙状态（关闭后显示not running，开启后显示running）
firewall-cmd --state  
```

####  关闭每台机器的Selinux

禁用selinux，也就是修改/etc/selinux/config文件，修改后的内容为：

``` nginx
vi /etc/selinux/config 

修改：SELINUX=disabled
```

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543387206447.png)

重启系统：`reboot`

查看SELinux 的状态：

	[root@master ~]# sestatus
	SELinux status:  disabled

####  关闭每台机器的THP服务
 
 为了解决已启用“透明大页面”，它可能会导致重大的性能问题。


``` groovy
vi /etc/rc.local
添加一下内容
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```
检查：有[never]则表示THP被禁用

	[root@master ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
	always madvise [never]
	

#### 集群时间同步

 查看是否安装时间同步服务器`rpm -qa |grep ntpd` 》》没有安装则需要安装此服务 》》`yum install ntp`
	 

 1. 在主节点

``` vim
		vi /etc/ntp.conf

		1. 去掉这个注释，将地址改成网段地址
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
		
		
		
		2.配置boot时间和系统时间同步
		vi /etc/sysconfig/ntpd  加入下面一句话，用于
		SYNC_HWCLOCK=yes
		
	
	
		3.启动ntpd服务器	

        ntpdate -u 3.asia.pool.ntp.org
		sudo service ntpd start
		sudo service ntpd status
		chkconfig ntpd on
		chkconfig --list |grep ntpd
```
2. 客户机配置（从节点）

``` lsl
		vi /etc/ntp.conf
		
		1.去掉这个注释，将地址改成网段地址
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
        
	    2. 启动ntpd客服端	
        ntpdate -u 192.168.1.46
		sudo service ntpd start
		sudo service ntpd status
		chkconfig ntpd on
		chkconfig --list |grep ntpd
```

#### 搭建Yum源服务器

##### 安装yum源

 1. 选择master服务器作为http服务器：

``` crmsh
[root@master ~]# mkdir -p /var/www/html
```
使用安装系统的ISO镜像文件CentOS-7-x86_64-DVD-1804.iso把CentOS-7-x86_64-DVD-1804.iso镜像复制到http服务器(选择master机器）的默认目录/var/www/html下
	

``` crmsh
	[root@master ~]# cd /var/www/html
	[root@master html]# ls
	CentOS-7-x86_64-DVD-1804.iso
```
在/var/www/html目录下创建文件夹CentOS

``` crmsh
[root@master html]# mkdir CentOS 
```

将ISO文件挂载至文件夹/var/www/html/CentOS下

``` crmsh
[root@master html]# cd  /var/www/html  
[root@master html]# mount -o loop CentOS-7-x86_64-DVD-1804.iso CentOS 
```
(取消挂载 umount /var/www/html/CentOS)

查看文件夹/var/www/html/CentOS

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543388518095.png)

``` crmsh
[root@master CentOS]# cd /etc/yum.repos.d/
[root@master yum.repos.d]# mkdir -p /etc/yum.repos.d/bak
[root@master yum.repos.d]# cp *.repo ./bak 
```
修改CentOS-Media.repo，删去原有内容并写入如下内容：

``` crmsh
[root@master yum.repos.d]# vi CentOS-Media.repo 
```
原有内容：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543388680886.png)

修改后内容：

``` ini
[c7-media]
name=CentOS-$releasever - Everything_ISO
enabled=1
baseurl=file:///var/www/html/CentOS
gpgcheck=1
gpgkey=file:///var/www/html/CentOS/RPM-GPG-KEY-CentOS-7
```

修改CentOS-Base.repo，在每一组中添加一行如下内容：enabled=0

``` crmsh
[root@master yum.repos.d]# vi CentOS-Base.repo
```
![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543388783761.png)

运行如下命令：

``` glsl
#清除yum的缓存、头文件、已下载的软件包等等
yum clean all


#重建yum缓存
yum makecache


#查看已启用的镜像源
 yum repolist all
```
![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543388900071.png)

ISO镜像yum源搭建OK！！！

##### 安装http服务器

本文在集群机器中任选一台机器作为源服务器，以master机器为例检查系统是否已经安装http服务 

``` crmsh
[root@master ~]# which httpd
/usr/sbin/httpd
```
若没有出现上述/usr/sbin/httpd目录信息，则说明没有安装；如果有，则跳过该步骤！

``` nginx
#如果没有安装则执行
yum install httpd
```

http服务使用80端口，检查端口是否占用

``` lsl
netstat -nltp | grep 80
```
如果有占用情况，安装完毕后需要修改http服务的端口号

``` vim
vi /etc/httpd/conf/httpd.conf
```

 修改监听端口，Listen 80为其他端口

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543389158112.png)

启动httpd服务器：

``` crmsh
[root@master ~]# systemctl start httpd.service
```
打开浏览器，访问http://192.168.3.100:80 ，能正确打开网页，服务正常启动

#### 制作离线源

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543389439503.png)

将这四个Ambari及HDP相关安装包复制到http服务器(这里我们选择master机器）的/var/www/html目录下，解压Ambari及HDP相关rpm包，生成相应的目录：

``` css
[root@master html]# tar -xzvf ambari-2.6.2.0-centos7.tar.gz
[root@master html]# tar -xzvf HDP-2.6.5.0-centos7-rpm.tar.gz
[root@master html]# tar -xzvf HDP-GPL-2.6.5.0-centos7-gpl.tar.gz
[root@master html]# tar -xzvf HDP-UTILS-1.1.0.22-centos7.tar.gz
```
![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543389600749.png)


目录说明：
ambari目录：包含ambari-server，ambari-agent，ambari-log4j等rpm包
HDP目录：包含hadoop的生态圈的组件，比如hdfs，hive，hbase，mahout等
HDP-UTILS目录：包含HDP平台所包含的工具组件等，比如nagios，ganglia，puppet等

##### 配置Ambari源

复制ambari.repo文件到/etc/yum.repos.d（该步骤在master机器上执行）
复制hdp.repo文件到/etc/yum.repos.d（该步骤在master机器上执行）
复制hdp-gpl.gpl.repo文件到/etc/yum.repos.d（该步骤在master机器上执行）
``` groovy
cp /var/www/html/ambari/centos7/2.6.2.0-155/ambari.repo /etc/yum.repos.d/
cp /var/www/html/HDP/centos7/2.6.5.0-292/hdp.repo
/etc/yum.repos.d/
cp /var/www/html/HDP-GPL/centos7/2.6.5.0-292/hdp-gpl.gpl.repo
/etc/yum.repos.d/
```
编辑刚拷贝的文件文件：

ambari.repo文件

``` ini
#VERSION_NUMBER=2.6.2.0-155
[ambari-2.6.2.0]
name=ambari Version - ambari-2.6.2.0
baseurl=http://192.168.3.100/ambari/centos7/2.6.2.0-155/
gpgcheck=0
gpgkey=http://192.168.3.100/ambari/centos7/2.6.2.0-155/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```
hdp.repo文件

``` ini
#VERSION_NUMBER=2.6.5.0-292
[HDP-2.6.5.0]
name=HDP Version - HDP-2.6.5.0
baseurl=http://192.168.3.100/HDP/centos7/2.6.5.0-292/
gpgcheck=0
gpgkey=http://192.168.3.100/HDP/centos7/2.6.5.0-292/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1


[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
baseurl=http://192.168.3.100/HDP-UTILS/centos7/1.1.0.22
gpgcheck=0
gpgkey=http://192.168.3.100/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```
hdp-gpl.gpl.repo文件

``` x86asm
#VERSION_NUMBER=2.6.5.0-292
[HDP-GPL-2.6.5.0]
name=HDP-GPL Version - HDP-GPL-2.6.5.0
baseurl=http://192.168.3.100/HDP-GPL/centos7/2.6.5.0-292
gpgcheck=0
gpgkey=http://192.168.3.100/HDP-GPL/centos7/2.6.5.0-292/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```
配置CentOS-Base.repo为aliyun的yum源

``` tcl
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#
 
[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/os/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
#released updates 
[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/updates/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/extras/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/centosplus/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
#contrib - packages by Centos Users
[contrib]
name=CentOS-$releasever - Contrib - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/contrib/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/contrib/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/contrib/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
```

CentOS-Media.repo文件

``` shell
# CentOS-Media.repo
#
#  This repo can be used with mounted DVD media, verify the mount point for
#  CentOS-7.  You can use this repo and yum to install items directly off the
#  DVD ISO that we release.
#
# To use this repo, put in your DVD and use it with the other repos too:
#  yum --enablerepo=c7-media [command]
#  
# or for ONLY the media repo, do this:
#
#  yum --disablerepo=\* --enablerepo=c7-media [command]

[c7-media]
name=CentOS-$releasever - Media
baseurl=http:///192.168.3.100/CentOS/
gpgcheck=1
enabled=0
gpgkey=http:///192.168.3.100/CentOS/RPM-GPG-KEY-CentOS-7
```



WARNING: Before starting Ambari Server, you must run the following DDL against the database to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql