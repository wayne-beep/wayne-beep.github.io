

目录
测试环境

Hadoop 组织框架

HDFS架构

YARN架构

HA集群部署规划

自动故障转移

关于集群主机时间

Linux环境搭建

配置Java环境

安装单机版Hadoop

Zookeeper集群安装

配置环境变量

关闭防火墙

修改hosts文件

配置SSH免密登录

修改Hadoop配置文件

Hadoop集群的初始化

Hadoop集群的启动

测试环境
Linux系统版本：CentOS 7 64位

Hadoop版本：hadoop-2.7.3

Java版本：jdk-8u181-linux-x64

ZooKeeper版本：zookeeper-3.4.10.tar.gz

配置HA高可用集群建议先看一下完全分布式集群的部署过程，整个流程大致一样。

CentOS 7部署Hadoop集群（完全分布式）

Hadoop 组织框架
Hadoop主要包括两部分：

一部分是HDFS（Hadoop Distributed File System），主要负责分布式存储和计算；

另一部分是YARN（Yet Another Resource Negotiator， 从Hadoop2.0开始引入），主要负责集群的资源管理和调度。

HDFS架构
架构图：



1. Active Name Node

主Master，整个Hadoop集群只能有一个

管理HDFS文件系统的命名空间

维护元数据信息

管理副本的配置和信息（默认三个副本）

处理客户端读写请求

2. Standby Name Node

Active Name Node的热备节点

Active Name Node故障时可快速切换成新的Active Name Node

周期性同步edits编辑日志，定期合并fsimage与edits到本地磁盘

3. Journal Node

可以被Active Name Node和StandBy Name Node同时访问，用以支持Active Name Node高可用

Active Name Node在文件系统被修改时，会向Journal Node写入操作日志（edits）

Standby Name Node同步Journal Node edits日志，使集群中的更新操作可以被共享和同步。

4. Data Node

Slave 工作节点，集群一般会启动多个

负责存储数据块和数据块校验

执行客户端的读写请求

通过心跳机制定期向NameNode汇报运行状态和本地所有块的列表信息

在集群启动时DataNode项NameNode提供存储Block块的列表信息

5. Block数据块

HDSF固定的最小的存储单元（默认128M，可配置修改）

写入到HDFS的文件会被切分成Block数据块（若文件大小小于数据块大小，则不会占用整个数据块）

默认配置下，每个block有三个副本

6. Client

与Name Node交互获取文件的元数据信息

与Data Node，读取或者写入数据

通过客户端可以管理HDFS

YARN架构
架构图：



1. Resource Manager

整个集群只有一个Master。Slave可以有多个，支持高可用

处理客户端Client请求

启动／管理／监控ApplicationMaster

监控NodeManager

资源的分配和调度

2. Node Manager

每个节点只有一个，一般与Data Node部署在同一台机器上且一一对应

定时向Resource Manager汇报本机资源的使用状况

处理来自Resource Manager的作业请求，为作业分配Container

处理来自Application Master的请求，启动和停止Container

3. Application Master

每个任务只有一个，负责任务的管理，资源的申请和任务调度

与Resource Manager协商，为任务申请资源

与Node Manager通信，启动／停止任务

监控任务的运行状态和失败处理

4. Container

任务运行环境的抽象，只有在分配任务时才会抽象生成一个Container

负责任务运行资源和环境的维护（节点，内存，CPU）

负责任务的启动

虽然在架构图中没有画出，但Hadoop高可用都是基于Zookeeper来实现的。如NameNode高可用，Block高可用，ResourceManager高可用等

以上部分内容来自：https://baijiahao.baidu.com/s?id=1589175554246101619&wfr=spider&for=pc

HA集群部署规划
主机名称	IP地址	用户名称	进程	安装的软件
node200	192.168.33.200	hadoop	NameNode（Active）、ResourceManager（StandBy）、ZKFC、JobHistoryServer	JDK、Hadoop
node201	192.168.33.201	hadoop	NameNode（StandBy）、ResourceManager（Active）、ZKFC、WebProxyServer	JDK、Hadoop
node202	192.168.33.202	hadoop	DataNode、NodeManager、JournalNode、QuorumPeerMain	JDK、Hadoop、Zookeeper
node203	192.168.33.203	hadoop	DataNode、NodeManager、JournalNode、QuorumPeerMain	JDK、Hadoop、Zookeeper
node204	192.168.33.204	hadoop	DataNode、NodeManager、JournalNode、QuorumPeerMain	JDK、Hadoop、Zookeeper
规划说明：

HDFS HA通常由两个NameNode组成，一个处于Active状态，另一个处于Standby状态。

Active NameNode对外提供服务，而Standby NameNode则不对外提供服务，仅同步Active NameNode的状态，以便能够在它失败时快速进行切换。

Hadoop 2.0官方提供了两种HDFS HA的解决方案，一种是NFS，另一种是QJM。这里我们使用简单的QJM。在该方案中，主备NameNode之间通过一组JournalNode同步元数据信息，一条数据只要成功写入多数JournalNode即认为写入成功。通常配置奇数个JournalNode，这里还配置了一个Zookeeper集群，用于ZKFC故障转移，当Active NameNode挂掉了，会自动切换Standby NameNode为Active状态。

YARN的ResourceManager也存在单点故障问题，这个问题在hadoop-2.4.1得到了解决：有两个ResourceManager，一个是Active，一个是Standby，状态由Zookeeper进行协调。

YARN框架下的MapReduce可以开启JobHistoryServer来记录历史任务信息，否则只能查看当前正在执行的任务信息。

Zookeeper的作用是负责HDFS中NameNode主备节点的选举，和YARN框架下ResourceManaer主备节点的选举。

部分内容来自：https://www.linuxidc.com/Linux/2016-08/134180.htm

自动故障转移
Zookeeper集群作用：

一是故障监控。每个NameNode将会和Zookeeper建立一个持久session，如果NameNode失效，那么此session将会过期失效，此后Zookeeper将会通知另一个Namenode，然后触发Failover；

二是NameNode选举。ZooKeeper提供了简单的机制来实现Acitve Node选举，如果当前Active失效，Standby将会获取一个特定的排他锁，那么获取锁的Node接下来将会成为Active。

ZKFC：

ZKFC是一个Zookeeper的客户端，它主要用来监测和管理NameNodes的状态，每个NameNode机器上都会运行一个ZKFC程序

主要职责：

一是健康监控。ZKFC间歇性的ping NameNode，得到NameNode返回状态，如果NameNode失效或者不健康，那么ZKFS将会标记其为不健康；

