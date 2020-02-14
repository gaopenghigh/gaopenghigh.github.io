---
layout: post
title: "Mesos 源码学习(5) Slave Recovery"
date: 2016-12-07 18:00:00 +0800
categories: Mesos
toc: true
---


在 Slave 初始化的最后，会进行 recovery ：


```c++
void Slave::initialize()
{
  ...
  // Do recovery.
  async(&state::recover, metaDir, flags.strict)
    .then(defer(self(), &Slave::recover, lambda::_1))
    .then(defer(self(), &Slave::_recover))
    .onAny(defer(self(), &Slave::__recover, lambda::_1));
}
```

其中 `async()` 是 libprocess 提供的一种实现异步调用的方法。

## 调用 state::recover 从硬盘拿到 State

首先调用 `state::recover` 获得需要 recover 的信息，这些信息可能被 checkpointed 在硬盘上。
该方法定义和实现在 `src/slave/state.hpp` 和 `src/slave/state.cpp` 中。
大概做的事情就是从硬盘目录中读取 checkpoint 的内容，然后反序列化为一个 `State` 结构。

```c++
// This function performs recovery from the state stored at 'rootDir'.
// If the 'strict' flag is set, any errors encountered while
// recovering a state are considered fatal and hence the recovery is
// short-circuited and returns an error. There might be orphaned
// executors that need to be manually cleaned up. If 'strict' flag is
// not set, any errors encountered are considered non-fatal and the
// recovery continues by recovering as much of the state as possible,
// while increasing the 'errors' count. Note that 'errors' on a struct
// includes the 'errors' encountered recursively. In other words,
// 'State.errors' is the sum total of all recovery errors. If the
// machine has rebooted since the last slave run, None is returned.
Result<State> recover(const std::string& rootDir, bool strict);
```

## 调用 Slave::recover

该方法实现在 `src/slave/slave.cpp` 中，首先做一些对 State 的检查，
然后逐个恢复 SlaveInfo，Framwork，Fetcher 和 StatusUpdateManager：

```c++
Future<Nothing> Slave::recover(const Result<state::State>& state)
{
...
    info = slaveState.get().info.get(); // Recover the slave info.
	...
    Try<Nothing> recovered = Fetcher::recover(slaveState.get().id, flags);
    ...
    // Recover the frameworks.
    foreachvalue (const FrameworkState& frameworkState,
                  slaveState.get().frameworks) {
      recoverFramework(frameworkState);
    }
...
  return statusUpdateManager->recover(metaDir, slaveState)
    .then(defer(self(), &Slave::_recoverContainerizer, slaveState));
}
```

### Recover SlaveInfo

SlaveInfo 存在 `Slave` 类的 `info` 属性中，这里就简单地做一下赋值。

```c++
SlaveInfo info;
```

### Recover Fetcher

Fetcher 的 recover 比较简单，就是把 fetcher cache 清理掉。
代码在 `src/slave/containerizer/fetcher.cpp`。

### Recover Frameworks

`Slave` 类中有一个字段 `frameworks`，是一个 hashmap，保持在这个 slave 上有 executor 的所有的
Framework。定义是：

```c++
hashmap<FrameworkID, Framework*> frameworks;
```

Framework recover 实现在 `Slave::recoverFramework` 中，主要逻辑是：
1. 如果从 checkpoint 拿到的 state 中 executor 的数量为空，也就是之前没有 task 运行，那么就把
   Framework 的 work dir 和 meta dir 做一次垃圾回收，然后返回。
2. 根据 state 重新创建一个 Framework 实例，然后放在 Slave 的 `frameworks` 属性中。
3. 如果 state 中 executors 不为空，则调用 `framework->recoverExecutor` 方法恢复 executor。
4. 最后检查一下 framework 中恢复出来的 executors ，如果数量为空，那么就把这个 framework 移除。

