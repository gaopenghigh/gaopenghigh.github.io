---
layout: post
title:  "Cloud Design Patterns 学习笔记"
date:   2017-07-20 14:30:00 +0800
categories: Tech
toc: true
---

写于 2016-12-21 。

Cloud Design Patterns 学习笔记
----


微软发布了 "[Cloud Design Patterns: Prescriptive Architecture Guidance for Cloud Applications](https://msdn.microsoft.com/en-us/library/dn568099.aspx) "。
书中，针对云应用总结出了多种设计模式和 Guidance 。本文简要记录一下这些 pattern 和 guidance 的思想。


# Ambassador Pattern

对于遗留服务，它们可能缺少一些我们需要的功能，但我们又离不开它们所提供的的服务。同时这些遗留服务往往都不方便进行改造。
这时候就可以使用 Ambassador Patter，在客户端和服务端之间加一个“代理”，把原来的服务包装起来，提供我们需要的额外功能，比如重试、断路、监控、安全等等。

当然，加一个“代理”会代理更多的成本，有时候提供一个客户端的类库会是更好的方式。


# Anti-Corruption Layer Pattern

这个模式也是适用于遗留服务。简单地说就是在新服务和遗留服务之间建立一个转换层，所谓的 Anti-Corruption Layer。通过这一层保证新服务设计的干净，同时又能继续使用遗留服务，知道遗留服务都改造完成。


# Backends for Frontends Pattern

即为每一种前端，建立专门的后端服务，这些后端服务又依赖同一套基础服务。
好处是可以针对某种前端方便灵活地提供一些专有功能，而无需考虑其他前端的情况。
代价显然是架构更复杂、模块更多、成本更大。


# Bulkhead Pattern

Bulkhead 就是船舱里面的隔板。这些隔板把船舱分隔成很多段，当其中一个段损坏时，海水只会灌进这一个段，而其他段仍然保持完好，船也不会因此沉没。

这个模式就是把服务分成多个组，组与组之间互相隔离，一个组中的失败不会导致整体的失败。比如同一个服务部署多个集群，集群之间互相隔离，每个集群提供给特定的客户使用。

客户端也可以把资源分组，比如为每个依赖的服务分配一个连接池，这样当其中一个服务不响应时，只会耗尽分配给那个服务的连接池，而不会导致整个客户端的连接耗尽。

另外，讲服务分组，还可以为每个组设置优先级，优先保证重要的业务。


# Cache-aside Pattern

如果一个 cache 服务没有提供 read-through 和 write-through/write-behind 的功能，比如存粹的缓存服务如
redis 或 memcached 等。可以通过 cache-aside pattern 来模拟 read-through 和
write-through/write-behind 的功能。

大概的过程是：
1. APP 查看 cache 中是否有需要的数据，有的话直接拿到；
2. 如果没有，就从数据存储服务中拿到这个数据；
3. 在主动的往 cache 中存入这份数据。


# Circuit Breaker Pattern

Circuite Breaker 就是断路器，该设计模式用来避免不必要的重试，从而避免雪崩效应。

断路器有3种状态：

- **Closed**：表示断路器是接通的，即认为后端的服务都是正常的，此时请求将会被正常地提交到服务。当短时间内针对某个服务或资源的失败调用过多时，即当短期内有超过阈值的失败出现时，状态变为 Open；
- **Open**：表示断路器是断开的，即认为后端的服务出现问题了，此时请求不会提交到服务，而是直接返回失败。进入该状态时，一般会设置一个定时器，定时器到期时把断路器置为 Half-Open 状态；
- **Half-Open**：表示一种中间状态，断路器不确定后端的服务是否正常，此时会允许部分请求提交到后端，如果这些请求成功了，那就把状态设置为 Closed，否则就回到 Open 状态。

代码实现上，可以创建一个 Circuit Breaker ，把对服务请求相关的逻辑包裹起来。如果是被断路器阻止了，则返回一种特殊的异常，做特殊的处理。

一般还需要一个手工开关，可以把断路器强制设置为某种状态。

在 Half-Open 状态时，除了被动地检查成功和失败数，还可以通过主动的健康检查，来决定下一步的状态。


# Command and Query Responsibility Segregation (CQRS) Pattern

