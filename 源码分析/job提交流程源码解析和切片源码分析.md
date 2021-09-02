```
//job等待完成参数true 是否打印流程
boolean result = job.waitForCompletion(true);
    submit();
    //设置使用新的api
    setUseNewAPI();
    //连接,返回cluster
    connect();
        return new Cluster(getConfiguration());
            //初始化,判断是本地环境还是yarn集群运行环境
            initialize(jobTrackAddr, conf);
    //提交job
    submitter.submitJobInternal(Job.this, cluster);
        //检查output是否合法(是否设置,是否已存在)
        checkSpecs(job);
        //初始化暂存目录,
        //创建给集群提交数据的 Stag 路径
        Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
        //获取jobID,并穿件job路径
        JobID jobId = submitClient.getNewJobID();
        //拷贝jar到集群
        copyAndConfigureFiles(job, submitJobDir);
        Path submitJobFile = JobSubmissionFiles.getJobConfPath(submitJobDir);
        //计算切片,生成切片规划文件
        int maps = writeSplits(job, submitJobDir);
            maps = writeNewSplits(job, jobSubmitDir);
                List<InputSplit> splits = input.getSplits(job);
        //向stag路径写入xml配置文件
        writeConf(conf, submitJobFile);
        
        //提交Job,返回提交状态
        status = submitClient.submitJob(...)
```
### 切片源码解析
```
//块大小=切片大小
//本地环境默认块大小32m
//conf中默认maxSize没有所以maxSize为Long.MAX_VALUE
//minSize默认为1
//1.默认切片大小为blocksize
//2.maxsize小于blocksize,则选maxsize和minsize的最大值
splitSize = Math.max(minSize, Math.min(maxSize, blockSize));
//maxSize小于blockSize才有意义,minSize大于BlockSize才有意义
//((double) bytesRemaining)/splitSize > SPLIT_SLOP(1.1)
```