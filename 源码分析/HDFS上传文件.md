## HDFS上传文件
#### create创建过程
1. DD向DN发起创建请求
2. NN处理DN的创建请求

#### write上传过程
1. 向DataStream队列写数据
2. 建立管道(机架感知,存储位置)
3. 建立管道socket发送
4. 建立管道socket接收
5. 客户端接收DN写数据应答Response   
```
    //创建目录结构,创建线程文件输出流DataStreamer,并且DataStreamer.wait()
    FSDataOutputStream fsDataOutputStream = fs.create(new Path("/inputtnew"));
    //写数据
    fsDataOutputStream.write("hello world".getBytes());
```
### block,chunk和packet
1. block最终存储在DataNode上数据粒度,参数dfs.clock.size,默认64M,一般设置为128或256
2. packet是,DFSClient流向DataNode的数据粒度,参数dfs.write.packet.size,默认64k,注：这个参数为参考值，是指真正在进行数据传输时，会以它为基准进行调整，调整的原因是一个packet有特定的结构，调整的目标是这个packet的大小刚好包含结构中的所有成员，同时也保证写到DataNode后当前block的大小不超过设定值；
3. chunk单位最小,是DFSClient到DataNode进行数据校验的粒度,参数io.bytes.per.checksum,默认512B,注：事实上一个chunk还包含4B的校验值，因而chunk写入packet时是516B；数据与检验值的比值为128:1，所以对于一个128M的block会有一个1M的校验文件与之对应；

## 写数据的三层buffer
1. FSOutputSummer.write,DFSOutputStream内有一个chunk大小的buffer,当数据写满,或者强制flush,会计算checksum的值,然后放进packet
2. 当一个个chunk将packet填满后,会将这个packet放进dataqueue队列
3. 进入dataqueue队列的packet会被另一个线程(DataStreammer)取出然后发送到datanode



