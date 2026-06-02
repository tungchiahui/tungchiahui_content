---
title: "queue / stack / deque"
---

## 本节解决什么问题

当你需要"先进先出"（FIFO）或者"先进后出"（LIFO）的数据结构时，用 vector 或 array 不是不行，但语义不清晰，操作不方便（从头部删除元素很慢）。

STL 提供了 `std::queue`、`std::stack`、`std::deque` 三种容器适配器，让代码意图清晰、操作高效。

## 这个特性是什么

- `std::deque<T>`：**双端队列**，可以在头部和尾部快速插入/删除。它是 `queue` 和 `stack` 的默认底层容器。
- `std::queue<T>`：**先进先出**（FIFO）队列，只能在尾部插入，头部取出。
- `std::stack<T>`：**先进后出**（LIFO）栈，只能在顶部插入和取出。

## C++ 标准版本

C++98

## 需要的头文件

```cpp
#include <queue>   // for std::queue, std::deque
#include <stack>   // for std::stack
```

## 基本语法

```cpp
// queue（队列）
std::queue<int> q;
q.push(1);           // 入队（尾部）
q.pop();             // 出队（头部）
q.front();           // 查看队首
q.back();            // 查看队尾

// stack（栈）
std::stack<int> s;
s.push(1);           // 入栈
s.pop();             // 出栈
s.top();             // 查看栈顶

// deque（双端队列）
std::deque<int> d = {1, 2, 3};
d.push_front(0);     // 头部插入
d.push_back(4);      // 尾部插入
d.pop_front();       // 头部删除
d.pop_back();        // 尾部删除
d[0];                // 随机访问（deque 独有！）
```

## 常用用法

| 容器 | 插入 | 删除 | 访问 | 特点 |
|:---|:---|:---|:---|:---|
| `queue` | `push()` | `pop()` | `front()` / `back()` | 先进先出 |
| `stack` | `push()` | `pop()` | `top()` | 先进后出 |
| `deque` | `push_front/back()` | `pop_front/back()` | `d[i]` / `front()` / `back()` | 双端 + 随机访问 |

> **注意**：`queue` 和 `stack` 的 `pop()` 不返回值，需要先 `front()`/`top()` 获取值再 `pop()`。

## 示例代码

### 示例 1：queue 先进先出

```cpp
#include <iostream>
#include <queue>

int main()
{
    std::queue<int> q;

    // 入队
    q.push(10);
    q.push(20);
    q.push(30);

    std::cout << "front = " << q.front() << "\n";
    std::cout << "back = " << q.back() << "\n";

    // 逐个出队
    std::cout << "dequeue: ";
    while (!q.empty())
    {
        std::cout << q.front() << " ";
        q.pop();
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
front = 10
back = 30
dequeue: 10 20 30 
```

### 示例 2：stack 先进后出

```cpp
#include <iostream>
#include <stack>

int main()
{
    std::stack<int> s;

    // 入栈
    s.push(10);
    s.push(20);
    s.push(30);

    std::cout << "top = " << s.top() << "\n";

    // 逐个出栈
    std::cout << "pop: ";
    while (!s.empty())
    {
        std::cout << s.top() << " ";
        s.pop();
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
top = 30
pop: 30 20 10 
```

### 示例 3：在示例 1、2 基础上，deque 双端操作

```cpp
#include <iostream>
#include <deque>

int main()
{
    std::deque<int> d = {20, 30};

    // 两端插入
    d.push_front(10);   // 头部插入
    d.push_back(40);    // 尾部插入

    // 随机访问
    std::cout << "d[0] = " << d[0] << "\n";
    std::cout << "d[2] = " << d[2] << "\n";

    // 遍历
    std::cout << "deque: ";
    for (int n : d)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // 两端删除
    d.pop_front();
    d.pop_back();
    std::cout << "after pop_front and pop_back: ";
    for (int n : d)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
d[0] = 10
d[2] = 30
deque: 10 20 30 40 
after pop_front and pop_back: 20 30 
```

### 示例 4：用 stack 实现简单的括号匹配检查

```cpp
#include <iostream>
#include <stack>
#include <string>

bool is_balanced(const std::string& expr)
{
    std::stack<char> s;
    for (char c : expr)
    {
        if (c == '(')
        {
            s.push(c);
        }
        else if (c == ')')
        {
            if (s.empty())
            {
                return false;  // 多了右括号
            }
            s.pop();
        }
    }
    return s.empty();  // 栈为空说明完全匹配
}

int main()
{
    std::cout << std::boolalpha;
    std::cout << "\"(())()\" balanced? " << is_balanced("(())()") << "\n";   // true
    std::cout << "\"(()\" balanced? " << is_balanced("(()") << "\n";         // false
    std::cout << "\"())\" balanced? " << is_balanced("())") << "\n";         // false

    return 0;
}
```

**运行结果**：

```
"(())()" balanced? true
"(()" balanced? false
"())" balanced? false
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | queue FIFO 行为 | `push()`、`front()`、`back()`、`pop()` | queue 只能从尾部进、头部出 | `pop()` 不返回值，要先 `front()` 取值 |
| 示例 2 | stack LIFO 行为 | `push()`、`top()`、`pop()` | stack 只能从顶部进出 | `pop()` 不返回值，要先 `top()` 取值 |
| 示例 3 | deque 双端操作和随机访问 | `push_front/back`、`pop_front/back`、`d[i]` | deque 支持两端操作且支持下标 | deque 是唯一一个两者都支持的 |
| 示例 4 | stack 实际应用 | 用 stack 实现括号匹配 | 利用 LIFO 特性，遇到右括号弹出最近的左括号 | stack 非常适合"最近未匹配"的场景 |

## 常见错误

**错误 1：以为 `pop()` 会返回值**

```cpp
int x = q.pop();  // ❌ pop() 返回 void！
```

正确做法：`int x = q.front(); q.pop();`

**错误 2：队/栈为空时调用 front/top**

```cpp
std::queue<int> q;
q.front();  // ❌ 未定义行为！
```

正确做法：先检查 `!q.empty()`。

**错误 3：用 queue/front 访问中间元素**

`queue` 和 `stack` 是**受限访问**容器，不能遍历也不能随机访问。如果需要这些功能，考虑 `deque` 或 `vector`。

## 使用建议

1. **queue 适合任务排队、消息缓冲**。
2. **stack 适合括号匹配、撤销操作、函数调用栈**。
3. **deque 适合需要两端插入删除 + 随机访问的场景**。
4. **C++23 起 `stack` 和 `queue` 支持 `push_range`**，可以批量插入。

## 小结

- `queue`：先进先出，插入用 `push`，查看用 `front`，删除用 `pop`。
- `stack`：先进后出，插入用 `push`，查看用 `top`，删除用 `pop`。
- `deque`：双端队列，支持两端操作和下标随机访问。
- `pop()` 都不返回值，需要先取值再删除。