```c++
void Slave::recoverFramework(const FrameworkState& state)
{
  LOG(INFO) << "Recovering framework " << state.id;

  if (state.executors.empty()) {
    // GC the framework work directory.
    garbageCollect(
        paths::getFrameworkPath(flags.work_dir, info.id(), state.id));

    // GC the framework meta directory.
    garbageCollect(
        paths::getFrameworkPath(metaDir, info.id(), state.id));

    return;
  }
  ...

  Framework* framework = new Framework(
      this, flags, frameworkInfo, pid);

  frameworks[framework->id()] = framework;

  ...
  // Now recover the executors for this framework.
  foreachvalue (const ExecutorState& executorState, state.executors) {
    framework->recoverExecutor(executorState);
  }

  // Remove the framework in case we didn't recover any executors.
  if (framework->executors.empty()) {
    removeFramework(framework);
  }
}
```

这里 `Framework` 是一个 struct ，同样实现在 `src/slave/slave.cpp` 中。

`Framework::recoverExecutor` 的主要逻辑是：
1. 一个 executor 可能 run 了很多次，而我们只需要恢复最近的一次，所以会先做一次 GC，清理除了最近一次
   run 的 executor work dir 和 executor meta dir
2. 根据 executorState 的 latest run，创建一个 `Executor` struct，状态为默认的 `REGISTERING`，
   然后恢复 executor process 的 PID。
   `Executor` struct 也同样实现在 `src/slave/slave.cpp` 中。
3. 调用 `executor->recoverTask` 对 executor 的每一个 task 进行恢复
4. 把这个 executor 放在 Framework 的 `executors` 属性里面，
   这是一个 hasmap：`hashmap<ExecutorID, Executor*> executors`，表示正在 running 的所有 executor
5. 如果发现这个 executor 现在已经 completed，则把它的状态置为 `Executor::TERMINATED`，然后做垃圾回收

```c++
void Framework::recoverExecutor(const ExecutorState& state)
{
  ...
  // We are only interested in the latest run of the executor!
  // So, we GC all the old runs.
  // NOTE: We don't schedule the top level executor work and meta
  // directories for GC here, because they will be scheduled when
  // the latest executor run terminates.
  const ContainerID& latest = state.latest.get();
  foreachvalue (const RunState& run, state.runs) {
    CHECK_SOME(run.id);
    const ContainerID& runId = run.id.get();
    if (latest != runId) {
      // GC the executor run's work directory.
      slave->garbageCollect(paths::getExecutorRunPath(
          slave->flags.work_dir, slave->info.id(), id(), state.id, runId));

      // GC the executor run's meta directory.
      slave->garbageCollect(paths::getExecutorRunPath(
          slave->metaDir, slave->info.id(), id(), state.id, runId));
    }
  }
  ...
  Option<RunState> run = state.runs.get(latest);
  ...
  Executor* executor = new Executor(
      slave,
      id(),
      state.info.get(),
      latest,
      directory,
      info.user(),
      info.checkpoint());
  ...
  executor->pid = run.get().libprocessPid.get();
  ...
  // And finally recover all the executor's tasks.
  foreachvalue (const TaskState& taskState, run.get().tasks) {
    executor->recoverTask(taskState);
  }
  ...
  // Add the executor to the framework.
  executors[executor->id] = executor;
  // If the latest run of the executor was completed (i.e., terminated
  // and all updates are acknowledged) in the previous run, we
  // transition its state to 'TERMINATED' and gc the directories.
  if (run.get().completed) {
    ...
	// Move the executor to 'completedExecutors'.
    destroyExecutor(executor->id);
  }
  return;
}
```

`Executor` struct 里面通过4个数据结构管理它的 task，一个 task 会存放在4个数据结构中的某一个：

