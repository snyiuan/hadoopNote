### hdfs高可用
1. hadoop2.0之前,hdfs集群中namenode存在单点故障(SPOF)
2. hadoop2.x,可在同意集群运行两个冗余namenode来解决问题
3. hadoop3.0起可超两个冗余namenode
4. HDFS HA通过配置Active/Standby两种namenode集群实现对namenode热备
   
## HDFS-HA工作机制的变更
1. 元数据管理变更,Edits日志只有active状态的namenode节点可写,其他可读,共享的Edits放在一个共享存储中管理(jounalNode或NFS两个主流实现)
2. 每个namenode节点上实现了一个zkfailover(进程DFSZKFailoverController,zookeeper客户端),zkfailover负责监控自己所在的namenode节点,利用zookeeper进行状态标识,当需要状态切换时,zkfailover负责切换,同时防止brain split发生
3. 每个namenode节点需要ssh免密登录,hdfs-site.xml设置私钥位置
4. 隔离(fence),同一时刻只有一个namenode对外提供服务

```
//官网高可用详情
https://hadoop.apache.org/docs/r3.3.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html
```
## HDFS-HA集群配置
### 1. 配置并开启zookeeper集群hadoop102,103,104
### 2. 配置HDFS-HA集群
#### 2.1 core-site.xml
```
<configuration>
        <!-- 指定 NameNode 的地址(两个或多个namenode地址合并) -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://mycluster</value>
        </property>

         <!-- 指定 hadoop 数据的存储目录 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/opt/module/hadoop-3.1.3/HA/data/tmp</value>
        </property>
         <!-- 配置 HDFS 网页登录使用的静态用户为 Z-->
        <property>
                <name>hadoop.http.staticuser.user</name>
                <value>z</value>
        </property>
         <!-- 垃圾回收时间60分钟 -->
        <property>
                <name>fs.trash.interval</name>
                <value>60</value>
        </property>
		<!-- 配置zookeeper集群 -->
        <property>
                <name>ha.zookeeper.quorum</name>
                <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
        </property>
</configuration>
```
#### 2.2 hdfs-site.xml
```
<configuration>
	<!-- 副本数 -->
	<property>
		  <name>dfs.replication</name>
		  <value>3</value>
	</property>
	<!-- 关闭用户权限检查(一般不用) -->
	<property>
		<name>dfs.permissions.enable</name>
		<value>false</value>
	</property>
	<!-- namenode rpc服务监听客户端请求数 -->
	<property>
		 <name>dfs.namenode.handler.count</name>
		 <value>21</value>
	</property>
	<!-- 白名单 -->
	<property>
		<name>dfs.hosts</name>
		<value>/opt/module/hadoop-3.1.3/etc/hadoop/whitelist</value>
	</property>
	<!-- 黑名单 -->
	<property>
        	<name>dfs.hosts.exclude</name>
	        <value>/opt/module/hadoop-3.1.3/etc/hadoop/blacklist</value>
    </property>
	<!-- 完全分布式集群名称 -->
	<property>
		  <name>dfs.nameservices</name>
		  <value>mycluster</value>
	</property>
	<!-- 集群中namennode节点 -->
	<property>
		  <name>dfs.ha.namenodes.mycluster</name>
		  <value>nn1,nn2</value>
	</property>
	<!-- nn1rpc通信端口 -->
	<property>
		  <name>dfs.namenode.rpc-address.mycluster.nn1</name>
		  <value>hadoop102:8020</value>
	</property>
	<property>
		  <name>dfs.namenode.rpc-address.mycluster.nn2</name>
		  <value>hadoop104:8020</value>
	</property>
	<!-- nn1http通信端口,网页访问 -->
	<property>
		  <name>dfs.namenode.http-address.mycluster.nn1</name>
		  <value>hadoop102:9870</value>
	</property>
	<property>
		  <name>dfs.namenode.http-address.mycluster.nn2</name>
		  <value>hadoop104:9870</value>
	</property>
	<!-- 指定namenode元数据存放在journalnode的位置 -->
	<property>
		  <name>dfs.namenode.shared.edits.dir</name>
		  <value>qjournal://hadoop102:8485;hadoop103:8485;hadoop104:8485/mycluster</value>
	</property>
	//确保隔离防止脑裂,sshfence强制杀死namenode进程,
	//若失败,若提供了shell可执行文件.sh,则调用指定.sh文件
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>sshfence</value>
		<value>shell(bin/true)</value>
	</property>
	<!-- ssh连接超时时间 -->
	<property>
		<name>dfs.ha.fencing.ssh.connect-timeout</name>
		<value>30000</value>
	</property>
	<!-- ssh私钥位置 -->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/home/z/.ssh/id_rsa</value>
	</property>
	<!-- journalnode服务器存储目录 -->
	<property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/opt/module/hadoop-3.1.3/HA/data/tmp/jn</value>
    </property>
	<!-- 在safemode不成为active -->
	<property>
		<name>dfs.ha.nn.not-become-active-in-safemode</name>
		<value>true</value>
	</property>
	<!-- 是否失败自动转移 -->
	<property>
		  <name>dfs.ha.automatic-failover.enabled</name>
		  <value>true</value>
	</property>
	<!-- 访问代理类：client，mycluster，active配置失败自动切换实现方式 -->
	<property>
		  <name>dfs.client.failover.proxy.provider.mycluster</name>
		  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
</configuration>
```
### 启动HDFS-HA,手动激活active
1. 启动journalnode, hdfs --daemon start journalnode
2. nn1格式化namenode,hdfs namenode -format(只需要执行一次)并启动namenode
3. nn2..nnn上同步nn1的元数据 hdfs namenode -bootstrapStandby(只需要执行一次)
4. 启动nn2,hdfs --daemon start namenode
5. 启动所有datanode, hdfs --daemon start datanode
6. 手动切换active, hdfs haadmin -transitionToActive nn1
7. 查看状态hdfs haadmin -getServerState nn1

