---
layout: post
title:  "Cloud Design Patterns 学习笔记"
date:   2016-12-21 14:00:00 +0800
categories: Tech
toc: true
---

写于 2016-12-21 。

Cloud Design Patterns 学习笔记
----


微软发布了 "[Cloud Design Patterns: Prescriptive Architecture Guidance for Cloud Applications](https://msdn.microsoft.com/en-us/library/dn568099.aspx) "。
书中，针对云应用总结出了多种设计模式和 Guidance 。本文简要记录一下这些 pattern 和 guidance 的思想。

## Patterns

### Ambassador Pattern

对于遗留服务，它们可能缺少一些我们需要的功能，但我们又离不开它们所提供的的服务。同时这些遗留服务往往都不方便进行改造。
这时候就可以使用 Ambassador Patter，在客户端和服务端之间加一个“代理”，把原来的服务包装起来，提供我们需要的额外功能，比如重试、断路、监控、安全等等。

当然，加一个“代理”会代理更多的成本，有时候提供一个客户端的类库会是更好的方式。


### Anti-Corruption Layer Pattern

这个模式也是适用于遗留服务。简单地说就是在新服务和遗留服务之间建立一个转换层，所谓的 Anti-Corruption Layer。通过这一层保证新服务设计的干净，同时又能继续使用遗留服务，知道遗留服务都改造完成。


### Backends for Frontends Pattern

即为每一种前端，建立专门的后端服务，这些后端服务又依赖同一套基础服务。
好处是可以针对某种前端方便灵活地提供一些专有功能，而无需考虑其他前端的情况。
代价显然是架构更复杂、模块更多、成本更大。


### Bulkhead Pattern

Bulkhead 就是船舱里面的隔板。这些隔板把船舱分隔成很多段，当其中一个段损坏时，海水只会灌进这一个段，而其他段仍然保持完好，船也不会因此沉没。

这个模式就是把服务分成多个组，组与组之间互相隔离，一个组中的失败不会导致整体的失败。比如同一个服务部署多个集群，集群之间互相隔离，每个集群提供给特定的客户使用。

客户端也可以把资源分组，比如为每个依赖的服务分配一个连接池，这样当其中一个服务不响应时，只会耗尽分配给那个服务的连接池，而不会导致整个客户端的连接耗尽。

另外，讲服务分组，还可以为每个组设置优先级，优先保证重要的业务。


### Cache-aside Pattern

如果一个 cache 服务没有提供 read-through 和 write-through/write-behind 的功能，比如存粹的缓存服务如
redis 或 memcached 等。可以通过 cache-aside pattern 来模拟 read-through 和
write-through/write-behind 的功能。

大概的过程是：
1. APP 查看 cache 中是否有需要的数据，有的话直接拿到；
2. 如果没有，就从数据存储服务中拿到这个数据；
3. 在主动的往 cache 中存入这份数据。


### Circuit Breaker Pattern

Circuite Breaker 就是断路器，该设计模式用来避免不必要的重试，从而避免雪崩效应。

断路器有3种状态：

- **Closed**：表示断路器是接通的，即认为后端的服务都是正常的，此时请求将会被正常地提交到服务。当短时间内针对某个服务或资源的失败调用过多时，即当短期内有超过阈值的失败出现时，状态变为 Open；
- **Open**：表示断路器是断开的，即认为后端的服务出现问题了，此时请求不会提交到服务，而是直接返回失败。进入该状态时，一般会设置一个定时器，定时器到期时把断路器置为 Half-Open 状态；
- **Half-Open**：表示一种中间状态，断路器不确定后端的服务是否正常，此时会允许部分请求提交到后端，如果这些请求成功了，那就把状态设置为 Closed，否则就回到 Open 状态。

代码实现上，可以创建一个 Circuit Breaker ，把对服务请求相关的逻辑包裹起来。如果是被断路器阻止了，则返回一种特殊的异常，做特殊的处理。

一般还需要一个手工开关，可以把断路器强制设置为某种状态。

在 Half-Open 状态时，除了被动地检查成功和失败数，还可以通过主动的健康检查，来决定下一步的状态。


### Command and Query Responsibility Segregation (CQRS) Pattern

即所谓的“命令与查询职责分离”。日常经常说的“读写分离”，其实也算是 CQRS。

CQRS 的核心内容是针对命令（写）和查询（读），使用不同的数据模型，最终数据也存储在不同的存储服务上。

