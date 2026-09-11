---
title: "jthread 与 stop_token"
---

C++20 提供了：

```cpp
std::jthread
```

它可以理解成“更安全的 `std::thread`”：

- 析构时自动处理线程，不要求每条控制路径都手写 `join()`；
- 内置协作式停止机制；
- 可以与 `std::stop_token`、`std::stop_source`、`std::stop_callback` 配合。

如果项目使用 C++20，并且线程是“启动后持续运行，退出时需要安全停止”的类型，`std::jthread` 通常比裸 `std::thread` 更合适。

## 1. `std::thread` 的生命周期风险

使用 `std::thread` 时：

```cpp
std::thread t(work);
```

必须保证对象析构前已经：

```cpp
t.join();
```

或者：

```cpp
t.detach();
```

否则 joinable 的 `std::thread` 析构会调用：

```cpp
std::terminate()
```

这意味着异常、提前 `return`、复杂分支都可能让“是否已经 join”变成维护负担。

## 2. `std::jthread` 会自动 join

最简单的写法：

```cpp
#include <iostream>
#include <thread>

void work()
{
    std::cout << "working\n";
}

int main()
{
    std::jthread t(work);
}
```

离开作用域时，不需要手动写：

```cpp
t.join();
```

`jthread` 的析构函数会负责等待线程结束。

这让线程管理更符合 RAII 思想。

## 3. `jthread` 不等于“强制杀死线程”

`std::jthread` 的停止机制是：

> 协作式停止（cooperative cancellation）

也就是说：

```text
某个线程发出停止请求
       ↓
工作线程主动检查这个请求
       ↓
工作线程自己决定在安全位置退出
```

标准库不会粗暴地强制终止线程。

这是非常重要的设计，因为强制杀线程可能发生在：

- 正持有 mutex；
- 正修改对象状态；
- 正写文件；
- 正执行资源管理代码；

如果在任意位置突然中断，很容易破坏程序状态。

## 4. `std::stop_token`

`std::stop_token` 表示：

> 我可以观察某个停止请求是否已经发出。

`jthread` 可以自动把 stop token 作为线程函数的第一个参数传入。

例如：

```cpp
#include <chrono>
#include <iostream>
#include <stop_token>
#include <thread>

using namespace std::chrono_literals;

void worker(std::stop_token stop_token)
{
    while (!stop_token.stop_requested())
    {
        std::cout << "working\n";
        std::this_thread::sleep_for(100ms);
    }

    std::cout << "stopping\n";
}

int main()
{
    std::jthread thread(worker);

    std::this_thread::sleep_for(350ms);
    thread.request_stop();
}
```

这里线程函数的第一个参数：

```cpp
std::stop_token
```

由 `jthread` 自动提供，不需要调用者手动传。

## 5. `request_stop()`

调用：

```cpp
thread.request_stop();
```

只是“发出停止请求”。

它不会等待线程结束，也不会强制终止线程。

工作线程需要主动检查：

```cpp
stop_token.stop_requested()
```

并自行退出。

## 6. `stop_possible()`

`stop_token` 还可以查询：

```cpp
stop_token.stop_possible()
```

表示这个 token 是否关联着一个能够产生停止请求的停止状态。

普通 `jthread` 自动提供的 token 通常是可停止的。

## 7. `jthread` 析构时会发生什么

如果一个 `jthread` 在析构时仍然 joinable，析构过程概念上会：

```text
request_stop()
      ↓
join()
```

也就是说，它不只是自动等待，还会先请求线程停止。

因此：

```cpp
{
    std::jthread thread(worker);
} // 离开作用域时自动请求停止并等待结束
```

如果 `worker` 正确响应 stop token，就能自然退出。

## 8. 析构请求停止不等于一定能立刻退出

如果工作线程完全不检查 stop token：

```cpp
void worker(std::stop_token)
{
    while (true)
    {
    }
}
```

那么 `jthread` 析构时虽然调用了 `request_stop()`，线程依然不会退出。

随后析构中的 `join()` 会一直等待。

所以：

> `jthread` 提供停止协议，但线程函数必须主动合作。

## 9. 阻塞操作也要考虑停止

下面虽然检查了停止请求：

```cpp
while (!stop_token.stop_requested())
{
    blocking_operation();
}
```

但如果：

```cpp
blocking_operation()
```

一次就能阻塞几十秒，那么停止响应仍然会非常慢。

线程设计不仅要考虑“有没有 stop token”，还要考虑：

- 阻塞调用是否支持超时；
- 是否能被唤醒；
- 是否可以分阶段检查停止状态。

## 10. `std::stop_source`

`std::stop_source` 是停止请求的“控制端”。

可以：

```cpp
std::stop_source source;
std::stop_token token = source.get_token();
```

然后：

```cpp
source.request_stop();
```

所有关联到这个停止状态的 token 都能观察到请求。

示例：

```cpp
#include <iostream>
#include <stop_token>

int main()
{
    std::stop_source source;
    std::stop_token token = source.get_token();

    std::cout << std::boolalpha
              << token.stop_requested()
              << '\n';

    source.request_stop();

    std::cout << token.stop_requested() << '\n';
}
```

输出：

```text
false
true
```

## 11. `std::stop_callback`

有时不希望某个线程一直轮询：

```cpp
stop_requested()
```

