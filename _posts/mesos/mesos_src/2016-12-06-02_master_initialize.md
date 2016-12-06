---
layout: post
title:  "Mesos 源码学习(2) Mesos Master 初始化"
date:   2016-12-06 12:00:00 +0800
categories: Mesos
toc: true
---

# Master Process 的初始化

Master 初始化的代码实现在 `src/master/master.cpp` 中：

```c++
void Master::initialize()
{
    ...
}
```

下面列出主要步骤。

## 参数合法性检查

不合法时直接退出。

## 权限相关的初始化

初始化了如下一些东西：

```c++
Option<Credentials> credentials;
Option<Authenticator*> authenticator;
```

## 初始化 framework 的 rate limit

通过 rate limit，Mesos 可以控制用户的使用频率。

## 初始化 role 和 weight

初始化了下面两个属性：
```c++
  // Configured role whitelist if using the (deprecated) "explicit
  // roles" feature. If this is `None`, any role is allowed.
  Option<hashset<std::string>> roleWhitelist;

  // Configured weight for each role, if any. If a role does not
  // appear here, it has the default weight of 1.
  hashmap<std::string, double> weights;
```

## 初始化 allocator

```c++
  // Initialize the allocator.
  allocator->initialize(
      flags.allocation_interval,
      defer(self(), &Master::offer, lambda::_1, lambda::_2),
      defer(self(), &Master::inverseOffer, lambda::_1, lambda::_2),
      weights,
      flags.fair_sharing_excluded_resource_names);
```

`allocator` 类型是 `mesos::allocator::Allocator*` 。
上一节说过，该类型包装了一个 Process-based Allocator，用了 libprocess 来提供 allocator 的服务。

`allocator->initialize` 方法定义在 `src/master/allocator/mesos/allocator.hpp` 中：

```c++
template <typename AllocatorProcess>
inline void MesosAllocator<AllocatorProcess>::initialize(
    const Duration& allocationInterval,
    const lambda::function<
        void(const FrameworkID&,
             const hashmap<SlaveID, Resources>&)>& offerCallback,
    const lambda::function<
        void(const FrameworkID&,
              const hashmap<SlaveID, UnavailableResources>&)>&
      inverseOfferCallback,
    const hashmap<std::string, double>& weights,
    const Option<std::set<std::string>>& fairnessExcludeResourceNames)
{
  process::dispatch(
      process,
      &MesosAllocatorProcess::initialize,
      allocationInterval,
      offerCallback,
      inverseOfferCallback,
      weights,
      fairnessExcludeResourceNames);
}
```

最终会调用到某个 `MesosAllocatorProcess` 的 `initialize` 方法。

需要注意，Master 在这里初始化 allocator 时设置的 offerCallback 是 `Master::offer`。
每过一段时间(allocationInterval)，Allocator 都会计算应该分给每个 framework 什么样的 offer，
然后调用 offerCallback 把 offer 发出去。这里调用的就是 `Master:offer`。

## 发送 offer 的回调方法

`Master:offer` 方法生成了一个 `ResourceOffersMessage`，然后把这个 message 传递给 framework。

```c++
  void Master::offer(const FrameworkID& frameworkId,
                   const hashmap<SlaveID, Resources>& resources)
{
  // Create an offer for each slave and add it to the message.
  ResourceOffersMessage message;
  ...
  Slave* slave = slaves.registered.get(slaveId);
  ...
    mesos::URL url;
    url.set_scheme("http");
    url.mutable_address()->set_hostname(slave->info.hostname());
    url.mutable_address()->set_ip(stringify(slave->pid.address.ip));
    url.mutable_address()->set_port(slave->pid.address.port);
    url.set_path("/" + slave->pid.id);

    Offer* offer = new Offer();
    offer->mutable_id()->MergeFrom(newOfferId());
    offer->mutable_framework_id()->MergeFrom(framework->id());
    offer->mutable_slave_id()->MergeFrom(slave->id);
    offer->set_hostname(slave->info.hostname());
    offer->mutable_url()->MergeFrom(url);
    offer->mutable_resources()->MergeFrom(offered);
    offer->mutable_attributes()->MergeFrom(slave->info.attributes());

    // Add all framework's executors running on this slave.
    if (slave->executors.contains(framework->id())) {
      const hashmap<ExecutorID, ExecutorInfo>& executors =
        slave->executors[framework->id()];
      foreachkey (const ExecutorID& executorId, executors) {
        offer->add_executor_ids()->MergeFrom(executorId);
      }
    }
    ...
    offers[offer->id()] = offer;

    framework->addOffer(offer);
    slave->addOffer(offer);

    if (flags.offer_timeout.isSome()) {
      // Rescind the offer after the timeout elapses.
      offerTimers[offer->id()] =
        delay(flags.offer_timeout.get(),
              self(),
              &Self::offerTimeout,
              offer->id());
    }

    // TODO(jieyu): For now, we strip 'ephemeral_ports' resource from
    // offers so that frameworks do not see this resource. This is a
    // short term workaround. Revisit this once we resolve MESOS-1654.
    Offer offer_ = *offer;
    offer_.clear_resources();

    foreach (const Resource& resource, offered) {
      if (resource.name() != "ephemeral_ports") {
        offer_.add_resources()->CopyFrom(resource);
      }
    }

    // Add the offer *AND* the corresponding slave's PID.
    message.add_offers()->MergeFrom(offer_);
    message.add_pids(slave->pid);
  }

  if (message.offers().size() == 0) {
    return;
  }

  LOG(INFO) << "Sending " << message.offers().size()
            << " offers to framework " << *framework;

  framework->send(message);
}
```

