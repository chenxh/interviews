## spark 任务运行全流程
master，worker，driver 和 executor 之间怎么协调来完成整个 job 的运行。

## 示例代码

```
 spark.sparkContext.parallelize(0 until numMappers, numMappers).flatMap { p =>
      val ranGen = new Random
      val arr1 = new Array[(Int, Array[Byte])](numKVPairs)
      for (i <- 0 until numKVPairs) {
        val byteArr = new Array[Byte](valSize)
        ranGen.nextBytes(byteArr)
        arr1(i) = (ranGen.nextInt(Int.MaxValue), byteArr)
      }
      arr1
    }.cache()
    // Enforce that everything has been calculated and in cache
    pairs1.count()

```

## driver 流程

在函数式语法运行时已经构造了RDD血缘关系。


count() 是个 action 算子， 执行时调用

```
sc.runJob(this, Utils.getIteratorSize _).sum 
->  SparkContext.runJob(rdd, func, 0 until rdd.partitions.length)
->  DAGScheduler.runJob(rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties)
-> DAGScheduler.submitJob()
-> DAGSchedulerEventProcessLoop.post(JobSubmitted)
-> DAGScheduler#handleJobSubmitted()  # JobSubmitted 事件的处理方法
-> DAGScheduler.createResultStage() #创建finalStage
    -> DAGScheduler.getOrCreateShuffleMapStage() # 创建 ShuffleMapStage
    -> new ResultStage(id, rdd, func, partitions, parents, jobId,
        callSite, resourceProfile.id) # 创建最后的 ResultStage,也就是 
-> DAGScheduler.submitStage(finalStage) # 提交 stage， 
    -> DAGScheduler.submitMissingTasks(stage, jobId.get) //提交任务
        ->  new ShuffleMapTask/ new ResultTask
        ->  TaskSchedulerImpl.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties,
        stage.resourceProfileId))

//任务执行完成后
DAGScheduler#taskEnd()
  -> postEvent()
  -> DAGScheduler#handleTaskCompletion
```



### DAGScheduler 流程

JobSubmitted的处理方法，org.apache.spark.scheduler.DAGScheduler#handleJobSubmitted
```
 try {
      // New stage creation may throw an exception if, for example, jobs are run on a
      // HadoopRDD whose underlying HDFS files have been deleted.
      finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite) //创建finalStage
    } catch {
       //处理代码
    }
    // Job submitted, clear internal data.
    barrierJobIdToNumTasksCheckFailures.remove(jobId)

    val job = new ActiveJob(jobId, finalStage, callSite, listener, properties) //创建ActiveJob
    clearCacheLocs()
    logInfo("Got job %s (%s) with %d output partitions".format(
      job.jobId, callSite.shortForm, partitions.length))
    logInfo("Final stage: " + finalStage + " (" + finalStage.name + ")")
    logInfo("Parents of final stage: " + finalStage.parents)
    logInfo("Missing parents: " + getMissingParentStages(finalStage))

    val jobSubmissionTime = clock.getTimeMillis()
    jobIdToActiveJob(jobId) = job
    activeJobs += job
    finalStage.setActiveJob(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    listenerBus.post(
      SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos,
        Utils.cloneProperties(properties)))
    submitStage(finalStage) 

```

* 创建 finalStage ： 调用createResultStage方法创建ResultStage。创建Stage的过程可能发生异常（比如运行在HadoopRDD上的任务所依赖的底层HDFS文件被删除了），当异常发生时需要主动调用JobWaiter的jobFailed方法。createResultStage方法在创建ResultStage的过程中会引起创建一系列Stage的连锁反应，Job与这些Stage的关系将被放入jobIdToStageIds中。

* 创建ActiveJob。ActiveJob用来表示已经激活的Job，即被DAGScheduler接收处理的Job。

* 向LiveListenerBus投递SparkListenerJobStart事件，进而引发所有关注此事件的监听器执行相应的操作。

* 调用submitStage方法提交ResultStage。


**createResultStage**

```
val (shuffleDeps, resourceProfiles) = getShuffleDependenciesAndResourceProfiles(rdd)
val resourceProfile = mergeResourceProfilesForStage(resourceProfiles)
checkBarrierStageWithDynamicAllocation(rdd)
checkBarrierStageWithNumSlots(rdd, resourceProfile)
checkBarrierStageWithRDDChainPattern(rdd, partitions.toSet.size)


val parents = getOrCreateParentStages(shuffleDeps, jobId)
val id = nextStageId.getAndIncrement()
val stage = new ResultStage(id, rdd, func, partitions, parents, jobId,
    callSite, resourceProfile.id)
stageIdToStage(id) = stage
updateJobIdStageIdMaps(jobId, stage)
stage
```

