---
title: "mutex 与 RAII 锁"
---

多个线程同时访问共享可变数据时，必须考虑同步问题。最常用的同步工具之一就是互斥量（mutex）。

这一节重点学习：

- 什么是临界区和数据竞争；
- `std::mutex` 的 `lock()` / `try_lock()` / `unlock()`；
- 为什么不推荐手动成对调用 `lock()` / `unlock()`；
- `std::lock_guard`、`std::unique_lock`、`std::scoped_lock` 的区别；
- 多把锁为什么会死锁；
- `std::shared_mutex` 的多读单写；
- `std::call_once` 与一次性初始化。

## 1. 为什么需要互斥量

下面这段代码存在数据竞争：

```cpp
#include <iostream>
#include <thread>

int counter = 0;

void add_many()
{
    for (int i = 0; i < 100000; ++i)
    {
        ++counter;
    }
}

int main()
{
    std::thread t1(add_many);
    std::thread t2(add_many);

    t1.join();
    t2.join();

    std::cout << counter << '\n';
}
```

`++counter` 通常包含读取、加一、写回几个步骤。两个线程可能交错执行，导致数据竞争。

在 C++ 中，数据竞争通常意味着未定义行为。

## 2. `std::mutex`

`std::mutex` 定义在：

```cpp
#include <mutex>
```

最基础的三个成员函数是：

```cpp
m.lock();
m.try_lock();
m.unlock();
```

### 2.1 `lock()`

```cpp
m.lock();
```

如果互斥量当前没有被其他线程持有，当前线程获得锁并继续执行。

如果已经被其他线程持有，当前线程会阻塞，直到能够获得锁。

### 2.2 `unlock()`

```cpp
m.unlock();
```

释放当前线程已经持有的锁。

### 2.3 `try_lock()`

```cpp
if (m.try_lock())
{
    // 成功获得锁
    m.unlock();
}
else
{
    // 没有获得锁，不会阻塞
}
```

`try_lock()` 尝试获得锁，但如果暂时无法获得，会立即返回 `false`，而不是一直等待。

## 3. 为什么不推荐手写 `lock()` / `unlock()`

下面这种代码看起来没问题：

```cpp
mutex.lock();
change_shared_state();
mutex.unlock();
```

但如果中间：

- 提前 `return`；
- 抛出异常；
- 后续代码修改时忘记解锁；

就可能导致锁永远无法释放。

这也是 RAII 在并发中的典型应用场景：

> 让一个对象在构造时加锁，在析构时自动解锁。

## 4. `std::lock_guard`

最简单、最常用的 RAII 锁是：

```cpp
std::lock_guard<std::mutex>
```

例如：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int counter = 0;
std::mutex counter_mutex;

void add_many()
{
    for (int i = 0; i < 100000; ++i)
    {
        std::lock_guard<std::mutex> lock(counter_mutex);
        ++counter;
    }
}

int main()
{
    std::thread t1(add_many);
    std::thread t2(add_many);

    t1.join();
    t2.join();

    std::cout << counter << '\n';
}
```

进入这一层作用域时：

```text
lock_guard 构造
      ↓
mutex.lock()
```

离开作用域时：

```text
lock_guard 析构
      ↓
mutex.unlock()
```

因此即使中间抛出异常，RAII 对象析构时也会正常释放锁。

## 5. 锁保护的是共享状态，不只是某一行代码

假设对象有两个成员：

```cpp
int count;
double average;
```

如果这两个值必须保持一致，那么应该由同一把锁一起保护，而不是分别加锁。

例如：

```cpp
std::mutex state_mutex;

