## Yarn ，客户端可以有多个，集群上可以运行多个ApplicationMaster，每个NodeManager上可以有多个Container
- ResourceManager（RM）：整个集群资源（内存，cpu等）的老大
- NodeManager（NM）：单个节点服务器资源老大
- ApplicationMaster（AM）：单个任务运行的老大
- Container：容器，相当于一台独立的服务器，里面封装了任务运行所需要的资源，如内存，cpu，磁盘，网络等

### 调度器
- FIFO调度器(先进先出调度器)
- 容量(Capacity Scheduler)apache Hadoop3.1.3默认调度器,多队列,每个队列可配置一定资源.每个队列按FIFO
- 公平(Fair Scheduler)CDH默认调度器 

#### 容量调度器
1. 多队列,每队列采用FIFO调度策略
2. 容量保证,每队列可以设置资源最低和资源使用上限
3. 灵活,一个队列资源有剩余,可以暂时共享给需要资源的队列,而该队列一旦有新的应用程序提交,则其他队列借调的资源会归还给该队列
4. 多租户,支持多用户共享集群和多应用程序同时运行,为防止同一个用户的作业独占队列中的资源,该调度器会对同一用户提交的作业所占资源进行限定

#### 公平调度器
- 同队列所有任务共享资源,在时间尺度上获得公平的资源(平分资源)
- 与容量调度器相同点,多队列,容量保证,灵活,多租户
- 与容量调度器不同的,核心策略不同
- 容量调度器: 优先选择资源利用率低的队列
- 公平调度器: 优先选择资源缺额比例大的
- 每个队列可以单独设置资源的分配方式
    - 容量调度器(FIFO,DRF)
    - 公平调度器:FIFO,FAIR,DRF ,(DRF 内存+CPU双重因素配比)

## DRF

### 容量调度器配置(capacity-scheduler.xml)
### 更新yarn-site.xml需要重启yarn,
### 更新队列信息,yarn rmadmin -refreshQueues
### 代码中设置
```
public class WcDrvier {
 public static void main(String[] args) throws IOException,
ClassNotFoundException, InterruptedException {
 Configuration conf = new Configuration();
 conf.set("mapreduce.job.queuename","hive");
 //1. 获取一个 Job 实例
 Job job = Job.getInstance(conf);
 。。。 。。。
 //6. 提交 Job
 boolean b = job.waitForCompletion(true);
 System.exit(b ? 0 : 1);
 }
}
```
```
  <property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <!-- 配置队列 -->
    <value>default,hive</value>
    <description>
      The queues at the this level (root is the root queue).
    </description>
  </property>

  <property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>40</value>
    <description>Default queue target capacity.</description>
  </property>
  
  <property>
    <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>60</value>
    <description>Default queue target capacity.</description>
  </property>
	<!-- 用户最大占有队列资源比	-->
  <property>
    <name>yarn.scheduler.capacity.root.default.user-limit-factor</name>
    <value>1</value>
    <description>
      Default queue user limit a percentage from 0.0 to 1.0.
    </description>
  </property>
```

## 公平调度器
##### 修改yarn-site.xml
```
<property>
 <name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
 <description>配置使用公平调度器</description>
</property>
<property>
 <name>yarn.scheduler.fair.allocation.file</name>
 <value>/opt/module/hadoop-3.1.3/etc/hadoop/fair-scheduler.xml</value>
 <description>指明公平调度器队列分配配置文件</description>
</property>
<property>
 <name>yarn.scheduler.fair.preemption</name>
 <value>false</value>
 <description>禁止队列间资源抢占</description>
</property>
```
##### 配置fair-scheduler.xml
```
<?xml version="1.0"?>
<allocations>
 <!-- 单个队列中 Application Master 占用资源的最大比例,取值 0-1 ，企业一般配置 0.1
-->
 <queueMaxAMShareDefault>0.5</queueMaxAMShareDefault>
 <!-- 单个队列最大资源的默认值 test atguigu default -->
 <queueMaxResourcesDefault>4096mb,4vcores</queueMaxResourcesDefault>
 <!-- 增加一个队列 test -->
 <queue name="test">
 <!-- 队列最小资源 -->
 <minResources>2048mb,2vcores</minResources>
 <!-- 队列最大资源 -->
 <maxResources>4096mb,4vcores</maxResources>
 <!-- 队列中最多同时运行的应用数，默认 50，根据线程数配置 -->
 <maxRunningApps>4</maxRunningApps>
 <!-- 队列中 Application Master 占用资源的最大比例 -->
 <maxAMShare>0.5</maxAMShare>
 <!-- 该队列资源权重,默认值为 1.0 -->
 <weight>1.0</weight>
 <!-- 队列内部的资源分配策略 -->
 <schedulingPolicy>fair</schedulingPolicy>
 </queue>
 <!-- 增加一个队列 atguigu -->
 <queue name="atguigu" type="parent">
 <!-- 队列最小资源 -->
 <minResources>2048mb,2vcores</minResources>
 <!-- 队列最大资源 -->
 <maxResources>4096mb,4vcores</maxResources>
 <!-- 队列中最多同时运行的应用数，默认 50，根据线程数配置 -->
 <maxRunningApps>4</maxRunningApps>
 <!-- 队列中 Application Master 占用资源的最大比例 -->
 <maxAMShare>0.5</maxAMShare>
 <!-- 该队列资源权重,默认值为 1.0 -->
 <weight>1.0</weight>
 <!-- 队列内部的资源分配策略 -->
 <schedulingPolicy>fair</schedulingPolicy>
 </queue>
 <!-- 任务队列分配策略,可配置多层规则,从第一个规则开始匹配,直到匹配成功 -->
 <queuePlacementPolicy>
 <!-- 提交任务时指定队列,如未指定提交队列,则继续匹配下一个规则; false 表示：如果指
定队列不存在,不允许自动创建-->
 <rule name="specified" create="false"/>
 <!-- 提交到 root.group.username 队列,若 root.group 不存在,不允许自动创建；若
root.group.user 不存在,允许自动创建 -->
 <rule name="nestedUserQueue" create="true">
 <rule name="primaryGroup" create="false"/>
 </rule>
 <!-- 最后一个规则必须为 reject 或者 default。Reject 表示拒绝创建提交失败，
default 表示把任务提交到 default 队列 -->
 <rule name="reject" />
 </queuePlacementPolicy>
</allocations>
```
### 提交任务时指定队列，按照配置规则，任务会到指定的 root.test 队列 
```
 hadoop jar /opt/module/hadoop3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi -D mapreduce.job.queuename=root.test 1 1
```
### 提交任务时不指定队列，按照配置规则，任务会到 root.atguigu.atguigu 队列
```
 hadoop jar /opt/module/hadoop3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi 1 1
  hadoop jar wc.jar WordCountDriver -D mapreduce.job.queuename=root.test /input /output1
   hadoop jar wc.jar com.z.mapreduce.wordcount2.WordCountDriver /input /output1

```
