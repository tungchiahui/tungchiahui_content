---
title: "future、async、promise 与 packaged_task"
---

`std::thread` 很适合表达“启动一个线程”，但它本身不直接提供：

- 线程函数返回值；
- 异常自动传回调用方；
- 一个统一的“未来结果”对象。

C++11 在 `<future>` 中提供了一组更偏“任务”的并发工具：

```cpp
std::future
std::async
std::promise
std::packaged_task
std::shared_future
```

它们的核心思想是：

> 某个结果现在还没有，但未来会产生；调用方先拿到一个 `future`，之后再等待或取得结果。

## 1. `std::future<T>` 是什么

`std::future<T>` 表示：

```text
未来某个时刻会得到一个 T
```

例如：

```cpp
std::future<int>
```

表示未来会产生一个 `int`。

如果任务没有返回值，则使用：

```cpp
std::future<void>
```

`future` 最重要的成员函数包括：

```cpp
get()
wait()
wait_for()
wait_until()
valid()
```

## 2. 最简单的 `std::async`

```cpp
#include <future>
#include <iostream>

int calculate(int a, int b)
{
    return a + b;
}

int main()
{
    std::future<int> result = std::async(
        std::launch::async,
        calculate,
        10,
        20);

    std::cout << "main continues\n";
    std::cout << result.get() << '\n';
}
```

这里：

```cpp
std::async(...)
```

启动一个异步任务，并返回：

```cpp
std::future<int>
```

调用：

```cpp
result.get()
```

时，如果结果还没准备好，当前线程会等待；准备好后取得返回值。

输出可能是：

```text
main continues
30
```

## 3. `async` 的参数和 `thread` 很像

概念上：

```cpp
std::async(policy, callable, arg1, arg2, ...)
```

其中：

- `policy`：启动策略；
- `callable`：要执行的可调用对象；
- 后续参数：传给该可调用对象。

可调用对象同样可以是：

- 普通函数；
- Lambda；
- 函数对象；
- 成员函数。

## 4. `std::launch::async` 和 `std::launch::deferred`

`std::async` 支持两种重要启动策略。

### 4.1 `std::launch::async`

```cpp
std::async(std::launch::async, task);
```

要求任务异步执行。

可以把它理解成：

> 现在就安排任务独立执行。

### 4.2 `std::launch::deferred`

```cpp
std::async(std::launch::deferred, task);
```

表示延迟执行。

任务不会立刻运行，而是在第一次：

```cpp
future.get()
future.wait()
```

时，由调用这些函数的线程执行。

例如：

```cpp
#include <future>
#include <iostream>
#include <thread>

int task()
{
    std::cout << "task thread: "
              << std::this_thread::get_id()
              << '\n';
    return 42;
}

int main()
{
    std::cout << "main thread: "
              << std::this_thread::get_id()
              << '\n';

    auto result = std::async(std::launch::deferred, task);

    std::cout << result.get() << '\n';
}
```

这里 `task()` 会在调用 `get()` 的线程中执行。

## 5. 不写 policy 会怎样

可以写：

```cpp
auto result = std::async(task);
```

此时标准库允许实现选择：

```text
async
或
deferred
```

因此，如果你明确要求任务真正异步执行，建议显式写：

```cpp
std::launch::async
```

否则不要假设“不写策略就一定创建新线程”。

## 6. `future::get()` 只能取得一次

普通 `std::future` 的结果通常只能 `get()` 一次：

```cpp
auto result = std::async(std::launch::async, [] {
    return 42;
});

int value = result.get();
// result.get(); // 不应再次 get
```

第一次 `get()` 后，future 通常不再关联共享状态。

可以用：

```cpp
result.valid()
```

检查它是否仍然关联有效状态。

## 7. `wait()`：只等待，不取结果

```cpp
future.wait();
```

只等待任务完成，不消费结果。

之后仍然可以：

```cpp
future.get();
```

例如：

```cpp
result.wait();
std::cout << "ready\n";
std::cout << result.get() << '\n';
```

## 8. `wait_for()`：等待一段时间

```cpp
auto status = result.wait_for(std::chrono::milliseconds(100));
```

返回：

```cpp
std::future_status::ready
std::future_status::timeout
std::future_status::deferred
```

示例：

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <thread>

using namespace std::chrono_literals;

int main()
{
    auto result = std::async(std::launch::async, [] {
        std::this_thread::sleep_for(1s);
        return 42;
    });

    if (result.wait_for(100ms) == std::future_status::timeout)
    {
        std::cout << "not ready yet\n";
    }

    std::cout << result.get() << '\n';
}
```

## 9. 异常会通过 future 传播

这是 `future` 相比裸 `std::thread` 很方便的一点。

如果异步任务抛异常：

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int main()
{
    auto result = std::async(std::launch::async, []() -> int {
        throw std::runtime_error("task failed");
    });

    try
    {
        std::cout << result.get() << '\n';
    }
    catch (const std::exception& e)
    {
        std::cout << e.what() << '\n';
    }
}
```

异常会被保存到共享状态中，然后在：

```cpp
result.get()
```

时重新抛出。

这比让工作线程自己想办法把错误传回主线程方便很多。

## 10. `std::promise<T>`

`std::promise<T>` 可以理解成 future 通道的“写入端”。

对应关系：

```text
promise<T>  ----写入结果---->  shared state  ----读取结果----> future<T>
```

通过：

```cpp
promise.get_future()
```

取得与它关联的 future。

示例：

