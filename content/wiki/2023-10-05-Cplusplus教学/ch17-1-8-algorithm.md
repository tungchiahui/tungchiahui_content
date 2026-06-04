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

算法库最重要的思想是：**算法不直接拥有容器，它只处理一段迭代器范围**。

所以同一个算法可以处理整个容器：

```cpp
std::sort(v.begin(), v.end());
```

也可以只处理一部分：

```cpp
std::sort(v.begin(), v.begin() + 3);  // 只排序前 3 个元素
```

这和普通函数“接收一个 vector”不一样。算法更通用，但也要求你准确传入范围。

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

### 常用算法怎么选

| 你想做什么 | 推荐算法 | 返回什么 |
|:---|:---|:---|
| 找一个等于某值的元素 | `std::find` | 迭代器，找不到返回 `end()` |
| 找一个满足条件的元素 | `std::find_if` | 迭代器，找不到返回 `end()` |
| 判断有没有某类元素 | `std::any_of` | `bool` |
| 判断是否全部满足条件 | `std::all_of` | `bool` |
| 统计等于某值的个数 | `std::count` | 个数 |
| 统计满足条件的个数 | `std::count_if` | 个数 |
| 排序 | `std::sort` | 原地修改范围 |
| 稳定排序 | `std::stable_sort` | 原地修改范围 |
| 删除满足条件的元素 | `std::remove_if` + `erase` | 真正删除要靠容器的 `erase` |
| 每个元素变成另一个值 | `std::transform` | 写入目标范围 |

注意：算法名里的 `_if` 通常表示“按条件处理”。条件一般用 Lambda 表达式写。

## 示例代码

### 示例 1：sort 排序和 find 查找

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

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

### 示例 5：find 和 find_if 的区别

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

struct Student
{
    std::string name;
    int score;
};

int main()
{
    std::vector<int> nums = {10, 20, 30, 40};

    auto n_it = std::find(nums.begin(), nums.end(), 30);
    if (n_it != nums.end())
    {
        std::cout << "found number: " << *n_it << "\n";
    }

    std::vector<Student> students = {
        {"Alice", 85},
        {"Bob", 92},
        {"Charlie", 78}
    };

    auto s_it = std::find_if(students.begin(), students.end(),
                             [](const Student& s) {
                                 return s.score >= 90;
                             });

    if (s_it != students.end())
    {
        std::cout << "first high score: "
                  << s_it->name << " " << s_it->score << "\n";
    }

    return 0;
}
```

**运行结果**：

```
found number: 30
first high score: Bob 92
```

`find` 适合“找某个具体值”，`find_if` 适合“找满足某个条件的元素”。真实工程里，结构体、类对象、消息包、传感器数据通常都不是直接按一个值比较，所以 `find_if` 更常用。

### 示例 6：sort 和 stable_sort 的区别要在相同关键字时才看得出来

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

struct Student
{
    std::string name;
    int score;
};

void print(const std::vector<Student>& students)
{
    for (const auto& s : students)
    {
        std::cout << s.name << "(" << s.score << ") ";
    }
    std::cout << "\n";
}

int main()
{
    std::vector<Student> students = {
        {"Alice", 90},
        {"Bob", 80},
        {"Charlie", 90},
        {"David", 80}
    };

    std::stable_sort(students.begin(), students.end(),
                     [](const Student& a, const Student& b) {
                         return a.score < b.score;
                     });

    print(students);

    return 0;
}
```

**运行结果**：

```
Bob(80) David(80) Alice(90) Charlie(90)
```

`stable_sort` 的“稳定”指的是：如果两个元素比较结果相等，它们原来的相对顺序会保留。这里 Bob 和 David 都是 80，排序后仍然是 Bob 在 David 前面；Alice 和 Charlie 都是 90，排序后仍然是 Alice 在 Charlie 前面。

如果只是排一组互不相同的数字，`sort` 和 `stable_sort` 看起来没区别；一旦有“相同分数但要保留原报名顺序”这种需求，区别就很明显。

