---
title: C++ 并发二 线程间共享数据
date: 2021-11-01 02:09:41
tags:
---

# 线程间共享数据

解决恶性条件竞争最简单的办法就是对数据结构采用某种保护机制，确保只有进行修改的线程才能看到不变量被破坏时的中间状态。从其他访问线程的角度来看，修改不是已经完成了，就是还没开始。C++标准库提供很多类似的机制。

另一个选择是对数据结构和不变量的设计进行修改，修改完的结构必须能完成一系列不可分割的变化，也就是保证每个不变量保持稳定的状态，这就是所谓的无锁编程。

### 互斥量

C++ 中通过实例化 `std::mutex` 创建互斥量实例，通过成员函数 `lock()` 对互斥量上锁，`unlock()` 进行解锁。不过，实践中不推荐直接去调用成员函数，调用成员函数就意味着，必须在每个函数出口都要去调用 `unlock()`，也包括异常的情况。C++ 标准库为互斥量提供了一个 RAII 语法的模板类 `std::lock_guard`，在构造时就能提供已锁的互斥量，并在析构的时候进行解锁，从而保证了一个已锁互斥量能被正确解锁。

```cpp
#include <list>
#include <mutex>
#include <algorithm>
std::list<int> some_list;    // 1
std::mutex some_mutex;    // 2
void add_to_list(int new_value)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 3
  some_list.push_back(new_value);
}
bool list_contains(int value_to_find)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 4
  return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end();
}
```

当成员函数返回的是保护数据的指针或引用时，会破坏数据。具有访问能力的指针或引用可以访问(并可能修改)被保护的数据，而不会被互斥锁限制。这就需要对接口有相当谨慎的设计，要确保互斥量能锁住数据的访问，并且不留后门。