---
title: "std::map / std::unordered_map"
---

## 本节解决什么问题

你需要存储"键值对"数据——比如学生姓名→成绩、单词→出现次数、ID→对象。如果用数组或 vector，每次查找都要遍历一遍，效率很低（O(n)）。你需要一种能**按键快速查找**的容器。

`std::map` 和 `std::unordered_map` 解决了这个问题，它们提供了 O(log n) 和 O(1) 平均查找时间。

## 这个特性是什么

- `std::map<Key, Value>`：有序键值对容器，底层是**红黑树**，按键排序。
- `std::unordered_map<Key, Value>`：无序键值对容器，底层是**哈希表**，不排序但查找更快（平均 O(1)）。

## C++ 标准版本

- `std::map`：C++98
- `std::unordered_map`：C++11

## 需要的头文件

```cpp
#include <map>             // for std::map
#include <unordered_map>   // for std::unordered_map
```

## 基本语法

```cpp
// map：键有序
std::map<std::string, int> m;
m["key"] = 100;

// unordered_map：键无序，查找更快
std::unordered_map<std::string, int> um;
um["key"] = 100;
```

## 常用用法

| 操作 | 说明 |
|:---|:---|
| `m[key] = val` | 插入或更新键值对（key 不存在则插入） |
| `m.at(key)` | 访问（key 不存在抛异常） |
| `m[key]` | 访问（key 不存在则**默认构造一个**，慎用） |
| `m.find(key)` | 查找（返回迭代器，找不到返回 `end()`） |
| `m.insert({k, v})` | 插入（key 已存在则插入失败） |
| `m.erase(key)` | 删除键值对 |
| `m.size()` | 元素个数 |
| `m.count(key)` | 判断键是否存在（返回 0 或 1） |

## 示例代码

### 示例 1：用 map 存储学生成绩，用姓名查找

```cpp
#include <iostream>
#include <map>
#include <string>

int main()
{
    // 姓名 → 成绩
    std::map<std::string, int> scores;

    // 插入数据
    scores["Alice"] = 85;
    scores["Bob"] = 92;
    scores["Charlie"] = 78;

    // 按键查找
    std::cout << "Alice's score: " << scores["Alice"] << "\n";
    std::cout << "Bob's score: " << scores["Bob"] << "\n";

    return 0;
}
```

**运行结果**：

```
Alice's score: 85
Bob's score: 92
```

### 示例 2：在示例 1 基础上，用 find 安全查找和遍历

```cpp
#include <iostream>
#include <map>
#include <string>

int main()
{
    std::map<std::string, int> scores = {
        {"Alice", 85},
        {"Bob", 92},
        {"Charlie", 78}
    };

    // find() 安全查找（不会在找不到时插入新元素）
    auto it = scores.find("David");
    if (it != scores.end())
    {
        std::cout << "David's score: " << it->second << "\n";
    }
    else
    {
        std::cout << "David not found\n";
    }

    // 遍历整个 map（按键的字母顺序排列）
    std::cout << "All scores:\n";
    for (const auto& pair : scores)
    {
        std::cout << "  " << pair.first << ": " << pair.second << "\n";
    }

    return 0;
}
```

**运行结果**：

```
David not found
All scores:
  Alice: 85
  Bob: 92
  Charlie: 78
```

### 示例 3：在示例 2 基础上，对比 map 和 unordered_map 的顺序

```cpp
#include <iostream>
#include <map>
#include <string>
#include <unordered_map>

int main()
{
    std::map<std::string, int> ordered = {
        {"Charlie", 78},
        {"Alice", 85},
        {"Bob", 92}
    };

    std::unordered_map<std::string, int> unordered = {
        {"Charlie", 78},
        {"Alice", 85},
        {"Bob", 92}
    };

    // map：按键排序
    std::cout << "map (ordered by key):\n";
    for (const auto& p : ordered)
    {
        std::cout << "  " << p.first << ": " << p.second << "\n";
    }

    // unordered_map：顺序不确定
    std::cout << "unordered_map (arbitrary order):\n";
    for (const auto& p : unordered)
    {
        std::cout << "  " << p.first << ": " << p.second << "\n";
    }

    return 0;
}
```

**运行结果**（unordered_map 顺序可能不同）：

```
map (ordered by key):
  Alice: 85
  Bob: 92
  Charlie: 78
unordered_map (arbitrary order):
  Bob: 92
  Alice: 85
  Charlie: 78
```

### 示例 4：用 map 统计单词出现次数

```cpp
#include <iostream>
#include <map>
#include <string>

int main()
{
    std::map<std::string, int> word_count;
    std::string words[] = {"apple", "banana", "apple", "cherry", "banana", "apple"};

    for (const auto& w : words)
    {
        ++word_count[w];  // 不存在自动插入并初始化为 0，然后 +1
    }

    for (const auto& pair : word_count)
    {
        std::cout << pair.first << ": " << pair.second << "\n";
    }

    return 0;
}
```

**运行结果**：

```
apple: 3
banana: 2
cherry: 1
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 用 map 存储键值对 | `m[key] = val` | 下标操作符可直接插入或修改 | `m[key]` 在 key 不存在时会自动创建 |
| 示例 2 | find() 安全查找和遍历 | `find()`、`pair.first`/`pair.second`、`const auto&` | `find` 不会意外插入空值 | map 遍历时是按键排序的 |
| 示例 3 | map vs unordered_map | 两种容器的区别 | 需要排序用 map，需要更快查找用 unordered_map | unordered_map 元素顺序不确定 |
| 示例 4 | 统计频率的应用 | `++m[key]` | `m[key]` 不存在返回 0（int 的默认值） | 依赖 int 默认值为 0 的特性 |

## 常见错误

**错误 1：读取时误用 `m[key]` 造成意外插入**

```cpp
std::map<std::string, int> m = {{"Alice", 85}};
std::cout << m["Bob"];  // ❌ Bob 不存在，会被插入，值为 0！
```

正确做法：查找时用 `find()` 或 `count()`。

**错误 2：遍历时修改 key**

```cpp
for (auto& p : m)
{
    p.first = "new_key";  // ❌ 编译错误！key 是 const 的
}
```

正确做法：key 不可修改，只能修改 value（`p.second`）。

**错误 3：用自定义类型做 key 没有定义比较运算符**（只对 map）

```cpp
struct Point { int x, y; };
std::map<Point, int> m;  // ❌ 编译错误！Point 没有 operator<
```

正确做法：定义 `operator<` 或提供自定义比较函数。

## 使用建议

1. **需要有序遍历用 `map`**，不需要排序且追求速度用 `unordered_map`。
2. **查找用 `find()` 而不是 `m[key]`**，避免意外插入。
3. **int 类型的 value 默认是 0**，可以直接 `++m[key]` 做统计。
4. **插入用 `insert` 或 `emplace`**（有则不变），覆盖用 `m[key] = val`（有则覆盖）。
5. **C++17 起可以用 `insert_or_assign`**，语义更明确。

## 小结

- `std::map` 是按键排序的键值对容器（红黑树），`std::unordered_map` 是哈希表版本。
- 查找用 `find()`，不要用 `m[key]`（会意外插入）。
- 遍历用 `const auto&`，`pair.first` 是 key，`pair.second` 是 value。
- 统计频率是 map 的经典应用场景。
