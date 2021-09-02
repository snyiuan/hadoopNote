## DNFS优点
1. 高容错性
    - 数据自动保存多副本,提高容错性
    - 某一个副本丢失,可以自动恢复
2. 适合处理大数据
    - 数据处理规模GB,TB,PB级别的数据
    - 文件规模,能够处理百万规模以上的文件数量,数量相当大
3. 可构建在廉价的机器上,通过多副本机制,提高可靠性

## 缺点
1. 不适合低延时数据访问,如毫秒级的存储数据,是做不到的
2. 无法高效的对大量小文件进行存储
    - 存储大量小文件,它会占用NameNode大量的内存来存储文件目录和块信息.这样是不可取的,因为NameNode的内存是有限的(一般为128GB,小文件9亿个)
    - 小文件的寻址时间会超过读取时间,它违反了HDFS的设计目标
3. 不支持并发写入,文件的随机修改
    - 一个文件只能一个写,不允许多个线程同时写
    - 仅支持数据的append(追加),不支持文件的随机修改

## HDFS的组成
1. NameNode 主管,管理者
    - 管理HDFS的名称空间
    - 配置副本策略(设置维持几块副本)
    - 管理块数据映射信息
    - 处理客户端读写请求
2. DataNode ,NameNode下达命令,DataNode执行实际操作
    - 存储实际的数据库
    - 执行数据的读写操作
3. Client客户端
    - 文件切分,文件上传到DDFS的时候,Client将文件切分成一个个block,然后进行上传
    - 与NameNode交互,获取文件的位置信息
    - 与DataNode交互,读写数据
    - Client提供的一些命令来管理HDFS,比如NameNode格式化hdfs namenode -format
    - client可以通过一些命令来访问HDFS,比如对HDFS增删改查操作

4. Secondary NameNode: 并非NameNode的热备,当NameNode挂掉的时候,它并不能马上替换NameNode并提供服务
    - 辅助NameNode,分担其工作量,比如定期合并Fsimage和Edits,并推送给NameNode
    - 在紧急情况下,可辅助恢复NameNode

### HDFS文件块大小
1. 默认128mb 可以通过hdfs-site.xml参数dfs.blcoksize规定
2. 如果寻址时间约为10ms,即查到目标block的时间为10ms
3. 寻址时间为传输时间的1%时,则为最佳状态.因此传输时间=10ms/0.01=1000ms=1s
4. 目前磁盘的传输速率普遍为100MB/S,所以设置块大小为128M,(1s左右传完)

### 块大小问题
1. 如果块太小,会增加寻址时间,程序一直在找块的开始位置,(100kb文件,块设置1kb,需要存储100个文件)
2. 如果设置过大,从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间.导致程序处理这块数据时会非常慢
3. HDFS块的大小设置主要取决于磁盘传输速率(机械硬盘128mb,固态硬盘256mb)

### HDFS读数据,读取不同块是串行,先读block1,再读block2,然后拼接

### 副本节点选择
- 第一个副本在Client所在节点,如果客户端在集群外,则随机选一个
- 第二个副本在另一个机架的随机一个节点
- 第三个副本在第二个副本所在机架的随机节点

### HDFS写数据流程
1. 客户端通过DistributeFileSystem模块箱NameNode请求上传文件
2. NameNode检车客户端用户是否有权限,目录是否存在,返回是否同意上传
3. 客户端请求上传第一个(最大128mb)Block数据,
4. NameNode返回DataNode的3个节点,dn1,dn2,dn3
5. 客户端通过FSOutPutStream模块请求dn1上传数据,dn1收到请求继续调用dn2,dn2调用dn3,将通信管道建立
6. dn1,dn2,dn3逐级应答客户端
7. 客户端开始往dn1上传第一个Block(先从磁盘读取放到一个本地内存缓存),以Packet为单位,dn1收到一个Packet就会传给dn2,dn2传给dn3,dn1每传一个packet会放入一个应答队列等待应答
8. 当一个Block传输完成之后,客户端再次请求NameNode上传第二个Block服务,重复执行3-7步骤

