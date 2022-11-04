---
title: starrocks离线构建
date: 2022-01-19 19:04:11
tags:
  - linux
  - cpp
categories:
  - sre
---
### 问题描述

+ 入口文件->StarRocksFE.java
```
.....
feServer.start();
httpServer.start();
qeService.start();
```
主要是初始化配置和启动服务，分别是mysql server端口、thrift server端口、http端口

+ mysq服务启动->QeService.java
```
public void start() throws IOException {
        if (!mysqlServer.start()) {
            LOG.error("mysql server start failed");
            System.exit(-1);
        }
        LOG.info("QE service start.");
    }
```

+ MysqlServer.java
```
// start MySQL protocol service
// return true if success, otherwise false
public boolean start() {
    if (scheduler == null) {
        LOG.warn("scheduler is NULL.");
        return false;
    }

    // open server socket
    try {
        serverChannel = ServerSocketChannel.open();
        serverChannel.socket().bind(new InetSocketAddress("0.0.0.0", port), 2048);
        serverChannel.configureBlocking(true);
    } catch (IOException e) {
        LOG.warn("Open MySQL network service failed.", e);
        return false;
    }

    // start accept thread
    listener = ThreadPoolManager.newDaemonCacheThreadPool(1, "MySQL-Protocol-Listener", true);
    running = true;
    listenerFuture = listener.submit(new Listener());

    return true;
}
```

+ Listener 监听线程，实现Runnable接口
```
public void run() {
    while (running && serverChannel.isOpen()) {
        SocketChannel clientChannel;
        try {
            clientChannel = serverChannel.accept();
            if (clientChannel == null) {
                continue;
            }
            // submit this context to scheduler
            ConnectContext context = new ConnectContext(clientChannel);
            // Set catalog here.
            context.setCatalog(Catalog.getCurrentCatalog());
            if (!scheduler.submit(context)) {
                LOG.warn("Submit one connect request failed. Client=" + clientChannel.toString());
                // clear up context
                context.cleanup();
            }
        } catch (IOException e) {
            // ClosedChannelException
            // AsynchronousCloseException
            // ClosedByInterruptException
            // Other IOException, for example "to many open files" ...
            LOG.warn("Query server encounter exception.", e);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e1) {
                // Do nothing
            }
        } catch (Throwable e) {
            // NotYetBoundException
            // SecurityException
            LOG.warn("Query server failed when calling accept.", e);
        }
    }
}
``` 

+ submit提交连接的上下文给线程池
```
// submit one MysqlContext to this scheduler.
// return true, if this connection has been successfully submitted, otherwise return false.
// Caller should close ConnectContext if return false.
public boolean submit(ConnectContext context) {
    if (context == null) {
        return false;
    }

    context.setConnectionId(nextConnectionId.getAndAdd(1));
    // no necessary for nio.
    if (context instanceof NConnectContext) {
        return true;
    }
    if (executor.submit(new LoopHandler(context)) == null) {
        LOG.warn("Submit one thread failed.");
        return false;
    }
    return true;
}
```

+ LoopHandler.java (实现Runnable接口)
```
public void run() {
    try {
        // Set thread local info
        context.setThreadLocalInfo();
        context.setConnectScheduler(ConnectScheduler.this);
        // authenticate check failed.
        if (!MysqlProto.negotiate(context)) {
            return;
        }

        if (registerConnection(context)) {
            MysqlProto.sendResponsePacket(context);
        } else {
            context.getState().setError("Reach limit of connections");
            MysqlProto.sendResponsePacket(context);
            return;
        }

        context.setStartTime();
        ConnectProcessor processor = new ConnectProcessor(context);
        processor.loop();
    } catch (Exception e) {
        // for unauthrorized access such lvs probe request, may cause exception, just log it in debug level
        if (context.getCurrentUserIdentity() != null) {
            LOG.warn("connect processor exception because ", e);
        } else {
            LOG.debug("connect processor exception because ", e);
        }
    } finally {
        unregisterConnection(context);
        context.cleanup();
    }
}
```

+ ConnectProcessor.java -> loop
```
public void loop() {
    while (!ctx.isKilled()) {
        try {
            processOnce();
        } catch (Exception e) {
            // TODO(zhaochun): something wrong
            LOG.warn("Exception happened in one seesion(" + ctx + ").", e);
            ctx.setKilled();
            break;
        }
    }
}
```

+ ConnectProcessor.java -> processOnce
```
// handle one process
public void processOnce() throws IOException {
    // set status of query to OK.
    ctx.getState().reset();
    executor = null;

    // reset sequence id of MySQL protocol
    final MysqlChannel channel = ctx.getMysqlChannel();
    channel.setSequenceId(0);
    // read packet from channel
    try {
        packetBuf = channel.fetchOnePacket();
        if (packetBuf == null) {
            throw new IOException("Error happened when receiving packet.");
        }
    } catch (AsynchronousCloseException e) {
        // when this happened, timeout checker close this channel
        // killed flag in ctx has been already set, just return
        return;
    }

    // dispatch
    dispatch();
    // finalize
    finalizeCommand();

    ctx.setCommand(MysqlCommand.COM_SLEEP);
}
```