即所谓的“命令与查询职责分离”。日常经常说的“读写分离”，其实也算是 CQRS。

CQRS 的核心内容是针对命令（写）和查询（读），使用不同的数据模型，最终数据也存储在不同的存储服务上。

CQRS 和 Event Sourcing 模式经常结合在一起使用。
写模型就是把数据转换为 Event 存入 Event Store。
读模型就是从 Event 中读取 Event stream，转换为需要的业务结构。
这两个模式结合的缺点也很明显：读和写的延迟、额外的复杂性和处理 Event 需要的额外资源。


# Compensating Transaction Pattern

补偿事务模式。

一个事务，可能包含一系列的步骤，每个步骤需要调用不同的服务完成。
通过这一系列的步骤，整体上实现最终一致性（eventually consistent）。

所谓的 Compensating Transaction Pattern 就是指，当其中的某一步失败时（经过了一些重试后），
系统自动进行一些列的补偿步骤，比如把之前已经成功的那些步骤回滚掉，实现最终一致性。
这些补偿步骤自己也是有可能会失败的，实现过程中会有一些重试，所以补偿步骤需要是幂等的。

补偿步骤不一定要正好是事务步骤的逆序，而是应该根据具体业务具体实现。


# Competing Consumers Pattern

这个模式很简单，其实就是引入一个消息队列。Consumers 从队列中获取任务执行。

# Compute Resource Consolidation Pattern

业务逻辑经常包含多个任务，一般情况下，这些任务会部署在独立的计算单元中（虚拟机、容器等）。
这个模式就是把一些任务打包在一起，放在同一个计算单元中运行。
带来的好处就是能够提升资源利用率，降低成本。

但是把任务放到一起，同时也可能带来新的问题，最直接的就是可能增加了复杂度。
原来解耦的任务现在放到了一起，任务之间互相影响，对于容错、安全有更高的要求，架构也变得复杂了。

所以把什么样的任务放在一起，需要仔细选择。一般适用于那些对资源要求不高的，大部分时间都在等待的任务。


# Event Sourcing Pattern

一般情况下，我们会存储数据的最新状态。
Event Sourcing 模式利用 Log 的思想，把对数据的所有操作，当做一个个的 Event 存储起来。
通过订阅并且回放这些 Event，我们可以知道数据在在任何时刻的状态。

Event Stream 的量可能会比较大，这时可以考虑在合适的间隔（比如每多少个 Event）时创建 snapshot。

Event 的发布，很可能是“至少一次（at least once”的策略，所以订阅方需要保证 Event 的消费处理时幂等的。


# External Configuration Store Pattern

多个 App 之间，一个 App 的多个实例之间往往需要共享配置。
这个模式就是指把配置存到一个外部的专门服务中。该服务提供接口，实现对配置的读和写。
配置服务的实现上，需要考虑很多细节问题，比如权限控制、配置的多版本、配置的数据类型等等。


# Federated Identity Pattern

联合身份模式，指 App 把身份认证教给专门的 ID Provider 来进行。优点是简化了 App 的开发，提升了用户体验。

常用的认证方式是“claims-based access control”：
1. App/Service 信任 ID Provider（IdP)；
2. Client 与 IdP 练习，进行身份认证；
3. 如果认证通过，IdP 会向 Security Token Service(STS) 返回一个 token，STS 通过该 token 能得到用户的信息（IdP 和 STS 往往是同一个服务）和用户想要的到的权限；
4. STS 根据预先设定的规则，对 token 进行转换，添加一些必要的信息，生成一个新的 token，然后返回给 Client；
5. Client 使用这个 token 来请求 App/Service；


# Gatekeeper Pattern

看门人模式，就是在 App/Service 之前专门增加一个 Gatekeeper 服务，该服务做请求的合法性校验、对请求进行一些清洗（sanitizes requests），然后在 Client 和 App/Services 之间传递数据。

该模式的目的在于增加一层安全性保障。


# Gateway Aggregation Pattern

有时候，Client 依赖多个 service，Client 需要请求这些 service 然后合并成为一个结果。
当 Client 与 Service 之间网络不够好时，这会引入严重的性能问题。

Gateway Aggration Pattern 是指在 Client 和 Services 之间引入一个 Gateway，Client 发送一个请求给 Gateway，Gateway 再去请求依赖的所有服务，把这些服务的返回聚合成为一个结果，最后返回给 Client。