### HDFS读数据流程
1. 客户端通过DistributedFileSystem向NameNode请求下载文件
2. NameNode检查客户端权限,查询元数据,返回目标文件元数据
3. 挑选一台DataNode(就近原则,然后随机)服务器,请求读取数据
4. DataNode开始传输数据给客户端(从磁盘里面读取数据输入流,以Packet为单位来做校验)
5. 客户端以Packet为单位接收,先在本地缓存,然后写入目标文件

### NN和2NN工作机制
### 内存计算快,可靠性低,断电就没了,磁盘计算慢,可靠性高
### NN采用内存和磁盘向结合的方式
1. 磁盘中FsImage存储元数据
2. 磁盘中edits(只进行追加操作),每当有更新或者添加元数据时,修改内存中的元数据并追加到edits中,一旦NameNode断电,可通过FsImage和edits合并,合成元数据
3. 防止edits数据量过大,导致恢复元数据需要的时间长,需要定期合并FsImage和eidts,由另一个节点SecondaryNameNode完成,防止当前节点效率低
4. 启动时内存中同时加载FsImage和eidts


### SecondaryNameNode 合并工作
1. SecondaryNameNode询问NameNode是否需要CheckPoint,直接带回NameNode是否检查的结果
2. SecondaryNameNode请求执行checkPoint
3. NameNode滚动正在写的Eidts日志生成edits01,edits02(记录新的元数据修改),
4. edits01和镜像文件FsImage拷贝到SecondaryNameNode
5. SecondaryNameNode加载编辑日志和镜像文件到内存,并合并
6. 合并的结果生成新的FsImage.chkpoint
7. 拷贝FsImage.chkpoint到NameNode
8. NameNode将FsImage.chkpoint重新命名为FsImage

### 查看eidts和Fsimage命令
```
hdfs oev -p XML edits_001 -i -o /opt/software/output.xml
hdfs oIv -p XML fsimage_001 -i -o /opt/software/output.xml
```

### CheckPoint时间设置,hdfs-default.xml
```
//每小时检查一次
<property>
 <name>dfs.namenode.checkpoint.period</name>
 <value>3600s</value>
</property>
//操作次数达到100万SecondaryNameNode执行一次
<property>
 <name>dfs.namenode.checkpoint.txns</name>
 <value>1000000</value>
<description>操作动作次数</description>
</property>
//每分钟检查一次操作次数
<property>
 <name>dfs.namenode.checkpoint.check.period</name>
 <value>60s</value>
<description> 1 分钟检查一次操作次数</description>
</property>
```

### DataNode工作机制
1. 一个数据块在DataNode上以文件的形式存储在磁盘中,包括两个文件,一个是数据本身,一个是数据包括数据块的长度,块数据的校验和,以及时间戳
2. DataNode启动后向NameNode注册,通过后每隔6小时向NameNode上报所有的块信息
3. 心跳是每 3 秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode的心跳，则认为该节点不可用。
4. 超过10分钟+30秒没有收到DataNode2的心跳，则认为该节点不可用
```
//hdfs-site.xml
//DN 向 NN 汇报当前解读信息的时间间隔，默认 6 小时；
<property>
    <name>dfs.blockreport.intervalMsec</name>
    <value>21600000</value>
    <description>Determines block reporting interval in
milliseconds.</description>
</property>
//DN 扫描自己节点块信息列表的时间，默认 6 小时
<property>
    <name>dfs.datanode.directoryscan.interval</name>
    <value>21600s</value>
    <description>Interval in seconds for Datanode to scan data 
directories and reconcile the difference between blocks in memory and on
the disk.
Support multiple time unit suffix(case insensitive), as described
in dfs.heartbeat.interval.
    </description>
</property>
```
## 掉线时限参数
```
//TimeOut = 2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval。
//单位毫秒
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
//单位秒
<property>
    <name>dfs.heartbeat.interval</name>
    <value>3</value>
</property>
```