### 示例 7：remove_if 不是真删，它只是把要保留的元素挪到前面

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

void print(const std::vector<int>& v)
{
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";
}

int main()
{
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    auto new_end = std::remove_if(nums.begin(), nums.end(),
                                  [](int n) { return n % 2 == 0; });

    std::cout << "after remove_if, size = " << nums.size() << "\n";
    print(nums);

    nums.erase(new_end, nums.end());

    std::cout << "after erase, size = " << nums.size() << "\n";
    print(nums);

    return 0;
}
```

**可能的运行结果**：

```
after remove_if, size = 6
1 3 5 4 5 6
after erase, size = 3
1 3 5
```

这个例子要仔细看：`remove_if` 之后 `size` 还是 6，因为算法不能直接改变 `vector` 的大小。它只是把要保留的 `1 3 5` 挪到了前面，并返回新的逻辑结尾 `new_end`。真正删除尾部无效区域，要靠 `nums.erase(new_end, nums.end())`。

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | sort 排序和 find 查找 | `sort()`、`find()`、`it - v.begin()` | sort 默认升序；find 返回迭代器 | `it - begin` 只对连续存储容器有效（vector/array/deque） |
| 示例 2 | 降序排序和统计 | `std::greater<int>()`、`count()` | sort 第三个参数是比较器 | `greater<int>()` 是预定义的降序比较器 |
| 示例 3 | transform 变换和条件删除 | `transform()`、`remove_if()` + `erase()` | `remove_if` 不真正删除，要配合 `erase` | "erase-remove" 是经典惯用法 |
| 示例 4 | for_each 和极值查找 | `for_each()`、`min_element()`、`reverse()` | for_each 返回传入的可调用对象 | `min/max_element` 返回迭代器不是值 |
| 示例 5 | 值查找和条件查找 | `find()`、`find_if()` | 具体值用 find，复杂条件用 find_if | 找不到时返回 `end()` |
| 示例 6 | 稳定排序 | `stable_sort()` | 相等元素保持原顺序 | 数据有相同关键字时才体现区别 |
| 示例 7 | erase-remove 惯用法 | `remove_if()` 返回新逻辑结尾 | 算法不能直接改变容器大小 | 必须再调用容器的 `erase` |

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

**错误 4：比较函数写成 `<=`**

```cpp
std::sort(v.begin(), v.end(),
          [](int a, int b) { return a <= b; });  // ❌ 错误比较器
```

排序比较器表达的是“a 是否应该排在 b 前面”，必须满足严格弱序。简单记法：升序写 `a < b`，降序写 `a > b`，不要写 `<=` 或 `>=`。

**错误 5：忘记检查 find 的返回值**

```cpp
auto it = std::find(v.begin(), v.end(), 100);
std::cout << *it;  // ❌ 如果没找到，it == v.end()
```

正确做法：先判断 `it != v.end()`。

## 使用建议

1. **先用标准算法而不是自己写循环**：代码更简洁，意图更清晰，还不容易出错。
2. **`sort` + `unique` + `erase` 可以高效去重**。
3. **Lambda 和算法是绝配**：`find_if`、`remove_if`、`transform` 等结合 lambda 很强大。
4. **C++20 引入 ranges**：`std::ranges::sort(v)` 更简洁，但底层算法是一样的。

## 小结

- `<algorithm>` 提供排序（`sort`）、查找（`find`）、统计（`count`）、变换（`transform`）、遍历（`for_each`）等几十种算法。
- 所有算法操作 `[first, last)` 迭代器范围，与容器解耦。
- `remove`/`remove_if` 需要配合 `erase` 才能真正删除元素。
- `find_if`、`count_if`、`remove_if` 这类条件算法通常配合 Lambda 使用。
- `sort` 和 `stable_sort` 在普通数字上差异不明显，在相同关键字需要保序时差异明显。
- 先用标准算法，不要重复造轮子。