# Gateway Offloading Pattern

这个模式就是把一些 Service 之间共享的特性，集中到一个新的 Gateway 层进行。
这个特性尤其适用于证书管理、鉴权、SSL termination、监控、协议转换、流量控制等等。


# Gateway Routing Pattern

就是最常见的的 Gateway 路由模式，service 不直接暴露给 client，而是统一经由 Gateway 进行路由。


# Health Endpoint Monitoring Pattern

这是非常有用的一个模式，Service 对外暴露一个接口，可以查询当前 Service 的健康状况。

要点：
* 返回信息的多少、实现难度以及对性能的影响，这三个方面需要有合适的的折中；
* 可以考虑暴露多个健康检查接口，分别检查不同重要性的健康情况；
* 健康检查的结果可以适当缓存，以减少对性能的影响；
* 健康检查接口需要考虑安全性；


# Index Table Pattern

这是最常用的设计模式之一。就是为需要查询的数据生成一个索引表。


# Leader Election Pattern

就是常用的选主模式。大量的分布式系统都用了该模式，并且可以基于 ZooKeeper，etcd，Consul 等开源软件快速地实现 Leader Election。


# Materialized View Pattern

当做数据的存储是，基于不同的存储服务，开发者考虑的往往是“写友好”，而不是“读友好”。
这就导致一些查询的请求，可能需要从存储服务读很多次，才能合并出需要的结果。

该模式的思路，就是主动地生成一些方便读取查询的结果（Materialized View）。这些结果永远不会被直接更新，可以看做是一种特殊的缓存。这些缓存可能存储在完全不同的存储服务上。

需要考虑的点：
1. 什么时候去更新这些 Materialized View，一帮情况下，可以使用一个任务，或通过某种触发来生成 View。
2. 考虑数据一致性，这是所有缓存都需要考虑的点。
3. 考虑为 Materialized View 创建索引表，进一步优化查询性能。


# Pipes and Filters Pattern

就是把一个任务分解成一系列的小任务（Filter），然后用管道（Pipe）串联起来，数据在管道中流通。
好处是：
1. 可以根据每一个 Filter 自身的情况为其分配资源；
2. 利于扩展改造；

由于每一个 Filter 实例可能会失败，这会导致同一份数据在 Filter 的两个实例上被执行两次，这就：
1. Filter 需要是幂等的。
2. Pipeline 需要能够检查并且去除重复消息。

该模式可以和补偿事务模式（Compensating Transaction Pattern）一起使用，来实现分布式事务。
因为一个分布式事务可以拆分成为多 Filter，每个 Filter 自身再实现补偿事务模式。


# Priority Queue Pattern

对于需要区分优先级的任务的处理，可以通过引入一个优先级队列来实现。

这种模式还有几个变种。

第一个变种是使用多个队列，每个优先级对应一个队列。有一个 Customer Pool，里面的 Customer 总是先试图从高优先级的队列中获取任务，没有的话在尝试下一优先级的队列。

第二个变种是每个优先级对应一个队列，同时还对应一个 Customer Pool，根据优先级的大小确定 Pool 的大小。

第三个变种是动态地改变优先级队列中元素的优先级，随着一个元素在队列中待的时间增加，它的优先级也会逐渐增加。

多队列的方式，有利于系统性能和可扩展性的最大化。


# Queue-Based Load Leveling Pattern

基于队列的负载均化模式。
就是在 Service 之前引入一个队列，作为任务的缓冲区。这样可以避免突如其来的负载高峰。
Service 根据自己的能力，按照一定的速率从队列中获取任务执行。


# Retry Pattern

就是考虑失败的情况并按照一定策略重试。
关键是根据业务情况，制定合适的重试策略。
可以联合断路器模式（Circuit Breaker pattern）一起使用。


# Scheduler Agent Supervisor Pattern

这个模式比较复杂，但很有用。

在分布式的环境中，一个任务，经常会由一系列的步骤组成。
每一个步骤又可能依赖于某个远程服务或远程资源。
这些步骤按照一定的次序执行，只有当所有的步骤都成功时，这个任务才算作成功。

