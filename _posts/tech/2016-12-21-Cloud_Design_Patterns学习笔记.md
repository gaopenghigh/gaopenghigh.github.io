---
layout: post
title:  "Cloud Design Patterns 学习笔记"
date:   2012-12-21 14:00:00 +0800
categories: Tech
toc: true
---

写于 2012-12-21 。

Cloud Design Patterns 学习笔记

微软发布了 "[Cloud Design Patterns: Prescriptive Architecture Guidance for Cloud Applications](https://msdn.microsoft.com/en-us/library/dn568099.aspx) "。书中，针对云应用总结出了 24 个设计模式和 10 个 Guidance 。本文简要记录一下这些 pattern 和 guidance 的思想。

## Patterns

### 1. Cache-aside Pattern

如果一个 cache 服务没有提供 read-through 和 write-through/write-behind 的功能，比如存粹的缓存服务如
redis 或 memcached 等。可以通过 cache-aside pattern 来模拟 read-through 和
write-through/write-behind 的功能。

大概的过程是：
1. APP 查看 cache 中是否有需要的数据，有的话直接拿到；
2. 如果没有，就从数据存储服务中拿到这个数据；
3. 在主动的往 cache 中存入这份数据。


### 2. Circuit Breaker Pattern

Circuite Breaker 就是断路器，该设计模式用来避免不必要的重试，从而避免雪崩效应。

断路器有3种状态：
- **Closed**：表示断路器是接通的，即认为后端的服务都是正常的，此时请求将会被正常地提交到服务。当短时间内针对某个服务或资源的失败调用过多时，即当短期内有超过阈值的失败出现时，状态变为 Open；
- **Open**：表示断路器是断开的，即认为后端的服务出现问题了，此时请求不会提交到服务，而是直接返回失败。进入该状态时，一般会设置一个定时器，定时器到期时把断路器置为 Half-Open 状态；
- **Half-Open**：表示一种中间状态，断路器不确定后端的服务是否正常，此时会允许部分请求提交到后端，如果这些请求成功了，那就把状态设置为 Closed，否则就回到 Open 状态。

代码实现上，可以创建一个 Circuit Breaker ，把对服务请求相关的逻辑包裹起来。如果是被断路器阻止了，则返回一种特殊的异常，做特殊的处理。

一般还需要一个手工开关，可以把断路器强制设置为某种状态。

在 Half-Open 状态时，除了被动地检查成功和失败数，还可以通过主动的健康检查，来决定下一步的状态。


### 3. Compensating Transaction Pattern

补偿事务模式。

一个事务，可能包含一系列的步骤，每个步骤需要调用不同的服务完成。
通过这一系列的步骤，整体上实现最终一致性（eventually consistent）。

所谓的 Compensating Transaction Pattern 就是指，当其中的某一步失败时（经过了一些重试后），
系统自动进行一些列的补偿步骤，比如把之前已经成功的那些步骤回滚掉，实现最终一致性。
这些补偿步骤自己也是有可能会失败的，实现过程中会有一些重试，所以补偿步骤需要是幂等的。

补偿步骤不一定要正好是事务步骤的逆序，而是应该根据具体业务具体实现。


### 4. Competing Consumers Pattern

### 5. Compute Resource Consolidation Pattern

### 6. Command and Query Responsibility Segregation (CQRS) Pattern

### 7. Event Sourcing Pattern

### 8. External Configuration Store Pattern

### 9. Federated Identity Pattern

### 10. Gatekeeper Pattern

### 11. Health Endpoint Monitoring Pattern

### 12. Index Table Pattern

### 13. Leader Election Pattern

### 14. Materialized View Pattern

### 15. Pipes and Filters Pattern

### 16. Priority Queue Pattern

### 17. Queue-Based Load Leveling Pattern

### 18. Retry Pattern

### 19. Runtime Reconfiguration Pattern

### 20. Scheduler Agent Supervisor Pattern

### 21. Sharding Pattern

### 22. Static Content Hosting Pattern

### 23. Throttling Pattern

### 24. Valet Key Pattern


## Guidances

### 1. Asynchronous Messaging Primer

### 2. Autoscaling Guidance



### 3. Caching Guidance

### 4. Compute Partitioning Guidance

### 5. Data Consistency Primer

### 6. Data Partitioning Guidance

### 7. Data Replication and Synchronization Guidance

### 8. Instrumentation and Telemetry Guidance

### 9. Multiple Datacenter Deployment Guidance

### 10. Service Metering Guidance


