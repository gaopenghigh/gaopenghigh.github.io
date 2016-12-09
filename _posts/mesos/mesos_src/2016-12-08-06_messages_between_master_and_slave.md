---
layout: post
title: "Mesos 源码学习(6) Master 和 Slave 之间的消息"
date: 2016-12-08 17:10:00 +0800
categories: Mesos
toc: true
---

## Slave 向 Master 注册

### 注册总流程

Slave 初始化的最后，会做 Recovery ，而 Recovery 的最后，则调用 `detector->detect()` 方法找到
leader master ，找到后，就回调 `Slave:detected` 方法。改方法的主要逻辑是：

1. 暂停 StatusUpdateManager
2. 调用 `Slave::authenticate()` 做认证：
    1. 如果正在做 `authenticating`，就取消然后返回
	2. 调用 libprocess 中的 `link` 方法确保到 master 有一个 link，简单地说就是创建到 master 的一个
	   长连接。
	3. 使用默认的或者自定义 module 中的 authenticatee 进行认证
	4. 认证完成后调用 `Slave::_authenticate` 方法，该方法：
	    1. 如果认证失败，就等待一段 backoff 时间，再次调用 authenticate 做认证；
		2. 如果认证成功，则调用 `Slave::doReliableRegistration` 向 master 进行注册，详见见下一节；
	5. 如果认证超时（写死的5秒），就调用 `Slave::authenticationTimeout` 方法，该方法会导致重试再一次
	   认证。
3. 观察 leader master 的变化，如果 leader 变了，就再次调用一次 `Slave::detected` 方法。

```c++
void Slave::detected(const Future<Option<MasterInfo>>& _master)
{
  ...
  // Pause the status updates.
  statusUpdateManager->pause();
  ...
  Option<MasterInfo> latest;
  ...
  latest = _master.get();
  master = UPID(_master.get().get().pid());
  ...
  // Authenticate with the master.
  authenticate();
  ...
  // Keep detecting masters.
  LOG(INFO) << "Detecting new master";
  detection = detector->detect(latest)
    .onAny(defer(self(), &Slave::detected, lambda::_1));
}
...

void Slave::authenticate()
{
  authenticated = false;
  ...
  

```

### ReliablRegistration: 可依赖的注册

`Slave::doReliableRegistration` 的主要逻辑如下：
1. 做一些状态检查，比如当 slave 的状态是 RUNNING 时就直接退出；
2. 调用 libprocess 的 `link` 方法确保到 master process 有一个长连接；
3. 如果 Slave 没有一个 ID，说明这是它的第一次注册，则：
    1. 创建一个 `RegisterSlaveMessage` 消息发送给 master process；
4. 如果 Slave 已经有了一个 ID（比如是 recover 回来的），说明这是 re-register，则：
    1. 创建一个 `ReregisterSlaveMessage` 消息
	2. 把 checkpointed resources，frameworks，tasks 和 executors 等信息填充到消息体中
	3. 然后把消息发给 master process
5. 在适当的 backoff 时间后，再次调用 `doReliableRegistration` 方法，这样当 slave 没有正常 run 起来
   时，会再次注册，而当 slave 已经 RUNNNIG 时，就完成了注册过程。

```c++
void Slave::doReliableRegistration(Duration maxBackoff)
{
  ...
    if (state == RUNNING) { // Slave (re-)registered with the master.
    return;
  }
  ...
  // Ensure there is a link to the master before we start
  // communicating with it. We want to link after the initial
  // registration backoff in order to avoid all of the agents
  // establishing connections with the master at once.
  // See MESOS-5330.
  link(master.get());
  
    if (!info.has_id()) {
    // Registering for the first time.
    RegisterSlaveMessage message;
    message.set_version(MESOS_VERSION);
    message.mutable_slave()->CopyFrom(info);

    // Include checkpointed resources.
    message.mutable_checkpointed_resources()->CopyFrom(checkpointedResources);

    send(master.get(), message);
  } else {
    // Re-registering, so send tasks running.
    ReregisterSlaveMessage message;
    message.set_version(MESOS_VERSION);
	...
	message.mutable_checkpointed_resources()->CopyFrom(checkpointedResources);
	message.mutable_slave()->CopyFrom(info);
	...
	send(master.get(), message);
  }
  ...
  process::delay(delay, self(), &Slave::doReliableRegistration, maxBackoff * 2);
}
```