+ ConnectProcessor.java -> dispatch
```
int code = packetBuf.get();
MysqlCommand command = MysqlCommand.fromCode(code);
....
ctx.setCommand(command);
....

switch (command) {
    case COM_INIT_DB:
        handleInitDb();
        break;
    case COM_QUIT:
        handleQuit();
        break;
    case COM_QUERY:
        handleQuery();
        ctx.setStartTime();
        break;
    case COM_FIELD_LIST:
        handleFieldList();
        break;
    case COM_CHANGE_USER:
        handleChangeUser();
        break;
    case COM_RESET_CONNECTION:
        handleResetConnnection();
        break;
    case COM_PING:
        handlePing();
        break;
    default:
        ctx.getState().setError("Unsupported command(" + command + ")");
        LOG.warn("Unsupported command(" + command + ")");
        break;
}
```
+ ConnectProcessor.java -> handleQuery
```
....
List<StatementBase> stmts = analyze(originStmt);
for (int i = 0; i < stmts.size(); ++i) {
    ctx.getState().reset();
    if (i > 0) {
        ctx.resetRetureRows();
        ctx.setQueryId(UUIDUtil.genUUID());
    }
    parsedStmt = stmts.get(i);
    parsedStmt.setOrigStmt(new OriginStatement(originStmt, i));

    executor = new StmtExecutor(ctx, parsedStmt);
    ctx.setExecutor(executor);

    ctx.setIsLastStmt(i == stmts.size() - 1);

    executor.execute();

    // do not execute following stmt when current stmt failed, this is consistent with mysql server
    if (ctx.getState().getStateType() == QueryState.MysqlStateType.ERR) {
        break;
    }

    if (i != stmts.size() - 1) {
        // NOTE: set serverStatus after executor.execute(),
        //      because when execute() throws exception, the following stmt will not execute
        //      and the serverStatus with MysqlServerStatusFlag.SERVER_MORE_RESULTS_EXISTS will
        //      cause client error: Packet sequence number wrong
        ctx.getState().serverStatus |= MysqlServerStatusFlag.SERVER_MORE_RESULTS_EXISTS;
        finalizeCommand();
    }
}
....

```

+ ConnectProcessor.java -> analyze
```
// analyze the origin stmt and return multi-statements
private List<StatementBase> analyze(String originStmt) throws AnalysisException {
    LOG.debug("the originStmts are: {}", originStmt);
    // Parse statement with parser generated by CUP&FLEX
    SqlScanner input = new SqlScanner(new StringReader(originStmt), ctx.getSessionVariable().getSqlMode());
    SqlParser parser = new SqlParser(input);
    try {
        return SqlParserUtils.getMultiStmts(parser);
    } catch (Error e) {
        throw new AnalysisException("Please check your sql, we meet an error when parsing.", e);
    } catch (AnalysisException e) {
        LOG.warn("origin_stmt: " + originStmt + "; Analyze error message: " + parser.getErrorMsg(originStmt), e);
        String errorMessage = parser.getErrorMsg(originStmt);
        if (errorMessage == null) {
            throw e;
        } else {
            throw new AnalysisException(errorMessage, e);
        }
    } catch (Exception e) {
        // TODO(lingbin): we catch 'Exception' to prevent unexpected error,
        // should be removed this try-catch clause future.
        LOG.warn("origin_stmt: " + originStmt + "; exception: ", e);
        String errorMessage = e.getMessage();
        if (errorMessage == null) {
            throw new AnalysisException("Internal Error");
        } else {
            throw new AnalysisException("Internal Error: " + errorMessage);
        }
    }
}
```

+ StmtExecutor.java -> execute
```
........
} else if (parsedStmt instanceof DdlStmt) {
    handleDdlStmt();
} 
......
```

