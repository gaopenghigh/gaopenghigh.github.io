---
layout: post
title:  "Mesos 源码学习(1) Mesos Master 的启动"
date:   2016-12-05 11:00:00 +0800
categories: Mesos
toc: true
---

Mesos Master 启动相关的代码在 `src/master/main.cpp` 中。

## 解析 flags 参数

通过解析以 `MESOS_` 开头的环境变量，来初始化一些参数，并验证参数的合法性。
具体的参数参考 [Mesos Configuration](http://mesos.apache.org/documentation/latest/configuration/)。


## 初始化 libprocess

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


## 初始化 Version process

## 初始化 firewall rules

## 加载模块

## Anonymous modules

## Hooks

## Allocator

## Registry storage

## State

## Master contendor and detector

## Authorizer

## Slave removal rate limiter

## `Master` process