```c++
  // Tasks can be found in one of the following four data structures:

  // Not yet launched tasks. This also includes tasks from `queuedTaskGroups`.
  LinkedHashMap<TaskID, TaskInfo> queuedTasks;

  // Not yet launched task groups. This is needed for correctly sending
  // TASK_KILLED status updates for all tasks in the group if any of the
  // tasks were killed before the executor could register with the agent.
  //
  // TODO(anand): Replace this with `LinkedHashSet` when it is available.
  std::list<TaskGroupInfo> queuedTaskGroups;

  // Running.
  LinkedHashMap<TaskID, Task*> launchedTasks;

  // Terminated but pending updates.
  LinkedHashMap<TaskID, Task*> terminatedTasks;
```

另外还有一个环形 buffer 存放已经 completed 的 task：
```c++
  // Terminated and updates acked.
  // NOTE: We use a shared pointer for Task because clang doesn't like
  // Boost's implementation of circular_buffer with Task (Boost
  // attempts to do some memset's which are unsafe).
  boost::circular_buffer<std::shared_ptr<Task>> completedTasks;
```

`Executor::recoverTask` 的主要逻辑是：
1. 根据 `taskState` 创新创建出一个 `Task`，加到 `launchedTasks` 里面。
   这是 `Executor` struct 的一个属性，是一个 hasmap：`LinkedHashMap<TaskID, Task*> launchedTasks`，
   表示所有 running 的 Task。
2. 恢复 resources 统计
3. 调用 `Executor::updateTaskState` 更新 task 状态，事实上就是更新那4个存放 task 的数据结构。
4. 当一个 task 已经 terminal 并 terminal 的状态更新已经被 acknowledged，那就把它从 `terminatedTasks`
   中移除并且加入到 `completedTasks` 中。


### Recover StatusUpdateManager

这是 `Slave::recover` 的最后一步，代码是：

```
Future<Nothing> Slave::recover(const Result<state::State>& state)
{
  ...
  return statusUpdateManager->recover(metaDir, slaveState)
    .then(defer(self(), &Slave::_recoverContainerizer, slaveState));
}
```

就是先调用 `StatusUpdateManager::recover` 方法，成功后调用 `Slave::_recoverContainerizer`。

#### StatusUpdateManager::recover

`StatusUpdateManagerProcess` 维护了一个叫 `streams` 的 hashmap ：
```c++
  hashmap<FrameworkID, hashmap<TaskID, StatusUpdateStream*>> streams;
```
这个 map 为每个 Framework 的每一个 Task 存了一个 `StatusUpdateStream*`。
该结构负责处理 task 状态的更新和确认。大概就是里面维护了一个 pending 的队列，每隔一段时间会从队列中
取出一个 Update 来去 forward ，也就是调用从 slave 传进来的 forward 回调函数，最终会发给 master。

`StatusUpdateManager::recover` 方法实现在 `src/slave/src/slave/status_update_manager.cpp` 中。
主要逻辑就是对 slave 上每个 framework 的每个 latest executor 的每个 task，
生成一个 `StatusUpdateStream`，然后去 replay，也就是向 master 去汇报。

```c++
Future<Nothing> StatusUpdateManager::recover(
    const string& rootDir,
    const Option<SlaveState>& state)
{
  return dispatch(
      process, &StatusUpdateManagerProcess::recover, rootDir, state);
}
...
Future<Nothing> StatusUpdateManagerProcess::recover(
    const string& rootDir,
    const Option<SlaveState>& state)
{
  ...
  foreachvalue (const FrameworkState& framework, state.get().frameworks) {
    foreachvalue (const ExecutorState& executor, framework.executors) {
	...
	const ContainerID& latest = executor.latest.get();
    Option<RunState> run = executor.runs.get(latest);
	...
	foreachvalue (const TaskState& task, run.get().tasks) {
	  ...
	  // Create a new status update stream.
      StatusUpdateStream* stream = createStatusUpdateStream(
            task.id, framework.id, state.get().id, true, executor.id, latest);
	  // Replay the stream.
      Try<Nothing> replay = stream->replay(task.updates, task.acks);
	  ...
	  if (stream->terminated) {
        cleanupStatusUpdateStream(task.id, framework.id);
      }
	}
	...
}
```