+ StmtExecutor.java -> handleDdlStmt
```
private void handleDdlStmt() {
  try {
      DdlExecutor.execute(context.getCatalog(), (DdlStmt) parsedStmt);
      context.getState().setOk();
  } catch (QueryStateException e) {
      if (e.getQueryState().getStateType() != MysqlStateType.OK) {
          LOG.warn("DDL statement(" + originStmt.originStmt + ") process failed.", e);
      }
      context.setState(e.getQueryState());
  } catch (UserException e) {
      LOG.warn("DDL statement(" + originStmt.originStmt + ") process failed.", e);
      // Return message to info client what happened.
      context.getState().setError(e.getMessage());
  } catch (Exception e) {
      // Maybe our bug
      LOG.warn("DDL statement(" + originStmt.originStmt + ") process failed.", e);
      context.getState().setError("Unexpected exception: " + e.getMessage());
  }
}
```
+ DdlExecutor.java -> execute
```
........
else if (ddlStmt instanceof LoadStmt) {
    LoadStmt loadStmt = (LoadStmt) ddlStmt;
    EtlJobType jobType = loadStmt.getEtlJobType();
    if (jobType == EtlJobType.UNKNOWN) {
        throw new DdlException("Unknown load job type");
    }
    if (jobType == EtlJobType.HADOOP && Config.disable_hadoop_load) {
        throw new DdlException("Load job by hadoop cluster is disabled."
                + " Try using broker load. See 'help broker load;'");
    }

    catalog.getLoadManager().createLoadJobFromStmt(loadStmt);
} 
........
```

+ LoadManager.java -> createLoadJobFromStmt
```
public void createLoadJobFromStmt(LoadStmt stmt) throws DdlException {
  Database database = checkDb(stmt.getLabel().getDbName());
  long dbId = database.getId();
  LoadJob loadJob = null;
  writeLock();
  try {
      checkLabelUsed(dbId, stmt.getLabel().getLabelName());
      if (stmt.getBrokerDesc() == null && stmt.getResourceDesc() == null) {
          throw new DdlException("LoadManager only support the broker and spark load.");
      }
      if (loadJobScheduler.isQueueFull()) {
          throw new DdlException(
                  "There are more than " + Config.desired_max_waiting_jobs + " load jobs in waiting queue, "
                          + "please retry later.");
      }
      loadJob = BulkLoadJob.fromLoadStmt(stmt);
      createLoadJob(loadJob);
  } finally {
      writeUnlock();
  }
  Catalog.getCurrentCatalog().getEditLog().logCreateLoadJob(loadJob);

  // The job must be submitted after edit log.
  // It guarantee that load job has not been changed before edit log.
  loadJobScheduler.submitJob(loadJob);
}
```

通过 `createLoadJobFromStmt` 创建load任务
+ LoadJobScheduler.java -> process
注意：LoadJobScheduler 继承自 MasterDaemon，MasterDaemon 继承自 Daemon，
Daemon继承自Thread，重载了run方法，里面有一个loop，主要执行runOneCycle
MasterDaemon 又重写了 runOneCycle，执行 runAfterCatalogReady 函数
LoadJobScheduler 又重写了 runAfterCatalogReady 主要就是干process处理，里面是一个死循环，不断从LinkedBlockingQueue类型的needScheduleJobs里出栈取要珍惜的job
```
while (true) {
  // take one load job from queue
  LoadJob loadJob = needScheduleJobs.poll();
  if (loadJob == null) {
      return;
  }

  // schedule job
  try {
      loadJob.execute();
  } 
```
+ LoadJob.java -> execute
```
/**
  * create pending task for load job and add pending task into pool
  * if job has been cancelled, this step will be ignored
  *
  * @throws LabelAlreadyUsedException  the job is duplicated
  * @throws BeginTransactionException  the limit of load job is exceeded
  * @throws AnalysisException          there are error params in job
  * @throws DuplicatedRequestException
  */
public void execute() throws LabelAlreadyUsedException, BeginTransactionException, AnalysisException,
        DuplicatedRequestException, LoadException {
    writeLock();
    try {
        unprotectedExecute();
    } finally {
        writeUnlock();
    }
}
```


+ SparkLoadJob.java -> unprotectedExecuteJob
```
protected void unprotectedExecuteJob() throws LoadException {
    // create pending task
    LoadTask task = new SparkLoadPendingTask(this, fileGroupAggInfo.getAggKeyToFileGroups(),
            sparkResource, brokerDesc);
    task.init();
    idToTasks.put(task.getSignature(), task);
    submitTask(Catalog.getCurrentCatalog().getPendingLoadTaskScheduler(), task);
}
```

+ SparkLoadPendingTask.java -> init

 ```
public void init() throws LoadException {
    createEtlJobConf();
}
```
+ LoadTask -> exec
```
@Override
protected void exec() {
    boolean isFinished = false;
    try {
        // execute pending task
        executeTask();
        // callback on pending task finished
        callback.onTaskFinished(attachment);
        isFinished = true;
    } catch (UserException e) {
        failMsg.setMsg(e.getMessage() == null ? "" : e.getMessage());
        LOG.warn(new LogBuilder(LogKey.LOAD_JOB, callback.getCallbackId())
                .add("error_msg", "Failed to execute load task").build(), e);
    } catch (Exception e) {
        failMsg.setMsg(e.getMessage() == null ? "" : e.getMessage());
        LOG.warn(new LogBuilder(LogKey.LOAD_JOB, callback.getCallbackId())
                .add("error_msg", "Unexpected failed to execute load task").build(), e);
    } finally {
        if (!isFinished) {
            // callback on pending task failed
            callback.onTaskFailed(signature, failMsg);
        }
    }
}
```
+ SparkLoadPendingTask.java -> executeTask
```
void executeTask() throws LoadException {
    LOG.info("begin to execute spark pending task. load job id: {}", loadJobId);
    submitEtlJob();
}
```