1. ***Spark 为了支持深度学习而引入的屏障调度器***， 前面的check都是屏障调度器的检测。
2. getOrCreateParentStages 获取所有父Stage的列表，父Stage主要是宽依赖（ShuffleDependency）对应的Stage，此列表内的Stage包含以下几种：
   1. 当前RDD的直接或间接的依赖是ShuffleDependency且已经注册过的Stage
   2. 当前RDD的直接或间接的依赖是ShuffleDependency且没有注册过Stage的。对于这种ShuffleDependency，则根据ShuffleDependency中的RDD，找到它的直接或间接的依赖是ShuffleDependency且没有注册过Stage的所有ShuffleDependency，为它们创建并注册Stage
   3. 当前RDD的直接或间接的依赖是ShuffleDependency且没有注册过Stage的。为此ShuffleDependency创建并注册Stage。
3. 生成Stage的身份标识，并创建ResultStage。
4. 将ResultStage注册到stageIdToStage中。
5. 调用updateJobIdStageIdMaps方法，更新Job的身份标识与ResultStage及其所有祖先的映射关系。

**getOrCreateParentStages**
```
private def getOrCreateParentStages(shuffleDeps: HashSet[ShuffleDependency[_, _, _]],
    firstJobId: Int): List[Stage] = {
  shuffleDeps.map { shuffleDep =>
    getOrCreateShuffleMapStage(shuffleDep, firstJobId)
  }.toList
}

private def getOrCreateShuffleMapStage(
  shuffleDep: ShuffleDependency[_, _, _],
  firstJobId: Int): ShuffleMapStage = {
shuffleIdToMapStage.get(shuffleDep.shuffleId) match {
  case Some(stage) =>  //1. 当前RDD的直接或间接的依赖是ShuffleDependency且已经注册过的Stage
    stage

  case None => //当前RDD的直接或间接的依赖是ShuffleDependency且没有注册过Stage的。
    // Create stages for all missing ancestor shuffle dependencies.
    getMissingAncestorShuffleDependencies(shuffleDep.rdd).foreach { dep =>
      // Even though getMissingAncestorShuffleDependencies only returns shuffle dependencies
      // that were not already in shuffleIdToMapStage, it's possible that by the time we
      // get to a particular dependency in the foreach loop, it's been added to
      // shuffleIdToMapStage by the stage creation process for an earlier dependency. See
      // SPARK-13902 for more information.
      if (!shuffleIdToMapStage.contains(dep.shuffleId)) {
        createShuffleMapStage(dep, firstJobId)  //创建ShuffleMapStage
      }
    }
    // Finally, create a stage for the given shuffle dependency.
    createShuffleMapStage(shuffleDep, firstJobId)
}
}
```

1）如果已经创建了ShuffleDependency对应的ShuffleMapStage，则直接返回此ShuffleMapStage。
2）否则调用getMissingAncestorShuffleDependencies方法找到所有还未创建过ShuffleMapStage的祖先ShuffleDependency，并调用createShuffleMapStage方法,创建ShuffleMapStage并注册。最后还会为当前ShuffleDependency调用方法createShuffleMapStage创建ShuffleMapStage并注册。