CQRS 和 Event Sourcing 模式经常结合在一起使用。
写模型就是把数据转换为 Event 存入 Event Store。
读模型就是从 Event 中读取 Event stream，转换为需要的业务结构。
这两个模式结合的缺点也很明显：读和写的延迟、额外的复杂性和处理 Event 需要的额外资源。


### Compensating Transaction Pattern

补偿事务模式。

一个事务，可能包含一系列的步骤，每个步骤需要调用不同的服务完成。
通过这一系列的步骤，整体上实现最终一致性（eventually consistent）。

所谓的 Compensating Transaction Pattern 就是指，当其中的某一步失败时（经过了一些重试后），
系统自动进行一些列的补偿步骤，比如把之前已经成功的那些步骤回滚掉，实现最终一致性。
这些补偿步骤自己也是有可能会失败的，实现过程中会有一些重试，所以补偿步骤需要是幂等的。

补偿步骤不一定要正好是事务步骤的逆序，而是应该根据具体业务具体实现。


### Competing Consumers Pattern

这个模式很简单，其实就是引入一个消息队列。Consumers 从队列中获取任务执行。

### Compute Resource Consolidation Pattern

业务逻辑经常包含多个任务，一般情况下，这些任务会部署在独立的计算单元中（虚拟机、容器等）。
这个模式就是把一些任务打包在一起，放在同一个计算单元中运行。
带来的好处就是能够提升资源利用率，降低成本。

但是把任务放到一起，同时也可能带来新的问题，最直接的就是可能增加了复杂度。
原来解耦的任务现在放到了一起，任务之间互相影响，对于容错、安全有更高的要求，架构也变得复杂了。

所以把什么样的任务放在一起，需要仔细选择。一般适用于那些对资源要求不高的，大部分时间都在等待的任务。


### Event Sourcing Pattern

一般情况下，我们会存储数据的最新状态。
Event Sourcing 模式利用 Log 的思想，把对数据的所有操作，当做一个个的 Event 存储起来。
通过订阅并且回放这些 Event，我们可以知道数据在在任何时刻的状态。

Event Stream 的量可能会比较大，这时可以考虑在合适的间隔（比如每多少个 Event）时创建 snapshot。

Event 的发布，很可能是“至少一次（at least once”的策略，所以订阅方需要保证 Event 的消费处理时幂等的。


### External Configuration Store Pattern

多个 App 之间，一个 App 的多个实例之间往往需要共享配置。
这个模式就是指把配置存到一个外部的专门服务中。该服务提供接口，实现对配置的读和写。
配置服务的实现上，需要考虑很多细节问题，比如权限控制、配置的多版本、配置的数据类型等等。


### Federated Identity Pattern

联合身份模式，指 App 把身份认证教给专门的 ID Provider 来进行。优点是简化了 App 的开发，提升了用户体验。

常用的认证方式是“claims-based access control”：
1. App/Service 信任 ID Provider（IdP)；
2. Client 与 IdP 练习，进行身份认证；
3. 如果认证通过，IdP 会向 Security Token Service(STS) 返回一个 token，STS 通过该 token 能得到用户的信息（IdP 和 STS 往往是同一个服务）和用户想要的到的权限；
4. STS 根据预先设定的规则，对 token 进行转换，添加一些必要的信息，生成一个新的 token，然后返回给 Client；
5. Client 使用这个 token 来请求 App/Service；


### Gatekeeper Pattern

看门人模式，就是在 App/Service 之前专门增加一个 Gatekeeper 服务，该服务做请求的合法性校验、对请求进行一些清洗（sanitizes requests），然后在 Client 和 App/Services 之间传递数据。

该模式的目的在于增加一层安全性保障。


### Health Endpoint Monitoring Pattern

### Index Table Pattern

### Leader Election Pattern

### Materialized View Pattern

### Pipes and Filters Pattern

### Priority Queue Pattern

### Queue-Based Load Leveling Pattern

### Retry Pattern

### Runtime Reconfiguration Pattern

### Scheduler Agent Supervisor Pattern

### Sharding Pattern

### Static Content Hosting Pattern

### Throttling Pattern

### Valet Key Pattern


## Guidances

### Asynchronous Messaging Primer

### Autoscaling Guidance



### Caching Guidance

### Compute Partitioning Guidance

### Data Consistency Primer

### Data Partitioning Guidance

### Data Replication and Synchronization Guidance

### Instrumentation and Telemetry Guidance

### Multiple Datacenter Deployment Guidance

### Service Metering Guidance


