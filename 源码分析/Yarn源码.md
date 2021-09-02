## Yarn源码
## job提交流程()
```
boolean result = job.waitForCompletion(true);
submit();
return submitter.submitJobInternal(Job.this, cluster);
    checkSpecs(job);
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
    JobID jobId = submitClient.getNewJobID();
    job.setJobID(jobId);
    Path submitJobDir = new Path(jobStagingArea, jobId.toString());
    copyAndConfigureFiles(job, submitJobDir);
    Path submitJobFile = JobSubmissionFiles.getJobConfPath(submitJobDir);
    int maps = writeSplits(job, submitJobDir);
    writeConf(conf, submitJobFile);
    //实际提交job,本地用LocalJobRunner,集群用YARNRunner
    status = submitClient.submitJob(
          jobId, submitJobDir.toString(), job.getCredentials());
```

## YARNRunner.java
```
@Override
  public JobStatus submitJob(JobID jobId, String jobSubmitDir, Credentials ts)
  throws IOException, InterruptedException {
    //RPC历史服务器
    addHistoryToken(ts);
    //构建启动 MR AM(MRAPPMASTER) 所需的所有信息。
    //
    // Setup LocalResources
    // Setup security tokens
    // Setup ContainerLaunchContext for AM container
    // Set up the ApplicationSubmissionContext
    // Setup the AM ResourceRequests
    // set labels for the AM container requests if present
    // set labels for the Job containers
    ApplicationSubmissionContext appContext =
      createApplicationSubmissionContext(conf, jobSubmitDir, ts);

    //提交到资源管理器ResourceManager
    try {
      ApplicationId applicationId =
          resMgrDelegate.submitApplication(appContext);//->YarnClientImpl.java:304  rmClient.submitApplication(request);
      //获取提交响应
      ApplicationReport appMaster = resMgrDelegate
          .getApplicationReport(applicationId);
      String diagnostics =
          (appMaster == null ?
              "application report is null" : appMaster.getDiagnostics());
      if (appMaster == null
          || appMaster.getYarnApplicationState() == YarnApplicationState.FAILED
          || appMaster.getYarnApplicationState() == YarnApplicationState.KILLED) {
        throw new IOException("Failed to run job : " +
            diagnostics);
      }
      return clientCache.getClient(jobId).getJobStatus(jobId);
    } catch (YarnException e) {
      throw new IOException(e);
    }
  }
```

## createApplicationSubmissionContext
```

  public ApplicationSubmissionContext createApplicationSubmissionContext(
      Configuration jobConf, String jobSubmitDir, Credentials ts)
      throws IOException {
    ApplicationId applicationId = resMgrDelegate.getApplicationId();

    // 本地资源相关路径
    Map<String, LocalResource> localResources =
        setupLocalResources(jobConf, jobSubmitDir);

    // Setup security tokens
    DataOutputBuffer dob = new DataOutputBuffer();
    ts.writeTokenStorageToStream(dob);
    ByteBuffer securityTokens =
        ByteBuffer.wrap(dob.getData(), 0, dob.getLength());

    //启动MRAppMaster和运行Container的相关命令和参数
    // Setup ContainerLaunchContext for AM container
    //"/bin/java" -Djava.io.tmpdir=
    //vargs.add(MRJobConfig.APPLICATION_MASTER_CLASS);
    //public static final String APPLICATION_MASTER_CLASS ="org.apache.hadoop.mapreduce.v2.app.MRAppMaster";
    //设置启动的java类MRAppMaster
    List<String> vargs = setupAMCommand(jobConf);
    ContainerLaunchContext amContainer = setupContainerLaunchContextForAM(
        jobConf, localResources, securityTokens, vargs);

    String regex = conf.get(MRJobConfig.MR_JOB_SEND_TOKEN_CONF);
    if (regex != null && !regex.isEmpty()) {
      setTokenRenewerConf(amContainer, conf, regex);
    }


    Collection<String> tagsFromConf =
        jobConf.getTrimmedStringCollection(MRJobConfig.JOB_TAGS);

    // Set up the ApplicationSubmissionContext
    ApplicationSubmissionContext appContext =
        recordFactory.newRecordInstance(ApplicationSubmissionContext.class);
    appContext.setApplicationId(applicationId);                // ApplicationId
    appContext.setQueue(                                       // Queue name
        jobConf.get(JobContext.QUEUE_NAME,
        YarnConfiguration.DEFAULT_QUEUE_NAME));
    // add reservationID if present
    ReservationId reservationID = null;
    try {
      reservationID =
          ReservationId.parseReservationId(jobConf
              .get(JobContext.RESERVATION_ID));
    } catch (NumberFormatException e) {
      // throw exception as reservationid as is invalid
      String errMsg =
          "Invalid reservationId: " + jobConf.get(JobContext.RESERVATION_ID)
              + " specified for the app: " + applicationId;
      LOG.warn(errMsg);
      throw new IOException(errMsg);
    }
    if (reservationID != null) {
      appContext.setReservationID(reservationID);
      LOG.info("SUBMITTING ApplicationSubmissionContext app:" + applicationId
          + " to queue:" + appContext.getQueue() + " with reservationId:"
          + appContext.getReservationID());
    }
    appContext.setApplicationName(                             // Job name
        jobConf.get(JobContext.JOB_NAME,
        YarnConfiguration.DEFAULT_APPLICATION_NAME));
    appContext.setCancelTokensWhenComplete(
        conf.getBoolean(MRJobConfig.JOB_CANCEL_DELEGATION_TOKEN, true));
    appContext.setAMContainerSpec(amContainer);         // AM Container
    appContext.setMaxAppAttempts(
        conf.getInt(MRJobConfig.MR_AM_MAX_ATTEMPTS,
            MRJobConfig.DEFAULT_MR_AM_MAX_ATTEMPTS));

    // Setup the AM ResourceRequests
    List<ResourceRequest> amResourceRequests = generateResourceRequests();
    appContext.setAMContainerResourceRequests(amResourceRequests);

    // set labels for the AM container requests if present
    String amNodelabelExpression = conf.get(MRJobConfig.AM_NODE_LABEL_EXP);
    if (null != amNodelabelExpression
        && amNodelabelExpression.trim().length() != 0) {
      for (ResourceRequest amResourceRequest : amResourceRequests) {
        amResourceRequest.setNodeLabelExpression(amNodelabelExpression.trim());
      }
    }
    // set labels for the Job containers
    appContext.setNodeLabelExpression(jobConf
        .get(JobContext.JOB_NODE_LABEL_EXP));
    //public static final String MR_APPLICATION_TYPE = "MAPREDUCE";
    appContext.setApplicationType(MRJobConfig.MR_APPLICATION_TYPE);
    if (tagsFromConf != null && !tagsFromConf.isEmpty()) {
      appContext.setApplicationTags(new HashSet<>(tagsFromConf));
    }
    //设置job优先级
    String jobPriority = jobConf.get(MRJobConfig.PRIORITY);
    if (jobPriority != null) {
      int iPriority;
      try {
        iPriority = TypeConverter.toYarnApplicationPriority(jobPriority);
      } catch (IllegalArgumentException e) {
        iPriority = Integer.parseInt(jobPriority);
      }
      appContext.setPriority(Priority.newInstance(iPriority));
    }

    return appContext;
```

