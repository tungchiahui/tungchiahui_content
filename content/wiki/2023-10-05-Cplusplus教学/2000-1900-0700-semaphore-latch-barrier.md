---
title: "semaphore、latch 与 barrier"
---

C++20 在 `<semaphore>`、`<latch>`、`<barrier>` 中加入了三类更高层的同步工具：

```cpp
std::counting_semaphore
std::binary_semaphore
std::latch
std::barrier
```

它们解决的问题和 `mutex` 不同：

- `mutex`：保护共享临界区；
- `semaphore`：限制同时进入某个区域的线程数量，或表示可用资源数量；
- `latch`：一次性等待若干参与者全部到达；
- `barrier`：多个线程反复进行“阶段同步”。

这些工具不是并发入门第一优先级，但理解后可以避免手写复杂的条件变量计数逻辑。

## `std::counting_semaphore`

计数信号量内部维护一个计数值。

线程进入前调用：

```cpp
semaphore.acquire();
```

如果计数大于 0：

```text
计数减 1
线程继续
```

如果计数为 0：

```text
线程阻塞等待
```

使用完资源后调用：

```cpp
semaphore.release();
```

计数增加，并可能唤醒等待线程。

## 限制并发数量

例如只允许最多两个线程同时执行某段任务：

```cpp
#include <chrono>
#include <iostream>
#include <semaphore>
#include <thread>
#include <vector>

using namespace std::chrono_literals;

std::counting_semaphore<2> slots(2);

void worker(int id)
{
    slots.acquire();

    std::cout << "start " << id << '\n';
    std::this_thread::sleep_for(500ms);
    std::cout << "end " << id << '\n';

    slots.release();
}

int main()
{
    std::vector<std::thread> threads;

    for (int i = 0; i < 5; ++i)
    {
        threads.emplace_back(worker, i);
    }

    for (auto& thread : threads)
    {
        thread.join();
    }
}
```

即使创建了 5 个线程，真正进入受限区域的线程最多只有 2 个。

## 模板参数和初始计数不是同一个东西

```cpp
std::counting_semaphore<10> semaphore(3);
```

其中：

```text
10
```

表示实现至少支持的最大计数上界；

```text
3
```

表示当前初始计数。

不要把两者混在一起理解。

## `try_acquire()`

如果不想阻塞，可以使用：

```cpp
if (semaphore.try_acquire())
{
    // 成功获得一个许可
    semaphore.release();
}
```

还有带时间限制的版本：

```cpp
try_acquire_for(...)
try_acquire_until(...)
```

适合“等不到资源就做其他事情”的场景。

## `std::binary_semaphore`

二值信号量只有两种状态，可以理解为计数最多为 1 的 semaphore。

标准库提供别名：

```cpp
std::binary_semaphore
```

例如：

```cpp
std::binary_semaphore signal(0);
```

一个线程：

```cpp
signal.acquire();
```

另一个线程：

```cpp
signal.release();
```

它可以用于简单的“一次允许一个等待者继续”的同步。

不过二值信号量和 mutex 的语义仍然不同：

> mutex 强调所有权：由获得锁的线程释放；semaphore 强调许可数量，释放许可的线程不一定是获得许可的那个线程。

## semaphore 不能自动保护复杂共享状态

有了 semaphore 并不意味着共享容器就自动线程安全。

例如：

```cpp
std::vector<int> data;
```

如果多个线程同时修改这个 vector，仍然需要正确同步。

semaphore 常用于：

```text
控制并发数量
限制资源池容量
表示可用资源数
线程间发放许可
```

而不是简单替代 mutex。

## `std::latch`

`std::latch` 是一个一次性倒计时同步点。

创建时指定计数：

```cpp
std::latch done(3);
```

参与者完成后调用：

```cpp
done.count_down();
```

等待方调用：

```cpp
done.wait();
```

当计数降到 0 后，所有等待者继续执行。

## latch 示例

```cpp
#include <iostream>
#include <latch>
#include <thread>
#include <vector>

std::latch done(3);

void worker(int id)
{
    std::cout << "worker " << id << " done\n";
    done.count_down();
}

int main()
{
    std::vector<std::thread> threads;

    for (int i = 0; i < 3; ++i)
    {
        threads.emplace_back(worker, i);
    }

    done.wait();
    std::cout << "all workers reached the latch\n";

    for (auto& thread : threads)
    {
        thread.join();
    }
}
```

这里：

```cpp
done.wait();
```

只关心“3 个参与者是否都已经完成某个阶段”。

## `count_down()`、`wait()`、`arrive_and_wait()`

常见操作：

```cpp
latch.count_down();
latch.wait();
latch.arrive_and_wait();
```

