---
title: "condition_variable"
---

`std::condition_variable` 用于让一个线程等待某个共享条件发生变化，而不是不停循环检查。

典型场景包括：

- 生产者把数据放入队列，消费者没有数据时等待；
- 一个线程等待初始化完成；
- 一个工作线程等待新任务到来；
- 多个线程等待某个状态变为 ready。

条件变量最重要的不是记住 `wait()` 和 `notify_one()`，而是理解它必须围绕下面三件东西一起使用：

```text
共享状态 + mutex + condition_variable
```

## 1. 为什么不能一直轮询

最直接的等待方式可能是：

```cpp
while (!ready)
{
}
```

这种写法有两个问题：

1. 如果 `ready` 是普通变量，多线程读写本身就可能产生数据竞争；
2. 即使用 `atomic`，这种忙等也会持续占用 CPU。

例如：

```cpp
while (!ready.load())
{
}
```

在等待时间较长时通常不是理想方案。

条件变量允许线程进入阻塞等待，等到其他线程通知后再继续。

## 2. 基本组成

通常会同时存在：

```cpp
std::mutex mutex;
std::condition_variable cv;
bool ready = false;
```

这里：

- `ready`：真正表示业务条件的共享状态；
- `mutex`：保护 `ready`；
- `cv`：负责睡眠和唤醒等待线程。

注意：

> `condition_variable` 本身不是状态。

它只是通知机制。真正决定线程能否继续的是谓词所检查的共享状态。

## 3. 最简单的等待与通知

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mutex;
std::condition_variable cv;
bool ready = false;

void worker()
{
    std::unique_lock<std::mutex> lock(mutex);

    cv.wait(lock, [] {
        return ready;
    });

    std::cout << "worker start\n";
}

int main()
{
    std::thread t(worker);

    {
        std::lock_guard<std::mutex> lock(mutex);
        ready = true;
    }

    cv.notify_one();
    t.join();
}
```

执行逻辑：

```text
worker 获得 mutex
      ↓
检查 ready == false
      ↓
wait 暂时释放 mutex，并进入等待
      ↓
main 获得 mutex
      ↓
修改 ready = true
      ↓
main 释放 mutex
      ↓
notify_one
      ↓
worker 被唤醒并重新获得 mutex
      ↓
再次检查 ready
      ↓
条件成立，wait 返回
```

## 4. 为什么 `wait()` 要配合 `std::unique_lock`

常见写法是：

```cpp
std::unique_lock<std::mutex> lock(mutex);
cv.wait(lock, predicate);
```

而不是：

```cpp
std::lock_guard<std::mutex>
```

原因在于 `wait()` 必须完成一组特殊操作：

1. 当前线程已经持有 mutex；
2. `wait()` 在睡眠前临时释放 mutex；
3. 线程被唤醒后重新获取 mutex；
4. 获得 mutex 后再返回给调用者。

`unique_lock` 支持这种“暂时释放、重新获得”的所有权管理，而 `lock_guard` 不提供手动解锁和重新加锁能力。

## 5. `wait(lock)` 与 `wait(lock, predicate)`

条件变量有两种常见等待形式。

### 5.1 不带谓词

```cpp
cv.wait(lock);
```

线程被唤醒后直接返回。

如果使用这种形式，就必须自己写循环检查条件：

```cpp
while (!ready)
{
    cv.wait(lock);
}
```

### 5.2 带谓词

推荐写：

```cpp
cv.wait(lock, [] {
    return ready;
});
```

它可以理解为标准库帮你写了：

```cpp
while (!ready)
{
    cv.wait(lock);
}
```

因此大多数普通场景优先使用带谓词版本。

## 6. 虚假唤醒（spurious wakeup）

等待线程有可能在没有对应业务事件发生时从 `wait()` 醒来，这叫虚假唤醒。

因此下面这种逻辑不可靠：

```cpp
cv.wait(lock);
use_shared_data();
```

正确思路是：

> 每次醒来都重新检查真正的共享条件。

也就是：

```cpp
cv.wait(lock, [] {
    return ready;
});
```

这也是为什么“谓词”不是可有可无的装饰。

## 7. 条件变量没有记忆：不要把 notify 当状态

另一个常见误解是：

> 调用了 `notify_one()`，以后某个线程再 `wait()` 时应该能收到这次通知。

不是。

条件变量不会像消息队列一样永久保存通知。

例如：

```text
线程 A notify_one()
        ↓