**createShuffleMapStage**
```
 val rdd = shuffleDep.rdd
    val (shuffleDeps, resourceProfiles) = getShuffleDependenciesAndResourceProfiles(rdd)
    val resourceProfile = mergeResourceProfilesForStage(resourceProfiles)
    checkBarrierStageWithDynamicAllocation(rdd)
    checkBarrierStageWithNumSlots(rdd, resourceProfile)
    checkBarrierStageWithRDDChainPattern(rdd, rdd.getNumPartitions)
    val numTasks = rdd.partitions.length
    val parents = getOrCreateParentStages(shuffleDeps, jobId)
    val id = nextStageId.getAndIncrement()
    val stage = new ShuffleMapStage(
      id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep, mapOutputTracker,
      resourceProfile.id)

    stageIdToStage(id) = stage
    shuffleIdToMapStage(shuffleDep.shuffleId) = stage
    updateJobIdStageIdMaps(jobId, stage)

    if (!mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
      // Kind of ugly: need to register RDDs with the cache and map output tracker here
      // since we can't do it in the RDD constructor because # of partitions is unknown
      logInfo(s"Registering RDD ${rdd.id} (${rdd.getCreationSite}) as input to " +
        s"shuffle ${shuffleDep.shuffleId}")
      mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length,
        shuffleDep.partitioner.numPartitions)
    }
    stage
```
createShuffleMapStage的执行步骤如下。
1）创建ShuffleMapStage，具体如下。
  1. 获取ShuffleDependency的rdd属性，作为将要创建的ShuffleMapStage的rdd。
  2. 调用rdd（即RDD）的partitions方法得到rdd的分区数组，此分区数组的长度即为要创建的ShuffleMapStage的numTasks（Task数量）。这说明了***map任务数量与RDD的各个分区一一对应***。
  3. 调用getOrCreateParentStages方法获得要创建ShuffleMapStage的所有父Stage（即parents）。
  4. 生成将要创建的ShuffleMapStage的身份标识。
  5. 创建ShuffleMapStage。
2）更新刚创建的ShuffleMapStage的映射关系。具体如下：
1. 将新创建的ShuffleMapStage的身份标识与ShuffleMapStage的映射关系放入stageId-ToStage中。
2. 将shuffleId与ShuffleMapStage的映射关系放入shuffleIdToMapStage中。
3. 调用updateJobIdStageIdMaps方法更新Job的身份标识与Shuffle-MapStage及其所有祖先的映射关系。

3） 在 MapOutputTrackerMaster 中 注册或者更新MapStatus


**submitStage 提交 stage 运行**
handleJobSubmitted方法中处理Job提交的最后一步是调用submit-Stage方法提交ResultStage。
```
val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug(s"submitStage($stage (name=${stage.name};" +
        s"jobs=${stage.jobIds.toSeq.sorted.mkString(",")}))")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
```
1）调用activeJobForStage方法找到使用当前Stage的所有ActiveJob的身份标识。
2）如果存在Stage对应的ActiveJob的身份标识并且当前Stage还未提交（即waiting-Stages、runningStages、failedStages中都不包含当前Stage），则执行以下操作：
1. 调用getMissingParentStages方法获取当前Stage的所有未提交的父Stage。
2. 如果不存在未提交的父Stage，那么调用submitMissingTasks方法提交当前Stage所有未提交的Task。
3. 如果存在未提交的父Stage，那么将多次调用submitStage方法提交所有未提交的父Stage，并且将当前Stage加入waitingStages集合中（这表示当前Stage必须等待所有父Stage执行完成）。

3）如果不存在Stage对应的 ActiveJob 的身份标识，则调用abortStage方法终止依赖于当前Stage的所有Job。

**submitMissingTasks 提交对应 stage 的为计算的 task**

