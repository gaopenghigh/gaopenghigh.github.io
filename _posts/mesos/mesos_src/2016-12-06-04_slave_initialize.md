---
layout: post
title: "Mesos 源码学习(4) Mesos Slave 初始化"
date: 2016-12-06 18:00:00 +0800
categories: Mesos
toc: true
---


Slave Process 初始化的代码在 `src/slave/slave.cpp` 中：

```c++
void Slave::initialize()
{...}
```

## cgourps 初始化

可以通过参数 `--agent_subsystems` 指定需要使用的 cgroup 子系统，默认问空。
Mesos Slave 会运行在这些 cgroup 子系统里面。主要是用于资源监控。
这些子系统都继承自 root mesos cgroup 。

注意这里的 cgroup 初始化都是针对 slave 进程本身的，而不是针对 executor。

## Credential 和 Authenticaton 初始化

```c++
// 在 src/slave/slave.hpp 中
Option<Credential> credential;

// 在 src/slave/slave.cpp 中
  if (flags.credential.isSome()) {
    Result<Credential> _credential =
      credentials::readCredential(flags.credential.get());
    ...
    credential = _credential.get();
  }

  Option<Credentials> httpCredentials;
  if (flags.http_credentials.isSome()) {
    Result<Credentials> credentials =
      credentials::read(flags.http_credentials.get());
    ...
    httpCredentials = credentials.get();
  }
  if (flags.authenticate_http_readonly) {
    Try<Nothing> result = initializeHttpAuthenticators(
        READONLY_HTTP_AUTHENTICATION_REALM,
        strings::split(flags.http_authenticators, ","),
        httpCredentials);

    if (result.isError()) {
      EXIT(EXIT_FAILURE) << result.error();
    }
  }
  ...
```

## Resource Estimator 和 Qos Controller 初始化

```c++
  Try<Nothing> initialize =
    resourceEstimator->initialize(defer(self(), &Self::usage));

  if (initialize.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to initialize the resource estimator: " << initialize.error();
  }

  initialize = qosController->initialize(defer(self(), &Self::usage));

  if (initialize.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to initialize the QoS Controller: " << initialize.error();
  }
```

## Resources 初始化

下面的代码把 flags 传给 Containerizer，得到 resource：

```c++
  Try<Resources> resources = Containerizer::resources(flags);
  if (resources.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to determine agent resources: " << resources.error();
  }
```

`Containerizer::resources` 定义在 `src/slave/containerizer/containerizer.cpp` 中：

```c++
Try<Resources> Containerizer::resources(const Flags& flags)
{
  Try<Resources> parsed = Resources::parse(
      flags.resources.getOrElse(""), flags.default_role);

  if (parsed.isError()) {
    return Error(parsed.error());
  }

  Resources resources = parsed.get();
  ...
  return resources;
}
...
```

如果命令行中没有明确指定 `cpus`, `gpus`, `mem`, `ports` 和 `disk` 这5种资源，则它们会被自动添加。

另外还会检查 `disk` 类型的资源是否真的存在于硬盘上。

`Resource` 定义在 `include/mesos/resources.hpp` 和 `src/common/resources.cpp` 中。

## Attributes 初始化

```c++
  Attributes attributes;
  if (flags.attributes.isSome()) {
    attributes = Attributes::parse(flags.attributes.get());
  }
```

`Attributes` 定义在 `include/mesos/attributes.hpp` 和 `src/common/attributes.cpp` 中。

## 初始化 slave info

就是把之前初始化的很多东西填充到 `info` 中：

```c++
// 在 src/slave/slave.hpp 中
SlaveInfo info;

// 在 src/slave/slave.cpp 中
  // Initialize slave info.
  info.set_hostname(hostname);
  info.set_port(self().address.port);

  info.mutable_resources()->CopyFrom(resources.get());
  if (HookManager::hooksAvailable()) {
    info.mutable_resources()->CopyFrom(
        HookManager::slaveResourcesDecorator(info));
  }

  // Initialize `totalResources` with `info.resources`, checkpointed
  // resources will be applied later during recovery.
  totalResources = resources.get();

  LOG(INFO) << "Agent resources: " << info.resources();

  info.mutable_attributes()->CopyFrom(attributes);
  if (HookManager::hooksAvailable()) {
    info.mutable_attributes()->CopyFrom(
        HookManager::slaveAttributesDecorator(info));
  }

  LOG(INFO) << "Agent attributes: " << info.attributes();

  // Checkpointing of slaves is always enabled.
  info.set_checkpoint(true);
```

这里有两个 Hooks 有可能会被调用到（如果在命令行设置了的话）：