## 调用 `Slave::_recover`

`Slave::_recover` 方法负责 Containerizer 的恢复。主要逻辑如下：

1. 针对 Slave 中记录的所有 framework 的所有 executor：
    1. 设置一个回调函数 `Slave::executorTerminated`，该回调函数在 executor 终结时被调用。
    2. 如果 slave 启动时指定了 `--rocover=reconnect`，就找到 executor process 的 PID，然后创建出一个
	   `ReconnectExecutorMessage` 消息，发送给 executor process 。
	3. 如果 slave 启动时参数 `--recover` 的值不是 `reconnect`，就调用 `_shutdownExecutor` 把这个
       executor 关闭
2. 在等待一段时间后，调用 `Slave::reregisterExecutorTimeout` 清理所有没有重新注册的 executor 。

```c++
Future<Nothing> Slave::_recover()
{
  // Alow HTTP based executors to subscribe after the
  // containerizer recovery is complete.
  recoveryInfo.reconnect = true;

  foreachvalue (Framework* framework, frameworks) {
    foreachvalue (Executor* executor, framework->executors) {
      // Set up callback for executor termination.
      containerizer->wait(executor->containerId)
        .onAny(defer(self(),
                     &Self::executorTerminated,
                     framework->id(),
                     executor->id,
                     lambda::_1));

      if (flags.recover == "reconnect") {
	    ...
		ReconnectExecutorMessage message;
        message.mutable_slave_id()->MergeFrom(info.id());
        send(executor->pid.get(), message);
		...
	  } else {
	    ...
		_shutdownExecutor(framework, executor);
		...
	  }
	  ...
	}

    if (!frameworks.empty() && flags.recover == "reconnect") {
    // Cleanup unregistered executors after a delay.
    delay(EXECUTOR_REREGISTER_TIMEOUT,
          self(),
          &Slave::reregisterExecutorTimeout);

    // We set 'recovered' flag inside reregisterExecutorTimeout(),
    // so that when the slave re-registers with master it can
    // correctly inform the master about the launched tasks.
    return recoveryInfo.recovered.future();
  }

  return Nothing();
}
```

### 设置回调函数，当 executor 终结时调用


`containerizer->wait` 定义在 `src/slave/containerizer/containerizer.hpp`，它会等待 executor 终结，
然后做一些容器上下文的清理（destroy the containerized context）。

```c++
  // Wait on the 'ContainerTermination'. If the executor terminates,
  // the containerizer should also destroy the containerized context.
  // Returns None if the container cannot be found.
  // The future may be failed if an error occurs during termination of
  // the executor or destruction of the container.
  virtual process::Future<Option<mesos::slave::ContainerTermination>> wait(
      const ContainerID& containerId) = 0;
```

这里传进去的回调函数是 `Self::executorTerminated` ，它的主要逻辑是：

1. 做一些 termination 的检查。
2. 通过 `sendExecutorTerminatedStatusUpdate` 方法把 executor 中所有未完成的 task 的状态设置为
   `TASK_FAILED` 或者 `TASK_GONE`。
3. 如果这是一个 Command Executor，则向 master 发送一个 `ExitedExecutorMessage` 消息。
4. 如果 slave 或者 framework 的状态是 `TERMINATING`，即正在终结，就调用 `removeExecutor` 把
   executor 删掉。
5. 如果这个 executor 的 framework 已经没有其他 pending 的 executor 和 task 了，就调用
   `removeFramework` 把这个 framework 删除。