#### crate
```java
FSDataOutputStream fsDataOutputStream = fs.create(new Path("/input"));
return create(f, true);
return create(f, overwrite,
                  getConf().getInt(IO_FILE_BUFFER_SIZE_KEY,
                      IO_FILE_BUFFER_SIZE_DEFAULT),
                  getDefaultReplication(f),
                  getDefaultBlockSize(f));
return create(f, overwrite, bufferSize, replication, blockSize, null);
return this.create(f, FsCreateModes.applyUMask(
        FsPermission.getFileDefault(), FsPermission.getUMask(getConf())),
        overwrite, bufferSize, replication, blockSize, progress);

/**
*DistributeFileSystem.java
*/
@Override
  public FSDataOutputStream create(Path f, FsPermission permission,
      boolean overwrite, int bufferSize, short replication, long blockSize,
      Progressable progress) throws IOException {
    return this.create(f, permission,
        overwrite ? EnumSet.of(CreateFlag.CREATE, CreateFlag.OVERWRITE)
            : EnumSet.of(CreateFlag.CREATE), bufferSize, replication,
        blockSize, progress, null);
  }

return this.create(f, permission,
        overwrite ? EnumSet.of(CreateFlag.CREATE, CreateFlag.OVERWRITE)
            : EnumSet.of(CreateFlag.CREATE), bufferSize, replication,
        blockSize, progress, null);

final DFSOutputStream dfsos = dfs.create(getPathName(p), permission,
            cflags, replication, blockSize, progress, bufferSize,
            checksumOpt);
        return dfs.createWrappedOutputStream(dfsos, statistics);
->create->create->create_>newStreamForCreate
final DFSOutputStream result = DFSOutputStream.newStreamForCreate(this,
        src, masked, flag, createParent, replication, blockSize, progress,
        dfsClientConf.createChecksum(checksumOpt),
        getFavoredNodesStr(favoredNodes), ecPolicyName);

  static DFSOutputStream newStreamForCreate(DFSClient dfsClient, String src,
      FsPermission masked, EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize, Progressable progress,
      DataChecksum checksum, String[] favoredNodes, String ecPolicyName)
      throws IOException {
    try (TraceScope ignored =
             dfsClient.newPathTraceScope("newStreamForCreate", src)) {
      HdfsFileStatus stat = null;
    shouldRetry = false;
          stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
              new EnumSetWritable<>(flag), createParent, replication,
              blockSize, SUPPORTED_CRYPTO_VERSIONS, ecPolicyName);
          break;
      final DFSOutputStream out;
        out = new DFSStripedOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes);
        out = new DFSOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes, true);
      out.start();
      return out;
    }
}


//NN代理调用create方法
stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
              new EnumSetWritable<>(flag), createParent, replication,
              blockSize, SUPPORTED_CRYPTO_VERSIONS, ecPolicyName);

@Override // ClientProtocol
public HdfsFileStatus create(String src, FsPermission masked,
      String clientName, EnumSetWritable<CreateFlag> flag,
      boolean createParent, short replication, long blockSize,
      CryptoProtocolVersion[] supportedVersions, String ecPolicyName)
      throws IOException {
    checkNNStartup();
    String clientMachine = getClientMachine();    
    namesystem.checkOperation(OperationCategory.WRITE);
    CacheEntryWithPayload cacheEntry = RetryCache.waitForCompletion(retryCache, null);
    if (cacheEntry != null && cacheEntry.isSuccess()) {
      return (HdfsFileStatus) cacheEntry.getPayload();
    }

    HdfsFileStatus status = null;
    try {
      PermissionStatus perm = new PermissionStatus(getRemoteUser()
          .getShortUserName(), null, masked);
      status = namesystem.startFile(src, perm, clientName, clientMachine,
          flag.get(), createParent, replication, blockSize, supportedVersions,
          ecPolicyName, cacheEntry != null);
    } finally {
      RetryCache.setState(cacheEntry, status != null, status);
    }

    metrics.incrFilesCreated();
    metrics.incrCreateFileOps();
    return status;
}

status = namesystem.startFile(src, perm, clientName, clientMachine,
          flag.get(), createParent, replication, blockSize, supportedVersions,
          ecPolicyName, cacheEntry != null);

status = startFileInt(src, permissions, holder, clientMachine, flag,
          createParent, replication, blockSize, supportedVersions, ecPolicyName,
          logRetryCache);

stat = FSDirWriteFileOp.startFile(this, iip, permissions, holder,
            clientMachine, flag, createParent, replication, blockSize, feInfo,
            toRemoveBlocks, shouldReplicate, ecPolicyName, logRetryCache);

iip = addFile(fsd, parent, iip.getLastLocalName(), permissions,
          replication, blockSize, holder, clientMachine, shouldReplicate,
          ecPolicyName);

INodeFile newNode = newINodeFile(fsd.allocateNewInodeId(), permissions,
          modTime, modTime, replicationFactor, ecPolicyID, preferredBlockSize,
          blockType);
newiip = fsd.addINode(existing, newNode, permissions.getPermission());
//创建过程完成
->
//构造一个新的输出流来创建一个文件。
out = new DFSStripedOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes);

out.start();->getStreamer().start();
DataStreamer.java run(){}
//等待数据队列,DataStreamer.java:1948:queuePacket dataQueue.notifyAll
dataQueue.wait(timeout);
//dataQueue有数据后
one = dataQueue.getFirst(); // regular data packet

setPipeline(nextBlockOutputStream());
      nextBlockOutputStream(){
      //查找块
      lb = locateFollowingBlock(
          excluded.length > 0 ? excluded : null, oldBlock);

      // Connect to first DataNode in the list.
      success = createBlockOutputStream(nodes, nextStorageTypes, nextStorageIDs,0L, false);}
          s = createSocketForPipeline(nodes[0], nodes.length, dfsClient);

      new Sender(out).writeBlock(blockCopy, nodeStorageTypes[0], accessToken,
            dfsClient.clientName, nodes, nodeStorageTypes, null, bcs,
            nodes.length, block.getNumBytes(), bytesSent, newGS,
            checksum4WriteBlock, cachingStrategy.get(), isLazyPersistFile,
            (targetPinnings != null && targetPinnings[0]), targetPinnings,
            nodeStorageIDs[0], nodeStorageIDs);

private LocatedBlock locateFollowingBlock(DatanodeInfo[] excluded,
      ExtendedBlock oldBlock) throws IOException {
    return DFSOutputStream.addBlock(excluded, dfsClient, src, oldBlock,
        stat.getFileId(), favoredNodes, addBlockFlags);
}

//NameNodeRpcServer机架感知获取块存储位置
return dfsClient.namenode.addBlock(src, dfsClient.clientName, prevBlock,
            excludedNodes, fileId, favoredNodes, allocFlags);
LocatedBlock locatedBlock = namesystem.getAdditionalBlock(src, fileId,
        clientName, previous, excludedNodes, favoredNodes, addBlockFlags);
//验证块信息
r = FSDirWriteFileOp.validateAddBlock(this, pc, src, fileId, clientName,
                                            previous, onRetryBlock);
//为要分配的新块选择目标。                                        
DatanodeStorageInfo[] targets = FSDirWriteFileOp.chooseTargetForNewBlock(
        blockManager, src, excludedNodes, favoredNodes, flags, r);                                          
// choose targets for the new block to be allocated.
//Choose target datanodes for creating a new block.        
return bm.chooseTarget4NewBlock(src, r.numTargets, clientNode,
                                    excludedNodesSet, r.blockSize,
                                    favoredNodesList, r.storagePolicyID,
                                    r.blockType, r.ecPolicy, flags);

final DatanodeStorageInfo[] targets = blockplacement.chooseTarget(src,
        numOfReplicas, client, excludedNodes, blocksize, 
        favoredDatanodeDescriptors, storagePolicy, flags);

return chooseTarget(src, numOfReplicas, writer, 
        new ArrayList<DatanodeStorageInfo>(numOfReplicas), false,
        excludedNodes, blocksize, storagePolicy, flags);
return chooseTarget(numOfReplicas, writer, chosenNodes, returnChosenNodes,
        excludedNodes, blocksize, storagePolicy, flags, null);

  /** This is the implementation. */
  private DatanodeStorageInfo[] chooseTarget(...) {
    ...
   localNode = chooseTarget(numOfReplicas, writer,
          excludedNodeCopy, blocksize, maxNodesPerRack, results,
          avoidStaleNodes, storagePolicy,
          EnumSet.noneOf(StorageType.class), results.isEmpty(), sTypes);
          ...
  }

  writer = chooseTargetInOrder(numOfReplicas, writer, excludedNodes, blocksize,
          maxNodesPerRack, results, avoidStaleNodes, newBlock, storageTypes);


 protected Node chooseTargetInOrder(...)throws NotEnoughReplicasException {
    final int numOfResults = results.size();
    if (numOfResults == 0) {
      //选择本地
      DatanodeStorageInfo storageInfo = chooseLocalStorage(writer,
          excludedNodes, blocksize, maxNodesPerRack, results, avoidStaleNodes,
          storageTypes, true);
      writer = (storageInfo != null) ? storageInfo.getDatanodeDescriptor(): null;
      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    final DatanodeDescriptor dn0 = results.get(0).getDatanodeDescriptor();
    if (numOfResults <= 1) {
      //选择远程机架
      chooseRemoteRack(1, dn0, excludedNodes, blocksize, maxNodesPerRack,
          results, avoidStaleNodes, storageTypes);
      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    if (numOfResults <= 2) {
      final DatanodeDescriptor dn1 = results.get(1).getDatanodeDescriptor();
      //如果dn0和dn1都在同一机架,选择另一个机架
      if (clusterMap.isOnSameRack(dn0, dn1)) {
        chooseRemoteRack(1, dn0, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      } else if (newBlock){
        //选择同机架的另一节点
        chooseLocalRack(dn1, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      } else {
        chooseLocalRack(writer, excludedNodes, blocksize, maxNodesPerRack,
            results, avoidStaleNodes, storageTypes);
      }
      if (--numOfReplicas == 0) {
        return writer;
      }
    }
    chooseRandom(numOfReplicas, NodeBase.ROOT, excludedNodes, blocksize,
        maxNodesPerRack, results, avoidStaleNodes, storageTypes);
    return writer;
  }

//机架选完回到setPipeline(nextBlockOutputStream())中的nextBlockOutputStream()方法中的
//createBlockOutputStream方法下
//connects to the first datanode in the pipeline
// Returns true if success, otherwise return failure.
//获取输入输出流,往出写输出流,获取应答信息输入流
createBlockOutputStream(){
   blockReplyStream = new DataInputStream(unbufIn);
     // send the request
        new Sender(out).writeBlock(blockCopy, nodeStorageTypes[0], accessToken,
            dfsClient.clientName, nodes, nodeStorageTypes, null, bcs,
            nodes.length, block.getNumBytes(), bytesSent, newGS,
            checksum4WriteBlock, cachingStrategy.get(), isLazyPersistFile,
            (targetPinnings != null && targetPinnings[0]), targetPinnings,
            nodeStorageIDs[0], nodeStorageIDs);
      // receive ack for connect
        BlockOpResponseProto resp = BlockOpResponseProto.parseFrom(
            PBHelperClient.vintPrefixed(blockReplyStream));
        Status pipelineStatus = resp.getStatus();

}

public void writeBlock() throws IOException {  
    send(out, Op.WRITE_BLOCK, proto.build());
}


private static void send(final DataOutputStream out, final Op opcode,
      final Message proto) throws IOException {
    LOG.trace("Sending DataTransferOp {}: {}",
        proto.getClass().getSimpleName(), proto);
    op(out, opcode);
    proto.writeDelimitedTo(out);
    out.flush();
}

private static void send(final DataOutputStream out, final Op opcode,
      final Message proto) throws IOException {
    LOG.trace("Sending DataTransferOp {}: {}",
        proto.getClass().getSimpleName(), proto);
    op(out, opcode);
    proto.writeDelimitedTo(out);
    out.flush();
}
  private static void op(final DataOutput out, final Op op) throws IOException {
    out.writeShort(DataTransferProtocol.DATA_TRANSFER_VERSION);
    op.write(out);
  }


//发送的数据由DataNode中的DataXceiverServer接收
DataXceiverServer.java:141
run(){
  new Daemon(datanode.threadGroup,Dataceiver.create(peer,datanode,false)).start();
}
Dataceiver.java:222 
run(){
  //获取对方发送的类型,OP.WRITE_BLOCK
  op = readOp();  
  processOp(op);
}

protected final void processOp(Op op) throws IOException {
    switch(op) {
    case WRITE_BLOCK:
      opWriteBlock(in);
      break;
    }
}
private void opWriteBlock(DataInputStream in) throws IOException {
   writeBlock(......);
}
DataXceiver.java:667
763 setCurrentBlockReceiver(getBlockReceiver(block, storageType, in,
            peer.getRemoteAddressString(),
            peer.getLocalAddressString(),
            stage, latestGenerationStamp, minBytesRcvd, maxBytesRcvd,
            clientname, srcDataNode, datanode, requestedChecksum,
            cachingStrategy, allowLazyPersist, pinning, storageId));
 BlockReceiver getBlockReceiver(){
   return new BlockReceiver()
 }
BlockReceiver(){
      switch (stage) {
        case PIPELINE_SETUP_CREATE:
          replicaHandler = datanode.data.createRbw(storageType, storageId,
              block, allowLazyPersist);
          datanode.notifyNamenodeReceivingBlock(
              block, replicaHandler.getReplica().getStorageUuid());
          break;
      }
}
FsDatasetSpi.java:1373 createRbw
public ReplicaHandler createRbw(){
       // First try to place the block on a transient volume.
        ref = volumes.getNextTransientVolume(b.getNumBytes());
}
//写完后回到DataXceiver.java的writeBlock方法,
if (targets.length > 0) {
    //往下一个节点继续发送
     new Sender(mirrorOut).writeBlock(originalBlock, targetStorageTypes[0],
                blockToken, clientname, targets, targetStorageTypes,
                srcDataNode, stage, pipelineSize, minBytesRcvd, maxBytesRcvd,
                latestGenerationStamp, requestedChecksum, cachingStrategy,
                allowLazyPersist, targetPinnings[0], targetPinnings,
                targetStorageId, targetStorageIds);
}


//查看DataStreamer的应答反馈
//Initialize for data streaming,创建ResponseProcessor
//处理来自datanodes的响应,响应到达溢出ackQueue
initDataStreaming();

//// Processes responses from the datanodes.  A packet is removed
// from the ackQueue when its response arrives.
ResponseProcessor.java: run(){
   lastAckedSeqno = seqno;
            pipelineRecoveryCount = 0;
            //响应到达移除ackQueue中的一个
            ackQueue.removeFirst();
            packetSendTime.remove(seqno);
            dataQueue.notifyAll();
            one.releaseBuffer(byteArrayManager);
}

//回到DataStreamer.java run(){
    ...
    initDataStreaming();
    ...
    //发送数据dataQueue移除一个,在响应队列ackQueue增加一个,待响应到达,将备份在响应队列的移除
    dataQueue.removeFirst();    
    ackQueue.addLast(one);
    packetSendTime.put(one.getSeqno(), Time.monotonicNow());
    dataQueue.notifyAll();
    ...
    // write out data to remote datanode
    try (TraceScope ignored = dfsClient.getTracer().
    newScope("DataStreamer#writeTo", spanId)) {
    one.writeTo(blockStream);

}





```
### write流程
```java
fsDataOutputStream.write("hello world".getBytes());
  write(b, 0, b.length);
   write(b[off + i]);
    out.write(b);
FSOutputSummer.java:79
flushBuffer();
flushBuffer(false, true);
protected synchronized int flushBuffer(boolean keep,
      boolean flushPartial) throws IOException {
    int bufLen = count;
    int partialLen = bufLen % sum.getBytesPerChecksum();
    int lenToFlush = flushPartial ? bufLen : bufLen - partialLen;
    if (lenToFlush != 0) {
      //DFSOutputStream
      writeChecksumChunks(buf, 0, lenToFlush);
      if (!flushPartial || keep) {
        count = partialLen;
        System.arraycopy(buf, bufLen - count, buf, 0, count);
      } else {
        count = 0;
      }
    }
    // total bytes left minus unflushed bytes left
    return count - (bufLen - lenToFlush);
  }
writeChecksumChunks(buf, 0, lenToFlush);

private void writeChecksumChunks(byte b[], int off, int len)
  throws IOException {
    sum.calculateChunkedSums(b, off, len, checksum, 0);
    TraceScope scope = createWriteTraceScope();
    try {
      for (int i = 0; i < len; i += sum.getBytesPerChecksum()) {
        int chunkLen = Math.min(sum.getBytesPerChecksum(), len - i);
        int ckOffset = i / sum.getBytesPerChecksum() * getChecksumSize();
        writeChunk(b, off + i, chunkLen, checksum, ckOffset,
            getChecksumSize());
      }
    } finally {
      if (scope != null) {
        scope.close();
      }
    }
  }
//writeChunk
//DFSOutputStream.java:421
@Override
  protected synchronized void writeChunk(byte[] b, int offset, int len,
      byte[] checksum, int ckoff, int cklen) throws IOException {
    writeChunkPrepare(len, ckoff, cklen);

    currentPacket.writeChecksum(checksum, ckoff, cklen);
    currentPacket.writeData(b, offset, len);
    currentPacket.incNumChunks();
    getStreamer().incBytesCurBlock(len);

    // If packet is full, enqueue it for transmission
    if (currentPacket.getNumChunks() == currentPacket.getMaxChunks() ||
        getStreamer().getBytesCurBlock() == blockSize) {
      /********************************/
      enqueueCurrentPacketFull();
    }
  }
enqueueCurrentPacketFull();

synchronized void enqueueCurrentPacketFull() throws IOException {
    LOG.debug("enqueue full {}, src={}, bytesCurBlock={}, blockSize={},"
            + " appendChunk={}, {}", currentPacket, src, getStreamer()
            .getBytesCurBlock(), blockSize, getStreamer().getAppendChunk(),
        getStreamer());
    enqueueCurrentPacket();
    adjustChunkBoundary();
    endBlock();
}

void enqueueCurrentPacket() throws IOException {
    getStreamer().waitAndQueuePacket(currentPacket);
    currentPacket = null;
}

void waitAndQueuePacket(DFSPacket packet) throws IOException {
    synchronized (dataQueue) {
      try {
        // If queue is full, then wait till we have enough space
        boolean firstWait = true;
        try {
            ......
            try {
              dataQueue.wait();
            } catch (InterruptedException e) {
            ......
            }
          }
        } finally {
          Span span = Tracer.getCurrentSpan();
          if ((span != null) && (!firstWait)) {
            span.addTimelineAnnotation("end.wait");
          }
        }
        checkClosed();
        /***************************************************/
        queuePacket(packet);
      } catch (ClosedChannelException ignored) {
      }
    }
  }

queuePacket(packet);
/*Put a packet to the data queue*/
void queuePacket(DFSPacket packet) {
    synchronized (dataQueue) {
      if (packet == null) return;
      packet.addTraceParent(Tracer.getCurrentSpanId());
      dataQueue.addLast(packet);
      lastQueuedSeqno = packet.getSeqno();
      LOG.debug("Queued {}, {}", packet, this);
      /*******************/
      dataQueue.notifyAll();
    }
}
```

```java
//hello
public static void main(String[] args){}

```