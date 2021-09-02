## hadoop 基本命令
1. 创建文件夹 hadoop fs -mkdir /filename
2. 上传文件 hadoop fs -put /diretory/filename 
3. 下载文件hadoop fs -get filename
4. 执行wordcount程序 hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.13.jar wordcount /input /output
5. 初始化namenode, hdfs namenode -format
6. 启动HDFS start-dfs.sh
7. (配置了ResourceManager的节点上)启动Yarn start-yarn.sh
8. mapred --daemon stop historyserver
9. hdfs --daemon start/stop namenode/datanode/secondarynamenode
10. yarn --daemon start/stop resourcemanager/nodemanager
11. hadoop fs -appendToFile  source.txt target.txt追加文件内容
12. hadoop fs -help command
13. hadoop fs -copyToLocal/-get targetfile renamefilename
14. (重要,末尾都是新的数据)显示末尾1kb数据hadoop fs -tail filename 
15. hadoop fs -setrep 10 设置副本数,此处设置副本数只是记录在NameNode的元数据,是否真的存那么多副本,还得看DataNode的数量,目前只有3台设备,最多就只有3个副本,只有节点增加到10台,副本数才能达到10
    
## 数据均衡
1. hdfs diskbalancer -plan hadoop102
2. hdfs diskbalancer -excute hadoop102.plan.json
3. hdfs diskbalancer -query hadoop102
4. hdfs diskbalancer -cancel hadoop102.plan.json
   
## 纠删码
1.  hdfs ec -listPolicies
2.  hdfs ec -enablePolicy -policy RS-3-2-1024k
3.  hdfs ec -setPolicy -path path -policy RS-3-2-1024k (-replicate 2)
   
## 异构存储
1. hdfs storagepolicies -listPolicies
2. hdfs storagepolicies -setStoragePolicy -path <path> -policy <policy>
3. hdfs storagepolicies -getStoragePolicy -path <path>
4. hdfs storagepolicies -unsetStoragePolicy -path <path>
5. hdfs fsck <filename> -files -blocks -locations//查看文件块的分布
6. hadoop dfsadmin report//查看集群节点
7. hdfs storagepolicies -setStoragePolicy-path /hdfsdata -policy lazy_persist//更改策略
8. hdfs mover /hdfsdata //策略改变移动文件块
9. hdfs fsck /hdfsdata -files -blocks -locations//查看文件块分布
    
## 安全模式
1. hdfs dfsadmin -safemode get
2. hdfs dfsadmin -safemode enter
3. hdfs dfsadmin -safemode leave
4. hdfs dfsadmin -safemode wait
   
## 测试读集群写性能
1. sudo yum install -y fio
2. sudo fio -filename=/home/atguigu/test.log -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=2G -numjobs=10 -runtime=60 -group_reporting -name=test_r // 测试读
3. sudo fio -filename=/home/atguigu/test.log -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=2G -numjobs=10 -runtime=60 -group_reporting -name=test_w//测试顺序写
4. sudo fio -filename=/home/atguigu/test.log -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=16k -size=2G -numjobs=10 -runtime=60 -group_reporting -name=test_randw//随机写
5.  sudo fio -filename=/home/atguigu/test.log -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=2G -numjobs=10 -runtime=60 -group_reporting -name=test_r_w -ioscheduler=noop//混合随机读写
   
## 小文件归档
1. //将input下所有文件归档为input.har文件,输出到/output
   1. hadoop archive -archiveName input.har -p /input /output
2. //查看信息,har://
   1. hadoop fs -ls har:///output/input.har
3. //解除归档
   1. hadoop fs -cp har:///output/input.har/* /
   2. hadoop fs -cp har:///output/input.har/a.txt /
   
## 集群迁移
1. apache和apache集群间数据拷贝 
   1. scp拷贝
   2. hadoop distcp hdfs://hadoop100:8020/user/z/hello.txt hdfs://hadoop101:8020/user/z/hello.txt
2. apache和CDH集群间数据拷贝
   1. todo...

## yarn命令
- yarn application -list
- yarn application -list-appStates <参数ALL,NEW,NEW_SAVING,SUBMITTED,ACCEPTED,RUNNING,FINISHED,FAILED,KILLED>
- yarn application -kill <applicationId>
- yarn logs -applicationId <ApplicationId>
- container日志,yarn logs -applicationId <applicationId> -containerId <ContainerId>
- 尝试运行的任务yarn applicationattempt -list <ApplicationId>
- 尝试状态yarn applicationattempt -status <ApplicationAttemptId>
- 列出所有Container,yarn container -list <ApplicationAttemptId>
- yarn container -status <ContainerId>
- 查看节点状态,yarn node -list -all
- 更新配置,yarn rmadmin -refreshQueues
- 查看队列yarn queue -status <QueueName>(例yarn queue -status default)

## yarn 配置
- ResourceManager
    - yarn.resourcemanager.scheduler.class 配置调度器，默认容量
    - yarn.resourcemanager.scheduler.client.thread-count ResourceManager处理调度器请求的线程数量，默认50
- NodeManager
    - yarn.nodemanager.resource.detect-hardware-capabilities 是否让yarn自己检测硬件进行配置，默认false
    - yarn.nodemanager.resource.count-logical-processors-as-cores 是否将虚拟核数当作CPU核数，默认false
    - yarn.nodemanager.resource.pcores-vcores-multiplier 虚拟核数和物理核数乘数，例如：4核8线程，该参数就应设为2，默认1.0
    - yarn.nodemanager.resource.memory-mb NodeManager使用内存，默认8G
    - yarn.nodemanager.resource.system-reserved-memory-mb NodeManager为系统保留多少内存以上二个参数配置一个即可
    - yarn.nodemanager.resource.cpu-vcores NodeManager使用CPU核数，默认8个
    - yarn.nodemanager.pmem-check-enabled 是否开启物理内存检查限制container，默认打开
    - yarn.nodemanager.vmem-check-enabled 是否开启虚拟内存检查限制container，默认打开
    - yarn.nodemanager.vmem-pmem-ratio 虚拟内存物理内存比例，默认2.1
- Container
    - yarn.scheduler.minimum-allocation-mb 容器最最小内存，默认1G
    - yarn.scheduler.maximum-allocation-mb 容器最最大内存，默认8G
    - yarn.scheduler.minimum-allocation-vcores 容器最小CPU核数，默认1个
    - yarn.scheduler.maximum-allocation-vcores 容器最大CPU核数，默认4个