## Master 处理 Slave 的注册

在 master process 初始化的最后，会注册一些处理方法，来处理不同类型的消息。

### 第一次注册，处理 RegisterSlaveMessage 消息

负责处理 `RegisterSlaveMessage` 的是 `Master::registerSlave` 方法：

```c++
  install<RegisterSlaveMessage>(
      &Master::registerSlave,
      &RegisterSlaveMessage::slave,
      &RegisterSlaveMessage::checkpointed_resources,
      &RegisterSlaveMessage::version);
```

#### Master::registerSlave 方法

`Master::registerSlave` 方法的主要逻辑是：
1. 检查认证：
    1. 如果该 slave 正在做认证就等一小会儿再次调用 `registerSlave` 方法；
	2. 如果设置了需要认证但 slave 没有通过认证，就向 slave 发送一个 `ShutdownMessage` 消息让它自己
	   关闭；
2. 如果该 slave 所在的机器被设置为 DOWN 的状态，就向 slave 发送一个 `ShutdownMessage` 消息让它自己
   关闭。DOWN 的状态用于维护机器，具体参考
   [Maintenance Primitives](http://mesos.apache.org/documentation/latest/maintenance/) 。
3. 如果该 slave 已经注册过了（存在于 `slaves.registered` 中），则
    1. 如果已经注册的那个 slave 现在已经没有连接了，就把它移除掉，说明使用了同样地址的一个新的 slave
	   来注册了。
	2. 如果已经注册的那个 slave 还在有连接，说明 slave 没有收到注册的 ACK，这时就在给 slave 发送一个
	   ACK，即发送一个 `SlaveRegisteredMessage` 消息，然后退出。
4. 如果该 slave 正在注册中，则忽略此次消息，直接退出；
5. 根据消息内容，创建一个 SlaveInfo ，通过 Registrar 把 SlaveInfo 存起来，然后再调用
   `Master::_registerSlave` 方法，该方法：
    1. 创建一个 `Slave` 结构，然后通过 `Master::addSlave` 方法把它加入进来。
	2. 发送给 slave 一个 ACK 消息，即 `SlaveRegisteredMessage`。


##### `Master::addSlave` 方法

`Master::addSlave` 方法：
1. 把这个 Slave 放到 `slaves.registered` 结构中；
2. 调用 libprocess 的 `link` 方法确保从 master 到 slave 有长连接；
3. 把这个 slave 的机器放到 `machines` 结构中；
4. 创建一个 `SlaveObserver`（这也是一个 libprocess process），来对这个 Slave 进行观察。
   slave observer process 的行为见下一节；
5. 把这个 slave 上的 executor 和 task 信息存到 `frameworks` 数据结构中；
6. 调用 `allocator->addSlave` 方法把这个 slave 加入到 Allocator 中，详见下面。
7. 创建一个 `AgentAdded` 消息，发给所有订阅者，订阅者的列表维护在 `subscribers` 结构中。
	
#### SlaveObserver 负责探测一个 Slave 是否 reachable

SlaveObserver 通过 ping-pong 机制。负责探测一个 Slave 是否 reachable。

这也是一个 libprocess process，它初始化之后就会每隔一小段时间给 slave 发送 `PingSlaveMessage`，
Slave 会返回一个 `PongSlaveMessage` 。如果没有及时返回，就把这个 slave 标记为 unreachable。

注意，把 slave 标记为 unreachable 是有速率限制的。

最终的标记过程由 `Master::markUnreachable` 方法实现，主要逻辑是：
1. 更新一下 `slaves` 结构，并且通过 `registrar` 把一些信息存储下来；
2. 调用 `allocator->removeSlave` 方法把 slave 移出 Allocator；
3. 调用 `Master::updateTask` 把该 slave 上的 task 状态设置为 `TASK_UNREACHABLE` 或 `TASK_LOST`。
   该方法会创建一个 `StatusUpdate` 消息，然后 TODO
4. 调用 `Master::removeTask` 把这个 task remove 掉；
5. 把 task 的状态通知它的 framework，也就是把 `StatusUpdate` 消息发送给 framework ；
6. 调用 `Master::removeExecutor` 把这个 slave 上的 executor 移除；
7. 针对涉及到这个 slave 的 offer，调用 `allocator->recoverResources` recover 这个 offer 的资源，
   然后调用 `Master::removeOffer` 方法把 offer 撤回；
8. 针对涉及到这个 slave 的 inverse offer，调用 `removeInverseOffer` 把它们 remove 掉；
9. 把这个 slave 从 `slaves.registered` 和 `authenticated` 中删除，
   添加进 `slaves.removed` 和 `unreachable` 中，把 slave 所在的机器从 `machines` 中删除；
10. Terminate 这个 slave 的 observer ；
11. 调用 `sendSlaveLost` 方法，发送一个 `LostSlaveMessage` 消息给所有已经注册的 Framework，然后如果
    有注册 Hook 的话，调用 `masterSlaveLostHook` 。



### 重新注册，处理 ReregisterSlaveMessage 消息

当一个 slave 重新注册时，会发送一个 `ReregisterSlaveMessage` 消息，
该消息由 `Master::reregisterSlave` 方法处理：

```c++
  install<ReregisterSlaveMessage>(
      &Master::reregisterSlave,
      &ReregisterSlaveMessage::slave,
      &ReregisterSlaveMessage::checkpointed_resources,
      &ReregisterSlaveMessage::executor_infos,
      &ReregisterSlaveMessage::tasks,
      &ReregisterSlaveMessage::frameworks,
      &ReregisterSlaveMessage::completed_frameworks,
      &ReregisterSlaveMessage::version);
```

#### Master::reregisterSlave 方法

`Master::reregisterSlave` 方法的主要逻辑是：
1. 检查认证：
    1. 如果该 slave 正在做认证就等一小会儿再次调用 `reregisterSlave` 方法；
	2. 如果设置了需要认证但 slave 没有通过认证，就向 slave 发送一个 `ShutdownMessage` 消息让它自己
	   关闭；
2. 如果该 slave 所在的机器被设置为 DOWN 的状态，就向 slave 发送一个 `ShutdownMessage` 消息让它自己
   关闭。DOWN 的状态用于维护机器，具体参考
   [Maintenance Primitives](http://mesos.apache.org/documentation/latest/maintenance/) 。
3. 如果该 slave 已经注册（存在于 `slaves.registered` 中），则：
    1. 如果发现这次注册的 hostname 和 ip 和之前注册的不一致，就发送一个 `ShutdownMessage` 消息给
	   slave 让它关闭，然后返回。
	2. 更新已经 slave process 的 PID，然后调用 libprocess 的 `link` 方法创建一个到 slave 的长连接；
	3. 调用 `Master::reconcileKnownSlave` 方法进行 reconcile，确保 master 和 slave 上保存的 task 
	   一致，TODO
	4. 如果这是一个 disconnected 的 slave ：
	    1. 调用 `SlaveObserver::reconnect` 方法重新连接，
		2. 调用 `allocator->activateSlave` 方法让这个 slave 在 Allocator 中重新 active 。
	5. 调用 `Master::__reregisterSlave` 方法，告诉 slave master 的 version，以及新的 framework PID，
	   然后退出；
4. 如果该 slave 正在注册，就忽略本次消息，然后退出；
5. 如果该 slave 即没有注册也没有正在注册，通常情况下是由于这个 slave unreachable 了。这是就通过
   `registrar` 把 slave 的信息存储下来，然后调用 `Master::_reregisterSlave` 方法。


##### `Master::_reregisterSlave` 方法

主要逻辑是：
1. 调用 `Master::addSlave` 方法把 slave 加进来；
2. 在 Mesos 1.1.0 版本中，新引入了实验性的 partition-aware 特性，具体可以参考：
   [Apache Mesos 1.1.0 Released](http://mesos.apache.org/blog/mesos-1-1-0-released/) 。
   在这里，如果这个 slave 是由当前这个 master remove 掉的，那么对于没有设置 partition-aware 的
   framework，就发送一个 `ShutdownFrameworkMessage` 消息给 slave 让它关闭这个 framework ，也就是
   把 slave 上这个 framework 的所有 task 杀掉。
3. 调用 `Master::__reregisterSlave` 方法


##### `Master::__reregisterSlave` 方法


#### Master::reconcileKnownSlave 方法

## `Slave::statusUpdate` 方法。