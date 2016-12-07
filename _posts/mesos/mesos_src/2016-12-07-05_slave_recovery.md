---
layout: post
title:  "Mesos 源码学习(5) Slave Recovery"
date:   2016-12-07 18:00:00 +0800
categories: Mesos
toc: true
---

# Slave Recovery

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

## 从硬盘拿到 State

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
2. 根据 executorState 的 latest run，创建一个 `Executor` struct，恢复 executor process 的 PID。
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

这是 Slave Recovery 的最后一步，代码是：

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