```c++
// Called by the isolator when an executor process terminates.
void Slave::executorTerminated(
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const Future<Option<ContainerTermination>>& termination)
{
  ...
  Framework* framework = getFramework(frameworkId);
  ...
  Executor* executor = framework->getExecutor(executorId);
  ...
  if (framework->state != Framework::TERMINATING) {
    // Transition all live launched tasks.
    foreach (Task* task, executor->launchedTasks.values()) {
      if (!protobuf::isTerminalState(task->state())) {
        sendExecutorTerminatedStatusUpdate(
            task->task_id(), termination, frameworkId, executor);
      }
    }

    // Transition all queued tasks.
    foreach (const TaskInfo& task, executor->queuedTasks.values()) {
      sendExecutorTerminatedStatusUpdate(
          task.task_id(), termination, frameworkId, executor);
    }
  }
  ...
  // Only send ExitedExecutorMessage if it is not a Command
  // Executor because the master doesn't store them; they are
  // generated by the slave.
  // TODO(vinod): Reliably forward this message to the master.
  if (!executor->isCommandExecutor()) {
    ExitedExecutorMessage message;
    message.mutable_slave_id()->MergeFrom(info.id());
    message.mutable_framework_id()->MergeFrom(frameworkId);
    message.mutable_executor_id()->MergeFrom(executorId);
    message.set_status(status);

    if (master.isSome()) { send(master.get(), message); }
  }
  ...
  // Remove the executor if either the slave or framework is
  // terminating or there are no incomplete tasks.
  if (state == TERMINATING ||
      framework->state == Framework::TERMINATING ||
      !executor->incompleteTasks()) {
    removeExecutor(framework, executor);
  }

  // Remove this framework if it has no pending executors and tasks.
  if (framework->executors.empty() && framework->pending.empty()) {
    removeFramework(framework);
  }
```

其中，`sendExecutorTerminatedStatusUpdate` 负责发送 executor 已经终结的消息。
该方法会判断 executor 的终结状态，然后创建出一个 `StatusUpdate` 消息，并最终发送给 master 。


```c++
void Slave::sendExecutorTerminatedStatusUpdate(
    const TaskID& taskId,
    const Future<Option<ContainerTermination>>& termination,
    const FrameworkID& frameworkId,
    const Executor* executor)
{
  ...
  statusUpdate(protobuf::createStatusUpdate(
      frameworkId,
      info.id(),
      taskId,
      state,
      TaskStatus::SOURCE_SLAVE,
      UUID::random(),
      message,
      reason,
      executor->id),
      UPID());
}
```

其中，`protobuf::createStatusUpdate` 创建一个 `StatusUpdate` 消息，该消息描述一个 task 的状态，
定义在 `src/messages/messages.proto` 中。

拿到消息后，调用 `Slave::statusUpdate` 方法把消息发送出去，最终会发给 master 。
具体的发送机制参考
[Mesos 源码学习(6) Master 和 Slave 之间的消息](2016-12-08-06-messages_between_master_and_slave.md)。

`removeExecutor` 负责把一个 Executor 从 Slave 中移除。主要逻辑就是做一些检查，进行垃圾回收，
做 checkpoint ，最后把 executor 的数据结构从 Slave 中去除。

`removeFramework` 负责把一个 Framework 从 Slave 中移除。主要逻辑类似：做一些检查，进行垃圾回收，
处理 framework 的数据结构。


### recover 为 reconnect 时，重新和 Executor 联系上

在 recovery 的这一步，需要和 executor 再次联系上，于是会向 Executor 发送 `ReconnectExecutorMessage`
消息。

### recover 不为 reconnect 时，关闭 Executor

这是调用 `_shutdownExecutor` 把这个 executor 关闭。它做的事情就是向 Executor 发送一个
`ShutdownExecutorMessage` 消息，然后等待一个 `executor_shutdown_grace_period` 时间，如果 executor
还活着的话就把它 destroy 掉，最终执行 destroy 的是 `Containerizer->destroy` 方法，它会 kill 掉这个
容器所有的进程，释放所有的资源。

### 清理没有及时重新注册的 Executor

这里会先等待一段时间 `EXECUTOR_REREGISTER_TIMEOUT` ，这个时间写死为2s，然后调用
`Slave::reregisterExecutorTimeout` 清理所有没有重新注册的 executor 。