## submitApplication
```
  ApplicationId applicationId =
          resMgrDelegate.submitApplication(appContext);//->YarnClientImpl.java:304  rmClient.submitApplication(request);
return client.submitApplication(appContext);
YarnClientImpl.java:286

public ApplicationId
      submitApplication(ApplicationSubmissionContext appContext)
          throws YarnException, IOException {
    ApplicationId applicationId = appContext.getApplicationId();    
    SubmitApplicationRequest request =
        Records.newRecord(SubmitApplicationRequest.class);
    request.setApplicationSubmissionContext(appContext);

//TODO: YARN-1763:Handle RM failovers during the submitApplication call.
    rmClient.submitApplication(request);
    ...
}

ClientRMService.java:554
public SubmitApplicationResponse submitApplication(
      SubmitApplicationRequest request) throws YarnException, IOException {
    ApplicationSubmissionContext submissionContext = request
        .getApplicationSubmissionContext();

    if (submissionContext.getQueue() == null) {
      submissionContext.setQueue(YarnConfiguration.DEFAULT_QUEUE_NAME);
    }
    if (submissionContext.getApplicationName() == null) {
      submissionContext.setApplicationName(
          YarnConfiguration.DEFAULT_APPLICATION_NAME);
    }
    if (submissionContext.getApplicationType() == null) {
      submissionContext
        .setApplicationType(YarnConfiguration.DEFAULT_APPLICATION_TYPE);
    } else {
      if (submissionContext.getApplicationType().length() > YarnConfiguration.APPLICATION_TYPE_LENGTH) {
        submissionContext.setApplicationType(submissionContext
          .getApplicationType().substring(0,
            YarnConfiguration.APPLICATION_TYPE_LENGTH));
      }
    }
```
## 启动MRAppMaster
```
//MRAppMaster.java:1640 main()
public static void main(String[] args) {
...
//初始化并启动AppMaster
initAndStartAppMaster(appMaster, conf, jobUserName);
...
}
appMaster.init(conf);
appMaster.start();
```
## appMaster.init(conf);
```
serviceInit(config);
MRAppMaster.java:280
@Override
  protected void serviceInit(final Configuration conf) throws Exception {
    createJobClassLoader(conf);
    initJobCredentialsAndUGI(conf);
    //创建调度器
    dispatcher = createDispatcher();
}
```
## appMaster.start();
```
serviceStart();
MRAppMater.java:1217
@Override
  protected void serviceStart() throws Exception {
    ...
// All components have started, start the job.
      startJobs();
      ...
}
protected void startJobs() {
    /** create a job-start event to get this ball rolling */
    JobEvent startJobEvent = new JobStartEvent(job.getID(),
        recoveredJobStartTime);
    /** send the job-start event. this triggers the job execution. */
    dispatcher.getEventHandler().handle(startJobEvent);
  }
->
AsyncDispatcher.java:GenericEventHandler.java:245
public void handle(Event event) {
//Yarn放进调度器中的时间队列
eventQueue.put(event);
}
```

## YarnChild
```
public static void main(String[] args) throws Throwable {
taskFinal.run(job, umbilical); // run the task
}
-> MapTask.run->runNewMapper();->mapper.run(mapperContext);->用户写的继承Mapper接口的run方法
-> ReduceTask.run->runNewReducer()->reducer.run(reducerContext);->用户的写继承Reducer接口的run方法 
```

```
if (isMapTask()) {
      // If there are no reducers then there won't be any sort. Hence the map 
      // phase will govern the entire attempt's progress.
      if (conf.getNumReduceTasks() == 0) {
        mapPhase = getProgress().addPhase("map", 1.0f);
      } else {
        // If there are reducers then the entire attempt's progress will be 
        // split between the map phase (67%) and the sort phase (33%).
        //资源比例66.7,map进度条先到66.7然后到100%,最后在开启Reduce阶段
        //如果低于66.7map阶段出问题,大于66.7,map逻辑没有问题
        mapPhase = getProgress().addPhase("map", 0.667f);
        sortPhase  = getProgress().addPhase("sort", 0.333f);
      }
    }
```