```cpp
#include <future>
#include <iostream>
#include <thread>

void producer(std::promise<int> promise)
{
    promise.set_value(42);
}

int main()
{
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::thread t(producer, std::move(promise));

    std::cout << future.get() << '\n';
    t.join();
}
```

注意 `std::promise` 不能随意复制，因此这里通过：

```cpp
std::move(promise)
```

把 promise 的所有权移动到工作线程。

## 11. `promise::set_value()`

```cpp
promise.set_value(value);
```

向共享状态写入结果。

对于 `std::promise<void>`：

```cpp
promise.set_value();
```

表示任务成功完成，但没有具体返回值。

## 12. `promise::set_exception()`

promise 也可以主动写入异常：

```cpp
try
{
    ...
}
catch (...)
{
    promise.set_exception(std::current_exception());
}
```

之后 future 调用：

```cpp
future.get()
```

会重新抛出这个异常。

完整示例：

```cpp
#include <exception>
#include <future>
#include <iostream>
#include <stdexcept>
#include <thread>

void producer(std::promise<int> promise)
{
    try
    {
        throw std::runtime_error("failed");
    }
    catch (...)
    {
        promise.set_exception(std::current_exception());
    }
}

int main()
{
    std::promise<int> promise;
    auto future = promise.get_future();

    std::thread t(producer, std::move(promise));

    try
    {
        std::cout << future.get() << '\n';
    }
    catch (const std::exception& e)
    {
        std::cout << e.what() << '\n';
    }

    t.join();
}
```

## 13. 什么是 broken promise

如果 `promise` 在既没有：

```cpp
set_value()
```

也没有：

```cpp
set_exception()
```

的情况下被销毁，那么对应 future 不会永远傻等。

它会得到一个“broken promise”错误状态。

例如：

```cpp
std::future<int> future;

{
    std::promise<int> promise;
    future = promise.get_future();
} // promise 销毁，但没有写入结果

future.get(); // 抛出 std::future_error
```

## 14. `std::packaged_task`

`std::packaged_task` 用来把一个可调用对象包装成：

> 执行后自动把返回值或异常写进共享状态的任务。

例如：

```cpp
#include <future>
#include <iostream>

int add(int a, int b)
{
    return a + b;
}

int main()
{
    std::packaged_task<int(int, int)> task(add);
    std::future<int> result = task.get_future();

    task(10, 20);

    std::cout << result.get() << '\n';
}
```

这里：

```cpp
std::packaged_task<int(int, int)>
```

表示：

```text
接收两个 int
返回一个 int
```

## 15. `packaged_task` 和 thread 配合

```cpp
#include <future>
#include <iostream>
#include <thread>

int calculate()
{
    return 42;
}

int main()
{
    std::packaged_task<int()> task(calculate);
    std::future<int> result = task.get_future();

    std::thread t(std::move(task));

    std::cout << result.get() << '\n';
    t.join();
}
```

`packaged_task` 本身是 move-only 的，因此传入线程时需要：

```cpp
std::move(task)
```

线程池内部经常会出现类似思想：

```text
把任务包装起来
      ↓
放入任务队列
      ↓
worker thread 取出并执行
      ↓
future 接收结果
```

## 16. `promise` 和 `packaged_task` 的区别

`promise` 更像：

> 我自己决定什么时候、在什么逻辑里把结果写进去。

而 `packaged_task` 更像：

> 我已经有一个可调用对象，执行它时自动把返回值写进 future。

例如：

```text
promise
适合手动生产结果

packaged_task
适合包装已有函数/任务
```

## 17. `std::shared_future`

普通 `std::future` 更偏单消费者：

```cpp
future.get()
```

通常只能消费一次。

如果多个地方都需要读取同一个异步结果，可以使用：

```cpp
std::shared_future<T>
```

可以由普通 future 转换：

```cpp
auto future = std::async(std::launch::async, [] {
    return 42;
});

std::shared_future<int> shared = future.share();
```

之后多个线程可以：

```cpp
shared.get()
```

读取同一个结果。

## 18. `async` 和 `thread` 怎么选

如果你关心的是：

```text
启动一个长期运行线程
控制线程生命周期
线程循环
明确 join/detach
```

使用：

```cpp
std::thread / std::jthread
```

如果你关心的是：

```text
提交一个任务
未来得到返回值
自动传播异常
```

可以优先考虑：

```cpp
std::async / std::future
```

它们表达的是不同层次的抽象。

## 19. 常见错误

1. 不写 launch policy，却默认认为 `std::async` 一定创建新线程。
2. 对同一个普通 `future` 重复调用 `get()`。
3. 忘记处理异步任务在 `get()` 时重新抛出的异常。
4. `promise` 销毁前既不 `set_value()` 也不 `set_exception()`，产生 broken promise。
5. 把 move-only 的 `promise` / `packaged_task` 当成可复制对象使用。
6. 为一个长期运行、需要明确停止控制的后台线程强行使用 `std::async`。

## 小结

- `std::future<T>` 表示未来产生的 `T` 结果。
- `std::async` 适合直接提交一个会返回结果的任务。
- 明确要求真正异步执行时，使用 `std::launch::async`。
- `future::get()` 会等待结果并取得值，也会重新抛出异步任务中的异常。
- `std::promise` 是结果通道的主动写入端。
- `std::packaged_task` 把可调用对象包装成能产生 future 的任务。
- `std::shared_future` 允许多个消费者读取同一个结果。
- `thread` 偏线程生命周期管理，`future/async` 偏任务与结果管理。
