yarn 在ResourceManager启动
1.创建虚拟机centos7.5
- 删除firewalld(yum remove firewalld)或者systemctl stop firewalld ,systemctl disable firewalld
- 设置虚拟机数据源 curl http://mirrors.aliyun.com/repo/Centos-7.repo > /etc/yum.repos.d/CentOS-Base.repo
- 设置虚拟机数据源curl http://mirrors.163.com/.help/CentOS7-Base-163.repo >/etc/yum.repos.d/CentOS7-Base-163.repo
- 网络工具包yum install -y net-tools
- vim编辑器 yum install -y vim
- 设置z用户sudo免密(sudo vim /etc/sudoers) z ALL=(ALL) NOPASSWD:ALL
- 设置静态ip vim /etc/sysconfig/network-scripts/ifcfg-ens33
- 设置主机名称映射 vim /etc/hosts 192.168.10.100 hadoop100 
- 设置主机名vim /etc/hostname
- ssh免密登录 ssh-keygen生成秘钥对,ssh-copy-id 用户@hostname,拷贝到目的主机.ssh/authorized_keys
- 创建目录 /opt/module /opt/software ,更改module和software所有权 sudo chown z:z -R /opt/module
- 克隆虚拟机 修改hostname
- window hosts  C:/Windows/System32/drivers/etc/hosts
- 若有桌面环境,删除自带的JDK ,复制 jdk-8u212-linux-x64.tar.gz,hadoop-3.1.3.tar.gz到/opt/software
- 配置JDK和hadoop环境变量 vim /etc/profile.d/my_env.sh
```
#!/bin/bash
#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_212
export PATH=$PATH:$JAVA_HOME/bin

#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```
- 编写集群分发脚本xsync ,放到/home/user/bin,添加可执行 chmod +x xsync
```
    /*
    *安全拷贝scure copy
    scp -r $pdir/$name $user@host:$pdir/$fname*/
    /*
    * 远程同步工具 rsync,对差异文件做更新
    * 参数 -a 归档拷贝 -v显示过程
    * rsync -av $pdir/$name $user@host:$pdir/$fname*
    * mkdir bin ,cd bin ,vim xsync 
    */
    
#!/bin/bash
#1. 判断参数个数
if [ $# -lt 1 ]
then
 echo Not Enough Arguement!
 exit;
fi
#2. 遍历集群所有机器
for host in hadoop102 hadoop103 hadoop104
do
 echo ==================== $host ====================
 #3. 遍历所有目录，挨个发送
 for file in $@
 do
	#4. 判断文件是否存在
	if [ -e $file ]
	then
		#5. 获取父目录
		pdir=$(cd -P $(dirname $file); pwd)
		#6. 获取当前文件的名称
		fname=$(basename $file)
		ssh $host "mkdir -p $pdir"
		rsync -av $pdir/$fname $host:$pdir
	else
		echo $file does not exists!
	fi
	done
done
```
## 集群配置
<table>
    <tr>
        <th> </th>
        <th>hadoop102</th>
        <th>hadoop103</th>
        <th>hadoop104</th>
    </tr>
    <tr>
        <td rowspan="2">HDFS</td>
        <td>NameNode</td>
        <td>DataNode</td>
        <td>SecondaryNameNode</td>
    </tr>
    <tr>
        <td>DataNode</td>
        <td></td>
        <td>DataNode</td>
    </tr>
    <tr>
        <td rowspan="2">YARN</td>
        <td></td>
        <td>ResourceManager</td>
        <td></td>
    </tr>
    <tr>
        <td>NodeManager</td>
        <td>NodeManager</td>
        <td>NodeManager</td>
    </tr>
</table>
## 默认配置文件

文件 | 位置
---|---
[core-default.xml] | hadoop-common-3.1.3.jar/core-default.xml
[hdfs-default.xml] | hadoop-hdfs-3.1.3.jar/hdfs-default.xml
[yarn-default.xml] | hadoop-yarn-common-3.1.3.jar/yarn-default.xml
[mapred-default.xml] | hadoop-mapreduce-client-core-3.1.3.jar/mapred-default.xml

- cd $HADOOP_HOME/etc/hadoop
- core-site.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
 <!-- 指定 NameNode 的地址 -->
 <property>
 <name>fs.defaultFS</name>
 <value>hdfs://hadoop102:8020</value>
 </property>
 <!-- 指定 hadoop 数据的存储目录 -->
 <property>
 <name>hadoop.tmp.dir</name>
 <value>/opt/module/hadoop-3.1.3/data</value>
 </property>
 <!-- 配置 HDFS 网页登录使用的静态用户为 atguigu -->
 <property>
 <name>hadoop.http.staticuser.user</name>
 <value>atguigu</value>
 </property>