Scheduler Agent Supervisor 模式包含了三种逻辑上的角色：
1. Scheduler 的职责是保证任务的各个步骤按照业务需要的正确顺序执行。 Scheduler 需要维护每个步骤的状态信息，包括步骤的过期时间。这些信息存放在持久型数据存储服务中，叫做 state store。如果一个步骤需要调用远程服务，则把相应的信息发送给对应的 Agent。 
2. Agent 负责代理一个远程服务或远程资源，收到一个消息/请求后，它会根据消息的内容，调用自己代理的远程服务，然后把结果返回给 Scheduler。 Agent需要自己实现重试等机制。每一个远程服务都
3. Supervisor 周期性地运行，负责监控由 Scheduler 执行的所有步骤是否在规定的时间内正确完成。

Scheduler，Agent 和 Supervisor 都可以以多实例运行。不过 Supervisor 的多实例之间需要互相同步状态，或者通过 Leader Election 模式选主运行。

这个设计模式的工作方式如下：
1. Application 需要运行一个任务时，就向 Scheduler 提交一个请求。
2. Scheduler 收到请求后，在 state store 中初始化该任务以及它所有步骤的状态，然后根据业务逻辑顺序来执行步骤。这些步骤必须是幂等的。
3. 如果一个步骤依赖远程服务，Scheduler 就像对应的 Agent 发送一个消息。
4. Agent 收到消息后，根据其内容，调用远程服务，同时还会根据消息中的过期时间，决定是否需要终止执行。如果一切正常，Agent 会向 Scheduler 发送一个运行成功的消息。如果 Agent 没在过期时间前完成工作，则不会向 Scheduler 发送任何消息。
5. Supervisor 会定期检查 state store 中的所有状态，看看有哪些步骤超时了或失败了，然后尝试恢复它们。这可以通过更新步骤的过期时间，然后通知 Scheduler 实现。也可以通过 Scheduler 主动轮询 state store 实现。如果一个步骤一直失败，Supervisor 还需要知道什么时候不应该再尝试，或者进一步实现更复杂的策略。Supervisor 最本质的责任，就是当一个步骤失败是，决定是重试该步骤，还是让整个任务失败掉。
6. Scheduler 本身也会异常，所以当一个 Scheduler Fail 时，新的或其他的 Scheduler 需要能够恢复那些执行了一半的任务。

Scheduler Agent Supervisor 设计模式最核心的优点，就是在分布式环境下的那些临时的、意外的和不可恢复的异常情况下，整个系统是可“自恢复”的。


# Sharding Pattern

就是常用的分片（Sharding）模式。关键是根据业务，选择合适的 Sharding 策略。


# Sidecar Pattern

一个 Application 可能需要一系列的附加功能，比如监控、日志、配置等等。
这些服务和 App 本身的关系不大，可以和 App 的每一个实例部署在一起。
就像边三轮摩托车的 Sidecar 一样。
容器天然适合 Sidecar 模式，每个 App 的容器都附带一个 Sidecar 容器。


# Static Content Hosting Pattern

就是简单的动静分离模式，把静态内容放在合适的服务上。


# Strangler Pattern

老的系统有时需要更新、升级、重构为新系统，这将是一个持续的过程。
Strangler 模式就是把新老系统都接入一个 Strangler 层，Strangler 把改造好了的接口路由给新系统，没有改造好的系统路由给老系统。当整个系统改造完成后，老系统就没有任何请求，可以下线了。


# Throttling Pattern

系统压力可能会出现突然的变化，Throttling Pattern 就是通过一定的限制策略，让整个系统的资源消耗保持在安全范围内。

限制策略包括：
1. 解决部分请求，比如当一个用户每秒请求数超过每个阈值之后，就不再提供服务；
2. 服务降级，次要的服务把资源让给重要的服务使用
3. 通过引入 Queue-Based Load Leveling Pattern 来避免负载高峰


# Valet Key Pattern

有些服务，比如文件的上传下载，往往是由专业的云服务来提供。
Application 只负责管理控制这些资源，而不负责资源本身的处理。
Valet Key 模式，就是指当 Client 请求对一个资源进行操作时，返回给 Client 一个 token，叫做 valet token，Client 拿着这个 token 去请求其他服务。
资源服务通过 Token 可以知道这个 Client 所拥有的权限，以及这些权限的有效期。
