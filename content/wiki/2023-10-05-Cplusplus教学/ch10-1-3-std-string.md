---
title: "std::string"
---

## 本节解决什么问题

C 语言用 `char[]` 或 `char*` 表示字符串，使用时很痛苦：需要手动管理内存（`malloc/free`）、不能用 `==` 比较（要用 `strcmp`）、不能直接用 `+` 拼接（要用 `strcat` 还要注意缓冲区溢出）、不知道字符串长度（每次 `strlen` 要遍历一遍）。

`std::string` 解决的就是这些痛点，让你像操作普通变量一样操作字符串。

## 这个特性是什么

`std::string` 是 STL 中的字符串类，自动管理内存，支持拼接、比较、查找、截取等操作，用法简单直观。

## C++ 标准版本

C++98（C++11 起增加了移动语义等性能优化）。

## 需要的头文件

```cpp
#include <string>
```

## 基本语法

```cpp
std::string s;                     // 空字符串
std::string s = "hello";           // 从 C 字符串构造
std::string s("hello");            // 同上
std::string s(5, 'a');             // "aaaaa"
std::string s2 = s;                // 拷贝
```

## 常用用法

| 操作 | 说明 |
|:---|:---|
| `s1 + s2` | 字符串拼接 |
| `s1 == s2` | 内容比较（不是比较地址！） |
| `s.size()` / `s.length()` | 获取长度 |
| `s.empty()` | 判断是否为空 |
| `s[i]` | 访问第 i 个字符 |
| `s.find(str)` | 查找子串（返回位置，找不到返回 `std::string::npos`） |
| `s.substr(pos, len)` | 截取子串 |
| `s.append(str)` | 追加字符串 |
| `s.insert(pos, str)` | 插入字符串 |
| `s.replace(pos, len, str)` | 替换子串 |
| `stoi(s)` / `stod(s)` | 字符串转 int / double |
| `std::to_string(n)` | 数字转字符串 |

## 示例代码

### 示例 1：创建 string、拼接和比较

```cpp
#include <iostream>
#include <string>

int main()
{
    std::string s1 = "Hello";
    std::string s2 = "World";

    // 拼接
    std::string s3 = s1 + " " + s2;
    std::cout << "s3 = " << s3 << "\n";

    // 长度
    std::cout << "length = " << s3.size() << "\n";

    // 比较（和 C 字符串比较内容，不是比较地址）
    if (s1 == "Hello")
    {
        std::cout << "s1 equals \"Hello\"\n";
    }

    return 0;
}
```

**运行结果**：

```
s3 = Hello World
length = 11
s1 equals "Hello"
```

### 示例 2：在示例 1 基础上，用 find 查找和 substr 截取

```cpp
#include <iostream>
#include <string>

int main()
{
    std::string s = "Hello World";

    // 查找子串
    size_t pos = s.find("World");
    if (pos != std::string::npos)
    {
        std::cout << "\"World\" found at position " << pos << "\n";
    }

    // 截取子串
    std::string sub = s.substr(0, 5);  // 从位置 0 开始取 5 个字符
    std::cout << "substr(0, 5) = " << sub << "\n";

    // 查找不存在的子串
    size_t pos2 = s.find("xyz");
    if (pos2 == std::string::npos)
    {
        std::cout << "\"xyz\" not found\n";
    }

    return 0;
}
```

**运行结果**：

```
"World" found at position 6
substr(0, 5) = Hello
"xyz" not found
```

### 示例 3：在示例 2 基础上，数字和字符串互转

```cpp
#include <iostream>
#include <string>

int main()
{
    // 数字转字符串
    int score = 95;
    double pi = 3.14159;
    std::string s1 = std::to_string(score);
    std::string s2 = std::to_string(pi);
    std::cout << "int to string: " << s1 << "\n";
    std::cout << "double to string: " << s2 << "\n";

    // 字符串转数字
    std::string s3 = "42";
    std::string s4 = "3.14";
    int n = std::stoi(s3);
    double d = std::stod(s4);
    std::cout << "string to int: " << n << "\n";
    std::cout << "string to double: " << d << "\n";

    return 0;
}
```

**运行结果**：

```
int to string: 95
double to string: 3.141590
string to int: 42
string to double: 3.14
```

### 示例 4：在函数中处理字符串（统计字符出现次数）

```cpp
#include <iostream>
#include <string>

int count_char(const std::string& s, char c)
{
    int count = 0;
    for (char ch : s)
    {
        if (ch == c)
        {
            ++count;
        }
    }
    return count;
}

int main()
{
    std::string text = "hello world, welcome to modern C++";

    // 统计字母 'o' 出现的次数
    int n = count_char(text, 'o');
    std::cout << "'o' appears " << n << " times\n";

    // 统计字母 'l' 出现的次数
    std::cout << "'l' appears " << count_char(text, 'l') << " times\n";

    return 0;
}
```

**运行结果**：

```
'o' appears 4 times
'l' appears 5 times
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 创建、拼接、比较 | `+`、`==`、`.size()` | string 支持运算符，和基础类型一样用 | `+` 至少有一个操作数是 string |
| 示例 2 | 查找和截取 | `.find()`、`std::string::npos`、`.substr()` | `find` 返回位置，找不到返回 `npos` | 始终检查返回值是否为 `npos` |
| 示例 3 | 数字与字符串互转 | `std::to_string()`、`std::stoi()`、`std::stod()` | 格式化输出或解析输入时常用 | `stoi` 若字符串非法会抛异常 |
| 示例 4 | 字符串作为函数参数 | `const std::string&`、范围 for 遍历 | 传引用避免拷贝，范围 for 遍历字符 | `const std::string&` 是最佳传参方式 |

## 常见错误

**错误 1：拼接时两个都是 C 字符串**

```cpp
std::string s = "Hello" + "World";  // ❌ 编译错误！两个都是 const char*
```

正确做法：至少一个是 `std::string`：

```cpp
std::string s = std::string("Hello") + "World";  // ✅
```

**错误 2：不检查 find 返回值**

```cpp
std::string s = "hello";
size_t pos = s.find("x");
std::cout << s.substr(pos);  // ❌ pos 可能是 npos（一个巨大的值）
```

正确做法：先检查 `pos != std::string::npos`。

**错误 3：用 `=` 给 string 赋数字**

```cpp
std::string s = 123;  // ❌ 不会变成 "123"！
```
正确做法：用 `std::to_string(123)`。

## 使用建议

1. **永远用 `std::string`**：除非是纯 C 项目或与 C API 交互，否则不要用 `char[]`。
2. **传参用 `const std::string&`**：避免不必要的拷贝。
3. **频繁拼接用 `+=` 或 `append`**：比反复 `+` 更高效。
4. **C++17 起可用 `std::string_view`**：只读字符串视图，性能更好（后面会讲）。
5. **C++20 起可用 `s.starts_with()` / `s.ends_with()`**：判断前缀/后缀更方便。

## 小结

- `std::string` 自动管理内存，支持拼接（`+`）、比较（`==`）、查找（`find`）、截取（`substr`）。
- 数字与字符串互转：`std::to_string()` 和 `std::stoi()` / `std::stod()`。
- 传参用 `const std::string&`，遍历用范围 for。
- 记住检查 `find` 的返回值是否为 `npos`。