void update_state()
{
    std::lock_guard<std::mutex> lock(state_mutex);
    ++count;
    average = calculate_average();
}
```

重点是维护“状态不变量”。

## 6. 临界区要尽量小

锁内只做真正需要互斥保护的工作。

```cpp
void worker(const std::vector<int>& data)
{
    int local_sum = 0;

    // 不需要访问共享状态，放在锁外
    for (int value : data)
    {
        local_sum += value;
    }

    // 只在更新共享结果时加锁
    {
        std::lock_guard<std::mutex> lock(total_mutex);
        total += local_sum;
    }
}
```

不要在持锁期间做：

- 长时间休眠；
- 文件 IO；
- 网络 IO；
- 大量计算；

除非这些操作本身确实必须位于同一个临界区内。

## 7. `std::unique_lock`

`std::unique_lock` 比 `lock_guard` 更灵活。

它支持：

- 手动 `lock()` / `unlock()`；
- 延迟加锁；
- 尝试加锁；
- 所有权转移；
- 与 `std::condition_variable` 配合使用。

最简单的写法：

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

作用域结束时仍然会自动解锁。

### 7.1 手动提前解锁

```cpp
std::unique_lock<std::mutex> lock(mutex);

read_shared_state();
lock.unlock();

// 后面的耗时工作不再占着锁
do_expensive_work();
```

### 7.2 延迟加锁：`std::defer_lock`

```cpp
std::unique_lock<std::mutex> lock(mutex, std::defer_lock);

// 此时还没有获得锁
lock.lock();
```

### 7.3 尝试加锁：`std::try_to_lock`

```cpp
std::unique_lock<std::mutex> lock(mutex, std::try_to_lock);

if (lock.owns_lock())
{
    // 成功获得锁
}
```

### 7.4 `owns_lock()`

```cpp
lock.owns_lock()
```

可以判断当前 `unique_lock` 是否实际拥有锁。

## 8. `lock_guard` 和 `unique_lock` 怎么选

| 工具 | 特点 | 推荐场景 |
|:---|:---|:---|
| `std::lock_guard` | 最简单，作用域内始终持锁 | 普通临界区，默认优先选择 |
| `std::unique_lock` | 可延迟、手动解锁、可移动 | 条件变量、需要灵活控制锁生命周期 |

能用 `lock_guard` 时优先用 `lock_guard`；确实需要更灵活的锁控制时再使用 `unique_lock`。

## 9. 死锁

如果两个线程分别持有一把锁，又等待对方持有的另一把锁，就可能产生死锁。

例如：

```cpp
std::mutex m1;
std::mutex m2;

void task1()
{
    std::lock_guard<std::mutex> lock1(m1);
    std::lock_guard<std::mutex> lock2(m2);
}

void task2()
{
    std::lock_guard<std::mutex> lock2(m2);
    std::lock_guard<std::mutex> lock1(m1);
}
```

一种可能的执行顺序：

```text
线程 A 获得 m1
线程 B 获得 m2
线程 A 等待 m2
线程 B 等待 m1
```

两个线程都无法继续。

## 10. 避免死锁的方法

### 10.1 固定加锁顺序

如果整个项目规定：

```text
永远先锁 m1，再锁 m2
```

就能避免由顺序不一致造成的循环等待。

### 10.2 `std::scoped_lock`（C++17）

同时锁多把互斥量时，优先考虑：

```cpp
std::scoped_lock lock(m1, m2);
```

例如：

```cpp
#include <mutex>

std::mutex m1;
std::mutex m2;

void task()
{
    std::scoped_lock lock(m1, m2);
    // 同时持有 m1 和 m2
}
```

`std::scoped_lock` 可以使用标准库的死锁避免策略获取多把锁。

如果只传一把锁，它也可以像 `lock_guard` 一样使用：

```cpp
std::scoped_lock lock(mutex);
```

## 11. `std::lock()`

C++11 还提供：

```cpp
std::lock(m1, m2);
```

它用于同时获取多把可锁对象，并使用死锁避免算法。

然后通常配合 `std::adopt_lock` 把已经获得的锁交给 RAII 对象：

```cpp
std::lock(m1, m2);

std::lock_guard<std::mutex> lock1(m1, std::adopt_lock);
std::lock_guard<std::mutex> lock2(m2, std::adopt_lock);
```

在 C++17 以后，如果只是需要同时管理多把互斥量，`std::scoped_lock` 往往更直接。

## 12. `std::recursive_mutex`

普通 `std::mutex` 不允许同一线程重复对同一把互斥量加锁。

如果代码确实需要同一个线程递归地重复进入同一临界区，可以使用：

```cpp
std::recursive_mutex
```

但它通常不是首选。

大量依赖 `recursive_mutex` 有时意味着函数职责或锁设计需要重新整理。能通过调整结构避免递归加锁时，通常更容易维护。

## 13. `std::timed_mutex`

`std::timed_mutex` 支持带超时的加锁：

```cpp
try_lock_for(...)
try_lock_until(...)
```

例如：

```cpp
std::timed_mutex mutex;