```
// 1)
  stage match {
      case sms: ShuffleMapStage if stage.isIndeterminate && !sms.isAvailable =>
        mapOutputTracker.unregisterAllMapAndMergeOutput(sms.shuffleDep.shuffleId)
        sms.shuffleDep.newShuffleMergeState()
      case _ =>
    }

    //2) Figure out the indexes of partition ids to compute. 
    val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()

    // 3.Use the scheduling pool, job group, description, etc. from an ActiveJob associated
    // with this Stage
    val properties = jobIdToActiveJob(jobId).properties
    addPySparkConfigsToProperties(stage, properties)
    
    //4)
    runningStages += stage
    // 5) SparkListenerStageSubmitted should be posted before testing whether tasks are
    // serializable. If tasks are not serializable, a SparkListenerStageCompleted event
    // will be posted, which should always come after a corresponding SparkListenerStageSubmitted
    // event.
    //5）
    stage match {
      case s: ShuffleMapStage =>
        outputCommitCoordinator.stageStart(stage = s.id, maxPartitionId = s.numPartitions - 1)
        // Only generate merger location for a given shuffle dependency once.
        if (s.shuffleDep.shuffleMergeAllowed) {
          if (!s.shuffleDep.isShuffleMergeFinalizedMarked) {
            prepareShuffleServicesForShuffleMapStage(s)
          } else {
            // Disable Shuffle merge for the retry/reuse of the same shuffle dependency if it has
            // already been merge finalized. If the shuffle dependency was previously assigned
            // merger locations but the corresponding shuffle map stage did not complete
            // successfully, we would still enable push for its retry.
            s.shuffleDep.setShuffleMergeAllowed(false)
            logInfo(s"Push-based shuffle disabled for $stage (${stage.name}) since it" +
              " is already shuffle merge finalized")
          }
        }
      case s: ResultStage =>
        outputCommitCoordinator.stageStart(
          stage = s.id, maxPartitionId = s.rdd.partitions.length - 1)
    }
    //6）
    val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
      stage match {
        case s: ShuffleMapStage =>
          partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
        case s: ResultStage =>
          partitionsToCompute.map { id =>
            val p = s.partitions(id)
            (id, getPreferredLocs(stage.rdd, p))
          }.toMap
      }
    } catch {
      case NonFatal(e) =>
        stage.makeNewStageAttempt(partitionsToCompute.size)
        listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo,
          Utils.cloneProperties(properties)))
        abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
        runningStages -= stage
        return
    }

    //7)
    stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)

    // If there are tasks to execute, record the submission time of the stage. Otherwise,
    // post the even without the submission time, which indicates that this stage was
    // skipped.
    if (partitionsToCompute.nonEmpty) {
      stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
    }
    listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo,
      Utils.cloneProperties(properties)))

    // TODO: Maybe we can keep the taskBinary in Stage to avoid serializing it multiple times.
    // Broadcasted binary for the task, used to dispatch tasks to executors. Note that we broadcast
    // the serialized copy of the RDD and for each task we will deserialize it, which means each
    // task gets a different copy of the RDD. This provides stronger isolation between tasks that
    // might modify state of objects referenced in their closures. This is necessary in Hadoop
    // where the JobConf/Configuration object is not thread-safe.
    var taskBinary: Broadcast[Array[Byte]] = null
    var partitions: Array[Partition] = null
    //8) 
    try {
      // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
      // For ResultTask, serialize and broadcast (rdd, func).
      var taskBinaryBytes: Array[Byte] = null
      // taskBinaryBytes and partitions are both effected by the checkpoint status. We need
      // this synchronization in case another concurrent job is checkpointing this RDD, so we get a
      // consistent view of both variables.
      RDDCheckpointData.synchronized {
        taskBinaryBytes = stage match {
          case stage: ShuffleMapStage =>
            JavaUtils.bufferToArray(
              closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
          case stage: ResultStage =>
            JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
        }

        partitions = stage.rdd.partitions
      }

      if (taskBinaryBytes.length > TaskSetManager.TASK_SIZE_TO_WARN_KIB * 1024) {
        logWarning(s"Broadcasting large task binary with size " +
          s"${Utils.bytesToString(taskBinaryBytes.length)}")
      }
      //9）
      taskBinary = sc.broadcast(taskBinaryBytes)
    } catch {
      // In the case of a failure during serialization, abort the stage.
      case e: NotSerializableException =>
        abortStage(stage, "Task not serializable: " + e.toString, Some(e))
        runningStages -= stage

        // Abort execution
        return
      case e: Throwable =>
        abortStage(stage, s"Task serialization failed: $e\n${Utils.exceptionString(e)}", Some(e))
        runningStages -= stage

        // Abort execution
        return
    }

    //10）
    val tasks: Seq[Task[_]] = try {
      val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
      stage match {
        case stage: ShuffleMapStage =>
          stage.pendingPartitions.clear()
          partitionsToCompute.map { id =>
            val locs = taskIdToLocations(id)
            val part = partitions(id)
            stage.pendingPartitions += id
            new ShuffleMapTask(stage.id, stage.latestInfo.attemptNumber,
              taskBinary, part, locs, properties, serializedTaskMetrics, Option(jobId),
              Option(sc.applicationId), sc.applicationAttemptId, stage.rdd.isBarrier())
          }

        case stage: ResultStage =>
          partitionsToCompute.map { id =>
            val p: Int = stage.partitions(id)
            val part = partitions(p)
            val locs = taskIdToLocations(id)
            new ResultTask(stage.id, stage.latestInfo.attemptNumber,
              taskBinary, part, locs, id, properties, serializedTaskMetrics,
              Option(jobId), Option(sc.applicationId), sc.applicationAttemptId,
              stage.rdd.isBarrier())
          }
      }
    } catch {
      case NonFatal(e) =>
        abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
        runningStages -= stage
        return
    }

    //11）
    if (tasks.nonEmpty) {
      logInfo(s"Submitting ${tasks.size} missing tasks from $stage (${stage.rdd}) (first 15 " +
        s"tasks are for partitions ${tasks.take(15).map(_.partitionId)})")
      taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties,
        stage.resourceProfileId))
    } else {
      // Because we posted SparkListenerStageSubmitted earlier, we should mark
      // the stage as completed here in case there are no tasks to run
      //12）
      markStageAsFinished(stage, None)

      stage match {
        case stage: ShuffleMapStage =>
          logDebug(s"Stage ${stage} is actually done; " +
              s"(available: ${stage.isAvailable}," +
              s"available outputs: ${stage.numAvailableOutputs}," +
              s"partitions: ${stage.numPartitions})")
          markMapStageJobsAsFinished(stage)
        case stage : ResultStage =>
          logDebug(s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})")
      }
      submitWaitingChildStages(stage)
    }
```

