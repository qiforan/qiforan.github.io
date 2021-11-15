---
title: C++并发(一)
date: 2021-10-29 23:46:32
tags:
---

## 线程管理

### 线程管理的基础

每个程序至少有一个线程：执行 `main()` 函数的(原始)线程。其余线程有其各自的入口函数。

使用 C++ 线程库启动线程，可以归结为构造 `std::thread` 对象：

```cpp
void do_some_work();
std::thread my_thread(do_some_work);
```

使用 `std::thread` 类需引入 `<thread>` 头文件。除了函数，`std::thread` 也可用其他 **可调用类型** 构造。

启动线程后，需要明确是要等待线程结束，<!--more-->还是让其自主运行。


#### 线程等待

如果要等待子线程结束，相关的 `std::thread` 实例需要使用 `join()`，此时调用线程处于阻塞模式。

`join()` 执行完成之后，底层线程 id 被设置为0，即 `joinable()` 变为 `false`。同时会清理线程相关的存储部分， 这样 `std::thread` 对象将不再与已经底层线程有任何关联。这意味着，只能对一个线程使用一次 `join()`;再次调用 `join()`，将返回 `false`。

如果不等待线程，就必须保证线程结束之前，可访问的数据的 **有效性**。很可能线程还没结束，函数已经退出，这时线程函数还持有函数局部变量的指针或引用。下面的代码就展示了这样的一种情况。

```cpp
struct func
{
  int& i;
  func(int& i_) : i(i_) {}
  void operator() ()
  {
    for (unsigned j=0 ; j<1000000 ; ++j)
    {
      do_something(i);           // 1 潜在访问隐患：悬空引用
    }
  }
};
void oops()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread my_thread(my_func);
  my_thread.detach();          // 2 不等待线程结束
}                              // 3 新线程可能还在运行
```

#### 线程分离

`std::thread` 实例对象调用 `detach()` 会导致线程分离。

分离子线程意味着与当前线程的连接被断开，子线程成为后台线程，被 C++ 运行时库接管。这意味着不可能再有 `std::thread` 对象能引用到子线程了。与 `join` 一样，`detach` 也只能调用一次，当 `detach` 以后其 `joinable()` 为 `false`。

#### `std::thread` 析构

`std::thread` 对象析构时，会先判断 `joinable()`，如果可联结，则程序会直接被终止（调用`std::terminate()`）。

联结状态：一个 `std::thread` 对象只可能处于可联结或不可联结两种状态之一。可用 `joinable()` 函数来判断，即 `std::thread` 对象是否与某个有效的底层线程关联（内部通过判断线程 id 是否为 0 来实现）。

* 可联结(joinable)
  - 当线程可运行、己运行或处于阻塞时是可联结的。注意，如果某个底层线程已经执行完任务，但是没有被 join 的话，该线程依然会被认为是一个活动的执行线程，仍然处于 joinable 状态。


* 不可联结（unjoinable)
  - 当不带参构造的 `std::thread` 对象为不可联结，因为底层线程还没创建。
  - 己移动的 `std::thread` 对象为不可联结。因为该对象的底层线程 id 已被设置为 0。
  - 己调用 join 或 detach 的对象为不可联结状态。因为调用 `join()` 以后，底层线程己结束，而 `detach()` 会把`std::thread` 对象和对应的底层线程之间的连接断开。

### 传递线程参数

向 `std::thread` 构造函数中的可调用对象，或函数传递一个参数很简单。

```cpp
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```

需要特别要注意，当指向动态变量的指针作为参数传递给线程的情况，代码如下：

```cpp
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024]; // 1
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer); // 2
  t.detach();
}
```

不能保证构造函数结束时新线程已经执行，只能保证构造函数完成时线程被安排执行，所以可能 `f` 函数开始执行时，`oops` 函数已退出。

参考：[std::thread construction and execution](https://stackoverflow.com/questions/17655880/stdthread-construction-and-execution)

解决方案就是在传递到 `std::thread` 构造函数之前就将字面值转化为 `std::string` 对象。


还可能遇到相反的情况：期望传递一个非常量引用(但这不会被编译)，但整个对象被复制了。

```cpp
void update_data_for_widget(widget_id w,widget_data& data); // 1
void oops_again(widget_id w)
{
  widget_data data;
  std::thread t(update_data_for_widget,w,data); // 2
  display_status();
  t.join();
  process_widget_data(data);
}
```

解决办法是：使用 `std::ref` 将参数转换成引用的形式，从而可将线程的调用改为以下形式：

```cpp
std::thread t(update_data_for_widget,w,std::ref(data));
```

`std::thread` 的传参类似 `std::bind`

```cpp
class X
{
public:
  void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x); // 1
```

这段代码中，新线程将 `my_x.do_lengthy_work()` 作为线程函数；`my_x` 的地址①作为指针对象提供给函数。也可以为成员函数提供参数：`std::thread` 构造函数的第三个参数就是成员函数的第一个参数，以此类推。

### `std::thread` 移动

每个 `std::thread` 实例都负责管理一个执行线程。执行线程的所有权可以在多个 `std::thread` 实例中互相转移，这是依赖于 `std::thread` 实例的可移动且不可复制性。不可复制保性证了在同一时间点，一个 `std::thread` 实例只能关联一个执行线程；可移动性使得开发者可以自己决定，哪个实例拥有实际执行线程的所有权。

```cpp
void some_function();
void some_other_function();
std::thread t1(some_function);            // 1
std::thread t2=std::move(t1);            // 2
t1=std::thread(some_other_function);    // 3
std::thread t3;                            // 4
t3=std::move(t2);                        // 5
t1=std::move(t3);                        // 6 赋值操作将使程序崩溃
```

为了确保线程程序退出前完成，下面的代码里定义了 `scoped_thread`类。

```cpp
class scoped_thread
{
  std::thread t;
public:
  explicit scoped_thread(std::thread t_):                 // 1
    t(std::move(t_))
  {
    if(!t.joinable())                                     // 2
      throw std::logic_error(“No thread”);
  }
  ~scoped_thread()
  {
    t.join();                                            // 3
  }
  scoped_thread(scoped_thread const&)=delete;
  scoped_thread& operator=(scoped_thread const&)=delete;
};
struct func; 
void f()
{
  int some_local_state;
  scoped_thread t(std::thread(func(some_local_state)));    // 4
  do_something_in_current_thread();
}                                                        // 5
```

### 决定线程数量

### 线程标识

线程标识类型为 `std::thread::id`，并可以通过两种方式进行检索。第一种，可以通过调用 `std::thread` 对象的成员函数 `get_id()` 来直接获取。如果 `std::thread` 对象没有与任何执行线程相关联，`get_id()` 将返回 `std::thread::type` 默认构造值，这个值表示“无线程”。第二种，当前线程中调用 `std::this_thread::get_id()` (这个函数定义在<thread>头文件中)也可以获得线程标识。

`std::thread::id` 对象可以自由的拷贝和对比,因为标识符就可以复用。如果两个对象的 `std::thread::id` 相等，那它们就是同一个线程，或者都“无线程”。如果不等，那么就代表了两个不同线程，或者一个有线程，另一没有线程。

这意味着允许程序员将其当做为容器的键值，做排序，或做其他方式的比较。

也可以使用输出流( `std::cout` )来记录一个 `std::thread::id` 对象的值。

```cpp
std::cout<<std::this_thread::get_id();
```

具体的输出结果是严格依赖于具体实现的，C++标准的唯一要求就是要保证ID比较结果相等的线程，必须有相同的输出。