if (mutex.try_lock_for(std::chrono::milliseconds(100)))
{
    mutex.unlock();
}
```

适合“不愿无限期等待一把锁”的场景。

## 14. `std::shared_mutex`：多读单写

C++17 提供：

```cpp
std::shared_mutex
```

它适合：

- 多个线程可以同时读；
- 但写操作必须独占。

读取时使用：

```cpp
std::shared_lock<std::shared_mutex>
```

写入时使用：

```cpp
std::unique_lock<std::shared_mutex>
```

示例：

```cpp
#include <shared_mutex>
#include <string>

std::string config;
std::shared_mutex config_mutex;

std::string read_config()
{
    std::shared_lock lock(config_mutex);
    return config;
}

void write_config(std::string value)
{
    std::unique_lock lock(config_mutex);
    config = std::move(value);
}
```

只有在“读远多于写”并且确实存在并发读取收益时，读写锁才可能有优势。不要默认认为它一定比普通 `mutex` 更快。

## 15. `std::call_once`：只执行一次

有些初始化逻辑要求：

> 不管多少线程同时到达，这段初始化代码只能执行一次。

标准库提供：

```cpp
std::once_flag
std::call_once
```

例如：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::once_flag flag;

void initialize()
{
    std::cout << "initialize once\n";
}

void worker()
{
    std::call_once(flag, initialize);
}

int main()
{
    std::thread t1(worker);
    std::thread t2(worker);
    std::thread t3(worker);

    t1.join();
    t2.join();
    t3.join();
}
```

虽然三个线程都会调用 `std::call_once`，但 `initialize()` 只会成功执行一次。

现代 C++ 中，函数局部 `static` 的初始化本身从 C++11 起也是线程安全的：

```cpp
MyObject& instance()
{
    static MyObject object;
    return object;
}
```

如果只是懒初始化一个函数局部静态对象，通常不需要额外手写 `call_once`。

## 16. mutex 保护的数据也要封装好

不推荐让调用方自己记住：

```text
调用函数 A 前必须先锁 mutex
函数 B 内部不能锁
函数 C 又必须锁
```

更好的设计通常是让拥有共享数据的类型自己负责同步：

```cpp
class Counter
{
public:
    void increment()
    {
        std::lock_guard<std::mutex> lock(mutex_);
        ++value_;
    }

    int get() const
    {
        std::lock_guard<std::mutex> lock(mutex_);
        return value_;
    }

private:
    mutable std::mutex mutex_;
    int value_ = 0;
};
```

这样更容易保证所有访问路径都遵守同一套同步规则。

## 17. 常见错误

1. 只有写操作加锁，读操作却不加锁。只要一边在写，另一边无同步读取仍可能产生数据竞争。
2. 手动 `lock()` 后存在提前返回或异常路径，导致忘记 `unlock()`。
3. 两个函数以不同顺序获取多把锁，产生死锁。
4. 持锁期间调用一个未知函数，而那个函数内部又尝试获取同一把锁。
5. 持锁期间执行长时间 IO 或休眠，导致其他线程长时间阻塞。
6. 为了“保险”到处加锁，导致锁顺序复杂、性能下降、死锁概率上升。
7. 认为 `shared_mutex` 一定比 `mutex` 快。

## 小结

- `std::mutex` 用于建立互斥临界区。
- 普通场景优先用 `std::lock_guard`，而不是手写 `lock()` / `unlock()`。
- 需要手动释放、延迟加锁或条件变量时使用 `std::unique_lock`。
- C++17 的 `std::scoped_lock` 很适合同时管理多把互斥量。
- 多把锁最重要的问题是死锁，应统一加锁顺序或使用标准库的多锁工具。
- `std::shared_mutex` 适合多读单写，但不应无脑替代 `mutex`。
- `std::call_once` 用于一次性初始化；函数局部 `static` 初始化从 C++11 起本身就是线程安全的。
