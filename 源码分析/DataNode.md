# NameNode启动流程
1. 初始化DataXceiverServer(DN用来接收客户端和其他DN发送过来的数据服务)
2. 初始化Http服务
3. 开启RPC服务端
4. DN向NN注册

```java
DataNode.java:2919 main(){
    secureMain(args, null);
  }
}

->  DataNode datanode = createDataNode(args, null, resources);

->  DataNode dn = instantiateDataNode(args, conf, resources);
    //instantiateDataNode方法返回makeInstance(dataLocations, conf, resources);
    //进入makeInstance方法,返回return new DataNode(conf, locations, storageLocationChecker, resources);    
    //DataNode构造函数startDataNode(dataDirs, resources);
    void startDataNode(List<StorageLocation> dataDirectories,
                     SecureResources resources
                     ) throws IOException {
                    
    registerMXBean();
    //初始化DataXceiverServer
    initDataXceiver();
    //初始化Http服务
    startInfoServer();
    //开启RPC服务端
    initIpcServer();
    //DN向NN注册
    blockPoolManager.refreshNamenodes(getConf());
    }
    //开启datanode守护线程
->  dn.runDatanodeDaemon();
```
## 开启Http服务 startInfoServer
```java
DataNode.java:758 
httpServer = new DatanodeHttpServer(getConf(), this, httpServerChannel);

DataNodeHttpServer.java:115
HttpServer2.Builder builder = new HttpServer2.Builder()
        .setName("datanode")
        .setConf(confForInfoServer)
        .setACL(new AccessControlList(conf.get(DFS_ADMIN, " ")))
        .hostName(getHostnameForSpnegoPrincipal(confForInfoServer))
        .addEndpoint(URI.create("http://localhost:" + proxyPort))
        .setFindPort(true);
```