1）清空当前Stage的pendingPartitions。由于当前Stage的任务刚开始提交，所以需要清空便于记录需要计算的分区任务。
2）调用Stage的findMissingPartitions方法，找出当前Stage的所有分区中还没有完成计算的分区的索引。
3）获取ActiveJob的properties。properties包含了当前Job的调度、group、描述等属性信息。
4）将当前Stage加入runningStages集合中，即当前Stage已经处于运行状态。
5）调用 OutputCommitCoordinator 的stageStart方法（见代码清单5-93），启动对当前Stage的输出提交到HDFS的协调。
***OutputCommitCoordinator***
6）调用DAGScheduler的getPreferredLocs方法，获取partitions-ToCompute中的每一个分区的偏好位置。如果发生任何异常，则调用Stage的makeNew-StageAttempt方法开始一次新的Stage执行尝试，然后向listenerBus投递SparkListener-StageSubmitted事件。
7）调用Stage的makeNewStageAttempt方法开始Stage的执行尝试，并向listenerBus投递SparkListenerStageSubmitted事件。
8）如果当前Stage是ShuffleMapStage，那么对Stage的rdd和ShuffleDependency进行序列化；如果当前Stage是ResultStage，那么对Stage的rdd和对RDD的分区进行计算的函数func进行序列化。
9）调用SparkContext的broadcast方法广播上一步生成的序列化对象。
10）如果当前Stage是ShuffleMapStage，则为ShuffleMapStage的每一个分区创建一个ShuffleMapTask。如果当前Stage是ResultStage，则为ResultStage的每一个分区创建一个ResultTask。
11）如果第10）步创建了至少1个Task，那么将此Task处理的分区索引添加到Stage的pendingPartitions中，然后为这批Task创建TaskSet（即任务集合），并调用TaskScheduler的submitTasks方法提交此TaskSet。
12）如果第10）步没有创建任何Task，这意味着当前Stage没有Task需要提交执行，因此调用DAGScheduler的markStageAsFinished方法，将当前Stage标记为完成。然后调用submitWaitingChildStages方法（见代码清单7-44），提交当前Stage的子Stage。

**taskEnded**
上面的步骤中 DagScheduler 最终把 TaskSet 提交到 TaskScheduler 调度运行。 运行完成后 TaskSetManager 会调用 DagScheduler#taskEnded 方法通知任务完成（失败）。
taskEnded方法将向DAGSchedulerEventProcessLoop投递Completion-Event事件。DAGSchedulerEventProcessLoop接收到CompletionEvent事件后，将调用DAGScheduler的handleTaskCompletion方法。handleTaskCompletion方法中实现了针对不同执行状态的处理，这里只介绍对成功状态的处理。

对于ShuffleMapTask而言，需要将它的状态信息MapStatus追加到ShuffleMapStage的outputLocs缓存中；如果ShuffleMapStage的所有分区的ShuffleMapTask都执行成功了，那么将 ***需要把ShuffleMapStage的outputLocs缓存中的所有MapStatus注册到MapOutput-TrackerMaster的mapStatuses中，以便于下游Stage中的Task读取输入数据所在的位置信息*** ；如果某个ShuffleMapTask执行失败了，则需要重新提交ShuffleMapStage；如果ShuffleMapStage的所有ShuffleMapTask都执行成功了，还需要唤醒下游Stage的执行。
对于ResultTask而言，如果ResultStage中的所有ResultTask都执行成功了，则将ResultStage标记为成功，并 ***通知JobWaiter对各个ResultTask的执行结果进行收集，然后根据应用程序的需要进行最终的处理（如打印到控制台、输出到HDFS）***。
















