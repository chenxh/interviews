
## TaskScheduler


### Pool
继承 Schedulable 接口， 实现了类似于 yarn 调度队列的调度池功能。调度池Pool通过调度算法对每个TaskSet进行调度，并将被调度的TaskSet交给TaskSchedulerImpl进行资源调度

### TaskSetManager
TaskSetManager对TaskSet进行管理，包括任务推断、Task本地性，并对Task进行资源分配。

**调度池与推断执行**
解决问题：某个子任务执行缓慢，导致整个作业执行慢的问题。 
方法：在检测到有任务执行缓慢，并且符合推断执行的标准时，重新启动一个同样的任务执行，最后以先执行完成的任务结果为准。

spark 中 Pool和TaskSetManager中对推断执行的操作分为两类：一类是可推断任务的检测与缓存；另一类是从缓存中找到可推断任务进行推断执行。

**TaskSetManager#checkSpeculatableTasks()**
实现对可推断任务的检测与缓存。TaskScheduler 定时调用此方法。
```
//如果 1. 处于 zombie 状态 2. 是 Barrier 3. 任务数量只有一个切推断执行的时间没有定义。 那么不需要推断运行
if (isZombie || isBarrier || (numTasks == 1 && !speculationTaskDurationThresOpt.isDefined)) {
      return false
    }
    var foundTasks = false
    logDebug("Checking for speculative tasks: minFinished = " + minFinishedForSpeculation)

    // It's possible that a task is marked as completed by the scheduler, then the size of
    // `successfulTaskDurations` may not equal to `tasksSuccessful`. Here we should only count the
    // tasks that are submitted by this `TaskSetManager` and are completed successfully.
    val numSuccessfulTasks = successfulTaskDurations.size()
    if (numSuccessfulTasks >= minFinishedForSpeculation) {
      val time = clock.getTimeMillis()
      val medianDuration = successfulTaskDurations.median
      val threshold = max(speculationMultiplier * medianDuration, minTimeToSpeculation)
      // TODO: Threshold should also look at standard deviation of task durations and have a lower
      // bound based on that.
      logDebug("Task length threshold for speculation: " + threshold)
      for (tid <- runningTasksSet) {
        var speculated = checkAndSubmitSpeculatableTask(tid, time, threshold)
        if (!speculated && executorDecommissionKillInterval.isDefined) {
          val taskInfo = taskInfos(tid)
          val decomState = sched.getExecutorDecommissionState(taskInfo.executorId)
          if (decomState.isDefined) {
            // Check if this task might finish after this executor is decommissioned.
            // We estimate the task's finish time by using the median task duration.
            // Whereas the time when the executor might be decommissioned is estimated using the
            // config executorDecommissionKillInterval. If the task is going to finish after
            // decommissioning, then we will eagerly speculate the task.
            val taskEndTimeBasedOnMedianDuration = taskInfos(tid).launchTime + medianDuration
            val executorDecomTime = decomState.get.startTime + executorDecommissionKillInterval.get
            val canExceedDeadline = executorDecomTime < taskEndTimeBasedOnMedianDuration
            if (canExceedDeadline) {
              speculated = checkAndSubmitSpeculatableTask(tid, time, 0)
            }
          }
        }
        foundTasks |= speculated
      }
    } else if (speculationTaskDurationThresOpt.isDefined && speculationTasksLessEqToSlots) {
      val time = clock.getTimeMillis()
      val threshold = speculationTaskDurationThresOpt.get
      logDebug(s"Tasks taking longer time than provided speculation threshold: $threshold")
      for (tid <- runningTasksSet) {
        foundTasks |= checkAndSubmitSpeculatableTask(tid, time, threshold)
      }
    }
    foundTasks

```

1）如果 1. 处于 zombie 状态 2. 是 Barrier 3. 任务数量只有一个切推断执行的时间没有定义。 那么不需要推断运行。
2）如果执行成功的Task数量（tasksSuccessful）大于等于minFinishedForSpeculation。
    * 找出执行成功的任务successfulTaskDurations 中执行时间处于中间的时间 medianDuration。
    * 计算进行推断的最小时间（threshold），即SPECULATION_MULTIPLIER与 median-Duration的乘积和minTimeToSpeculation中的最大值。
    * 遍历 runningTasksSet ，调用 checkAndSubmitSpeculatableTask() 检测和提交推断任务  。
    * checkAndSubmitSpeculatableTask 中先判断任务是否满足推断条件， 包括任务还未执行成功、copiesRunning为 ：任务运行时间大于threshold和 speculatableTasks（已提交的推断任务）中没有。这是先提交任务到speculatableTasks，然后***提交SpeculativeTaskSubmitted事件***。//todo，后续怎么运行？
    * 如果没有推断运行成功，判断 task 的executor 是否处于 Decommission 状态，如果是，也调用 checkAndSubmitSpeculatableTask 检测和提交推断运行任务。
4）如果执行成功的Task数量（tasksSuccessful）小于 minFinishedForSpeculation， 但是定义了参数```spark.speculation.task.duration.threshold```， 并且 task数量小于一个执行器的最大任务数。使用参数定义得时间推断运行。





