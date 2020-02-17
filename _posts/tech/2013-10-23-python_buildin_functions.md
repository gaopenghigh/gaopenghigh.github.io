---
layout: post
title: Python 内建函数和函数式编程
date:   2013-10-23 14:30:00 +0800
categories: Tech
toc: true
---

# Python 内建函数和函数式编程

关于Python内建函数的学习笔记。

**关键字：**
Python内建函数，函数式编程，Functional Programming


### iter()

把一切可以变成iterator的对象对变成iterator.

### dict()

    >>> L = [('Italy', 'Rome'), ('France', 'Paris'), ('US', 'Washington DC')]
    >>> dict(L)
    {'Italy': 'Rome', 'US': 'Washington DC', 'France': 'Paris'}

### Generator和yield

一个递归的generator，实现中根次序遍历树：

    def inorder(t):
        if t:
            for x in inorder(t.left):
                yield x
            yield t.label
            for x in inorder(t.right):
                yield x

Generator的状态还可以中途被改变，通过`send()`方法对generator传值，而在generator
中通过`val = (yield i)`方法获得这个值，然后根据需要做一些变化：

    def counter (maximum):
        i = 0
        while i < maximum:
            val = (yield i)
            # If value provided, change counter
            if val is not None:
                i = val
            else:
                i += 1

    >>> it = counter(10)
    >>> print it.next()
    0
    >>> print it.next()
    1
    >>> print it.send(8)
    8
    >>> print it.next()
    9
    >>> print it.next()
    Traceback (most recent call last):
    File "t.py", line 15, in ?
        print it.next()
    StopIteration

### map()

`map(f, iterA, iterB, ...)` 返回一个列表，该列表内容是：

    f(iterA[0], iterB[0]), f(iterA[1], iterB[1]), f(iterA[2], iterB[2]), ....


### filter()

    >>> def is_even(x):
    ...     return (x % 2) == 0

    >>>>>> filter(is_even, range(10))
    [0, 2, 4, 6, 8]

    >>> [x for x in range(10) if is_even(x)]
    [0, 2, 4, 6, 8]

    existing_files = filter(os.path.exists, file_list)


### reduce()

`reduce(func, iter, [initial_value])`返回对一个iterator不断地执行`func()`操作后
最终的结果。比如iter是`[1, 2, 3]`，则

    reduce(func, iter) 返回 func(func(1, 2), 3)
    reduce(func, iter, 4) 返回 func(func(func(4, 1), 2), 3)
    total = reduce(lambda a, b: (0, a[1] + b[1]), items)[1]


### enumerate()

    >>> for i in enumerate(['subject', 'verb', 'object']):
    ...     print(i)
    ...
    (0, 'subject')
    (1, 'verb')
    (2, 'object')


### any()和all()

*   any(iter)，当iter中有一个元素为真时返回True，相当于很多个`or`。
*   all(iter)，当iter中所有元素都为真时返回True，相当于很多个`and`。


### itertools模块

itertools模块包含一些常用的功能：

#### itertools.count(n)

    itertools.count() => 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, ...
    itertools.count(10) => 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, ...

#### itertools.cycle(iter)

    itertools.cycle([1,2,3,4,5]) => 1, 2, 3, 4, 5, 1, 2, 3, 4, 5, ...

#### itertools.repeat(elem, [n])

    itertools.repeat('abc') => abc, abc, abc, abc, abc, abc, abc, ...
    itertools.repeat('abc', 5) => abc, abc, abc, abc, abc

#### itertools.chain(iterA, iterB, ...)

    itertools.chain(['a', 'b', 'c'], (1, 2, 3)) => a, b, c, 1, 2, 3

#### itertools.izip(iterA, iterB, ...)

    itertools.izip(['a', 'b', 'c'], (1, 2, 3)) => ('a', 1), ('b', 2), ('c', 3)

内建函数zip()也做类似的事，但izip()和zip()不同的是，izip()不会在内存中构造出整
个要返回的列表，而是在需要的时候才去构造一个tuple。

####　itertools.islice(iter, [start], stop, [step])

    itertools.islice(range(10), 2, 8, 2) => 2, 4, 6

#### itertools.imap(f, iterA, iterB, ...)

    itertools.imap(f, iterA, iterB, ...) =>
    f(iterA[0], iterB[0]), f(iterA[1], iterB[1]), f(iterA[2], iterB[2]), ...

#### itertools.starmap(func, iter)

startmap用于iterable的东西会返回一个个tuple，而func使用这些tuple作为参数。

    itertools.starmap(os.path.join,
                      [('/usr', 'bin', 'java'), ('/bin', 'python'),
                       ('/usr', 'bin', 'perl'),('/usr', 'bin', 'ruby')])
    =>
    /usr/bin/java, /bin/python, /usr/bin/perl, /usr/bin/ruby


#### itertools.ifilter(predicate, iter)

类似于filter，不过是iterator。另外还有一个相反的

    itertools.ifilterfalse(predicate, iter)


#### itertools.takewhile(predicate, iter)

    def less_than_10(x):
        return (x < 10)

    itertools.takewhile(less_than_10, itertools.count()) =>
        0, 1, 2, 3, 4, 5, 6, 7, 8, 9

#### itertools.dropwhile(predicate, iter)

    itertools.dropwhile(less_than_10, itertools.count()) =>
        10, 11, 12, 13, 14, 15, 16, 17, 18, 19, ...

#### itertools.groupby(iter, key_func=None)

    city_list = [('Decatur', 'AL'), ('Huntsville', 'AL'), ('Selma', 'AL'),
                 ('Anchorage', 'AK'), ('Nome', 'AK'),
                 ('Flagstaff', 'AZ'), ('Phoenix', 'AZ'), ('Tucson', 'AZ'),
                 ...
                ]

    def get_state ((city, state)):
        return state

    itertools.groupby(city_list, get_state) =>
      ('AL', iterator-1),
      ('AK', iterator-2),
      ('AZ', iterator-3), ...

    where
    iterator-1 =>
      ('Decatur', 'AL'), ('Huntsville', 'AL'), ('Selma', 'AL')
    iterator-2 =>
      ('Anchorage', 'AK'), ('Nome', 'AK')
    iterator-3 =>
      ('Flagstaff', 'AZ'), ('Phoenix', 'AZ'), ('Tucson', 'AZ')


----

参考资料：
* [Functional Programming HOWTO](http://docs.python.org/2/howto/functional.html)