当时没有线程正在等待
        ↓
这次通知结束
        ↓
线程 B 之后才开始 wait()
```

线程 B 不会自动“补收到”之前的通知。

因此必须用共享状态记录事实：

```cpp
ready = true;
cv.notify_one();
```

即使通知时没有线程正在睡眠，未来线程进入：

```cpp
cv.wait(lock, [] { return ready; });
```

也会先检查到 `ready == true`，从而不需要睡眠。

这就是：

```text
共享状态保存事实
condition_variable 负责提高等待效率
```

## 8. `notify_one()` 和 `notify_all()`

### 8.1 `notify_one()`

```cpp
cv.notify_one();
```

唤醒一个正在等待的线程。

适合：

- 新增一个任务，只需要一个 worker 处理；
- 只需要一个消费者继续执行。

### 8.2 `notify_all()`

```cpp
cv.notify_all();
```

唤醒所有正在等待的线程。

适合：

- 全局状态改变，所有线程都需要重新检查条件；
- 程序停止，需要让所有等待线程退出。

被唤醒不等于所有线程能同时进入临界区。它们仍然需要竞争同一把 mutex。

## 9. 修改状态后什么时候 notify

一种常见推荐写法是：

```cpp
{
    std::lock_guard<std::mutex> lock(mutex);
    ready = true;
}

cv.notify_one();
```

即：

1. 持锁修改共享状态；
2. 释放锁；
3. 再通知。

这样等待线程醒来后更有机会直接获得 mutex，而不是醒来后马上又因为通知线程仍持锁而阻塞。

不过关键正确性原则不是“notify 必须永远写在锁外”，而是：

> 对共享谓词状态的访问必须正确同步，等待方必须在锁保护下检查谓词。

## 10. 生产者消费者

这是条件变量最经典的应用。

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>

std::queue<int> queue;
std::mutex mutex;
std::condition_variable cv;
bool finished = false;

void producer()
{
    for (int value = 1; value <= 5; ++value)
    {
        {
            std::lock_guard<std::mutex> lock(mutex);
            queue.push(value);
        }

        cv.notify_one();
    }

    {
        std::lock_guard<std::mutex> lock(mutex);
        finished = true;
    }

    cv.notify_all();
}

void consumer()
{
    while (true)
    {
        int value = 0;

        {
            std::unique_lock<std::mutex> lock(mutex);

            cv.wait(lock, [] {
                return !queue.empty() || finished;
            });

            if (queue.empty() && finished)
            {
                break;
            }

            value = queue.front();
            queue.pop();
        }

        // 真正处理数据放在锁外
        std::cout << "consume " << value << '\n';
    }
}

int main()
{
    std::thread p(producer);
    std::thread c(consumer);

    p.join();
    c.join();
}
```

这里谓词不是单纯：

```cpp
!queue.empty()
```

而是：

```cpp
!queue.empty() || finished
```

否则生产者彻底结束后，如果队列已经为空，消费者可能永远继续等待，再也没人通知它有数据。

## 11. 为什么取出任务后要尽快释放锁

消费者应该在锁内只做：

```text
检查队列
取出任务
修改共享状态
```

真正耗时的任务执行应尽量放到锁外。

如果消费者拿着 queue 的 mutex 执行一个耗时 2 秒的任务，那么生产者和其他消费者这 2 秒内都可能无法访问队列。

## 12. 超时等待：`wait_for()`

```cpp
cv.wait_for(lock, duration)
```

