# NameNode启动流程
1. 启动9870端口服务
2. 加载镜像文件和编辑日志
3. 初始化NameNode的RPC服务端
4. 启动资源检查
5. 对心跳超时判断
6. 安全模式
## 0. NameNode.jave-Main()
```
1755 NameNode namenode = createNameNode(argv, null);
1688 return new NameNode(conf);
949 initialize(getConf());
//启动9870端口服务
703 startHttpServer(conf);
//加载镜像文件和编辑日志
706 loadNamesystem(conf);
//初始化NameNode的RPC服务端
709 rpcServer = createRpcServer(conf);
//启动通用服务,资源检查,心跳超时判断,安全模式,开启RPC服务
726 startCommonServices(conf);
```
### 1. 启动9870端口服务
```
703 startHttpServer(conf);
880 httpServer = new NameNodeHttpServer(conf, this, getHttpServerBindAddress(conf));
    624 getHttpServerBindAddress(conf)
    625 InetSocketAddress bindAddress = getHttpServerAddress(conf);
    611 return getHttpAddress(conf);
        //"0.0.0.0:9870"
        639 return  NetUtils.createSocketAddr(conf.getTrimmed
        (DFS_NAMENODE_HTTP_ADDRESS_KEY, 
        DFS_NAMENODE_HTTP_ADDRESS_DEFAULT));
        public static final String  DFS_NAMENODE_HTTP_ADDRESS_DEFAULT = "0.0.0.0:" + DFS_NAMENODE_HTTP_PORT_DEFAULT;
        public static final int     DFS_NAMENODE_HTTP_PORT_DEFAULT =
        HdfsClientConfigKeys.DFS_NAMENODE_HTTP_PORT_DEFAULT;
        int DFS_NAMENODE_HTTP_PORT_DEFAULT = 9870;
881 httpServer.start();
    //Hadoop 自己封装了 HttpServer，形成自己的 HttpServer2
    NameNodeHttpServer.java:149 HttpServer2.Builder builder = DFSUtil.httpServerTemplateForNNAndJN(conf,
        httpAddr, httpsAddr, "hdfs",
        DFSConfigKeys.DFS_NAMENODE_KERBEROS_INTERNAL_SPNEGO_PRINCIPAL_KEY,
        DFSConfigKeys.DFS_NAMENODE_KEYTAB_FILE_KEY);
    NameNodeHttpServer.java:149 httpServer = builder.build();
    NameNodeHttpServer.java:180 setupServlets(httpServer, conf);
    NameNodeHttpServer.java:181 httpServer.start();
```
### 2. 加载镜像文件和编辑日志
```
NamaNode.java:706 loadNamesystem(conf);
NamaNode.java:644 this.namesystem = FSNamesystem.loadFromDisk(conf);
FSNameSystem.java:707 FSImage fsImage = new FSImage(conf,
        FSNamesystem.getNamespaceDirs(conf),
        FSNamesystem.getNamespaceEditsDirs(conf));
FSNameSystem.java:718 namesystem.loadFSImage(startOpt);
FSNameSystem.java:721 fsImage.close();
```
### 3. 初始化NameNode的RPC服务端,在下一步startCommonServices(conf);才启动rpcServer.start();
1. serviceRpcServer
2. lifelineRpcServer
3. clientRpcServer
```
 /** The RPC server that listens to requests from DataNodes */
  private final RPC.Server serviceRpcServer;
  private final InetSocketAddress serviceRPCAddress;

  /** The RPC server that listens to lifeline requests */
  private final RPC.Server lifelineRpcServer;
  private final InetSocketAddress lifelineRPCAddress;
  
  /** The RPC server that listens to requests from clients */
  protected final RPC.Server clientRpcServer;
  protected final InetSocketAddress clientRpcAddress;
```