+ SparkLoadPendingTask.java -> submitEtlJob
```
private void submitEtlJob() throws LoadException {
    SparkPendingTaskAttachment sparkAttachment = (SparkPendingTaskAttachment) attachment;
    // retry different output path
    etlJobConfig.outputPath = EtlJobConfig.getOutputPath(resource.getWorkingDir(), dbId, loadLabel, signature);
    sparkAttachment.setOutputPath(etlJobConfig.outputPath);

    // handler submit etl job
    SparkEtlJobHandler handler = new SparkEtlJobHandler();
    handler.submitEtlJob(loadJobId, loadLabel, etlJobConfig, resource, brokerDesc, sparkLoadAppHandle,
            sparkAttachment);
    LOG.info("submit spark etl job success. load job id: {}, attachment: {}", loadJobId, sparkAttachment);
}
```

+ SparkEtlJobHandler.java -> submitEtlJob
```
public void submitEtlJob(long loadJobId, String loadLabel, EtlJobConfig etlJobConfig, SparkResource resource,
                             BrokerDesc brokerDesc, SparkLoadAppHandle handle, SparkPendingTaskAttachment attachment)
      throws LoadException {
  // delete outputPath
  // init local dir
  // prepare dpp archive
  SparkLauncher launcher = new SparkLauncher(envs);
  // master      |  deployMode
  // ------------|-------------
  // yarn        |  cluster
  // spark://xx  |  client
  launcher.setMaster(resource.getMaster())
          .setDeployMode(resource.getDeployMode().name().toLowerCase())
          .setAppResource(appResourceHdfsPath)
          .setMainClass(SparkEtlJob.class.getCanonicalName())
          .setAppName(String.format(ETL_JOB_NAME, loadLabel))
          .setSparkHome(sparkHome)
          .addAppArgs(jobConfigHdfsPath)
          .redirectError();

  // spark configs

  // start app
  State state = null;
  String appId = null;
  String logPath = null;
  String errMsg = "start spark app failed. error: ";
  try {
      Process process = launcher.launch();
      handle.setProcess(process);
      if (!FeConstants.runningUnitTest) {
          SparkLauncherMonitor.LogMonitor logMonitor = SparkLauncherMonitor.createLogMonitor(handle);
          logMonitor.setSubmitTimeoutMs(GET_APPID_TIMEOUT_MS);
          logMonitor.setRedirectLogPath(logFilePath);
          logMonitor.start();
          try {
              logMonitor.join();
          } catch (InterruptedException e) {
              logMonitor.interrupt();
              throw new LoadException(errMsg + e.getMessage());
          }
      }
      appId = handle.getAppId();
      state = handle.getState();
      logPath = handle.getLogPath();
  } catch (IOException e) {
      LOG.warn(errMsg, e);
      throw new LoadException(errMsg + e.getMessage());
  }
..........
  // success
  attachment.setAppId(appId);
  attachment.setHandle(handle);
}
```

+ SparkLoadJob.java -> onTaskFinished
```
public void onTaskFinished(TaskAttachment attachment) {
    if (attachment instanceof SparkPendingTaskAttachment) {
        onPendingTaskFinished((SparkPendingTaskAttachment) attachment);
    }
}

private void onPendingTaskFinished(SparkPendingTaskAttachment attachment) {
    writeLock();
    try {
        // check if job has been cancelled
        if (isTxnDone()) {
            LOG.warn(new LogBuilder(LogKey.LOAD_JOB, id)
                    .add("state", state)
                    .add("error_msg", "this task will be ignored when job is: " + state)
                    .build());
            return;
        }

        if (finishedTaskIds.contains(attachment.getTaskId())) {
            LOG.warn(new LogBuilder(LogKey.LOAD_JOB, id)
                    .add("task_id", attachment.getTaskId())
                    .add("error_msg", "this is a duplicated callback of pending task "
                            + "when broker already has loading task")
                    .build());
            return;
        }

        // add task id into finishedTaskIds
        finishedTaskIds.add(attachment.getTaskId());

        sparkLoadAppHandle = attachment.getHandle();
        appId = attachment.getAppId();
        etlOutputPath = attachment.getOutputPath();

        executeEtl();
        // log etl state
        unprotectedLogUpdateStateInfo();
    } finally {
        writeUnlock();
    }
}

```