可以等待一段时间。

推荐同样使用谓词版本：

```cpp
bool ok = cv.wait_for(
    lock,
    std::chrono::seconds(1),
    [] {
        return ready;
    });
```

返回：

- `true`：谓词最终成立；
- `false`：超时且谓词仍未成立。

示例：

```cpp
if (!cv.wait_for(lock, std::chrono::seconds(1), [] {
        return ready;
    }))
{
    std::cout << "timeout\n";
}
```

## 13. 绝对时间等待：`wait_until()`

```cpp
cv.wait_until(lock, time_point, predicate)
```

适合已经有一个明确截止时间点的场景。

例如：

```cpp
auto deadline = std::chrono::steady_clock::now()
              + std::chrono::seconds(2);

cv.wait_until(lock, deadline, [] {
    return ready;
});
```

涉及超时时，通常优先使用 `steady_clock`，因为它不会受到系统墙上时钟被手动修改的影响。

## 14. 一个条件变量可以等待多个条件吗？

可以。

真正决定等待逻辑的是谓词，例如：

```cpp
cv.wait(lock, [] {
    return !queue.empty() || stopped || error;
});
```

醒来后再根据具体状态分支处理：

```cpp
if (error)
{
    ...
}
else if (stopped && queue.empty())
{
    ...
}
else
{
    ...
}
```

条件变量只是提示“相关状态可能变化了”，最终必须重新检查状态本身。

## 15. `std::condition_variable_any`

普通：

```cpp
std::condition_variable
```

主要配合：

```cpp
std::unique_lock<std::mutex>
```

标准库还提供：

```cpp
std::condition_variable_any
```

它可以配合更广泛的 BasicLockable 锁类型，灵活性更高，但一般也可能带来更多开销。

普通 `std::mutex` 场景优先使用 `std::condition_variable`。

C++20 中，`condition_variable_any` 还提供了能与 `std::stop_token` 配合的等待重载，这在 `std::jthread` 协作式停止里很有用。

## 16. 条件变量与 atomic 怎么选

如果只是：

```text
等待一个原子值改变
```

C++20 可以考虑：

```cpp
atomic.wait()
atomic.notify_one()
atomic.notify_all()
```

如果条件涉及：

- 队列是否为空；
- 多个字段组合；
- 复杂业务状态；

通常：

```text
mutex + condition_variable + predicate
```

更自然。

## 17. 常见错误

### 错误 1：把通知本身当成状态

```cpp
cv.notify_one();
```

不会永久保存一条“通知消息”。必须有共享状态记录事实。

### 错误 2：不用谓词处理虚假唤醒

```cpp
cv.wait(lock);
use_data();
```

应该重新检查条件。

### 错误 3：修改谓词状态时没有使用同一套同步规则

等待方在 mutex 下读取 `ready`，修改方却在没有 mutex 的情况下写普通 `bool ready`，仍然可能是数据竞争。

### 错误 4：持锁执行耗时任务

队列取任务后应尽量解锁，再处理任务。

### 错误 5：程序停止时只修改 `finished`，却忘记通知等待线程

等待线程可能一直睡眠。停止路径通常要配合 `notify_all()`。

### 错误 6：只检查 `queue.empty()`，没有设计“不会再产生数据”的结束状态

生产者消费者模型必须考虑线程如何正常退出。

## 小结

- `condition_variable` 必须围绕“共享状态 + mutex + 谓词”理解。
- `wait()` 在睡眠时会临时释放 mutex，醒来后重新获得，因此通常配合 `unique_lock`。
- 条件变量允许虚假唤醒，所以应始终重新检查真正的条件。
- `notify_one()` 唤醒一个等待者，`notify_all()` 唤醒全部等待者。
- 条件变量不会永久保存通知，真正的状态必须由受同步保护的数据记录。
- `wait_for()` / `wait_until()` 可以实现超时等待。
- 生产者消费者中，应在锁内快速取出任务，在锁外执行耗时工作。