</configuration>
```
- hdfs-site.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<!-- nn web 端访问地址-->
<property>
 <name>dfs.namenode.http-address</name>
 <value>hadoop102:9870</value>
 </property>
<!-- 2nn web 端访问地址-->
 <property>
 <name>dfs.namenode.secondary.http-address</name>
 <value>hadoop104:9868</value>
 </property>
</configuration>
```
- yarn-site.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
 <!-- 指定 MR 走 shuffle -->
 <property>
 <name>yarn.nodemanager.aux-services</name>
 <value>mapreduce_shuffle</value>
 </property>
 <!-- 指定 ResourceManager 的地址-->
 <property>
 <name>yarn.resourcemanager.hostname</name>
 <value>hadoop103</value>
 </property>
 <!-- 环境变量的继承(hadoop3.1.3BUG,hadoop3.2不用配置)-->
 <property>
 <name>yarn.nodemanager.env-whitelist</name>

<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CO
NF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAP
RED_HOME</value>
 </property>
</configuration>
```
- mapred-site.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<!-- 指定 MapReduce 程序运行在 Yarn 上 -->
 <property>
 <name>mapreduce.framework.name</name>
 <value>yarn</value>
 </property>
</configuration>
```
### 配置workers vim /opt/module/hadoop3.1.3/etchadoop/workers
```
hadoop102
hadoop103
hadoop104
```
### 集群上分发配置xsync /opt/module/hadoop3.1.3/etc/hadoo
### 启动集群,如果第一次启动,需要在102节点格式化NameNode(格式化NameNode,会产生新的集群id,导致NameNode和DataNode的集群id不一致,集群找不到以往的数据,如果集群运行中报错,需要重新格式化NameNode,需要先停止NameNode和DataNode进程,并删除所有机器中的data和logs目录)
### 命令hdfs namenode -format
```
//hadoop102启动HDFS,web查看NameNode http://hadoop102:9870
start-dfs.sh
//hadoop103启动ResourceManager,web查看ResourceManager http://hadoop103:8088
start-yarn.sh
```
### 配置历史服务器mapred-site.xml
```
<!-- 历史服务器端地址 -->
<property>
 <name>mapreduce.jobhistory.address</name>
 <value>hadoop102:10020</value>
</property>
<!-- 历史服务器 web 端地址 -->
<property>
 <name>mapreduce.jobhistory.webapp.address</name>
 <value>hadoop102:19888</value>
</property>
```
### 开启日志聚集功能 yarn-site.xml,
```
<!-- 开启日志聚集功能 -->
<property>
 <name>yarn.log-aggregation-enable</name>
 <value>true</value>
</property>
<!-- 设置日志聚集服务器地址 -->
<property>
 <name>yarn.log.server.url</name>
 <value>http://hadoop102:19888/jobhistory/logs</value>
</property>
<!-- 设置日志保留时间为 7 天 -->
<property>
 <name>yarn.log-aggregation.retain-seconds</name>
 <value>604800</value>
</property>
```
### 集群时间同步(公网环境不用)
- 安装ntp yum install ntp
- 修改配置vim /etc/ntp.conf 
```
//修改:
restrict 192.168.66.0 mask 255.255.255.0 nomodify notrap
//注释自动同步网上时间
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

//添加
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```
### 常见错误
1. 防火墙没关闭、或者没有启动 YARN
INFO client.RMProxy: Connecting to ResourceManager at hadoop108/192.168.10.108:8032
2. 主机名称配置错误
3. IP 地址配置错误
4. ssh 没有配置好
5. root 用户和 atguigu 两个用户启动集群不统一
6. 配置文件修改不细心
7. 不识别主机名称 /etc/hosts 添加ipaddr hostname ,hostname不用起特殊名称如hadoop命令
8. DataNode和NameNode进程只能工作一个,data(默认/tmp)和logs目录没删重新执行过hdfa namenode -format 导致重新生成clusterId(集群id)
9. 执行命令不生效，粘贴 Word 中命令时，遇到-和长–没区分开。导致命令失效
解决办法：尽量不要粘贴 Word 中代码。
10. jps 发现进程已经没有，但是重新启动集群，提示进程已经开启。
原因是在 Linux 的根目录下/tmp 目录中存在启动的进程临时文件，将集群相关进程删
除掉，再重新启动集群。
11. jps 不生效
原因：全局变量 hadoop java 没有生效。解决办法：需要 source /etc/profile 文件。
12. 8088 端口连接不上
```
[atguigu@hadoop102 桌面]$ cat /etc/hosts
注释掉如下代码
#127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
#::1 hadoop102

```