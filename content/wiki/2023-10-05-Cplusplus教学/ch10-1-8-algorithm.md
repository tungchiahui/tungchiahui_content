---
title: "algorithm"
---

## 本节解决什么问题

很多时候你需要对容器中的元素做操作：排序、查找、统计、遍历、替换等。自己写循环当然可以，但是又长又容易出错。

STL 的 `<algorithm>` 头文件提供了大量现成的、经过优化的通用算法，你只要告诉它"从哪里到哪里"、"做什么"，不用关心底层循环怎么写。

## 这个特性是什么

`<algorithm>` 是 STL 的算法库，包含了排序、查找、修改、排列等几十个常用算法。它们操作的是**迭代器范围 `[first, last)`**，与具体容器解耦。

## C++ 标准版本

C++98（C++11/14/17/20 不断增加新算法和 ranges 版本）。

## 需要的头文件

```cpp
#include <algorithm>
```

## 基本语法

```cpp
std::算法名(起始迭代器, 结束迭代器, 参数...);
```

所有算法操作的范围都是 `[first, last)`，左闭右开。

## 常用算法分类

| 分类 | 算法 | 说明 |
|:---|:---|:---|
| 查找 | `find` / `find_if` | 线性查找 |
| 查找 | `binary_search` | 二分查找（要求有序） |
| 查找 | `count` / `count_if` | 统计个数 |
| 排序 | `sort` | 快速排序（不稳定） |
| 排序 | `stable_sort` | 稳定排序 |
| 修改 | `copy` / `copy_if` | 拷贝 |
| 修改 | `transform` | 逐元素变换 |
| 修改 | `replace` / `replace_if` | 替换 |
| 修改 | `fill` | 填充 |
| 修改 | `remove` / `remove_if` | 移除（配合 erase 使用） |
| 遍历 | `for_each` | 对每个元素执行操作 |
| 极值 | `min_element` / `max_element` | 找最小/最大元素 |
| 其他 | `reverse` | 反转 |
| 其他 | `unique` | 去重（要求有序） |

## 示例代码

### 示例 1：sort 排序和 find 查找

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main()
{
    std::vector<int> v = {5, 2, 8, 1, 9, 3};

    // 排序（默认升序）
    std::sort(v.begin(), v.end());

    std::cout << "sorted: ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // 查找元素
    auto it = std::find(v.begin(), v.end(), 8);
    if (it != v.end())
    {
        std::cout << "Found " << *it << " at position " << (it - v.begin()) << "\n";
    }

    return 0;
}
```

**运行结果**：

```
sorted: 1 2 3 5 8 9 
Found 8 at position 4
```

### 示例 2：在示例 1 基础上，降序排序和 count 统计

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main()
{
    std::vector<int> v = {5, 2, 8, 1, 9, 3, 5, 5};

    // 降序排序
    std::sort(v.begin(), v.end(), std::greater<int>());

    std::cout << "descending: ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // 统计 5 出现的次数
    int cnt = std::count(v.begin(), v.end(), 5);
    std::cout << "5 appears " << cnt << " times\n";

    return 0;
}
```

**运行结果**：

```
descending: 9 8 5 5 5 3 2 1 
5 appears 3 times
```

### 示例 3：在示例 2 基础上，transform 变换和 remove_if + erase 删除

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main()
{
    std::vector<int> v = {1, 2, 3, 4, 5};

    // transform：每个元素乘以 2
    std::transform(v.begin(), v.end(), v.begin(),
                   [](int n) { return n * 2; });

    std::cout << "after transform (*2): ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // remove_if + erase：删除大于 5 的元素
    v.erase(std::remove_if(v.begin(), v.end(),
                           [](int n) { return n > 5; }),
            v.end());

    std::cout << "after remove_if (>5): ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
after transform (*2): 2 4 6 8 10 
after remove_if (>5): 2 4 
```

### 示例 4：用 for_each 遍历、min_element / max_element 找极值

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main()
{
    std::vector<int> v = {5, 2, 8, 1, 9, 3};

    // for_each：对每个元素执行操作
    std::cout << "for_each: ";
    std::for_each(v.begin(), v.end(),
                  [](int n) { std::cout << n << " "; });
    std::cout << "\n";

    // 找最小值
    auto min_it = std::min_element(v.begin(), v.end());
    std::cout << "min = " << *min_it << "\n";

    // 找最大值
    auto max_it = std::max_element(v.begin(), v.end());
    std::cout << "max = " << *max_it << "\n";

    // 反转
    std::reverse(v.begin(), v.end());
    std::cout << "reversed: ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
for_each: 5 2 8 1 9 3 
min = 1
max = 9
reversed: 3 9 1 8 2 5 
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | sort 排序和 find 查找 | `sort()`、`find()`、`it - v.begin()` | sort 默认升序；find 返回迭代器 | `it - begin` 只对连续存储容器有效（vector/array/deque） |
| 示例 2 | 降序排序和统计 | `std::greater<int>()`、`count()` | sort 第三个参数是比较器 | `greater<int>()` 是预定义的降序比较器 |
| 示例 3 | transform 变换和条件删除 | `transform()`、`remove_if()` + `erase()` | `remove_if` 不真正删除，要配合 `erase` | "erase-remove" 是经典惯用法 |
| 示例 4 | for_each 和极值查找 | `for_each()`、`min_element()`、`reverse()` | for_each 返回传入的可调用对象 | `min/max_element` 返回迭代器不是值 |

## 常见错误

**错误 1：`remove` 之后忘记 `erase`**

```cpp
v.erase(std::remove(v.begin(), v.end(), 5));  // ❌ 少了一个参数
```

正确做法：`v.erase(std::remove(v.begin(), v.end(), 5), v.end());`

**错误 2：对未排序的容器使用 `binary_search`**

```cpp
std::vector<int> v = {5, 2, 8, 1};
std::binary_search(v.begin(), v.end(), 5);  // ❌ 未排序，结果不确定！
```

正确做法：先 `sort` 再 `binary_search`。

**错误 3：算法操作用错迭代器范围**

```cpp
std::sort(v.begin(), v.begin() + 3);  // 只排序前 3 个（这本身 OK 但容易忘）
```

注意：所有算法操作的范围是 `[first, last)`，`last` 不包含在内。

## 使用建议

1. **先用标准算法而不是自己写循环**：代码更简洁，意图更清晰，还不容易出错。
2. **`sort` + `unique` + `erase` 可以高效去重**。
3. **Lambda 和算法是绝配**：`find_if`、`remove_if`、`transform` 等结合 lambda 很强大。
4. **C++20 引入 ranges**：`std::ranges::sort(v)` 更简洁，但底层算法是一样的。

## 小结

- `<algorithm>` 提供排序（`sort`）、查找（`find`）、统计（`count`）、变换（`transform`）、遍历（`for_each`）等几十种算法。
- 所有算法操作 `[first, last)` 迭代器范围，与容器解耦。
- `remove`/`remove_if` 需要配合 `erase` 才能真正删除元素。
- 先用标准算法，不要重复造轮子。