```
FSImage.java:709 rpcServer = createRpcServer(conf);
FSImage.java:795 return new NameNodeRpcServer(conf, this);
NameNodeRpcServer.java:356 serviceRpcServer = new RPC.Builder(conf)
          .setProtocol(
              org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolPB.class)
          .setInstance(clientNNPbService)
          .setBindAddress(bindHost)
          .setPort(serviceRpcAddr.getPort())
          .setNumHandlers(serviceHandlerCount)
          .setVerbose(false)
          .setSecretManager(namesystem.getDelegationTokenSecretManager())
          .build();
```
### 4. 启动通用服务
```
NameNode.java:726 startCommonServices(conf);
NameNode.java:800 namesystem.startCommonServices(conf, haContext);
void startCommonServices(Configuration conf, HAContext haContext) throws IOException {
      //NameNodeResourceChecker,资源检查器,标记需要检查的目录,HashMap属性volums有需要检查的edits目录和额外目录
      //每个卷保留空间duReserved=100m
      nnResourceChecker = new NameNodeResourceChecker(conf);
      //检查NameNode剩余空间是否小于100m(dfs.namenode.resource.du.reserved)
      //资源检查,缓存结果
      checkAvailableResources();
      assert !blockManager.isPopulatingReplQueues();
      StartupProgress prog = NameNode.getStartupProgress();
      //开始安全模式
      prog.beginPhase(Phase.SAFEMODE);
      //获取所有可正常使用的block
      long completeBlocksTotal = getCompleteBlocksTotal();
      prog.setTotal(Phase.SAFEMODE, STEP_AWAITING_REPORTED_BLOCKS,
          completeBlocksTotal);
      //检查心跳是否超时
      blockManager.activate(conf, completeBlocksTotal);
  }
```
#### 4.1 NameNode资源检查
```
//生成NameNode资源检查器,标记需要检查的目录
FSNamesystem.java:1172 nnResourceChecker = new NameNodeResourceChecker(conf);
    //public static final long    DFS_NAMENODE_DU_RESERVED_DEFAULT = 1024 * 1024 * 100; // 100 MB
    NameNodeResourceChecker.java:113 duReserved = conf.getLong(DFSConfigKeys.DFS_NAMENODE_DU_RESERVED_KEY,
        DFSConfigKeys.DFS_NAMENODE_DU_RESERVED_DEFAULT);

//Perform resource checks and cache the results.
//调用资源检查器执行资源检查
FSNamesystem.java:1172: checkAvailableResources();

void checkAvailableResources() {
    long resourceCheckTime = monotonicNow();
    Preconditions.checkState(nnResourceChecker != null,
    "nnResourceChecker not initialized");
    //检查磁盘空间是否可用
    hasResourcesAvailable = nnResourceChecker.hasAvailableDiskSpace();
    resourceCheckTime = monotonicNow() - resourceCheckTime;
    NameNode.getNameNodeMetrics().addResourceCheckTime(resourceCheckTime);
  }

NameNodeResourceChecker.java:180
public boolean hasAvailableDiskSpace() {
    return NameNodeResourcePolicy.areResourcesAvailable(volumes.values(),
        minimumRedundantVolumes);
}

NameNodeResourcePolicy.java:64 
if (!resource.isResourceAvailable()) 

CheckedVolume.java:82
isResourceAvailable(){
    CheckedVolume.java:88
    //可用空间是否小于duReserved(默认100m)
    if (availableSpace < duReserved) 
}

```
#### 4.2 对心跳超时判断
```
FSNamesystem.java:1180
void startCommonServices(Configuration conf, HAContext haContext) throws IOException {
    blockManager.activate(conf, completeBlocksTotal);
}
BlockManager.java:691
datanodeManager.activate(conf);
datanodeManager.java:373
void activate(final Configuration conf) {
    //初始化配置
    datanodeAdminManager.activate(conf);
    //心跳管理器激活启动
    heartbeatManager.activate();
}
HeartbeatManager.java:93
void activate() {
    heartbeatThread.start();
}

HeartbeatManager.java:62
private final Daemon heartbeatThread = new Daemon(new Monitor());
//HeartbeatManager内部类Monitor
HeartbeatManager.java:435 Monitor的run方法
    public void run() {
      while(namesystem.isRunning()) {
        //重启观察
        restartHeartbeatStopWatch();
        try {
          final long now = Time.monotonicNow();
          if (lastHeartbeatCheck + heartbeatRecheckInterval < now) {
            //心跳检查
            heartbeatCheck();
            lastHeartbeatCheck = now;
          }
          if (blockManager.shouldUpdateBlockKey(now - lastBlockKeyUpdate)) {
            synchronized(HeartbeatManager.this) {
              for(DatanodeDescriptor d : datanodes) {
                d.setNeedKeyUpdate(true);
              }
            }
            lastBlockKeyUpdate = now;
          }
        } catch (Exception e) {
          LOG.error("Exception while checking heartbeat", e);
        }
        try {
          Thread.sleep(5000);  // 5 seconds
        } catch (InterruptedException ignored) {
        }
        // avoid declaring nodes dead for another cycle if a GC pause lasts
        // longer than the node recheck interval
        if (shouldAbortHeartbeatCheck(-5000)) {
          LOG.warn("Skipping next heartbeat scan due to excessive pause");
          lastHeartbeatCheck = Time.monotonicNow();
        }
      }
    }

HeartbeatManager.java:441
heartbeatCheck();
HeartbeatManager.java:377
//判断DN节点是否断掉
if (dead == null && dm.isDatanodeDead(d))
  boolean isDatanodeDead(DatanodeDescriptor node) {

    return (node.getLastUpdateMonotonic() <
            (monotonicNow() - heartbeatExpireInterval));
}
//心跳超时判断
heartbeatRecheckInterval=5*60*1000=5分钟=5*60*1000毫秒
heartbeatIntervalSeconds=3秒
this.heartbeatExpireInterval=2*5*60*1000+10*1000*3=10分钟+30秒
this.heartbeatExpireInterval = 2 * heartbeatRecheckInterval
        + 10 * 1000 * heartbeatIntervalSeconds;
```
#### 4.3 安全模式
```
FSNamesystem.java:1176 
    //开始安全模式
    prog.beginPhase(Phase.SAFEMODE);
    //获取所有可以正常使用的block数
    long completeBlocksTotal = getCompleteBlocksTotal();
    //等待报告块
    prog.setTotal(Phase.SAFEMODE, STEP_AWAITING_REPORTED_BLOCKS,completeBlocksTotal);
    //启动块服务
    blockManager.activate(conf, completeBlocksTotal);

FSNamesystem.java:4578 
public long getCompleteBlocksTotal() {
    // Calculate number of blocks under construction
    long numUCBlocks = 0;
    readLock();
    try {
        //获取所有的block-正在构建的block=可以正常使用的block
      numUCBlocks = leaseManager.getNumUnderConstructionBlocks();
      return getBlocksTotal() - numUCBlocks;
    } finally {
      readUnlock("getCompleteBlocksTotal");
    }
  }
  进入blockManager.activate(conf, completeBlocksTotal)

BlockManager.java:698
  blockManager.activate(conf, completeBlocksTotal){
      ...
      bmSafeMode.activate(blockTotal);
      ...
}

bmSafeMode.active方法
BlockManagerSageMode.java:171
  void activate(long total) {
    assert namesystem.hasWriteLock();
    assert status == BMSafeModeStatus.OFF;

    startTime = monotonicNow();
    //设置可用block数
    setBlockTotal(total);
    if (areThresholdsMet()) {
      boolean exitResult = leaveSafeMode(false);
      Preconditions.checkState(exitResult, "Failed to leave safe mode.");
    } else {
      // enter safe mode
      status = BMSafeModeStatus.PENDING_THRESHOLD;
      initializeReplQueuesIfNecessary();
      reportStatus("STATE* Safe mode ON.", true);
      lastStatusReport = monotonicNow();
    }
}

this.threshold = conf.getFloat(DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_KEY,
        DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_DEFAULT);
public static final float   DFS_NAMENODE_SAFEMODE_THRESHOLD_PCT_DEFAULT = 0.999f;

BlockManagerSageMode.java:288
void setBlockTotal(long total) {
    assert namesystem.hasWriteLock();
    synchronized (this) {
      this.blockTotal = total;
      // 计算阈值：例如：1000 个正常的块 * 0.999 = 999
      //this.blockThreshold = (long) (total * 0.999f);
      this.blockThreshold = (long) (total * threshold);
    }
    this.blockReplQueueThreshold = (long) (total * replQueueThreshold);
}

BlockManagerSageMode.java:566
private boolean areThresholdsMet() {
    assert namesystem.hasWriteLock();
    // Calculating the number of live datanodes is time-consuming
    // in large clusters. Skip it when datanodeThreshold is zero.
    int datanodeNum = 0;
    if (datanodeThreshold > 0) {
      datanodeNum = blockManager.getDatanodeManager().getNumLiveDataNodes();
    }
    synchronized (this) {
      //  /** Safe mode minimum number of datanodes alive. */  
      //private final int datanodeThreshold;
      // 已经正常注册的块数 >= 块的最小阈值 && dataNode数量>=最小可用 DataNode
      return blockSafe >= blockThreshold && datanodeNum >= datanodeThreshold;
    }
  }

```