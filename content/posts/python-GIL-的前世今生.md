---
title: "Python GIL 的前世今生"
date: 2020-03-30T10:10:58+08:00
draft: false
categories: [Python, GIL]
---

简而言之，Python 的全局解释器（[GIL](https://wiki.python.org/moin/GlobalInterpreterLock)）是一种互斥锁，它仅允许一个线程持有 Python 解释器的控制权。这意味着在任何时间点只有一个线程处于执行状态。对于执行单线程程序的开发人员而言，GIL 的影响并不明显，但它可能是 CPU 密集和多线程代码的性能瓶颈。

由于即使在具有多个 CPU 内核的多线程体系结构中，GIL 一次仅允许一个线程执行，GIL 因此以 Python 的“臭名昭著”功能而闻名。

在本文中，您将学习到 GIL 如何影响 Python 程序的性能，以及如何减轻 GIL 对代码的影响。本文所有代码运行环境为

```shell
[I] ➜ python --version
Python 3.7.6
```

### GIL 为 Python 解决了什么问题

Python 使用引用计数进行内存管理。这意味着用 Python 创建的对象具有引用计数变量，该变量跟踪指向该对象的引用数，当词计数到达零时，将释放该对象占用的内存。

下面是一个简短的代码示例，以演示引用计数的工作原理：

```python &gt;&gt;&gt;
>>> a = []
>>> b = a
>>> sys.getrefcount(a)
3
```

在上面示例中，空列表对象 [] 的引用计数为3，它被 a，b 和传入 sys.getrefcount() 的参数引用。

回到GIL：

问题在于该引用计数变量需要保护，以防止两个线程同时增加或减少其值的竞争状态。如果发生这种情况，则可能导致未释放的内存泄漏，更糟糕的是，在仍然存在对该对象引用的情况下，错误地释放了内存。这可能会导致程序崩溃或其他奇怪的错误。

通过将锁添加到跨线程共享的所有数据结构中，以确保它们不会被不一致地修改，可以保持此引用计数变量的安全。但是对每个对象或者对象组加锁意味着多个锁将同时存在，这将导致另外一个问题——死锁（死锁只可能发生在有多个锁的情况下）。另一个副作用就是重复的请求和释放锁而导致性能降低。

GIL 是解释器本身的单一锁，它添加了一个规则，即任何 Python 字节码的执行都需要获取解释锁。这可以防止死锁（因为只有一个锁），并且不会带来太多的性能开销。但它也会使所有 CPU 密集型的 Python 程序成为单线程。

尽管解释器用于其他语言（例如 Ruby），但 GIL 并不是解决次问题的唯一方法。某些语言通过使用引用计数以外的方法（例如垃圾回收）来避免使用 GIL 对线程安全的内存管理。另一方面，这意味着这些语言必须通过添其他性能提升功能（例如 JIT 编译器）来弥补 GIL 的单线程性能优势的损失。

## 为什么选择 GIL 作为解决方案

那么，为什么在 Python 中使用了一种看起来如此碍事的方法？Python 开发者是否做了一个错误的决定？

用 [Larry Hastings](https://www.youtube.com/watch?v=KVKufdTphKs&feature=youtu.be&t=12m11s) 的话来说，GIL 的设计决定是使 Python 像今天一样流行的原因之一。

Python 使用 GIL 时多核还不常见，它被设计为易于使用，以加快开发速度。

现有的 C 库正在编写许多扩展，Python 需要这些 C 库中的功能。为了防止不一致的更改，这些 C 扩展需要 GIL 提供的线程安全内存管理。GIL 易于实现并且可以轻易的加入到 Python 中。由于只需要管理一个锁，因此它可以提高单线程程序的性能。

非线程安全的 C 库变得易于集成，而这些 C 扩展正是 Python 快速地被不同社区接受的原因之一。

所以说，GIL 是 CPython 开发人员在 Python 生命早期面临的一个难题的务实解决方案。

## 对多线程 Python 程序的影响

当您查看典型的 Python 程序（或与此相关的任何计算机程序）时，CPU 密集型与 I/O 密集型的性能之间是有区别的。

CPU 密集型程序是将 CPU 性能使用到极致的程序。这包括进行数学运算的程序，例如矩阵乘法，搜索，图像处理等。

I/O 密集型程序是花费时间等待输入/输出（这可能来自用户，文件，数据库，网络等）的程序。I/O 密集型程序有时候不得不等待大量的时间，知道他们从源中获取所需的内容位置，原因是在输入/输出准备就绪之前，源可能需要自行处理。举例而言，用户考虑要在自己的进程中运行的输入框或数据库查询中输入什么。

让我们看一个执行倒数计数的 CPU 密集型的简单程序：

```python single_thread.py
import time

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


start = time.time()
countdown(COUNT)
end = time.time()

print('Time taken in seconds -', end - start)
```

在我的电脑四核系统上运行代码得到如下输出：

```shell
[I] ➜ python single_thread.py
Time taken in seconds - 2.659637928009033
```

现在，我修改了一部分代码来并行地使用两个线程进行倒计时计数：

```python multi_thread.py
import time
from threading import Thread

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


t1 = Thread(target=countdown, args=(COUNT // 2, ))
t2 = Thread(target=countdown, args=(COUNT // 2, ))

start = time.time()
t1.start()
t2.start()
t1.join()
t2.join()
end = time.time()

print('Time taken in seconds -', end - start)
```

再次运行代码：

```shell
[I] ➜ python multi_thread.py
Time taken in seconds - 2.664177894592285
```

如你所见，两个版本运行时间几乎相同。在双线程的版本中，GIL 阻止 CPU 密集型线程并行运行。

GIL 对 I/O 密集型的多线程程序的性能影响不大，因为在线程等待 I/O 时他们之间共享锁。但是如上例所示，与将其编写为完全单线程的方案相比，CPU 密集型程序（例如使用线程处理图像的程序）不仅会由于锁而变为单线程，而且执行时间也会增加。这种增加是锁增加了获取和释放开销的结果。

## 为什么 GIL 至今还未被移除

Python 开发者对此有很多抱怨，但是像 Python 这样流行的语言不可能在移除 GIL 这么大的变化的情况下不导致向后不兼容的问题。

GIL 显然可以被移除并且已有一些开发者或研究院团队完成了，但这些尝试都破坏了现有的 C 扩展库，这些扩展库在很大程度上依赖于 GIL 提供的解决方案。

当然，对于 GIL 解决的问题，还有其他解决方案，但是其中一些解决方案降低了单线程和多线程 I/O 密集型程序的性能，其中有些太难实现了。

Python 的创建者 Guido Van Rossum 在 2007 年 9 月的文章[“It isn’t Easy to remove the GIL”](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)中向社区做出了回答：

>   “I’d welcome a set of patches into Py3k *only if* the performance for a single-threaded program (and for a multi-threaded but I/O-bound program) *does not decrease*”

此后的任何尝试都无法满足这一条件。

## 为什么不在 Python3 中移除

Python3 确实有机会从头开始实现许多功能，但是在此过程中破坏了一些现有的C扩展库，这些扩展库随后需要进行更新并移植到 Python3 才能使用，这就是 Python3 的早期版本被社区接受速度较慢的原因。

但是为什么不干脆移除掉 GIL？

相较于 Python2，移除 GIL 会使 Python3 的单线程性能变慢，并且可以想象会导致什么。您无法反对 GIL 的单线程性能优势。因此，结果是 Python3 仍然保留有GIL。

但是 Python3 确实对现有的 GIL 进行了重大的改进。

我们讨论了 GIL 对 “仅 CPU 密集型” 和 “仅 I/O 密集型”多线程程序的影响，但是其中一些线程时 I/O 密集型而某些线程时 CPU 密集型的程序又如何呢？ 

在这类程序中，众所周知，Python 的 GIL 会给 I/O 密集型线程带来饥饿，因为它们没有机会从 CPU 绑定线程获取 GIL。 

这是因为 Python 内置了一种机制，该机制强制线程在固定的连续使用时间间隔后释放 GIL，如果没有其他人获得 GIL，则同一线程可以继续使用它。

```python &gt;&gt;&gt;
>>> import sys
>>> # The interval is set to 100 instructions:
>>> sys.getswitchinterval()
0.005
```

该机制中的问题是 CPU 密集型的线程会在其他线程获取 GIL 之前先发起请求。这是由David Beazley进行的研究，可以在[此处](http://www.dabeaz.com/blog/2010/01/python-gil-visualized.html)找到可视化效果

这个问题在 2009 年的 Python3.2 中由 Antoine Pitrou 修复，他添加了一种[机制](https://mail.python.org/pipermail/python-dev/2009-October/093321.html)来查看被丢弃的其他线程的 GIL 获取请求的数量，并且不允许当前线程在其他线程有机会运行之前重新获取 GIL。

## 如何处理 Python 的 GIL

如果 GIL 导致您遇到问题，请尝试一下几种方法：

**多进程**：最流行的方法是使用多进程替代多线程。每个 Python 进程有一个自己的解释器和内存空间，所以 GIL 不会造成问题。Python 有一个 [`multiprocessing`](https://docs.python.org/2/library/multiprocessing.html)模块让我们可以像这样轻松的创建进程：

```python multi_process.py
import time
from multiprocessing import Pool

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


if __name__ == '__main__':
    pool = Pool(processes=2)
    start = time.time()
    r1 = pool.apply_async(countdown, [COUNT // 2])
    r2 = pool.apply_async(countdown, [COUNT // 2])
    pool.close()
    pool.join()
    end = time.time()
    print('Time taken in seconds -', end - start)
```

运行代码：

```shell
[I] ➜ python multi_process.py
Time taken in seconds - 1.4502887725830078
```

与多线程版本代码相比，性能显著提高了，不是吗？

时间并没有减少到我们上面看到的一半，因为进程管理有其自己的开销。多个进程比多个线程重，因此请记住，这可能会成为扩展瓶颈。

时间并没有减少到我们上面看到的一半，因为流程管理有其自己的开销。多个进程比多个线程重，因此请记住，这可能会成为扩展瓶颈。

**替换 Python 解释器**：Python 有多种解释器的实现。最受欢迎的分别是用 C，Java 和 C# 编写的 CPython，Jython，IronPython。 GIL 仅存在于原始 Python 实现中，即 CPython。如果您的程序及其库可用于其他实现之一，则也可以尝试一下。

**只需要等等**：尽管许多 Python 用户利用了 GIL 的单线程性能优势。多线程程序员也不必烦恼，因为 Python 社区中一些最聪明的人正在努力从 CPython 中删除 GIL。其中一种这样的尝试被称为 [Gilectomy](https://github.com/larryhastings/gilectomy)。

Python GIL通常被认为是一个神秘而困难的话题。但是请记住，作为 Pythonista，通常只有在编写 C 扩展或在程序中使用 CPU 密集型多线程时才受到它的影响。 在这种情况下，本文应为您提供了解 GIL 以及在您自己的项目中如何处理 GIL 所需的一切。而且，如果您想了解 GIL 的底层工作原理，我建议您观看 David Beazley 的 [Understanding the Python GIL](https://youtu.be/Obt-vMVdM8s) 演讲。

本文翻译自 [What is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil/)，有兴趣的可以去看看原文。

## 链接

1. [What is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil/)

2. [Gilectomy](https://github.com/larryhastings/gilectomy)

3. [Understanding the Python GIL](https://youtu.be/Obt-vMVdM8s)

4. [机制](https://mail.python.org/pipermail/python-dev/2009-October/093321.html)

5. [此处](http://www.dabeaz.com/blog/2010/01/python-gil-visualized.html)
6. [“It isn’t Easy to remove the GIL”](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)
7. [Larry Hastings](https://www.youtube.com/watch?v=KVKufdTphKs&feature=youtu.be&t=12m11s) 