而是希望停止请求到来时自动执行一段回调。

可以使用：

```cpp
std::stop_callback
```

例如：

```cpp
#include <iostream>
#include <stop_token>

int main()
{
    std::stop_source source;
    std::stop_token token = source.get_token();

    std::stop_callback callback(token, [] {
        std::cout << "stop requested\n";
    });

    source.request_stop();
}
```

`request_stop()` 发出停止请求时，已注册的 callback 会执行。

注意 callback 的执行上下文与停止请求有关，因此回调本身也应该保持简单，并避免制造新的锁顺序问题。

## 12. `jthread` 也支持普通线程函数

并不是所有 `jthread` 函数都必须接收 `stop_token`。

例如：

```cpp
#include <iostream>
#include <thread>

void work(int value)
{
    std::cout << value << '\n';
}

int main()
{
    std::jthread thread(work, 42);
}
```

如果可调用对象能够以：

```cpp
(stop_token, args...)
```

形式调用，`jthread` 会优先传入 stop token。

否则就像普通 `thread` 一样使用：

```cpp
(args...)
```

形式调用。

## 13. 带参数的 stop token 线程函数

```cpp
#include <iostream>
#include <stop_token>
#include <thread>

void worker(std::stop_token token, int id)
{
    std::cout << "worker " << id << '\n';

    while (!token.stop_requested())
    {
        // work
    }
}

int main()
{
    std::jthread thread(worker, 7);
    thread.request_stop();
}
```

这里：

```cpp
std::jthread thread(worker, 7);
```

调用效果相当于让线程执行：

```cpp
worker(自动提供的 stop_token, 7)
```

## 14. `jthread` 同样可以 join

自动 join 不代表不能手动 join。

```cpp
std::jthread thread(work);
thread.join();
```

调用后：

```cpp
thread.joinable() == false
```

之后析构时就无需再次等待。

同样可以查询：

```cpp
thread.joinable()
thread.get_id()
```

## 15. `jthread` 也可以 detach，但通常不推荐

`jthread` 仍然提供：

```cpp
thread.detach();
```

但一旦 detach，RAII 自动等待和结构化停止的优势就基本失去了。

除非确实明确理解生命周期，否则不要为了“后台运行”轻易 detach。

## 16. `condition_variable_any` 与 stop token

普通 `std::condition_variable` 没有直接接收 `stop_token` 的等待重载。

C++20 的：

```cpp
std::condition_variable_any
```

则可以把停止请求整合进等待条件。

例如：

```cpp
#include <condition_variable>
#include <mutex>
#include <stop_token>
#include <thread>

std::mutex mutex;
std::condition_variable_any cv;
bool ready = false;

void worker(std::stop_token token)
{
    std::unique_lock lock(mutex);

    bool condition_met = cv.wait(
        lock,
        token,
        [] {
            return ready;
        });

    if (!condition_met)
    {
        // 因停止请求结束等待
        return;
    }

    // ready == true
}
```

这样线程在等待条件时也能响应停止请求，不必额外写一个轮询循环。

## 17. `thread` 和 `jthread` 对比

| 特性 | `std::thread` | `std::jthread` |
|:---|:---:|:---:|
| 标准版本 | C++11 | C++20 |
| 创建线程 | ✅ | ✅ |
| 手动 `join()` | ✅ | ✅ |
| 析构自动等待 | ❌ | ✅ |
| 析构时 joinable 会 terminate | ✅ | ❌ |
| 内置停止请求 | ❌ | ✅ |
| 自动传 `stop_token` | ❌ | ✅ |
| 可 detach | ✅ | ✅ |

如果环境允许使用 C++20，新写的“可停止后台线程”通常优先考虑 `jthread`。

## 18. `atomic<bool> running` 和 stop token 怎么选

传统写法：

```cpp
std::atomic<bool> running{true};

while (running.load())
{
    ...
}
```

仍然完全合法，而且在简单场景下很直观。

`stop_token` 的优势是把：

```text
停止请求
停止观察
停止回调
jthread 生命周期
```

组织成标准协议。

因此：

- 只需要一个非常简单的原子标志：`atomic<bool>` 仍然很好用；
- 线程本身有明确的“请求停止”生命周期：优先考虑 `jthread + stop_token`。

## 19. 常见错误

1. 以为 `request_stop()` 会强制杀死线程。
2. 工作函数从不检查 `stop_requested()`，导致析构仍然一直等待。
3. 在线程内部执行长时间不可中断阻塞操作，导致停止响应很慢。
4. 使用了 `jthread` 却仍然随手 `detach()`，失去 RAII 生命周期管理优势。
5. stop callback 内执行复杂阻塞逻辑或获取大量锁，导致停止路径本身难以控制。
6. 把“收到停止请求”理解成“必须立即在任意位置退出”，而不是在安全点协作退出。

## 小结

- `std::jthread` 是 C++20 更现代的线程管理类。
- `jthread` 析构时会对 joinable 线程请求停止并等待结束。
- `std::stop_token` 用来观察停止请求，`request_stop()` 只是请求，不是强制终止。
- 停止必须由工作线程协作响应。
- `std::stop_source` 是请求端，`std::stop_callback` 可以在请求发生时执行回调。
- 对需要明确停止协议的后台线程，`jthread + stop_token` 通常比 `thread + atomic<bool>` 更结构化。
