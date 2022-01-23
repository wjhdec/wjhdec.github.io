# spark源码学习（一） —— Executor


> 不了解spark运行方式而写出的app是没有灵魂的  ———— XXX

其实两年前就想写，但是由于比较懒，一直没动笔，今天还是写点吧。顺便练习一下五笔。
<!--more-->

## 1. 环境信息

spark版本：2.4
[源码位置](https://github.com/apache/spark/blob/v2.4.0)

## 2. Executor 简介

executor为spark的执行单元，主要由driver进行调度。此类主要在包 *org.apache.spark.executor* 中。

[Executor类源码位置](https://github.com/apache/spark/blob/v2.4.0/core/src/main/scala/org/apache/spark/executor/Executor.scala)

下面是Executor类的注释

```java
/**
 * Spark executor, backed by a threadpool to run tasks.
 *
 * This can be used with Mesos, YARN, and the standalone scheduler.
 * An internal RPC interface is used for communication with the driver,
 * except in the case of Mesos fine-grained mode.
 */
```

上面的大意是 executor 支持在线程池中执行 task。它能够支持 mesos，yarn及standalone模式。
用RPC接口和drive端进行连接。
由于没有用过 mesos，最下面说的 fine-grained 模式不知道是什么，有经验的同学可以补充一下

## 3. Executor 结构

### 3.1. object

```scala
private[spark] object Executor {
  // This is reserved for internal use by components that need to read task properties before a
  // task is fully deserialized. When possible, the TaskContext.getLocalProperty call should be
  // used instead.
  val taskDeserializationProps: ThreadLocal[Properties] = new ThreadLocal[Properties]
}
```

很简单，定义了一个线程内参数，用于记录执行信息。
private[spark] 说明此类只能在spark包内部调用。

### 3.2. class

#### 3.2.1. 构造函数

```scala
private[spark] class Executor(
    executorId: String,
    executorHostname: String,
    env: SparkEnv,
    userClassPath: Seq[URL] = Nil,
    isLocal: Boolean = false,
    uncaughtExceptionHandler: UncaughtExceptionHandler = new SparkUncaughtExceptionHandler)
  extends Logging {
    // 省略源码... ...
}
```

同样是spark包内的内部类。

#### 3.2.2. Executor 内部类

主要有2个内部类：

##### 3.2.2.1. TaskRunner

```scala
  class TaskRunner(
    execBackend: ExecutorBackend,
    private val taskDescription: TaskDescription)
      extends Runnable {/*省略源码*/}
```

主要功能为执行task

##### 3.2.2.2. TaskReaper

```scala
  private class TaskReaper(
      taskRunner: TaskRunner,
      val interruptThread: Boolean,
      val reason: String)
    extends Runnable { /*省略源码*/ }
```

监控关闭task用，默认不启用，如果启用需指明 *spark.task.reaper.enabled=true*

### 3.3. 主要内容

#### 3.3.1. 初始化执行

```scala
private val threadPool = { /*省略源码*/ }
/*省略源码*/
private val taskReaperPool = ThreadUtils.newDaemonCachedThreadPool("Task reaper")
```

线程池的创建，threadPool是执行task时的线程池，taskReaperPool 监督正在关闭和取消的task。

```scala
  private val executorPlugins: Seq[ExecutorPlugin] = { /*省略源码*/ }
```

获取 *spark.executor.plugins* 中设置的插件。我目前还没用过这个，就不做进一步的解释了……

```scala
  private val heartbeater = ThreadUtils.newDaemonSingleThreadScheduledExecutor("driver-heartbeater")
  private val heartbeatReceiverRef =
    RpcUtils.makeDriverRef(HeartbeatReceiver.ENDPOINT_NAME, conf, env.rpcEnv)
  private val HEARTBEAT_MAX_FAILURES = conf.getInt("spark.executor.heartbeat.maxFailures", 60)
  startDriverHeartbeater()
```

创建心跳，和 driver 交互

#### 3.3.2. 执行Task入口

```scala
def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {
    val tr = new TaskRunner(context, taskDescription)
    runningTasks.put(taskDescription.taskId, tr)
    threadPool.execute(tr)
  }
```

开启线程执行task

### 3.4. TaskRunner类

task的执行类，主要内容在 run() 方法中

比较重要的源码有：

```scala
val value = Utils.tryWithSafeFinally {
    val res = task.run(
      taskAttemptId = taskId,
      attemptNumber = taskDescription.attemptNumber,
      metricsSystem = env.metricsSystem)
    threwException = false
    res
  } {
    val releasedLocks = env.blockManager.releaseAllLocksForTask(taskId)
    val freedMemory = taskMemoryManager.cleanUpAllAllocatedMemory()

    if (freedMemory > 0 && !threwException) {
      val errMsg = s"Managed memory leak detected; size = $freedMemory bytes, TID = $taskId"
      if (conf.getBoolean("spark.unsafe.exceptionOnMemoryLeak", false)) {
        throw new SparkException(errMsg)
      } else {
        logWarning(errMsg)
      }
    }
    /*省略部分源码*/
  }
```

上面执行的源码分2部分

1. task执行过程
2. 执行完成或错误后的资源释放等

run部分剩下的内容大部分都是状态的更新，内容序列化等内容

```scala
val serializedResult: ByteBuffer = {
  if (maxResultSize > 0 && resultSize > maxResultSize) {
    logWarning(s"Finished $taskName (TID $taskId). Result is larger than maxResultSize " +
      s"(${Utils.bytesToString(resultSize)} > ${Utils.bytesToString(maxResultSize)}), " +
      s"dropping it.")
    ser.serialize(new IndirectTaskResult[Any](TaskResultBlockId(taskId), resultSize))
  } else if (resultSize > maxDirectResultSize) {
    val blockId = TaskResultBlockId(taskId)
    env.blockManager.putBytes(
      blockId,
      new ChunkedByteBuffer(serializedDirectResult.duplicate()),
      StorageLevel.MEMORY_AND_DISK_SER)
    logInfo(
      s"Finished $taskName (TID $taskId). $resultSize bytes result sent via BlockManager)")
    ser.serialize(new IndirectTaskResult[Any](blockId, resultSize))
  } else {
    logInfo(s"Finished $taskName (TID $taskId). $resultSize bytes result sent to driver")
    serializedDirectResult
  }
}
```

把获得结果和累加器等返回 driver

## 4. 题外话

写的很水，因为打字太费劲，就不多写了，以后再补充。说多了都是泪 … …