* `slaveAttributesDecorator`：Slave 初始化时调用，该 hook 为这个 slave 创建 attributes，然后 slave 会把自身的信息（包含 attribute）通知到 master。
* `slaveResourcesDecorator`：slave 初始化时被调用，为 slave 生成 resource

## StatusUpdateManager 初始化

```c++
  statusUpdateManager->initialize(defer(self(), &Slave::forward, lambda::_1)
    .operator std::function<void(StatusUpdate)>());
```

`statusUpdateManager->initialize` 定义在 `src/slave/status_update_manager.cpp` 中：

```c++
// 在 src/slave/status_update_manager.hpp 中
  // Expects a callback 'forward' which gets called whenever there is
  // a new status update that needs to be forwarded to the master.
  void initialize(const lambda::function<void(StatusUpdate)>& forward);

// 在 src/slave/status_update_manager.cpp 中
void StatusUpdateManager::initialize(
    const function<void(StatusUpdate)>& forward)
{
  dispatch(process, &StatusUpdateManagerProcess::initialize, forward);
}
...
class StatusUpdateManagerProcess
  : public ProtobufProcess<StatusUpdateManagerProcess>
{
  ...
  function<void(StatusUpdate)> forward_;
  ...
}
...
void StatusUpdateManagerProcess::initialize(
    const function<void(StatusUpdate)>& forward)
{
  forward_ = forward;
}

```

可以看到，传给 `StatusUpdateManagerProcess` 的回调函数是 `Slave::forward` 方法。
该方法放到 `forward_`变量中，该变量在 `StatusUpdateManagerProcess::forward` 方法中被调用：

```c++
Timeout StatusUpdateManagerProcess::forward(
    const StatusUpdate& update,
    const Duration& duration)
{
  CHECK(!paused);

  VLOG(1) << "Forwarding update " << update << " to the agent";

  // Forward the update.
  forward_(update);

  // Send a message to self to resend after some delay if no ACK is received.
  return delay(duration,
               self(),
               &StatusUpdateManagerProcess::timeout,
               duration).timeout();
}
```

`StatusUpdateManagerProcess::forward` 方法所做的事，就是把更新消息传出去。
具体怎么传就是依靠初始化时设置的 `forward` 回调函数，这里设置的就是 `Slave::forward`：

```c++
// NOTE: An acknowledgement for this update might have already been
// processed by the slave but not the status update manager.
void Slave::forward(StatusUpdate update)
{
  ...
  update.mutable_status()->set_uuid(update.uuid());

  // Update the status update state of the task and include the latest
  // state of the task in the status update.
  Framework* framework = getFramework(update.framework_id());
  ...
  const TaskID& taskId = update.status().task_id();
  Executor* executor = framework->getExecutor(taskId);
  ...
  task = executor->launchedTasks[taskId];
  ...
  task->set_status_update_state(update.status().state());
  task->set_status_update_uuid(update.uuid());
  update.set_latest_state(task->state());
  ...
  // Forward the update to master.
  StatusUpdateMessage message;
  message.mutable_update()->MergeFrom(update);
  message.set_pid(self()); // The ACK will be first received by the slave.

  send(master.get(), message);
}
```

`Slave::forward` 做的事情，就是组装出一个 `StatusUpdateMessage` 消息，发送给 master 。

## 启动 disk 监控

```c++
  // Start disk monitoring.
  // NOTE: We send a delayed message here instead of directly calling
  // checkDiskUsage, to make disabling this feature easy (e.g by specifying
  // a very large disk_watch_interval).
  delay(flags.disk_watch_interval, self(), &Slave::checkDiskUsage);
```

其中， `Slave::checkDiskUsage` 会每隔一段时间，就检查一次硬盘使用情况。
如果硬盘空间使用量过大，就会出发一次垃圾回收，把早期的一些垃圾清理掉。
这些参数都可以在命令行中指定。

## 使用 libprocess 注册处理函数，基于 HTTP 监听消息

不赘。

## 注册信号处理方法

```c++
  auto signalHandler = defer(self(), &Slave::signaled, lambda::_1, lambda::_2)
    .operator std::function<void(int, int)>();
  if (os::internal::configureSignal(&signalHandler) < 0) {
    EXIT(EXIT_FAILURE)
      << "Failed to configure signal handlers: " << os::strerror(errno);
  }
```

当收到 `SIGUSR1` 时，slave 会 shutdown。其它信号不做处理。


## 异步地执行 recovery

详见 [Mesos 源码学习(5) Slave Recovery]({{ site.baseurl }}{% post_url /mesos/mesos_src/2016-12-07-05_slave_recovery %})。


## Slave Process 初始化完成

至此，Slave Process 完成了初始化。
接下来程序就依靠之前用 libprocess 的 `install` 和 `route` 方法注册的各个处理函数来运行了。