`arrive_and_wait()` 可以理解为：

```text
我已经到达
计数减一
然后我也等待其他参与者
```

## latch 只能使用一次

这是 `latch` 和 `barrier` 最关键的区别。

一旦 latch 的计数变为 0：

```text
它不会重新恢复到初始值
```

所以它适合：

- 等待多个初始化任务完成；
- 等待一组 worker 启动就绪；
- 等待一次性并行任务全部结束。

如果程序需要一轮又一轮地同步，应使用 `std::barrier`。

## `std::barrier`

`std::barrier` 是可重复使用的阶段同步点。

假设有 3 个线程，每一轮都必须：

```text
阶段 1 全部完成
      ↓
所有线程才能进入阶段 2
      ↓
阶段 2 全部完成
      ↓
所有线程才能进入阶段 3
```

这种场景非常适合 barrier。

## barrier 示例

```cpp
#include <barrier>
#include <iostream>
#include <thread>
#include <vector>

std::barrier sync_point(3);

void worker(int id)
{
    std::cout << "phase 1: " << id << '\n';
    sync_point.arrive_and_wait();

    std::cout << "phase 2: " << id << '\n';
    sync_point.arrive_and_wait();

    std::cout << "phase 3: " << id << '\n';
}

int main()
{
    std::vector<std::thread> threads;

    for (int i = 0; i < 3; ++i)
    {
        threads.emplace_back(worker, i);
    }

    for (auto& thread : threads)
    {
        thread.join();
    }
}
```

虽然每个阶段内部的打印顺序仍然不确定，但不会有某个线程提前跨过同步点进入下一阶段，而其他线程还没有完成上一阶段。

## barrier 会自动开始下一轮

当本轮所有参与者到达后，barrier 会完成当前阶段，然后自动为下一阶段重新准备。

这正是它与 latch 的不同：

```text
latch
一次性

barrier
可重复阶段同步
```

## completion function

`std::barrier` 还可以在每一阶段全部参与者到达后执行一个完成函数。

概念上：

```text
所有线程到达 barrier
        ↓
执行 completion function
        ↓
进入下一阶段
```

这适合每轮结束都需要统一做一次状态更新的算法。

使用这个功能时应确保 completion function 本身不会引入新的复杂阻塞或锁依赖。

## `arrive_and_drop()`

如果某个参与者之后不再参加后续阶段，可以调用：

```cpp
barrier.arrive_and_drop();
```

表示：

> 我完成本轮到达，并且从以后各轮的参与者数量中永久退出。

这和简单离开线程不同，因为 barrier 必须知道未来还应该等待多少参与者。

## 三种工具怎么选

| 问题 | 推荐工具 |
|:---|:---|
| 最多允许 N 个线程同时访问资源 | `counting_semaphore` |
| 一个线程给另一个线程发一个简单许可 | `binary_semaphore` |
| 等待 N 个一次性任务全部到达 | `latch` |
| 多个线程每一轮都要在阶段边界会合 | `barrier` |
| 保护共享对象的一致性 | `mutex` |
| 等待复杂谓词条件 | `condition_variable` |

## 和 condition_variable 的关系

这些工具很多都可以用：

```text
mutex + condition_variable + counter
```

手工实现。

例如 latch 本质上就是一种：

```text
计数递减到 0 后唤醒等待者
```

的同步模式。

但既然 C++20 已经提供标准抽象，就不必每次重新发明一套状态变量和通知协议。

标准工具的优势是：

- 意图更明确；
- 代码更短；
- 更不容易写错边界条件；
- 阅读代码的人一眼能看出同步模式。

## 常见错误

1. 把 semaphore 当成 mutex，误以为它自动保护共享对象。
2. `acquire()` 后忘记 `release()`，导致许可永久减少。
3. 认为 binary semaphore 具有 mutex 一样严格的线程所有权语义。
4. 把一次性的 `latch` 当成可以重置循环使用的同步器。
5. barrier 的参与者数量设计错误，导致永远等不到所有参与者。
6. 某个线程提前退出，却没有用 `arrive_and_drop()` 调整 barrier 后续参与数量。
7. 在 completion function 中执行复杂阻塞操作，让所有参与者都卡在阶段边界。

## 小结

- `counting_semaphore` 用计数控制可同时获得的许可数量。
- `binary_semaphore` 是计数最多为 1 的信号量语义。
- `latch` 是一次性倒计时同步点，计数归零后永久打开。
- `barrier` 是可重复使用的阶段同步点。
- 这些工具解决的是同步模式，不直接替代 mutex 对共享状态的保护。
- C++20 项目遇到对应模式时，优先考虑标准同步原语，而不是手写复杂条件变量计数器。