二是Zookeeper会话管理。当本地NaneNode运行良好时，ZKFC将会持有一个Zookeeper session，如果本地NameNode为Active，它同时也持有一个“排他锁”znode，如果session过期，那么次lock所对应的znode也将被删除；

三是选举。当集群中其中一个NameNode宕机，Zookeeper会自动将另一个激活。

此处内容来自：https://www.linuxidc.com/Linux/2016-08/134180.htm

关于集群主机时间
因为高可用集群的机制，各主机在集群中的时间需一致。

在下面Linux搭建前将虚拟机进行设置，设置方法如下：



安装完成后对每台主机的时间进行确认，确保每台主机时间一致。

Linux环境搭建
按如下方法部署五台主机，主机名与IP地址的对应关系见上文集群部署规划

VMware虚拟机安装Linux系统

配置完成之后各主机IP、主机名与时间信息如下：（时间不一致的自己百度同步集群时间的方法）

命令：

#查看系统ip信息
ip a
#查看系统时间
date
执行结果： 

[root@node200 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6a:0e:74 brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.200/24 brd 192.168.33.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe6a:e74/64 scope link 
       valid_lft forever preferred_lft forever
[root@node200 ~]# date
2018年 10月 11日 星期四 16:57:56 CST
[root@node201 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:60:36:3c brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.201/24 brd 192.168.33.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe60:363c/64 scope link 
       valid_lft forever preferred_lft forever
[root@node201 ~]# date
2018年 10月 11日 星期四 16:57:48 CST
[root@node202 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:41:4a:f4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.202/24 brd 192.168.33.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe41:4af4/64 scope link 
       valid_lft forever preferred_lft forever
[root@node202 ~]# date
2018年 10月 11日 星期四 16:58:00 CST
[root@node203 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:53:27:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.203/24 brd 192.168.33.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe53:2740/64 scope link 
       valid_lft forever preferred_lft forever
[root@node203 ~]# date
2018年 10月 11日 星期四 16:58:01 CST
[root@node204 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6e:ad:fa brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.204/24 brd 192.168.33.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe6e:adfa/64 scope link 
       valid_lft forever preferred_lft forever
[root@node204 ~]# date
2018年 10月 11日 星期四 16:57:54 CST
网络测试：

配置完成之后测试各主机网络互通情况，在每台主机上执行下面两条命令，运行过程中按Ctrl+C可以终止进程，下面就不贴测试效果了

ping 192.168.33.1
ping www.baidu.com
配置Java环境
node200、node201、node202、node203、node204都需要安装

为上面安装的系统配置Java环境变量，本文中就写关键配置步骤与执行命令了，想了解详细的配置过程可以查看：

Linux系统下安装Java环境

为了方便，本文就直接使用rpm包安装了，/etc/profile文件暂时不进行配置，到后面配置hadoop单机版时再进行配置

[1-3]均使用root用户执行

1、将安装包jdk-8u181-linux-x64.rpm上传到/usr/local目录下

2、安装rpm包，先设置权限，然后执行rpm命令安装

chmod 755 /usr/local/jdk-8u181-linux-x64.rpm
rpm -ivh /usr/local/jdk-8u181-linux-x64.rpm
3、校验安装情况

java -version
安装单机版Hadoop
node200、node201、node202、node203、node204都需要安装

详细步骤查看：CentOS 7部署Hadoop（单机版），这里只简单介绍安装步骤

[1-5]均使用root用户执行

1、将压缩包hadoop-2.7.3.tar.gz上传到/usr/local目录下

2、解压压缩包，进入/usr/local目录，对文件夹重命名

tar -zxvf /usr/local/hadoop-2.7.3.tar.gz
cd /usr/local
mv hadoop-2.7.3 hadoop
3、创建hadoop用户和hadoop用户组，并设置hadoop用户密码

useradd hadoop
passwd hadoop
4、为hadoop用户添加sudo权限

vi /etc/sudoers
 在root用户下面一行加上hadoop  ALL=(ALL)       ALL，保存并退出（这里需要用wq!强制保存退出）

## Next comes the main part: which users can run what software on
## which machines (the sudoers file can be shared between multiple
## systems).
## Syntax:
##
## user MACHINE=COMMANDS
##
## The COMMANDS section may have other options added to it.
##
## Allow root to run any commands anywhere
root    ALL=(ALL)    ALL
hadoop    ALL=(ALL)    ALL
5、将hadoop文件夹的主：组设置成hadoop，/usr目录与/usr/local目录所属主：组均为root，默认权限为755，也就是说其他用户（hadoop）没有写入（w）权限，在这里我们需要将这两个目录其他用户的权限设置为7。在这里将hadoop用户加入root组，然后将文件夹权限设置为775

###############################
#chown -R hadoop:hadoop hadoop
#chmod 757 /usr
#chmod 757 /usr/local
###############################
 
使用下面代码：
 
gpasswd -a hadoop root
chmod 771 /usr
chmod 771 /usr/local
chown -R hadoop:hadoop /usr/local/hadoop
Zookeeper集群安装
在node202、node203、node204上安装

Zookeeper是一个开源分布式协调服务，其独特的Leader-Follower集群结构，很好的解决了分布式单点问题。目前主要用于诸如：统一命名服务、配置管理、锁服务、集群管理等场景。大数据应用中主要使用Zookeeper的集群管理功能。

[1-5]均使用root用户执行

1、将压缩包zookeeper-3.4.10.tar.gz上传到node202的/usr/local目录下

2、解压压缩包，进入/usr/local目录，对文件夹重命名

tar -zxvf /usr/local/zookeeper-3.4.10.tar.gz
cd /usr/local
mv zookeeper-3.4.10 zookeeper
3、修改zookeeper的配置文件，命令如下：

cd /usr/local/zookeeper/conf/
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
配置文件如下：

# 客户端心跳时间(毫秒)
tickTime=2000
# 允许心跳间隔的最大时间
initLimit=10
# 同步时限
syncLimit=5
# 数据存储目录
dataDir=/usr/local/zookeeperdata
# 数据日志存储目录
dataLogDir=/usr/local/tmp/zookeeperlogs
# 端口号
clientPort=2181
# 集群节点和服务端口配置
server.1=node202:2888:3888
server.2=node203:2888:3888
server.3=node204:2888:3888
# 以下为优化配置
# 服务器最大连接数，默认为10，改为0表示无限制
maxClientCnxns=0
# 快照数
autopurge.snapRetainCount=3
# 快照清理时间，默认为0
autopurge.purgeInterval=1
 3、修改完zookeeper的配置文件，将node202上的zookeeper上传到其他两台服务器：

[root@node202 conf]# scp -r /usr/local/zookeeper 192.168.33.203:/usr/local/
[root@node202 conf]# scp -r /usr/local/zookeeper 192.168.33.204:/usr/local/
 
#这儿输入yes回车
Are you sure you want to continue connecting (yes/no)? yes
#这里输入远程主机的root密码
root@192.168.33.204's password:
4、在三台主机上创建 zookeeper数据存储目录：

mkdir /usr/local/zookeeperdata
在文件夹下面创建一个文件，叫myid，并且在文件里写入server.X对应的X

# 集群节点和服务端口配置
server.1=node202:2888:3888
server.2=node203:2888:3888
server.3=node204:2888:3888
上述配置中node202是1，node203是2，node204是3，所以按如下操作：

在node202 上执行：

echo "1" > /usr/local/zookeeperdata/myid
在node203 上执行： 

echo "2" > /usr/local/zookeeperdata/myid
在node204 上执行： 

echo "3" > /usr/local/zookeeperdata/myid
5、将zookeeper文件夹的主：组设置成hadoop

chown -R hadoop:hadoop /usr/local/zookeeper
chown -R hadoop:hadoop /usr/local/zookeeperdata
配置环境变量
node200、node201、node202、node203、node204都需要配置，第2步有所区别

[1-3]均使用root用户执行

1、编辑/etc/profile文件

vi /etc/profile
2、在末尾加上如下代码

node200、node201添加如下代码：

export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JRE_HOME=$JAVA_HOME/jre
node202、node203、node204添加如下代码：

export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
export HADOOP_HOME=/usr/local/hadoop
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JRE_HOME=$JAVA_HOME/jre
3、配置完环境变量之后保存退出，让环境变量立即生效

source /etc/profile
关闭防火墙
node200、node201、node202、node203、node204都需要配置

CentOS 7 使用的是firewalld作为防火墙，与CentOS 6 有所不同

下面三步均使用root用户执行

查看防火墙状态：

systemctl status firewalld
关闭防火墙：

systemctl stop firewalld
关闭防火墙开机自动启动：

systemctl disable firewalld
 更多详情可以了解：CentOS 7部署Hadoop（伪分布式）

修改hosts文件
修改所有主机的/etc/hosts文件，这里使用root用户操作

vi /etc/hosts
在文件最后面加上：

192.168.33.200    node200
192.168.33.201    node201
192.168.33.202    node202
192.168.33.203    node203
192.168.33.204    node204
注意：IP后面是Tab制表符，而不是空格，这里配置完后最好测试一下网络。如果复制粘贴配置完后无法ping通，可能是IP地址后面的空格问题。
 

以下为测试方法：

ping node200
ping node201
ping node202
ping node203
ping node204
 配置到这里，我们切换到hadoop用户，使用hadoop用户进行下面的操作

配置SSH免密登录
具体方法参照：

Linux系统配置SSH免密登录(多主机互通)

这里贴关键代码，不展示操作过程：

在node200、node201、node202、node203、node204上分别操作，一路回车就可以了：

[hadoop@node200 ~]$ ssh-keygen -t rsa
[hadoop@node201 ~]$ ssh-keygen -t rsa
[hadoop@node202 ~]$ ssh-keygen -t rsa
[hadoop@node203 ~]$ ssh-keygen -t rsa
[hadoop@node204 ~]$ ssh-keygen -t rsa
在node200上操作：

[hadoop@node200 ~]$ ssh-copy-id localhost
[hadoop@node200 ~]$ scp ~/.ssh/* node201:~/.ssh
[hadoop@node200 ~]$ scp ~/.ssh/* node202:~/.ssh
[hadoop@node200 ~]$ scp ~/.ssh/* node203:~/.ssh
[hadoop@node200 ~]$ scp ~/.ssh/* node204:~/.ssh
修改Hadoop配置文件
均使用hadoop用户操作，只需要在node200上修改即可

先进入/usr/local/hadoop/etc/hadoop/文件

[hadoop@node200 ~]$ cd /usr/local/hadoop/etc/hadoop/
1、修改hadoop-env.sh文件

[hadoop@node200 hadoop]$ vi hadoop-env.sh
找到export JAVA_HOME=${JAVA_HOME}，在前面加个#注释掉，将JAVA_HOME用路径代替，如下：

#export JAVA_HOME=${JAVA_HOME}
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
2、修改core-site.xml文件

[hadoop@node200 hadoop]$ vi core-site.xml
配置文件如下：

<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
 
<!-- Put site-specific property overrides in this file. -->
 
<configuration>
        <!-- 指定hdfs的nameservices名称为mycluster，与hdfs-site.xml的HA配置相同 -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://mycluster</value>
        </property>
 
        <!-- 指定缓存文件存储的路径 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/usr/local/tmp/hadoop/data</value>
        </property>
 
        <!-- 配置hdfs文件被永久删除前保留的时间（单位：分钟），默认值为0表明垃圾回收站功能关闭 -->
        <property>
                <name>fs.trash.interval</name>
                <value>1440</value>
        </property>
 
        <!-- 指定zookeeper地址，配置HA时需要 -->
        <property>
                <name>ha.zookeeper.quorum</name>
                <value>node202:2181,node203:2181,node204:2181</value>
        </property>
</configuration>
 3、修改hdfs-site.xml

[hadoop@node200 hadoop]$ vi hdfs-site.xml
 配置文件如下：

<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
 
<!-- Put site-specific property overrides in this file. -->
 
<configuration>
	<!-- 指定hdfs元数据存储的路径 -->
	<property>
		<name>dfs.namenode.name.dir</name>
		<value>/usr/local/hadoopdata/namenode</value>
	</property>
 
	<!-- 指定hdfs数据存储的路径 -->
	<property>
		<name>dfs.datanode.data.dir</name>
		<value>/usr/local/hadoopdata/datanode</value>
	</property>
 
	<!-- 数据备份的个数 -->
	<property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>
 
	<!-- 关闭权限验证 -->
	<property>
		<name>dfs.permissions.enabled</name>
		<value>false</value>
	</property>
 
	<!-- 开启WebHDFS功能（基于REST的接口服务） -->
	<property>
		<name>dfs.webhdfs.enabled</name>
		<value>true</value>
	</property>
 
	<!-- //以下为HDFS HA的配置// -->
	<!-- 指定hdfs的nameservices名称为mycluster -->
	<property>
		<name>dfs.nameservices</name>
		<value>mycluster</value>
	</property>
 
	<!-- 指定mycluster的两个namenode的名称分别为nn1,nn2 -->
	<property>
		<name>dfs.ha.namenodes.mycluster</name>
		<value>nn1,nn2</value>
	</property>
 
	<!-- 配置nn1,nn2的rpc通信端口 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn1</name>
		<value>node200:9000</value>
	</property>
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn2</name>
		<value>node201:9000</value>
	</property>
 
	<!-- 配置nn1,nn2的http通信端口 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn1</name>
		<value>node200:50070</value>
	</property>
	<property>
		<name>dfs.namenode.http-address.mycluster.nn2</name>
		<value>node201:50070</value>
	</property>
 
	<!-- 指定namenode元数据存储在journalnode中的路径 -->
	<property>
		<name>dfs.namenode.shared.edits.dir</name>
		<value>qjournal://node202:8485;node203:8485;node204:8485/mycluster</value>
	</property>
 
	<!-- 指定journalnode日志文件存储的路径 -->
	<property>
		<name>dfs.journalnode.edits.dir</name>
		<value>/usr/local/tmp/journalnodelogs</value>
	</property>
 
	<!-- 指定HDFS客户端连接active namenode的java类 -->
	<property>
		<name>dfs.client.failover.proxy.provider.mycluster</name>
		<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
 
	<!-- 配置隔离机制为ssh -->
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>sshfence</value>
	</property>
 
	<!-- 指定秘钥的位置 -->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/home/hadoop/.ssh/id_rsa</value>
	</property>
 
	<!-- 开启自动故障转移 -->
	<property>
		<name>dfs.ha.automatic-failover.enabled</name>
		<value>true</value>
	</property>
</configuration>
4、修改mapred-site.xml

/usr/local/hadoop/etc/hadoop文件夹中并没有mapred-site.xml文件，但提供了模板mapred-site.xml.template，将其复制一份重命名为mapred-site.xml 即可

[hadoop@node200 hadoop]$ cp mapred-site.xml.template mapred-site.xml
[hadoop@node200 hadoop]$ vi mapred-site.xml
配置文件如下： 

<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
 
<!-- Put site-specific property overrides in this file. -->
 
<configuration>
	<!-- 指定MapReduce计算框架使用YARN -->
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
 
	<!-- 指定jobhistory server的rpc地址 -->
	<property>
		<name>mapreduce.jobhistory.address</name>
		<value>node200:10020</value>
	</property>
 
	<!-- 指定jobhistory server的http地址 -->
	<property>
		<name>mapreduce.jobhistory.webapp.address</name>
		<value>node200:19888</value>
	</property>
 
	<!-- 开启uber模式（针对小作业的优化） -->
	<property>
		<name>mapreduce.job.ubertask.enable</name>
		<value>true</value>
	</property>
 
	<!-- 配置启动uber模式的最大map数 -->
	<property>
		<name>mapreduce.job.ubertask.maxmaps</name>
		<value>9</value>
	</property>
 
	<!-- 配置启动uber模式的最大reduce数 -->
	<property>
		<name>mapreduce.job.ubertask.maxreduces</name>
		<value>5</value>
	</property>
</configuration>
5、修改yarn-site.xml

[hadoop@master200 hadoop]$ vi yarn-site.xml
配置文件如下： 

<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
 
<configuration>
 
<!-- Site specific YARN configuration properties -->
 
	<!-- NodeManager上运行的附属服务，需配置成mapreduce_shuffle才可运行MapReduce程序 -->
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
 
	<!-- 配置Web Application Proxy安全代理（防止yarn被攻击） -->
	<property>
		<name>yarn.web-proxy.address</name>
		<value>node201:8888</value>
	</property>
 
	<!-- 开启日志 -->
	<property>
		<name>yarn.log-aggregation-enable</name>
		<value>true</value>
	</property>
 
	<!-- 配置日志删除时间为7天，-1为禁用，单位为秒 -->
	<property>
		<name>yarn.log-aggregation.retain-seconds</name>
		<value>604800</value>
	</property>
 
	<!-- 修改日志目录 -->
	<property>
		<name>yarn.nodemanager.remote-app-log-dir</name>
		<value>/usr/local/tmp/hadooplogs</value>
	</property>
 	
<!--配置nodemanager可用的资源内存 
	<property>
		<name>yarn.nodemanager.resource.memory-mb</name>
		<value>2048</value>
	</property>
	配置nodemanager可用的资源CPU 
	<property>
		<name>yarn.nodemanager.resource.cpu-vcores</name>
		<value>2</value>
	</property> 
 -->
 
	<!-- //以下为YARN HA的配置// -->
	<!-- 开启YARN HA -->
	<property>
		<name>yarn.resourcemanager.ha.enabled</name>
		<value>true</value>
	</property>
 
	<!-- 启用自动故障转移 -->
	<property>
		<name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
		<value>true</value>
	</property>
 
	<!-- 指定YARN HA的名称 -->
	<property>
		<name>yarn.resourcemanager.cluster-id</name>
		<value>yarncluster</value>
	</property>
 
	<!-- 指定两个resourcemanager的名称 -->
	<property>
		<name>yarn.resourcemanager.ha.rm-ids</name>
		<value>rm1,rm2</value>
	</property>
 
	<!-- 配置rm1，rm2的主机 -->
	<property>
		<name>yarn.resourcemanager.hostname.rm1</name>
		<value>node201</value>
	</property>
	<property>
		<name>yarn.resourcemanager.hostname.rm2</name>
		<value>node200</value>
	</property>
 
	<!-- 配置YARN的http端口 -->
	<property>
		<name>yarn.resourcemanager.webapp.address.rm1</name>
		<value>node201:8088</value>
	</property>	
	<property>
		<name>yarn.resourcemanager.webapp.address.rm2</name>
		<value>node200:8088</value>
	</property>
 
	<!-- 配置zookeeper的地址 -->
	<property>
		<name>yarn.resourcemanager.zk-address</name>
		<value>node202:2181,node203:2181,node204:2181</value>
	</property>
 
	<!-- 配置zookeeper的存储位置 -->
	<property>
		<name>yarn.resourcemanager.zk-state-store.parent-path</name>
		<value>/usr/local/zookeeperdata/rmstore</value>
	</property>
 
	<!-- 开启yarn resourcemanager restart -->
	<property>
		<name>yarn.resourcemanager.recovery.enabled</name>
		<value>true</value>
	</property>
 
	<!-- 配置resourcemanager的状态存储到zookeeper中 -->
	<property>
		<name>yarn.resourcemanager.store.class</name>
		<value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
	</property>
 
	<!-- 开启yarn nodemanager restart -->
	<property>
		<name>yarn.nodemanager.recovery.enabled</name>
		<value>true</value>
	</property>
 
	<!-- 配置nodemanager IPC的通信端口 -->
	<property>
		<name>yarn.nodemanager.address</name>
		<value>0.0.0.0:45454</value>
	</property>
</configuration>
6、修改slaves文件

[hadoop@node200 hadoop]$ vi slaves
配置文件如下： 

node202
node203
node204
上述关键的配置文件我已经上传至：https://github.com/PengShuaixin/hadoop-2.7.3_centos7

可以直接下载下来，通过上传到Linux直接覆盖原来文件的方式进行配置

7、通过scp将配置文件上传到其他主机

[hadoop@node200 hadoop]$ scp -r /usr/local/hadoop/etc/hadoop/* node201:/usr/local/hadoop/etc/hadoop
[hadoop@node200 hadoop]$ scp -r /usr/local/hadoop/etc/hadoop/* node202:/usr/local/hadoop/etc/hadoop
[hadoop@node200 hadoop]$ scp -r /usr/local/hadoop/etc/hadoop/* node203:/usr/local/hadoop/etc/hadoop
[hadoop@node200 hadoop]$ scp -r /usr/local/hadoop/etc/hadoop/* node204:/usr/local/hadoop/etc/hadoop
Hadoop集群的初始化
1、启动zookeeper集群（分别在node202、node203和node204上执行）

zkServer.sh start
ps：zookeeper其他命令

#查看状态
zkServer.sh status
#关闭
zkServer.sh stop
启动后查看状态如下：

#node202
[hadoop@node202 ~]$ zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: follower
 
#node203
[hadoop@node203 ~]$ zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: leader
 
#node204
[hadoop@node204 ~]$ zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: follower
 
#只有一台主机为leader，剩下的都是follower
#zookeeper配置的主机数一般为2n+1台，且最少需要3台
2、格式化ZKFC（在node200上执行）

[hadoop@node200 ~]$ hdfs zkfc -formatZK
出现如下代码说明执行成功：

18/10/12 09:19:11 INFO tools.DFSZKFailoverController: Failover controller configured for NameNode NameNode at node200/192.168.33.200:9000
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:zookeeper.version=3.4.6-1569965, built on 02/20/2014 09:09 GMT
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:host.name=node200
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.version=1.8.0_181
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.vendor=Oracle Corporation
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.home=/usr/java/jdk1.8.0_181-amd64/jre
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.class.path=/usr/local/hadoop/etc/hadoop:/usr/local/hadoop/share/hadoop/common/lib/jaxb-impl-2.2.3-1.jar:/usr/local/hadoop/share/hadoop/common/lib/jaxb-api-2.2.2.jar:/usr/local/hadoop/share/hadoop/common/lib/stax-api-1.0-2.jar:/usr/local/hadoop/share/hadoop/common/lib/activation-1.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-jaxrs-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-xc-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/common/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/common/lib/jets3t-0.9.0.jar:/usr/local/hadoop/share/hadoop/common/lib/httpclient-4.2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/httpcore-4.2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/java-xmlbuilder-0.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-configuration-1.6.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-digester-1.8.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-beanutils-1.7.0.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-beanutils-core-1.8.0.jar:/usr/local/hadoop/share/hadoop/common/lib/slf4j-api-1.7.10.jar:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar:/usr/local/hadoop/share/hadoop/common/lib/avro-1.7.4.jar:/usr/local/hadoop/share/hadoop/common/lib/paranamer-2.3.jar:/usr/local/hadoop/share/hadoop/common/lib/snappy-java-1.0.4.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/common/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/common/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/common/lib/gson-2.2.4.jar:/usr/local/hadoop/share/hadoop/common/lib/hadoop-auth-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/local/hadoop/share/hadoop/common/lib/apacheds-i18n-2.0.0-M15.jar:/usr/local/hadoop/share/hadoop/common/lib/api-asn1-api-1.0.0-M20.jar:/usr/local/hadoop/share/hadoop/common/lib/api-util-1.0.0-M20.jar:/usr/local/hadoop/share/hadoop/common/lib/zookeeper-3.4.6.jar:/usr/local/hadoop/share/hadoop/common/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-framework-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-client-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jsch-0.1.42.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-recipes-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/htrace-core-3.1.0-incubating.jar:/usr/local/hadoop/share/hadoop/common/lib/junit-4.11.jar:/usr/local/hadoop/share/hadoop/common/lib/hamcrest-core-1.3.jar:/usr/local/hadoop/share/hadoop/common/lib/mockito-all-1.8.5.jar:/usr/local/hadoop/share/hadoop/common/lib/hadoop-annotations-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/common/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-math3-3.1.1.jar:/usr/local/hadoop/share/hadoop/common/lib/xmlenc-0.52.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-httpclient-3.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-net-3.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-collections-3.2.2.jar:/usr/local/hadoop/share/hadoop/common/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/common/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/common/lib/jsp-api-2.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-json-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/jettison-1.1.jar:/usr/local/hadoop/share/hadoop/common/hadoop-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/hadoop-common-2.7.3-tests.jar:/usr/local/hadoop/share/hadoop/common/hadoop-nfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/hdfs:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xmlenc-0.52.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/htrace-core-3.1.0-incubating.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-daemon-1.0.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/netty-all-4.0.23.Final.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xercesImpl-2.9.1.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xml-apis-1.3.04.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-2.7.3-tests.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-nfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/lib/zookeeper-3.4.6-tests.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/yarn/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jaxb-api-2.2.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/stax-api-1.0-2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/activation-1.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-client-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-xc-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guice-servlet-3.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guice-3.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/javax.inject-1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/aopalliance-1.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-json-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jettison-1.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-guice-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/zookeeper-3.4.6.jar:/usr/local/hadoop/share/hadoop/yarn/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/yarn/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-collections-3.2.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-api-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-nodemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-web-proxy-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-applicationhistoryservice-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-resourcemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-tests-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-client-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-sharedcachemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-applications-unmanaged-am-launcher-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-registry-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/avro-1.7.4.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/paranamer-2.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/snappy-java-1.0.4.1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/hadoop-annotations-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/guice-3.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/javax.inject-1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/aopalliance-1.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-guice-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/guice-servlet-3.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/junit-4.11.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/hamcrest-core-1.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-shuffle-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-app-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-plugins-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3-tests.jar:/usr/local/hadoop/contrib/capacity-scheduler/*.jar
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.library.path=/usr/local/hadoop/lib/native
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.io.tmpdir=/tmp
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:java.compiler=<NA>
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:os.name=Linux
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:os.arch=amd64
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:os.version=3.10.0-862.el7.x86_64
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:user.name=hadoop
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:user.home=/home/hadoop
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Client environment:user.dir=/home/hadoop
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Initiating client connection, connectString=node202:2181,node203:2181,node204:2181 sessionTimeout=5000 watcher=org.apache.hadoop.ha.ActiveStandbyElector$WatcherWithClientRef@57a3af25
18/10/12 09:19:12 INFO zookeeper.ClientCnxn: Opening socket connection to server node202/192.168.33.202:2181. Will not attempt to authenticate using SASL (unknown error)
18/10/12 09:19:12 INFO zookeeper.ClientCnxn: Socket connection established to node202/192.168.33.202:2181, initiating session
18/10/12 09:19:12 INFO zookeeper.ClientCnxn: Session establishment complete on server node202/192.168.33.202:2181, sessionid = 0x16665c138780001, negotiated timeout = 5000
18/10/12 09:19:12 INFO ha.ActiveStandbyElector: Successfully created /hadoop-ha/mycluster in ZK.
18/10/12 09:19:12 INFO zookeeper.ZooKeeper: Session: 0x16665c138780001 closed
18/10/12 09:19:12 WARN ha.ActiveStandbyElector: Ignoring stale result from old client with sessionId 0x16665c138780001
18/10/12 09:19:12 INFO zookeeper.ClientCnxn: EventThread shut down
3、启动journalnode（分别在node202、node203和node204上执行）

hadoop-daemon.sh start journalnode
执行过程如下：

[hadoop@node202 ~]$ hadoop-daemon.sh start journalnode
starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node202.out
[hadoop@node202 ~]$ jps
1508 QuorumPeerMain
3239 Jps
3182 JournalNode
 
[hadoop@node203 ~]$ hadoop-daemon.sh start journalnode
starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node203.out
[hadoop@node203 ~]$ jps
3289 Jps
1484 QuorumPeerMain
3231 JournalNode
 
[hadoop@node204 ~]$ hadoop-daemon.sh start journalnode
starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node204.out
[hadoop@node204 ~]$ jps
3296 Jps
1490 QuorumPeerMain
3237 JournalNode
 
#这里用jps命令确认JournalNode进程启动情况
#QuorumPeerMain是第1步操作启动的zookeeper的进程
4、格式化HDFS（在node200上执行）

[hadoop@node200 ~]$ hdfs namenode -format
执行出现如下代码说明执行成功：

[hadoop@node200 ~]$ hdfs namenode -format
18/10/12 09:36:22 INFO namenode.NameNode: STARTUP_MSG: 
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = node200/192.168.33.200
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.7.3
STARTUP_MSG:   classpath = /usr/local/hadoop/etc/hadoop:/usr/local/hadoop/share/hadoop/common/lib/jaxb-impl-2.2.3-1.jar:/usr/local/hadoop/share/hadoop/common/lib/jaxb-api-2.2.2.jar:/usr/local/hadoop/share/hadoop/common/lib/stax-api-1.0-2.jar:/usr/local/hadoop/share/hadoop/common/lib/activation-1.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-jaxrs-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jackson-xc-1.9.13.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/common/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/common/lib/jets3t-0.9.0.jar:/usr/local/hadoop/share/hadoop/common/lib/httpclient-4.2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/httpcore-4.2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/java-xmlbuilder-0.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-configuration-1.6.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-digester-1.8.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-beanutils-1.7.0.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-beanutils-core-1.8.0.jar:/usr/local/hadoop/share/hadoop/common/lib/slf4j-api-1.7.10.jar:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar:/usr/local/hadoop/share/hadoop/common/lib/avro-1.7.4.jar:/usr/local/hadoop/share/hadoop/common/lib/paranamer-2.3.jar:/usr/local/hadoop/share/hadoop/common/lib/snappy-java-1.0.4.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/common/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/common/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/common/lib/gson-2.2.4.jar:/usr/local/hadoop/share/hadoop/common/lib/hadoop-auth-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/local/hadoop/share/hadoop/common/lib/apacheds-i18n-2.0.0-M15.jar:/usr/local/hadoop/share/hadoop/common/lib/api-asn1-api-1.0.0-M20.jar:/usr/local/hadoop/share/hadoop/common/lib/api-util-1.0.0-M20.jar:/usr/local/hadoop/share/hadoop/common/lib/zookeeper-3.4.6.jar:/usr/local/hadoop/share/hadoop/common/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-framework-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-client-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jsch-0.1.42.jar:/usr/local/hadoop/share/hadoop/common/lib/curator-recipes-2.7.1.jar:/usr/local/hadoop/share/hadoop/common/lib/htrace-core-3.1.0-incubating.jar:/usr/local/hadoop/share/hadoop/common/lib/junit-4.11.jar:/usr/local/hadoop/share/hadoop/common/lib/hamcrest-core-1.3.jar:/usr/local/hadoop/share/hadoop/common/lib/mockito-all-1.8.5.jar:/usr/local/hadoop/share/hadoop/common/lib/hadoop-annotations-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/common/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-math3-3.1.1.jar:/usr/local/hadoop/share/hadoop/common/lib/xmlenc-0.52.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-httpclient-3.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-net-3.1.jar:/usr/local/hadoop/share/hadoop/common/lib/commons-collections-3.2.2.jar:/usr/local/hadoop/share/hadoop/common/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/common/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/common/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/common/lib/jsp-api-2.1.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/jersey-json-1.9.jar:/usr/local/hadoop/share/hadoop/common/lib/jettison-1.1.jar:/usr/local/hadoop/share/hadoop/common/hadoop-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/common/hadoop-common-2.7.3-tests.jar:/usr/local/hadoop/share/hadoop/common/hadoop-nfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/hdfs:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xmlenc-0.52.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/htrace-core-3.1.0-incubating.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/commons-daemon-1.0.13.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/netty-all-4.0.23.Final.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xercesImpl-2.9.1.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/xml-apis-1.3.04.jar:/usr/local/hadoop/share/hadoop/hdfs/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-2.7.3-tests.jar:/usr/local/hadoop/share/hadoop/hdfs/hadoop-hdfs-nfs-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/lib/zookeeper-3.4.6-tests.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-lang-2.6.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guava-11.0.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jsr305-3.0.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-logging-1.1.3.jar:/usr/local/hadoop/share/hadoop/yarn/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-cli-1.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jaxb-api-2.2.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/stax-api-1.0-2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/activation-1.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/servlet-api-2.5.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-codec-1.4.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jetty-util-6.1.26.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-client-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jackson-xc-1.9.13.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guice-servlet-3.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/guice-3.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/javax.inject-1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/aopalliance-1.0.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-json-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jettison-1.1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jersey-guice-1.9.jar:/usr/local/hadoop/share/hadoop/yarn/lib/zookeeper-3.4.6.jar:/usr/local/hadoop/share/hadoop/yarn/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/yarn/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/yarn/lib/commons-collections-3.2.2.jar:/usr/local/hadoop/share/hadoop/yarn/lib/jetty-6.1.26.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-api-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-nodemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-web-proxy-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-applicationhistoryservice-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-resourcemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-tests-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-client-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-server-sharedcachemanager-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-applications-unmanaged-am-launcher-2.7.3.jar:/usr/local/hadoop/share/hadoop/yarn/hadoop-yarn-registry-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/protobuf-java-2.5.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/avro-1.7.4.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/paranamer-2.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/snappy-java-1.0.4.1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/commons-compress-1.4.1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/xz-1.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/hadoop-annotations-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/commons-io-2.4.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-core-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-server-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/asm-3.2.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/log4j-1.2.17.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/netty-3.6.2.Final.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/leveldbjni-all-1.8.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/guice-3.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/javax.inject-1.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/aopalliance-1.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/jersey-guice-1.9.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/guice-servlet-3.0.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/junit-4.11.jar:/usr/local/hadoop/share/hadoop/mapreduce/lib/hamcrest-core-1.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-shuffle-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-app-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-plugins-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar:/usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3-tests.jar:/usr/local/hadoop/contrib/capacity-scheduler/*.jar
STARTUP_MSG:   build = https://git-wip-us.apache.org/repos/asf/hadoop.git -r baa91f7c6bc9cb92be5982de4719c1c8af91ccff; compiled by 'root' on 2016-08-18T01:41Z
STARTUP_MSG:   java = 1.8.0_181
************************************************************/
18/10/12 09:36:22 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT]
18/10/12 09:36:22 INFO namenode.NameNode: createNameNode [-format]
18/10/12 09:36:23 WARN common.Util: Path /usr/local/hadoopdata/namenode should be specified as a URI in configuration files. Please update hdfs configuration.
18/10/12 09:36:23 WARN common.Util: Path /usr/local/hadoopdata/namenode should be specified as a URI in configuration files. Please update hdfs configuration.
Formatting using clusterid: CID-882887d1-b39c-419c-8945-f9c753f516ae
18/10/12 09:36:23 INFO namenode.FSNamesystem: No KeyProvider found.
18/10/12 09:36:23 INFO namenode.FSNamesystem: fsLock is fair:true
18/10/12 09:36:23 INFO blockmanagement.DatanodeManager: dfs.block.invalidate.limit=1000
18/10/12 09:36:23 INFO blockmanagement.DatanodeManager: dfs.namenode.datanode.registration.ip-hostname-check=true
18/10/12 09:36:23 INFO blockmanagement.BlockManager: dfs.namenode.startup.delay.block.deletion.sec is set to 000:00:00:00.000
18/10/12 09:36:23 INFO blockmanagement.BlockManager: The block deletion will start around 2018 十月 12 09:36:23
18/10/12 09:36:23 INFO util.GSet: Computing capacity for map BlocksMap
18/10/12 09:36:23 INFO util.GSet: VM type       = 64-bit
18/10/12 09:36:23 INFO util.GSet: 2.0% max memory 966.7 MB = 19.3 MB
18/10/12 09:36:23 INFO util.GSet: capacity      = 2^21 = 2097152 entries
18/10/12 09:36:23 INFO blockmanagement.BlockManager: dfs.block.access.token.enable=false
18/10/12 09:36:23 INFO blockmanagement.BlockManager: defaultReplication         = 3
18/10/12 09:36:23 INFO blockmanagement.BlockManager: maxReplication             = 512
18/10/12 09:36:23 INFO blockmanagement.BlockManager: minReplication             = 1
18/10/12 09:36:23 INFO blockmanagement.BlockManager: maxReplicationStreams      = 2
18/10/12 09:36:23 INFO blockmanagement.BlockManager: replicationRecheckInterval = 3000
18/10/12 09:36:23 INFO blockmanagement.BlockManager: encryptDataTransfer        = false
18/10/12 09:36:23 INFO blockmanagement.BlockManager: maxNumBlocksToLog          = 1000
18/10/12 09:36:23 INFO namenode.FSNamesystem: fsOwner             = hadoop (auth:SIMPLE)
18/10/12 09:36:23 INFO namenode.FSNamesystem: supergroup          = supergroup
18/10/12 09:36:23 INFO namenode.FSNamesystem: isPermissionEnabled = false
18/10/12 09:36:23 INFO namenode.FSNamesystem: Determined nameservice ID: mycluster
18/10/12 09:36:23 INFO namenode.FSNamesystem: HA Enabled: true
18/10/12 09:36:23 INFO namenode.FSNamesystem: Append Enabled: true
18/10/12 09:36:23 INFO util.GSet: Computing capacity for map INodeMap
18/10/12 09:36:23 INFO util.GSet: VM type       = 64-bit
18/10/12 09:36:23 INFO util.GSet: 1.0% max memory 966.7 MB = 9.7 MB
18/10/12 09:36:23 INFO util.GSet: capacity      = 2^20 = 1048576 entries
18/10/12 09:36:23 INFO namenode.FSDirectory: ACLs enabled? false
18/10/12 09:36:23 INFO namenode.FSDirectory: XAttrs enabled? true
18/10/12 09:36:23 INFO namenode.FSDirectory: Maximum size of an xattr: 16384
18/10/12 09:36:23 INFO namenode.NameNode: Caching file names occuring more than 10 times
18/10/12 09:36:23 INFO util.GSet: Computing capacity for map cachedBlocks
18/10/12 09:36:23 INFO util.GSet: VM type       = 64-bit
18/10/12 09:36:23 INFO util.GSet: 0.25% max memory 966.7 MB = 2.4 MB
18/10/12 09:36:23 INFO util.GSet: capacity      = 2^18 = 262144 entries
18/10/12 09:36:23 INFO namenode.FSNamesystem: dfs.namenode.safemode.threshold-pct = 0.9990000128746033
18/10/12 09:36:23 INFO namenode.FSNamesystem: dfs.namenode.safemode.min.datanodes = 0
18/10/12 09:36:23 INFO namenode.FSNamesystem: dfs.namenode.safemode.extension     = 30000
18/10/12 09:36:23 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10
18/10/12 09:36:23 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10
18/10/12 09:36:23 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25
18/10/12 09:36:23 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
18/10/12 09:36:23 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
18/10/12 09:36:23 INFO util.GSet: Computing capacity for map NameNodeRetryCache
18/10/12 09:36:23 INFO util.GSet: VM type       = 64-bit
18/10/12 09:36:23 INFO util.GSet: 0.029999999329447746% max memory 966.7 MB = 297.0 KB
18/10/12 09:36:23 INFO util.GSet: capacity      = 2^15 = 32768 entries
18/10/12 09:36:25 INFO namenode.FSImage: Allocated new BlockPoolId: BP-1162737530-192.168.33.200-1539308185287
18/10/12 09:36:25 INFO common.Storage: Storage directory /usr/local/hadoopdata/namenode has been successfully formatted.
18/10/12 09:36:25 INFO namenode.FSImageFormatProtobuf: Saving image file /usr/local/hadoopdata/namenode/current/fsimage.ckpt_0000000000000000000 using no compression
18/10/12 09:36:25 INFO namenode.FSImageFormatProtobuf: Image file /usr/local/hadoopdata/namenode/current/fsimage.ckpt_0000000000000000000 of size 352 bytes saved in 0 seconds.
18/10/12 09:36:25 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
18/10/12 09:36:25 INFO util.ExitUtil: Exiting with status 0
18/10/12 09:36:25 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at node200/192.168.33.200
************************************************************/
 5、将格式化后node200节点hadoop工作目录中的元数据目录复制到node201节点

[hadoop@node200 ~]$ scp -r /usr/local/hadoopdata node201:/usr/local/
6、初始化完毕后可关闭journalnode（分别在node202、node203和node204上执行）

hadoop-daemon.sh stop journalnode
Hadoop集群的启动
配置了好久，看到可以启动了是不是特别开心，下面就一步步启动我们的集群吧

启动步骤：

1、启动zookeeper集群（分别在node202、node203和node204上执行）

zkServer.sh start
在初始化过程中，如果启动了zookeeper没有关闭进程，在这里就不用重复启动了

2、启动HDFS（在node200上执行）

[hadoop@node200 ~]$ start-dfs.sh
此命令分别在node200、node201节点启动了NameNode和ZKFC，分别在node202、node203、node204节点启动了DataNode和JournalNode，如下所示。

[hadoop@node200 ~]$ start-dfs.sh
Starting namenodes on [node200 node201]
node200: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-node200.out
node201: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-node201.out
node204: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-node204.out
node202: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-node202.out
node203: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-node203.out
Starting journal nodes [node202 node203 node204]
node202: starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node202.out
node204: starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node204.out
node203: starting journalnode, logging to /usr/local/hadoop/logs/hadoop-hadoop-journalnode-node203.out
Starting ZK Failover Controllers on NN hosts [node200 node201]
node200: starting zkfc, logging to /usr/local/hadoop/logs/hadoop-hadoop-zkfc-node200.out
node201: starting zkfc, logging to /usr/local/hadoop/logs/hadoop-hadoop-zkfc-node201.out
3、启动YARN（在node201上执行）

[hadoop@node201 ~]$ start-yarn.sh
此命令在node201节点启动了ResourceManager，分别在node202、node203、node204节点启动了NodeManager。 

[hadoop@node201 ~]$ start-yarn.sh
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-resourcemanager-node201.out
node202: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-node202.out
node203: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-node203.out
node204: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-node204.out
4、启动YARN的另一个ResourceManager（在node200执行，用于容灾）

[hadoop@node200 ~]$ yarn-daemon.sh start resourcemanager
 
#执行过程如下
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-resourcemanager-node200.out
5、启动YARN的安全代理（在node201执行）

[hadoop@node201 ~]$ yarn-daemon.sh start proxyserver
 
#执行过程如下
starting proxyserver, logging to /usr/local/hadoop/logs/yarn-hadoop-proxyserver-node201.out
备注：proxyserver充当防火墙的角色，可以提高访问集群的安全性

6、启动YARN的历史任务服务（在node200执行）

#方法一：
[hadoop@node200 ~]$ mr-jobhistory-daemon.sh start historyserver
 
#执行过程如下
starting historyserver, logging to /usr/local/hadoop/logs/mapred-hadoop-historyserver-node200.out
 
#方法二：
[hadoop@node200 ~]$ yarn-daemon.sh start historyserver
备注：yarn-daemon.sh start historyserver已被弃用；CDH版本似乎有个问题，即mapred-site.xml配置的的mapreduce.jobhistory.address和mapreduce.jobhistory.webapp.address参数似乎不起作用，实际对应的端口号是10200和8188，而且部需要配置就可以在任意节点上开启历史任务服务。

集群进程查看：

[hadoop@node200 ~]$ jps
5313 DFSZKFailoverController
5842 ResourceManager
7049 Jps
5002 NameNode
6733 JobHistoryServer
[hadoop@node201 ~]$ jps
4726 NameNode
4839 DFSZKFailoverController
6794 Jps
5181 ResourceManager
6110 WebAppProxyServer
[hadoop@node202 ~]$ jps
1508 QuorumPeerMain
5111 NodeManager
4696 DataNode
4794 JournalNode
6318 Jps
[hadoop@node203 ~]$ jps
5080 NodeManager
1484 QuorumPeerMain
4668 DataNode
4767 JournalNode
6303 Jps
[hadoop@node204 ~]$ jps
1490 QuorumPeerMain
6307 Jps
4647 DataNode
5064 NodeManager
4746 JournalNode
Web界面截图：

HDFS

node200：http://192.168.33.200:50070，可看到NameNode为active状态



 

node201：http://192.168.33.201:50070，可看到NameNode为standby状态：



YARN

node201:http://192.168.33.201:8088，可看到ResourceManager为active状态：



node200:http://192.168.33.200:8088，此时ResourceManager为standby状态，

网页无法直接访问，会自动跳转到node201的页面