### 自动故障转移
1. 关闭hdfs服务,stop-dfs.sh
2. 启动zookeeper集群 zkServer.sh start
3. 初始化HA在zookeeper中的状态,hdfs zkfc -formatZK
4. 启动HDFS,start-dfs.sh
5. 各个节点启动zkfc,hdfs --daemon start zkfc

```
//在各个journalNode节点上,启动journalnode服务
hdfs --daemon start journalnode
//nn1格式化并启动
hdfs namenode -format
hdfs --daemon start namenode
//在nn2上同步nn1的元数据
//只在第一次启动集群需要
hdfs namenode -bootstrapStandby
hdfs --daemon start namenode

//初始化,在zookeeper集群创建hadoop-ha节点
//hadoop-ha节点下创建hdfs-site.xml中dfs.nameservices的值mycluster的节点
hdfs zkfc -formatZK
```
```
sshfence：防止namenode脑裂，当脑裂时，会自动通过ssh到old-active将其杀掉，将standby切换为active。但是只能在网络通畅时有效，一旦ipdown后fencing方法返回false，standby不会自动切换active，只能手动执行 hdfs haadmin failover namenode1 namenode2 进行切；所以需要加配shell(/bin/true)。想要kill掉namenode active后standby自动切换为active，需要安装psmisc(fuser)；因为sshfence方式是使用fuser通过ssh登录old-active进行诊断从而切换active/standby的。
shell(/bin/true)：如果出现故障并且fencing方法返回false，则会继续执行shell(true)，从而active/standby自动切换。fencing方法返回true，则不会执行shell。
  <property>
      <name>dfs.ha.fencing.methods</name>
      //sshfence 需要用fuser需要安装psmisc
      //yum install psmisc
      <value>sshfence</value>
      <value>shell(/path/to/my/script.sh arg1 arg2 ...)</value>
    </property>
  <property> 
    <name>dfs.ha.fencing.ssh.connect-timeout</name> 
    <value>30000</value> 
    </property>
```