在之前的 recover 过程中，所有 executor 的状态都是默认的 `REGISTERING`，所以这里会遍历所有的
Executor ，如果他们的状态仍然还是 `REGISTERING`，那就说明他们没有及时注册。
对于这些 Executor，会有如下操作：
1. 调用 `Containerizer->destroy` 方法把它终结掉。
2. executor 状态置为 `Executor::TERMINATING`，创建一个 `ContainerTermination` 结构，
   设置给 `executor->pendingTermination`。


最后，会发一个消息，说 recovery 过程完成了：

```c++
  // Signal the end of recovery.
  recoveryInfo.recovered.set(Nothing());
```


## 调用 `Slave::__recover`

当 recover 完成时，会调用 `Slave::__recover` 方法，主要逻辑是：
1. 找到之前的 slave 进程的目录，做垃圾回收；
2. 如果启动参数 `--recover=cleanup`，则调用 libprocess 的 `terminate` 方法终结 slave process ；
3. 如果启动参数 `--recover=reconnect`，则：
    1. 调用 `detector->detect()` 找到 leader master，找到后调用 `Slave::detected`，具体参考
	   [Mesos 源码学习(6) Master 和 Slave 之间的消息](2016-12-08-06_messages_between_master_and_slave)。
	2. 调用 `forwardOversubscribed()` 转发超发的资源，超发的资源从 ResourceEstimator 获取，经过一些
	   计算后生成一个 `UpdateSlaveMessage` 消息发送给 master。
	3. 调用 `Slave::qosCorrections()` 来启动 QoS Controller。该方法会自己调用自己实现不断地做检查，
	   它会调用 `qosController->corrections()` 的到一个 Correction 的列表，每个 Correction 表示对
	   某个 Executor 所采取的动作，杀还是不杀，如果决定要杀, slave 就会把它杀掉。

```c++
void Slave::__recover(const Future<Nothing>& future)
{
  ...
  // Schedule all old slave directories for garbage collection.
  const string directory = path::join(flags.work_dir, "slaves");
  Try<list<string>> entries = os::ls(directory);
  if (entries.isSome()) {
    foreach (const string& entry, entries.get()) {
      string path = path::join(directory, entry);
      // Ignore non-directory entries.
      if (!os::stat::isdir(path)) {
        continue;
      }

      // We garbage collect a directory if either the slave has not
      // recovered its id (hence going to get a new id when it
      // registers with the master) or if it is an old work directory.
      SlaveID slaveId;
      slaveId.set_value(entry);
      if (!info.has_id() || !(slaveId == info.id())) {
        // GC the slave work directory.
        os::utime(path); // Update the modification time.
        garbageCollect(path);

        // GC the slave meta directory.
        path = paths::getSlavePath(metaDir, slaveId);
        if (os::exists(path)) {
          os::utime(path); // Update the modification time.
          garbageCollect(path);
        }
      }
    }
  }
  ...
    if (flags.recover == "reconnect") {
    state = DISCONNECTED;

    // Start detecting masters.
    detection = detector->detect()
      .onAny(defer(self(), &Slave::detected, lambda::_1));

    // Forward oversubscribed resources.
    forwardOversubscribed();

    // Start acting on correction from QoS Controller.
    qosCorrections();
  } else {
    // Slave started in cleanup mode.
    CHECK_EQ("cleanup", flags.recover);
    state = TERMINATING;

    if (frameworks.empty()) {
      terminate(self());
    }

    // If there are active executors/frameworks, the slave will
    // shutdown when all the executors are terminated. Note that
    // the executors are guaranteed to terminate because they
    // are sent shutdown signal in '_recover()' which results in
    // 'Containerizer::destroy()' being called if the termination
    // doesn't happen within a timeout.
  }

  recoveryInfo.recovered.set(Nothing()); // Signal recovery.
}
```

## Slave Recovery 结束

到此，Slave Recovery 的过程结束。
正常的逻辑下（`--recover=reconnect`），Slave 开始和 Master 进行通信。