## 使用 libprocess 注册处理函数，基于 HTTP 监听消息

使用 libprocess 的 `install` 和 `route` 方法，
注册一些处理函数，来处理各种消息，以及提供 HTTP API。

消息和接口的种类很多，几个例子：

```c++
...
  install<RegisterFrameworkMessage>(
      &Master::registerFramework,
      &RegisterFrameworkMessage::framework);

  install<ReregisterFrameworkMessage>(
      &Master::reregisterFramework,
      &ReregisterFrameworkMessage::framework,
      &ReregisterFrameworkMessage::failover);

  install<UnregisterFrameworkMessage>(
      &Master::unregisterFramework,
      &UnregisterFrameworkMessage::framework_id);
...
  route("/api/v1/scheduler",
        DEFAULT_HTTP_FRAMEWORK_AUTHENTICATION_REALM,
        Http::SCHEDULER_HELP(),
        [this](const process::http::Request& request,
               const Option<string>& principal) {
          Http::log(request);
          return http.scheduler(request, principal);
        });
  route("/create-volumes",
        READWRITE_HTTP_AUTHENTICATION_REALM,
        Http::CREATE_VOLUMES_HELP(),
        [this](const process::http::Request& request,
               const Option<string>& principal) {
          Http::log(request);
          return http.createVolumes(request, principal);
        });
...
```

## 进行 leader 选举

在 master 初始化的最后，进行 leader 选举：

```c++
  contender->initialize(info_);

  // Start contending to be a leading master and detecting the current
  // leader.
  contender->contend()
    .onAny(defer(self(), &Master::contended, lambda::_1));
  detector->detect()
    .onAny(defer(self(), &Master::detected, lambda::_1));
```

调用 `contender->contend()` 进行选举，选举完成后调用 `Master::detected` 方法。
该方法会一直观察 leader 的变化。

```c++
void Master::detected(const Future<Option<MasterInfo>>& _leader)
{
  ...
  bool wasElected = elected();
  leader = _leader.get();

  if (elected()) { // 自己是 leader
    electedTime = Clock::now();

    if (!wasElected) {
      LOG(INFO) << "Elected as the leading master!";

      // Begin the recovery process, bail if it fails or is discarded.
      recover()
        .onFailed(lambda::bind(fail, "Recovery failed", lambda::_1))
        .onDiscarded(lambda::bind(fail, "Recovery failed", "discarded"));
    } else {
      // This happens if there is a ZK blip that causes a re-election
      // but the same leading master is elected as leader.
      LOG(INFO) << "Re-elected as the leading master";
    }
  } else {
    // A different node has been elected as the leading master.
    LOG(INFO) << "The newly elected leader is "
              << (leader.isSome()
                  ? (leader.get().pid() + " with id " + leader.get().id())
                  : "None");

    if (wasElected) {
      EXIT(EXIT_FAILURE) << "Lost leadership... committing suicide!";
    }
  }

  // Keep detecting.
  detector->detect(leader)
    .onAny(defer(self(), &Master::detected, lambda::_1));
}

```

## 完成初始化

至此，master 已经完成初始化，master process 开始工作，接受并处理来自其他组件或 API 的消息。