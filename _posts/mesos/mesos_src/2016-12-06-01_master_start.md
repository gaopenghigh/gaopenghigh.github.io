---
layout: post
title:  "Mesos 源码学习(1) Mesos Master 的启动"
date:   2016-12-06 11:00:00 +0800
categories: Mesos
toc: true
---

# Master 的启动

Mesos Master 启动相关的代码在 `src/master/main.cpp` 中。

## 解析 flags 参数

通过解析以 `MESOS_` 开头的环境变量，来初始化一些参数，并验证参数的合法性。
具体的参数参考 [Mesos Configuration](http://mesos.apache.org/documentation/latest/configuration/)。


## 初始化 master process

初始化一个 name 为 `master` 的 process :

```c++
int main(int argc, char** argv)
{
...
  // This should be the first invocation of `process::initialize`. If it returns
  // `false`, then it has already been called, which means that the
  // authentication realm for libprocess-level HTTP endpoints was not set to the
  // correct value for the master.
  if (!process::initialize(
          "master",
          READWRITE_HTTP_AUTHENTICATION_REALM,
          READONLY_HTTP_AUTHENTICATION_REALM)) {
    EXIT(EXIT_FAILURE) << "The call to `process::initialize()` in the master's "
                       << "`main()` was not the function's first invocation";
  }
...
}
```

其中，`process::initialize()` 定义在 `3rdparty/libprocess/include/process/process.hpp` 中：

```c++
/**
 * Initialize the library.
 *
 * **NOTE**: `libprocess` uses Google's `glog` and you can specify options
 * for it (e.g., a logging directory) via environment variables.
 *
 * @param delegate Process to receive root HTTP requests.
 * @param readwriteAuthenticationRealm The authentication realm that read-write
 *     libprocess-level HTTP endpoints will be installed under, if any.
 *     If this realm is not specified, read-write endpoints will be installed
 *     without authentication.
 * @param readonlyAuthenticationRealm The authentication realm that read-only
 *     libprocess-level HTTP endpoints will be installed under, if any.
 *     If this realm is not specified, read-only endpoints will be installed
 *     without authentication.
 * @return `true` if this was the first invocation of `process::initialize()`,
 *     or `false` if it was not the first invocation.
 *
 * @see [glog](https://google-glog.googlecode.com/svn/trunk/doc/glog.html)
 */
bool initialize(
    const Option<std::string>& delegate = None(),
    const Option<std::string>& readwriteAuthenticationRealm = None(),
    const Option<std::string>& readonlyAuthenticationRealm = None());
```

## 初始化 Version Process

```c++
int main(int argc, char** argv)
{
...
    spawn(new VersionProcess(), true);
...
}
```

其中 VersionProcess 就是一个 name 为 "version" 的 process，负责打印版本号的。
VersionProcess 定义在 `src/version/version.hpp` 和 `src/version/version.cpp` 中。

## 初始化 firewall rules

```c++
int main(int argc, char** argv)
{
...
  // Initialize firewall rules.
  if (flags.firewall_rules.isSome()) {
    vector<Owned<FirewallRule>> rules;

    const Firewall firewall = flags.firewall_rules.get();

    if (firewall.has_disabled_endpoints()) {
      hashset<string> paths;

      foreach (const string& path, firewall.disabled_endpoints().paths()) {
        paths.insert(path);
      }

      rules.emplace_back(new DisabledEndpointsFirewallRule(paths));
    }

    process::firewall::install(move(rules));
  }
...
}
```

通过 firewall 可以设置一些 endpoint 不可用。

## 加载 module 和 anonymous modules

```c++
  if (flags.modules.isSome()) {
    Try<Nothing> result = ModuleManager::load(flags.modules.get());
    if (result.isError()) {
      EXIT(EXIT_FAILURE) << "Error loading modules: " << result.error();
    }
  }

    // Create anonymous modules.
  foreach (const string& name, ModuleManager::find<Anonymous>()) {
    Try<Anonymous*> create = ModuleManager::create<Anonymous>(name);
    if (create.isError()) {
      EXIT(EXIT_FAILURE)
        << "Failed to create anonymous module named '" << name << "'";
    }
  }
```

`ModuleManager::load()` 定义在 `src/module/manager.hpp` 和 `src/module/manager.cpp`。

ModuleManager 会动态地加载命令行中指定的 module，就是一个个地加载动态链接库，
然后为每个 module 创建一个 singleton，同时还维护一个 module name 到 module 指针的映射表。

```c++
// Mesos module loading.
//
// Phases:

// 1. Load dynamic libraries that contain modules from the Modules
//    instance which may have come from a commandline flag.
// 2. Verify versions and compatibilities.
//   a) Library compatibility. (Module API version check)
//   b) Module compatibility. (Module Kind version check)
// 3. Instantiate singleton per module. (happens in the library)
// 4. Bind reference to use case. (happens in Mesos)
```

关于 module 参考 [Mesos Modules](http://mesos.apache.org/documentation/latest/modules/)。

这里简要说明通过 Mesos Module 可以做什么：
1. 实现自己的 Allocator：把自己实现的 Allocator 编译成 so 文件，通过命令行 `--modules` 和 `--allocator` 指定自己的 Allocator。
2. 权限认证模块 Authenticatee 和 Authenicator
3. Isolator，实现自己的资源隔离，当我们有某种特殊的资源时，就需要实现一个 Isolator
4. Master Contender 和 Detector，默认 Mesos Master 使用 ZK 进行选主，可以通过实现特定的 Contender 和 Detector 来使用其它的服务来实现选主，比如 etcd 和 consul。
5. 提供 Hook，Hook 是一些定义好的接口，这些 Hook 会在某些阶段被调用到，Module 可以选择实现其中的某些 Hook 来实现某种需求；

## 初始化 Hooks

```c++
  // Initialize hooks.
  if (flags.hooks.isSome()) {
    Try<Nothing> result = HookManager::initialize(flags.hooks.get());
    if (result.isError()) {
      EXIT(EXIT_FAILURE) << "Error installing hooks: " << result.error();
    }
  }
```

`HookManager` 定义在 `src/hook/manager.hpp` 和 `src/hook/manager.cpp` 中。

目前有如下一些 Hook：
* `masterLaunchTaskLabelDecorator`：在 Master launch task 时调用，为新的 task 提供一些 label，这些 label 会覆盖原来的那些。
* `masterSlaveLostHook`：当一个 slave lost 时，该 hook 被调用。
* `slaveAttributesDecorator`：Slave 初始化时调用，该 hook 为这个 slave 创建 attributes，然后 slave 会把自身的信息（包含 attribute）通知到 master。
* `slaveExecutorEnvironmentDecorator`：slave 启动一个 executor 时，会调用该 hook 为 executor 创建环境变量。
* `slavePostFetchHook`：当 slave 通过 fetcher 把需要的资源下载下来后会调用这个 hook 做一些处理
* `slavePreLaunchDockerTaskExecutorDecorator`：slave在执行 docker 任务前调用这个 hook。
* `slaveRemoveExecutorHook`：当一个 executor 被移除时，这个 hook 被调用，比如说可以做一些清理工作。
* `slaveResourcesDecorator`：slave 初始化时被调用，为 slave 生成 resource
* `slaveRunTaskLabelDecorator`：当 slave 从 master 收到要启动一个 task 的请求时，该 hook 被调用，生成一些 label 并且覆盖已有的 label。
* `slaveTaskStatusDecorator`：当 executor 向 slave 报告 task status 时，该 hook 被调用，生成 task 的 status ，覆盖原来的。


## 创建 Allocator

```c++
  // Create an instance of allocator.
  const string allocatorName = flags.allocator;
  Try<Allocator*> allocator = Allocator::create(allocatorName);
```

Allocator 接口定义在 `include/mesos/allocator/allocator.hpp` 中：

```c++
namespace mesos {
namespace allocator {

/**
 * Basic model of an allocator: resources are allocated to a framework
 * in the form of offers. A framework can refuse some resources in
 * offers and run tasks in others. Allocated resources can have offer
 * operations applied to them in order for frameworks to alter the
 * resource metadata (e.g. creating persistent volumes). Resources can
 * be recovered from a framework when tasks finish/fail (or are lost
 * due to an agent failure) or when an offer is rescinded.
 *
 * This is the public API for resource allocators.
 */
class Allocator
{
  ...
}
```

`Allocator::create` 方法实现在 `src/master/allocator/allocator.cpp` 中:

```c++
using mesos::allocator::Allocator;
using mesos::internal::master::allocator::HierarchicalDRFAllocator;
...
Try<Allocator*> Allocator::create(const string& name)
{
  // Create an instance of the default allocator. If other than the
  // default allocator is requested, search for it in loaded modules.
  // NOTE: We do not need an extra not-null check, because both
  // ModuleManager and built-in allocator factory do that already.
  if (name == mesos::internal::master::DEFAULT_ALLOCATOR) {
    return HierarchicalDRFAllocator::create();
  }

  return modules::ModuleManager::create<Allocator>(name);
}
```

默认的 HierarchicalDRFAllocator 实现在 `src/master/allocator/mesos/hierarchical.hpp` 中：

```c++
namespace mesos {
namespace internal {
namespace master {
namespace allocator {
...
class HierarchicalAllocatorProcess;

typedef HierarchicalAllocatorProcess<DRFSorter, DRFSorter, DRFSorter>
HierarchicalDRFAllocatorProcess;

typedef MesosAllocator<HierarchicalDRFAllocatorProcess>
HierarchicalDRFAllocator;
...
class HierarchicalAllocatorProcess : public MesosAllocatorProcess
{
    ...
}
```

`MesosAllocator` 和 `MesosAllocatorProcess` 都定义在 `src/master/allocator/mesos/allocator.hpp` :

```c++
namespace mesos {
namespace internal {
namespace master {
namespace allocator {

class MesosAllocatorProcess;
...
// The basic interface for all Process-based allocators.
class MesosAllocatorProcess : public process::Process<MesosAllocatorProcess>
{
  ...
}
```

`MesosAllocator` 包装了一个 Process-based Allocator，用了 libprocess 来提供 allocator 的服务：

```c++
// A wrapper for Process-based allocators. It redirects all function
// invocations to the underlying AllocatorProcess and manages its
// lifetime. We ensure the template parameter AllocatorProcess
// implements MesosAllocatorProcess by storing a pointer to it.
template <typename AllocatorProcess>
class MesosAllocator : public mesos::allocator::Allocator
{
    ...
private:
   MesosAllocatorProcess* process;
}
```

注意到 private 字段中包含了一个 process 的指针。
Allocator 对外的服务，都是通过这个 wrapper 提供的。比如：

```c++
template <typename AllocatorProcess>
inline void MesosAllocator<AllocatorProcess>::requestResources(
    const FrameworkID& frameworkId,
    const std::vector<Request>& requests)
{
  process::dispatch(
      process,
      &MesosAllocatorProcess::requestResources,
      frameworkId,
      requests);
}
```

这个 wrapper 通过 `dispatch`，调用了对应 AllocatorProcess 的 `MesosAllocatorProcess::requestResources`
方法，传入了两个参数 `frameworkId` 和 `requests`。

## Registry storage

Registry storage 负责存储数据，根据参数，数据可以存储在：
1. 内存
2. ZK + Replicated Log
3. 本地 Replicated Log

```c++
...
      Try<zookeeper::URL> url = zookeeper::URL::parse(flags.zk.get());
      if (url.isError()) {
        EXIT(EXIT_FAILURE) << "Error parsing ZooKeeper URL: " << url.error();
      }

      log = new Log(
          flags.quorum.get(),
          path::join(flags.work_dir.get(), "replicated_log"),
          url.get().servers,
          flags.zk_session_timeout,
          path::join(url.get().path, "log_replicas"),
          url.get().authentication,
          flags.log_auto_initialize,
          "registrar/");

...
    storage = new LogStorage(log);
```

`LogStorage` 定义在 `include/mesos/state/log.hpp` 中：

```c++
// Forward declarations.
class LogStorageProcess;

class LogStorage : public mesos::state::Storage
{
public:
  LogStorage(mesos::log::Log* log, size_t diffsBetweenSnapshots = 0);

  virtual ~LogStorage();

  // Storage implementation.
  virtual process::Future<Option<internal::state::Entry>> get(
      const std::string& name);
  virtual process::Future<bool> set(
      const internal::state::Entry& entry,
      const UUID& uuid);
  virtual process::Future<bool> expunge(const internal::state::Entry& entry);
  virtual process::Future<std::set<std::string>> names();

private:
  LogStorageProcess* process;
};

```

## State

创建一个 `state` 来记录状态:

```
  mesos::state::protobuf::State* state =
    new mesos::state::protobuf::State(storage);
  Registrar* registrar =
    new Registrar(flags, state, READONLY_HTTP_AUTHENTICATION_REALM);
```

`state` 为分布式服务提供了类似 key/value 的数据存储服务。
数据都是带版本的，所以每次 `set` 操作只有在上一次 fetch 到现在都没有被修改过才会成功。
`state` 定义在 `include/mesos/state/state.hpp` 中：

```c++
// An abstraction of "state" (possibly between multiple distributed
// components) represented by "variables" (effectively key/value
// pairs). Variables are versioned such that setting a variable in the
// state will only succeed if the variable has not changed since last
// fetched. Varying implementations of state provide varying
// replicated guarantees.
//
// Note that the semantics of 'fetch' and 'store' provide
// atomicity. That is, you cannot store a variable that has changed
// since you did the last fetch. That is, if a store succeeds then no
// other writes have been performed on the variable since your fetch.
//
// Example:
//
//   Storage* storage = new ZooKeeperStorage();
//   State* state = new State(storage);
//   Future<Variable> variable = state->fetch("slaves");
//   std::string value = update(variable.value());
//   variable = variable.mutate(value);
//   state->store(variable);

...
class State
{
public:
  explicit State(Storage* _storage) : storage(_storage) {}
  virtual ~State() {}

  // Returns a variable from the state, creating a new one if one
  // previously did not exist (or an error if one occurs).
  process::Future<Variable> fetch(const std::string& name);

  // Returns the variable specified if it was successfully stored in
  // the state, otherwise returns none if the version of the variable
  // was no longer valid, or an error if one occurs.
  process::Future<Option<Variable>> store(const Variable& variable);

  // Returns true if successfully expunged the variable from the state.
  process::Future<bool> expunge(const Variable& variable);

  // Returns the collection of variable names in the state.
  process::Future<std::set<std::string>> names();
  ...
}
```

## 创建 Master contendor 和 detector

简单地说，Contender 负责选主，Detector 负责探测谁是主。

```c++
  MasterContender* contender;
  MasterDetector* detector;

  Try<MasterContender*> contender_ = MasterContender::create(
      flags.zk, flags.master_contender);

  if (contender_.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to create a master contender: " << contender_.error();
  }

  contender = contender_.get();

  Try<MasterDetector*> detector_ = MasterDetector::create(
      flags.zk, flags.master_detector);

  if (detector_.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to create a master detector: " << detector_.error();
  }

  detector = detector_.get();

```

## 初始化 Authorizer

```c++
  ...
  authorizer = Authorizer::create(flags.acls.get());
  ...
      authorizer_ = authorizer.get();

    // Set the authorization callbacks for libprocess HTTP endpoints.
    // Note that these callbacks capture `authorizer_.get()`, but the master
    // creates a copy of the authorizer during construction. Thus, if in the
    // future it becomes possible to dynamically set the authorizer, this would
    // break.
    process::http::authorization::setCallbacks(
        createAuthorizationCallbacks(authorizer_.get()));
```

## 设置 Slave removal rate limiter

为了保证集群的稳定，可以设置 slave 不能以太快的速率被 remove 出集群。

```c++
Option<shared_ptr<RateLimiter>> slaveRemovalLimiter = None();
  if (flags.agent_removal_rate_limit.isSome()) {
      ...
      slaveRemovalLimiter = new RateLimiter(permits.get(), duration.get());
      ...
  }
```

## 创建 master process

```c++
  Master* master =
    new Master(
      allocator.get(),
      registrar,
      &files,
      contender,
      detector,
      authorizer_,
      slaveRemovalLimiter,
      flags);
...
  process::spawn(master);
  process::wait(master->self());
...
```

`Master` 的构造函数定义在 `src/master/master.hpp` 和 `src/master/master.cpp` 中：

```c++
Master::Master(
    Allocator* _allocator,
    Registrar* _registrar,
    Files* _files,
    MasterContender* _contender,
    MasterDetector* _detector,
    const Option<Authorizer*>& _authorizer,
    const Option<shared_ptr<RateLimiter>>& _slaveRemovalLimiter,
    const Flags& _flags)
  : ProcessBase("master"),
    flags(_flags),
    http(this),
    allocator(_allocator),
    registrar(_registrar),
    files(_files),
    contender(_contender),
    detector(_detector),
    authorizer(_authorizer),
    frameworks(flags),
    authenticator(None()),
    metrics(new Metrics(*this)),
    electedTime(None())
{
  slaves.limiter = _slaveRemovalLimiter;

  // NOTE: We populate 'info_' here instead of inside 'initialize()'
  // because 'StandaloneMasterDetector' needs access to the info.

  // Master ID is generated randomly based on UUID.
  info_.set_id(UUID::random().toString());

  // NOTE: Currently, we store ip in MasterInfo in network order,
  // which should be fixed. See MESOS-1201 for details.
  // TODO(marco): The ip, port, hostname fields above are
  //     being deprecated; the code should be removed once
  //     the deprecation cycle is complete.
  info_.set_ip(self().address.ip.in().get().s_addr);

  info_.set_port(self().address.port);
  info_.set_pid(self());
  info_.set_version(MESOS_VERSION);
  ...
  info_.set_hostname(hostname);

  // This uses the new `Address` message in `MasterInfo`.
  info_.mutable_address()->set_ip(stringify(self().address.ip));
  info_.mutable_address()->set_port(self().address.port);
  info_.mutable_address()->set_hostname(hostname);
}
```

主要工作就是把一些信息存到 `info_` 中，`info_` 的类型是 `MasterInfo`。

接下来，开始 master process 的初始化。