## 开启RPC服务端
```java
->  initIpcServer() 
-> ipcServer = new RPC.Builder(getConf())
        .setProtocol(ClientDatanodeProtocolPB.class)
        .setInstance(service)
        .setBindAddress(ipcAddr.getHostName())
        .setPort(ipcAddr.getPort())
        .setNumHandlers(
            getConf().getInt(DFS_DATANODE_HANDLER_COUNT_KEY,
                DFS_DATANODE_HANDLER_COUNT_DEFAULT)).setVerbose(false)
        .setSecretManager(blockPoolTokenSecretManager).build();
```
## 向Namenodes注册,refreshNamenodes(getConf());
```
DataNode.java:1441
blockPoolManager.refreshNamenodes(getConf());
-> BlockManager.java:170 
doRefreshNamenodes(newAddressMap, newLifelineAddressMap);
-> BlockManager.java:226
BPOfferService bpos = createBPOS(nsToAdd, addrs, lifelineAddrs);
-> BlockManager.java:289
return new BPOfferService(nameserviceId, nnAddrs, lifelineNnAddrs, dn);
  ->BPOfferService A thread per active or standby namenode to perform:
    this.bpServices.add(new BPServiceActor(nnAddrs.get(i),
-> BlockManager.java:231
startAll()
 for (BPOfferService bpos : offerServices) {
                bpos.start();
}
BPOFFerService.java:341
void start() {
    for (BPServiceActor actor : bpServices) {
      actor.start();
    }
}

void start() {
    if ((bpThread != null) && (bpThread.isAlive())) {
      //Thread is started already
      return;
    }
    bpThread = new Thread(this);
    bpThread.setDaemon(true); // needed for JUnit testing
    bpThread.start();

    if (lifelineSender != null) {
      lifelineSender.start();
    }
  }
BPServiceActor.java:814 
 public void run() {
   ...
   //向NN注册// setup storage
   connectToNNAndHandshake();
   //向NN发送心跳服务
   offerService();
   ...
 }
```
### connectToNNAndHandshake();
```
private void connectToNNAndHandshake() throws IOException {
    // get NN proxy
    // 获取NN代理
    bpNamenode = dn.connectToNN(nnAddr);

    // First phase of the handshake with NN - get the namespace info.
    // 与 NN 握手的第一阶段 - 获取命名空间信息。
    NamespaceInfo nsInfo = retrieveNamespaceInfo();

    // Verify that this matches the other NN in this HA pair.
    // This also initializes our block pool in the DN if we are
    // the first NN connection for this BP.
    bpos.verifyAndSetNamespaceInfo(this, nsInfo);

    /* set thread name again to include NamespaceInfo when it's available. */
    /* 再次设置线程名称以在可用时包含命名空间信息 */
    this.bpThread.setName(formatThreadName("heartbeating", nnAddr));

    // Second phase of the handshake with the NN.
    //注册,与NN第二次握手
    register(nsInfo);
}
//1.报告正在服务的存储,2.接收注册ID
void register(NamespaceInfo nsInfo) throws IOException {
  // Use returned registration from namenode with updated fields
//bpNamenode为Namenode代理
newBpRegistration = bpNamenode.registerDatanode(newBpRegistration);
}
-> NameNodeRpcServer.java:1517
public DatanodeRegistration registerDatanode(DatanodeRegistration nodeReg)
      throws IOException {
    checkNNStartup();
    verifySoftwareVersion(nodeReg);
    //注册Datanode
    namesystem.registerDatanode(nodeReg);
    return nodeReg;
  }
namesystem.registerDatanode(nodeReg);
->blockManager.registerDatanode(nodeReg);
->datanodeManager.registerDatanode(nodeReg);

DatanodeManager.java:1011
  /**
   * Register the given datanode with the namenode. NB: the given
   * registration is mutated and given back to the datanode.
   */
public void registerDatanode(DatanodeRegistration nodeReg)
      throws DisallowedDatanodeException, UnresolvedTopologyException {

  nodeS.updateRegInfo(nodeReg);
  // register new datanode
  addDatanode(nodeDescr);
  // also treat the registration message as a heartbeat
  heartbeatManager.register(nodeS);
}
//heartbeatManager.register
d.updateHeartbeatState(StorageReport.EMPTY_ARRAY, 0L, 0L, 0, 0, null);
->
  void updateHeartbeatState(StorageReport[] reports, long cacheCapacity,
      long cacheUsed, int xceiverCount, int volFailures,
      VolumeFailureSummary volumeFailureSummary) {
    updateStorageStats(reports, cacheCapacity, cacheUsed, xceiverCount,
        volFailures, volumeFailureSummary);
    //
    setLastUpdate(Time.now());
    setLastUpdateMonotonic(Time.monotonicNow());
    rollBlocksScheduled(getLastUpdateMonotonic());
  }

```
### offerService();
```java
  /**
   * Main loop for each BP thread. Run until shutdown,
   * forever calling remote NameNode functions.
   */
    private void offerService() throws Exception {
     ...
    resp = sendHeartBeat(requestBlockReportLease);
    ...
    }

-> sendHeartBeat(boolean requestBlockReportLease)
      throws IOException {
      HeartbeatResponse response = bpNamenode.sendHeartbeat(bpRegistration,
        reports,
        dn.getFSDataset().getCacheCapacity(),
        dn.getFSDataset().getCacheUsed(),
        dn.getXmitsInProgress(),
        dn.getXceiverCount(),
        numFailedVolumes,
        volumeFailureSummary,
        requestBlockReportLease,
        slowPeers,
        slowDisks);
}

-> NamenodeRpcServer.java:1526

  public HeartbeatResponse sendHeartbeat(DatanodeRegistration nodeReg,
      StorageReport[] report, long dnCacheCapacity, long dnCacheUsed,
      int xmitsInProgress, int xceiverCount,
      int failedVolumes, VolumeFailureSummary volumeFailureSummary,
      boolean requestFullBlockReportLease,
      @Nonnull SlowPeerReports slowPeers,
      @Nonnull SlowDiskReports slowDisks) throws IOException {
    checkNNStartup();
    verifyRequest(nodeReg);
    /****************************************/
    return namesystem.handleHeartbeat(nodeReg, report,
        dnCacheCapacity, dnCacheUsed, xceiverCount, xmitsInProgress,
        failedVolumes, volumeFailureSummary, requestFullBlockReportLease,
        slowPeers, slowDisks);
  }

  DatanodeCommand[] cmds = blockManager.getDatanodeManager().handleHeartbeat(
          nodeReg, reports, getBlockPoolId(), cacheCapacity, cacheUsed,
          xceiverCount, maxTransfer, failedVolumes, volumeFailureSummary,
          slowPeers, slowDisks);

->
    heartbeatManager.updateHeartbeat(nodeinfo, reports, cacheCapacity,
        cacheUsed, xceiverCount, failedVolumes, volumeFailureSummary);

->
    blockManager.updateHeartbeat(node, reports, cacheCapacity, cacheUsed,
        xceiverCount, failedVolumes, volumeFailureSummary);


->BlockManager.java:2517
  void updateHeartbeat(DatanodeDescriptor node, StorageReport[] reports,
      long cacheCapacity, long cacheUsed, int xceiverCount, int failedVolumes,
      VolumeFailureSummary volumeFailureSummary) {
      //更新存储
    for (StorageReport report: reports) {
      providedStorageMap.updateStorage(node, report.getStorage());
    }
    //更新心跳
    node.updateHeartbeat(reports, cacheCapacity, cacheUsed, xceiverCount,
        failedVolumes, volumeFailureSummary);
  }

  /**
   * Updates stats from datanode heartbeat.
   */
void updateHeartbeat(StorageReport[] reports, long cacheCapacity,
      long cacheUsed, int xceiverCount, int volFailures,
      VolumeFailureSummary volumeFailureSummary) {
    updateHeartbeatState(reports, cacheCapacity, cacheUsed, xceiverCount,
        volFailures, volumeFailureSummary);
    heartbeatedSinceRegistration = true;
  }

  void updateHeartbeatState(StorageReport[] reports, long cacheCapacity,
      long cacheUsed, int xceiverCount, int volFailures,
      VolumeFailureSummary volumeFailureSummary) {
    updateStorageStats(reports, cacheCapacity, cacheUsed, xceiverCount,
        volFailures, volumeFailureSummary);
    setLastUpdate(Time.now());
    setLastUpdateMonotonic(Time.monotonicNow());
    rollBlocksScheduled(getLastUpdateMonotonic());
  }
```