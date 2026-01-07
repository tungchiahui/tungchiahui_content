---
title: C/C++教程
date: 2023-10-05
path: /wiki/cplusplus-tutorial
---

> 📚 学习必须与实干相结合。—— 泰戈尔

## 前言
**C/C++介绍：**

C和C++是两种的高级计算机语言，常见的高级语言还有Python，Rust，Go，C#(C Sharp、C++++)，Java，JavaScript，LinuxShell等等。

C++语言是在C语言的基础上，添加了面向对象、模板等现代程序设计语言的特性而发展起来的。两者无论是从语法规则上，还是从运算符的数量和使用上，都非常相似，所以我们常常将这两门语言统称为“C/C++”。

C语言和C++并不是对立的竞争关系： 1)C++是C语言的加强，是一种更好的C语言，实际上C++和C语言是同一门语言的不同版本。 2)C++是以C语言为基础的，并且完全兼容C语言的特性。 C语言和C++语言的学习是可以相互促进。学好C语言，可以为我们将来进一步地学习C++语言打好基础，而C++语言的学习，也会促进我们对于C语言的理解，从而更好地运用C语言。

| 特性 | C 语言 | C++ 语言 |
|:---|:---|:---|
| 编程范式 | 面向过程 | 多范式，支持面向对象 |
| 内存管理 | 手动管理 | 手动管理，提供 RAII（资源获取即初始化） |
| 代码复用性 | 较低 | 高，通过类、继承、模板等实现 |
| 标准库 | 标准 C 库 | 标准模板库（STL）和 C 标准库 |
| 运行效率 | 高 | 稍低于 C，但差距不大 |
| 应用场景 | 操作系统、嵌入式 | 游戏开发、图形处理、大型应用 |
| 类型检查 | 较松散 | 较严格，提供更多类型检查 |

**本文只负责指导一些问题，学****C/C++****还是以下列视频为主:**

C/C++环境配置:[电控组环境搭建大全](https://sdutvincirobot.feishu.cn/wiki/FQszwXIR5iQgCfk7pRwc9rYpnqg)

1.  黑马程序员C++视频：

https://www.bilibili.com/video/BV1et411b73Z

2.  鹏哥C语言视频：

https://www.bilibili.com/video/BV1cq4y1U7sg

3.  菜鸟教程：

https://www.runoob.com/cprogramming/c-tutorial.html

https://www.runoob.com/cplusplus/cpp-tutorial.html

## 程序运行与变量生命周期
### 代码运行思路
从main函数开始，代码是一行一行运行的。（一个工程里有且只有一个main函数）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image1.webp)

### 生命周期
1.  **局部变量：**

    1.  位置：在某个函数或块的内部声明的变量称为局部变量。

    2.  作用域：它们只能被该函数或该代码块内部的语句使用。局部变量在函数外部是不可知的。

```cpp
#include <stdio.h>

 int add(int a,int b);

int main ()
{
  /* 局部变量声明 */
  int a, b;
  int c;

  /* 实际初始化 */
  a = 10;
  b = 20;
  c = add(a,b);

  printf ("value of a = %d, b = %d and c = %d\n", a, b, c);

  return 0;
}

int add(int x,int y)
{
    int z = x + y;
    return z;
}
```

2.  全局变量

    1.  位置：全局变量是定义在函数外部，通常是在程序的顶部。

    2.  作用域：全局变量在整个程序生命周期内都是有效的，在任意的函数内部能访问全局变量。

```cpp
#include <stdio.h>

/* 全局变量声明 */
int g;

int main ()
{
  /* 局部变量声明 */
  int a;

  /* 静态(全局)变量声明 */
  static int b;

  /* 实际初始化 */
  a = 10;
  b = 20;
  g = a + b;

  printf ("value of a = %d, b = %d and g = %d\n", a, b, g);

  return 0;
}
```

在程序中，局部变量和全局变量的名称可以相同，但是在函数内，如果两个名字相同，会使用局部变量值，全局变量不会被使用。

```cpp
#include <stdio.h>

 void tset();
/* 全局变量声明 */
int g = 20;

int main ()
{
  /* 局部变量声明 */
  int g = 10;

  printf ("value of g = %d\n",  g);

  test();

  return 0;
}

void test()
{
    printf("value of g = %d\n",g);
}
```

### 内存四区
暂时无法在飞书文档外展示此内容

1.  **静态存储区（全局区）** ：可以分为rodata区data区和bss区，已经初始化的只读常量被放在rodata,已经被初始化的非零变量被放在data区;没有被初始化的或者值为零的变量被放在bss区，通常默认初始化为 0(表示数字的数据类型)，'\\0'(char类型) 和 NULL(指针类型)。

2.  **栈区（stack）** ：局部变量被存放在该区，如果不初始化局部变量，那么局部变量是随机值。容量很小。

3.  **堆区（heap）** ：由程序员开辟内存空间给变量，由程序员分配和释放，如果程序员不进行释放内存，则会内存泄漏，当程序结束后，系统会帮忙释放没有被释放的内存。

4.  **代码区** ：存放 CPU 执行的机器指令，通常是只读的。

## 头文件
头文件的作用：头文件含有某个库的外部声明函数和变量，方便我们调用库中的API。

注意事项：

1.  常见的头文件stdio.h stdlib.h iostream string等

2.  头文件的扩展名：.h或者.hpp，其实没必要写扩展名，但是建议还是写。

3.  预处理：#include <> 和 #include " "

4.  条件编译

5.  extern "C" { } 用来实现C语言和C++的混合编译，表明它按照类C的编译和连接规约来编译和连接，而不是C++的编译的连接规约。

```cpp
#ifndef __FILE_NAME_H_    //头文件防止引用重复的条件编译
#define __FILE_NAME_H_   //头文件防止引用重复的条件编译

#ifdef __cplusplus    //混合编译的条件编译
extern "C"           //混合编译的条件编译
{                   //混合编译的条件编译
#endif             //混合编译的条件编译
/*  头文件内容开始   */

//头文件内容：预处理、函数声明、变量声明

/*   头文件内容结束  */
#ifdef __cplusplus     //混合编译的条件编译
}                      //混合编译的条件编译
#endif                 //混合编译的条件编译

#endif   //头文件防止引用重复的条件编译

```

## C语言 和 C++ 的I/O
1.C语言的stdio.h中的scanf和printf

**int scanf(const char \*format, ...)** 函数从标准输入流 **stdin** 读取输入，并根据提供的 **format** 来浏览输入。

**int printf(const char \*format, ...)** 函数把输出写入到标准输出流 **stdout ，并根据提供的格式产生输出。

```C
printf("输出内容(可含占位符)"，变量1，变量2)

printf("%d",a);      //输出一个整形变量a
printf("%f %f",a,b);   //输出二个单精度浮点数变量(fp32) a，b中间以空格隔开
printf("%d序号对应的值是%lf",a,b);   //输出二个单精度浮点数变量a，b中间以空格隔开
printf("%.2f",a);   //输出一个单精度浮点数(fp32) a,并保留两位小数
printf("你好")  //输出“你好”字符串
```

```cpp
scanf("%d",&a);           //输入一个整形数
scanf("%d %d",&a，&b);   //输入两个整形数，中间以空格隔开      
scanf("%d,%d",&a，&b);   //输入两个整形数，中间以逗号隔开
```

2.C++的iostream中的std::cin和std::cout

**cout** 是与流插入运算符 << 结合使用

```cpp
std::cout << a;   //输出一个变量a
std::cout << "你好"   //输出"你好"
std::cout << "结果是：" << a << std::endl  //输出 结果是: a  并换行
```

**cin** 是与流提取运算符 >> 结合使用

```cpp
std::cin >> a     //输入一个变量a
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image2.webp)

## C++命名空间
在 C++ 应用程序中。例如，您可能会写一个名为 xyz() 的函数，在另一个可用的库中也存在一个相同的函数 xyz()。这样，编译器就无法判断您所使用的是哪一个 xyz() 函数。

因此，引入了 **命名空间** 这个概念，专门用于解决上面的问题，它可作为附加信息来区分不同库中相同名称的函数、类、变量等。使用了命名空间即定义了上下文。本质上，命名空间就是定义了一个范围。

### 定义命名空间
命名空间的定义使用关键字 **namespace** ，后跟命名空间的名称，如下所示：

```cpp
namespace namespace_name {
   // 代码声明
}
```

为了调用带有命名空间的函数或变量，需要在前面加上命名空间的名称，如下所示：

```cpp
name::code;  // code 可以是变量或函数
```

### using 指令
您可以使用 **using namespace** 指令，这样在使用命名空间时就可以不用在前面加上命名空间的名称。这个指令会告诉编译器，后续的代码将使用指定的命名空间中的名称。

```cpp
#include <iostream>
using namespace std;

// 第一个命名空间
namespace first_space{
   void func(){
      cout << "Inside first_space" << endl;
   }
}
// 第二个命名空间
namespace second_space{
   void func(){
      cout << "Inside second_space" << endl;
   }
}

// 第三个命名空间
namespace third_space{
   void func()
   {
      cout << "Inside second_space" << endl;
   }
}

using namespace first_space;
using second_space::func;    //(与25行代码建议不能共存，因为都导入了func()函数)，（这种方法更好更推荐）

int main ()
{

   // 调用第一个命名空间中的函数
   func();
   // 调用第二个命名空间中的函数   
   func();         //(与32行代码不能共存)
   //调用第三个命名空间中的函数
   third_space::func();

   return 0;
}
```

### 嵌套的命名空间
命名空间可以嵌套，您可以在一个命名空间中定义另一个命名空间，如下所示：

```cpp
namespace namespace_name1 {
   // 代码声明
   namespace namespace_name2 {
      // 代码声明
   }
}
```

您可以通过使用 **::** 运算符来访问嵌套的命名空间中的成员：

```cpp
// 访问 namespace_name2 中的成员
using namespace namespace_name1::namespace_name2;

// 访问 namespace_name1 中的成员
using namespace namespace_name1;
```

## 联合体（共用体）
**共用体** 是一种特殊的数据类型，允许您在相同的内存位置存储不同的数据类型。您可以定义一个带有多成员的共用体，但是任何时候只能有一个成员带有值。共用体提供了一种使用相同的内存位置的有效方式。

### 定义共用体
为了定义共用体，您必须使用 **union** 语句，方式与定义结构类似。union 语句定义了一个新的数据类型，带有多个成员。union 语句的格式如下：

```cpp
union [union tag]
{
   member definition;
   member definition;
   ...
   member definition;
} [one or more union variables];
```
```cpp
//举例：
union Type_Name
{
   int i;
   float f;
   char str1[20];
   string str2;
} object_name;

//调用方式
object_name.i = 5;
object_name.f = 6.0f;
object_name.str2 = "你好！";
```

注意：共用体所占内存大小，按成员变量需占内存最大的来。

## typedef
C/C++ 提供了 **typedef** 关键字，您可以使用它来为类型取一个新的名字。下面的实例为单字节数字定义了一个术语 **BYTE** ：

```cpp
typedef unsigned char byte;
typedef unsigned char uint8_t;
typedef float fp32;
typedef double fp64;
```

## 结构体
### 定义结构体
结构体定义由关键字 **struct** 和结构体名组成，结构体名可以根据需要自行定义。

struct 语句定义了一个包含多个成员的新的数据类型，struct 语句的格式如下：

```cpp
struct type_name {
member_type1 member_name1;
member_type2 member_name2;
member_type3 member_name3;
.
.
} object_names;
```

### 访问结构体成员
下面是声明一个结构体类型 **Books** ，变量为 **book，** 为了访问结构的成员，我们使用 **成员访问运算符（.）** 。成员访问运算符是结构变量名称和我们要访问的结构成员之间的一个句号。

```cpp
struct Books
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
} book;

book.book_id = 1;
strcpy(book.title,"春天");
```

```cpp
typedef struct Books //Books可忽略不写
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
}Books_Rename;

struct Books book;   //第一种写法,不使用typedf必须加 struct 定义结构体
Books_Rename book;   //用typedef后的写法，9,11行不能共存，因为变量名相同。

book.book_id = 1;0
strcpy(book.title,"春天");
```

### 指向结构的指针
您可以定义指向结构的指针，方式与定义指向其他类型变量的指针相似，如下所示：

```cpp
struct Books *struct_pointer;
```

现在，您可以在上述定义的指针变量中存储结构变量的地址。为了查找结构变量的地址，请把 & 运算符放在结构名称的前面，如下所示：

```cpp
struct_pointer = &Book1;
```

为了使用指向该结构的指针访问结构的成员，您必须使用 -> 运算符，如下所示：

```cpp
struct_pointer->title;
(*struct_pointer).title;
```

### 结构体与函数
1.  作为函数参数

```cpp
struct Books
{
    ···
    int   book_id;
} book;

void fun1(Books book){
}
//通常使用指针传递
void fun2(Books *book){
}
```

2.  作为函数返回类型

### C++中的结构体
1.  定义 与上述定义一致，不同的是，在 C++ 中即使不使用 typedef struct 来定义结构体，定义结构体变量时也无需在变量前加 struct

```cpp
//定义结构体
struct Books
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
};
//定义结构体变量
book book1;
```

2.  面向对象 `struct` 默认的成员和继承是 `public`

```cpp
struct Books
{
  public://默认
    string title;
    string author;
    string subject;
    int    book_id;
  //构造函数
   Books(string t,string author,string subject,int id){
        title = t;
        author = author;
        subject = subject;
        book_id = id;
   }
   //析构函数
   ~Books(){
    ...
   }
  //成员函数
  void fun1(){
   ...
  }
   private:
   protected:
};
//初始化结构体
Books book1("title","author","subject",1);
```

## 数据的存放
1.  数据单位及其转换

    1.  **1字节byte = 8比特bit = 8位二进制**

    2.  数据是以 **二进制** 的形式存在电脑内存中的

    3.  进制转换

        1.  在C语言中，0b开头的数据代表2进制，0x开头的数代表16进制。因为2进制较长，所以我们在代码中，一般把2进制转化成16进制进行表示。

    4.  数据的原码

        1.  无符号(unsigned)的数据类型: 无符号数据所占的所有2进制都表示数据的大小，假设占8位2进制，那么该数最大是255（其对应的2进制:1111 1111)，最小是0（对应2进制是0000 0000)。

        2.  有符号（signed)的数据类型:有符号的数据其最高位二进制是符号位，符号位是0则是正数，符号位是1则为负数。假设占8位2进制，去掉1位2进制，还剩7位2进制来表示数据的大小。那么其最大是127（其2进制是0111 1111)，最小是-128（其2进制是1000 0000，-127的是1111 1111)，这个-128其实是用了-0的二进制表示，因为咱们有0（0000 0000)了，所以-0（理论是1000 0000)就没有啥意义了，所以我们规定把-0的二进制表示成-128。

    5.  数据的命名 :

        1.  由于unsigned char，int等数据类型名称无法突出 数据的二进制 有无符号，占多少二进制，所以我们用typedef给其起新的别名（比如 新别名 uint8\_t，int32\_t)

            1.  uint8\_t 中的u代表unsigned无符号，8代表占8位二进制，也就是1个字节。

            2.  int32\_t前面不带u，所以是有符号的，占32位二进制，所以也就是占4个字节，所以是int类型的别名

            3.  因为char 占 1 字节，8 比特，8 位二进制，而且有符号，所以我们叫他int8\_t

            4.  因为unsigned short int 无符号，占 2字节，16比特，16位2进制，所以叫uint16\_t，其他的类似。

            5.  float占4字节，32位二进制，所以叫fp32

            6.  double占8字节，64位二进制，所以叫fp64

        2.  具体的代码（在C++中自带一部分别名，可以不写一部分别名，但建议写):

            1.  typedef unsigned char uint8\_t;

            2.  typedef short int int16\_t;

            3.  typedef unsigned int uint32\_t;

            4.  typedef float fp32;

        ![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image3.webp)

2.  数据解析:

    1.  应用场景介绍:一般传感器会一直给我们的单片机、工控机发送数据，比如说发送我们机器人所走的路程（假设单位是毫米，是个短整型数)。

    2.  数据处理目标: 比如获取路程（短整型，2个字节，16位2进制)

    3.  传感器传数据: 一般的通信来说，传感器会一个字节一个字节的给单片机和工控机不断发数据，一般一个占n字节的物理量数据，要分成n个字节，也就是n个变量来发，又因为数据以2进制的形式存储，一般先发最高的8位二进制。

    4.  单片机、工控机接受到的数据: 比如获取机器人的路程，该数据占2字节，所以传感器要分成两个变量来发送。先接收到的变量我们叫DH（数据的高8位2进制)，后接收到的数据我们叫DL（数据的低8位2进制)。

    5.  数据处理思路: 我们现在拥有的是两个普通的8位二进制变量（uint8\_t类型的)，我们想要获得带符号的16位2进制的数据，机器人的路程（int16\_t类型的)。

        1.  位操作 : 假如说，DH是0x9D（对应2进制是1001 1101)，DL是0x57（对应2进制是0101 0111)。那么我们想要的16位数据就是把DH当16位中的最高的8位，DL当最低的8位，也就是我们想要的数据DATA（其二进制是 1001 1101 0101 0111)。我们想实现这个效果，就需要位操作，把DH向左移8位，让他变成1001 1101 0000 0000，然后再把左移后的DH 与 DL进行按位或运算，也就是1001 1101 0000 0000 与 0000 0000 0101 0111进行按位或，最后就得出来DATA是1001 1101 0101 0111了。

        2.  强转类型: 我们上面已经得出来data的2进制了，再进行强转化成有符号的数，也就是把最高位中的1变成符号位，具体代码是int16\_t DATA = （int16\_t) （（uint16\_t)DH << 8 | DL )，这样就处理成功了。

3.  数据在内存中的二进制表示方式

    1.  计算机存储数据的前置知识

        1.  计算机底层存储数据时使用的是二进制数字，但是计算机在存储一个数字时并不是直接存储该数字对应的二进制数字，而是存储该数字对应二进制数字的 **补码** 。

        2.  机器码：一个数在计算机的存储形式是二进制数，我们称这些二进制数为机器数，机器数是有符号，在计算机中用机器数的次最高位的左侧存放符号位，0表示正数，1表示负数

        3.  真值：因为机器数带有符号位，所以机器数的形式值不等于其真实表示的值（真值），以机器数1000 0001为例，其真正表示的值（首位为符号位）为-1，而形式值（首位就是代表1）为129； **因此将带符号的机器数的真正表示的值称为机器数的真值** 。

    2.  **原码、反码、补码：**

        1.  **原码** 的表示与机器数真值表示的一样，即用第一位表示符号，其余位表示数值。也就是

            正数：就是它对应的二进制数。 负数：将绝对值对应的二进制最左边位变为1。

        ```Plain Text
        【+1】= 原：[ 0000 0001 ]
        【-1】= 原：[ 1000 0001 ]
        ```
        4.  **反码：**

            正数 : 和原码相同。 负数 : 在其原码的基础上，符号位不变，其余各位取反。

        ```Java
        【+1】= 原： [ 0000 0001 ] = 反：[ 0000 0001 ]
        【-1】= 原： [ 1000 0001 ] = 反：[ 1111 1110 ]
        ```
        7.  **补码** ：

            正数 : 补码是其原码本身。 负数 : 补码是在其原码的基础上，符号位不变，其余各位取反后加1（即在反码的基础上加1）

        ```Java
        【+1】= 原： [ 0000 0001 ] = 反：[ 0000 0001 ] = 补：[ 0000 0001 ]
        【-1】= 原： [ 1000 0001 ] = 反：[ 1111 1110 ] = 补：[ 1111 1111 ]
        ```

    3.  数据在计算机中的存储形式

        1.  计算机实际只存储补码，所以原码转换为补码的过程，也可以理解为数据存储到计算机内存中的过程：

        ![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image4.webp)

        3.  正数：在原、反、补码中，正数的表示是一模一样的

        4.  负数：负数的表示是不相同的，所以对于负数的补码来说，我们是不能直接用进制转换将其转换为十进制数值的，因为这样是得不到计算机真正存储的十进制数的，所以应该将其转换为原码后，再将转换得到的原码进行进制转换为十进制数（机器数包含符号位）

        5.  为什么要诞生原码、反码和补码？

            1.  原因：对于只有正数的加减运算来说，用原码计算是没有任何问题的，但是当既有正数，也有负数的时候，计算机无法判断最高位是否是符号位。

    4.  总结：

        1.  **二进制的*****最高位*****是符号位：0表示正数，1表示负数**

        2.  正数的原码反码补码都一样，三码合一。

        3.  负数

            1.  负数的反码 = 它的原码符号位不变，其它位取反。

            2.  负数的补码 = 它的反码 + 1， 负数的反码 = 负数的补码 - 1 。

        4.  0 的反码、补码都是 0 。

        5.  **在计算机运算的时候都是以 “补码” 的方式来运算的。**

        6.  运算结果是以 **原码** 形式展现的。

## 现代C++
"现代C++"（Modern C++）指的是自 C++11 标准以来，C++ 语言的一系列重要更新和改进。现代 C++ 的目标是使代码更加简洁、安全、高效，并增强语言的功能。C++11、C++14、C++17、C++20 、C++23以及未来的C++26 标准，都包含了这些特性。

由于现代C++体系太过于庞大，本章只讲非常常用的现代C++的功能，其他你认为有用的功能请自学，你也可以把你认为很有用的功能补充在本文。

### 自动类型推导
通过使用 `auto` 关键字，编译器可以推断变量的类型，从而减少显式类型声明的冗余。(类似于Python)

这常用于那种返回值类型很复杂很长的情况，在ROS2中常用。

简单示例：

```cpp
auto x = 42; // 自动推导为 int 类型
auto str = "Hello"; // 推导为 const char*
```

### std::print这是C/C++在C++23新添加的功能,（来自print头文件，类似 Python 的 print）。
运行速度最快 → 最慢：
std::print ≈ fmt::print > printf > std::cout（关闭同步） > std::cout（默认） > std::endl

1. 特点:
    因为它本质上是 C++ 标准把 {fmt} 库（fmtlib）正式吸收进来。

    fmt 库以极高性能著称，特性包括：

    ✔ 零开销抽象（zero overhead）

    模板 + 智能优化，生成的代码非常紧凑。

    ✔ 直接写到 OS buffer

    不像 cout 那样层层包装和同步机制。

    ✔ 没有 printf 的可变参数开销

    printf 要解析格式字符串，是运行时开销；
    std::print 在编译期做了大量优化。

    ✔ 无需同步 C stdio

    和 cout 不同，它默认不做同步。

    ✔ 类型安全且格式强大

    性能比 printf 更高，但无格式化字符串崩溃风险。

2. 使用方法
   1. 打印一串字符串
   std::print 会直接将字符串输出到标准输出（stdout），不会自动换行。
    ```cpp
    #include <print>
    int main(int argc,char **argv)
    {
        std::print("Hello, world!");
        return 0;
    }
    ```
   2. 打印并自动换行：std::println
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        std::println(std::cout, "HelloWorld!");
    }
   ```
   3. 打印一个变量
   {} 是占位符，会被变量自动格式化。
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int value = 42;
        std::print("value = {}", value);
    }
    ```
    4. 打印多个变量
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int a = 10;
        float b = 3.14f;
        std::print("a = {}, b = {}", a, b);
    }
   ```
    5. 控制小数点输出
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        double pi = 3.1415926;
        std::print("pi = {:.2f}", pi);  // 输出：pi = 3.14
    }
   ```
   6. 对齐方式（左对齐 / 右对齐 / 居中）
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        std::print("[{:>10}]", 42);  // 右对齐
        std::print("[{:<10}]", 42);  // 左对齐
        std::print("[{:^10}]", 42);  // 居中
    }
   ```
   7. 打印整数为十六进制
   会加上 0x 前缀。
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int x = 255;
        std::print("hex = {:#x}", x);   // 输出：hex = 0xff
    }
   ```

### modules模块modules始于C++20，在C++26有了较大的变化。
扩展名为.cppm(社区常用)或.ixx(微软推广)
https://learn.microsoft.com/zh-cn/cpp/cpp/modules-cpp?view=msvc-170

### 安全空指针
`nullptr` 是一个类型安全的空指针常量，代替传统的 `NULL`，避免了空指针类型不匹配的问题。

主要是有的编译器不会把NULL识别成空指针，所以尽量不要使用`NULL`了，改用更加安全的`nullptr`。

```cpp
int* ptr = nullptr;  // 更安全的空指针
```

### 智能指针
#### 内存管理
**new 和 delete 运算符**

下面是使用 new 运算符来为任意的数据类型动态分配内存的通用语法：

```cpp
new data-type;
```

例如，我们可以定义一个指向 double 类型的指针，然后请求内存，该内存在执行时被分配。我们可以按照下面的语句使用 **new** 运算符来完成这点：

```cpp
double* pvalue  = NULL; // 初始化为 null 的指针   
//double* pvalue  = nullptr;   //更加安全
pvalue  = new double;   // 为变量请求内存

delete pvalue;        // 释放 pvalue 所指向的内存，如果用完不释放，则会内存泄漏
```

在某些情况下编译器会把NULL和int搞混，

所以用nullptr更为安全。（现代C++）

```cpp
#include <iostream>
using namespace std;

int main ()
{
   double* pvalue  = nulltr; // 初始化为 null 的指针
   pvalue  = new double;   // 为变量请求内存

   *pvalue = 29494.99;     // 在分配的地址存储值
   cout << "Value of pvalue : " << *pvalue << endl;

   delete pvalue;         // 释放内存，否则内存泄漏

   return 0;
}
```

#### 智能指针的类型
参考资料：https://learn.microsoft.com/zh-cn/cpp/cpp/smart-pointers-modern-cpp?view=msvc-170&source=recommendations

智能指针是在 [<memory>](https://learn.microsoft.com/zh-cn/cpp/standard-library/memory?view=msvc-170) 头文件中的 `std` 命名空间中定义的。

*   `unique_ptr` 只允许基础指针的一个所有者。 除非你确信需要 `shared_ptr`，否则请将该指针用作 POCO 的默认选项。 可以移到新所有者，但不会复制或共享。 替换已弃用的 `auto_ptr`。 与 `boost::scoped_ptr` 比较。 `unique_ptr` 小巧高效；大小等同于一个指针且支持 rvalue 引用，从而可实现快速插入和对 C++ 标准库集合的检索。 头文件：`<memory>`。 有关详细信息，请参阅[如何：创建和使用 unique\_ptr 实例](https://learn.microsoft.com/zh-cn/cpp/cpp/how-to-create-and-use-unique-ptr-instances?view=msvc-170)和 [unique\_ptr 类](https://learn.microsoft.com/zh-cn/cpp/standard-library/unique-ptr-class?view=msvc-170)。

*   `shared_ptr` 采用引用计数的智能指针。 如果你想要将一个原始指针分配给多个所有者（例如，从容器返回了指针副本又想保留原始指针时），请使用该指针。 直至所有 `shared_ptr` 所有者超出了范围或放弃所有权，才会删除原始指针。 大小为两个指针；一个用于对象，另一个用于包含引用计数的共享控制块。 头文件：`<memory>`。 有关详细信息，请参阅[如何：创建和使用 shared\_ptr 实例](https://learn.microsoft.com/zh-cn/cpp/cpp/how-to-create-and-use-shared-ptr-instances?view=msvc-170)和 [shared\_ptr 类](https://learn.microsoft.com/zh-cn/cpp/standard-library/shared-ptr-class?view=msvc-170)。

*   `weak_ptr` 结合 `shared_ptr` 使用的特例智能指针。 `weak_ptr` 提供对一个或多个 `shared_ptr` 实例拥有的对象的访问，但不参与引用计数。 如果你想要观察某个对象但不需要其保持活动状态，请使用该实例。 在某些情况下，需要断开 `shared_ptr` 实例间的循环引用。 头文件：`<memory>`。 有关详细信息，请参阅[如何：创建和使用 weak\_ptr 实例](https://learn.microsoft.com/zh-cn/cpp/cpp/how-to-create-and-use-weak-ptr-instances?view=msvc-170)和 [weak\_ptr 类](https://learn.microsoft.com/zh-cn/cpp/standard-library/weak-ptr-class?view=msvc-170)。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image5.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image6.webp)

当没有指向该内存的，程序会自动释放内存。

#### 如何创建和使用 shared\_ptr 实例
第一次创建内存资源时，请使用 [make\_shared](https://learn.microsoft.com/zh-cn/cpp/standard-library/memory-functions?view=msvc-170#make_shared) 函数创建 `shared_ptr`。 `make_shared` 异常安全。 它使用同一调用为控制块和资源分配内存，这会减少构造开销。 如果不使用 `make_shared`，则必须先使用显式 `new` 表达式来创建对象，然后才能将其传递到 `shared_ptr` 构造函数。

```cpp
//记得引头文件
#include <memory>
//下行是微软官方例子
auto ptr_name = std::make_shared<template>();
//下方为实例
auto struct_pointer = std::make_shared<struct Books>();
struct_pointer->title;
(*struct_pointer).title;
```

### **右值引用和移动语义**
**右值引用（`&&`）** 和 **`std::move`** 使得对象的资源可以被移动而不是复制，提高了性能，尤其是在容器类和资源管理类中。

```cpp
std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1); // 移动 v1 的数据到 v2
```

### **范围 for 循环**
范围 for 循环（Range-based for loop）用于简化迭代容器的代码。

```cpp
std::vector<int> v = {1, 2, 3, 4};
for (int n : v) 
{
    std::cout << n << std::endl;
}
```

### Constexpr
`constexpr`是从C++11开始引入的新特性，在后续的好几个C++版本做出了增强：

| 标准版本 | constexpr 相关支持 |
|:---|:---|
| C++11 | 支持 constexpr 变量和简单函数 |
| C++14 | constexpr 函数支持更多语句和循环 |
| C++17 | 支持 constexpr lambda，更多扩展 |
| C++20 | consteval 和更强的 constexpr |

`const` 和 `constexpr` 虽然都表示“不可变”，但本质和用途有明显区别，给你一份简明对比：

| 方面 | const | constexpr |
|:---|:---|:---|
| 含义 | 对象不可修改 | 表示值或函数可以在编译期求值 |
| 变量修饰 | 变量值不可变，但不保证编译期计算 | 变量必须是编译期常量 |
| 函数修饰 | 不用于函数（只能修饰成员函数） | 函数可在编译期执行 |
| 初始化要求 | 可以在运行时初始化 | 必须在编译时初始化 |
| 用途 | 防止变量被修改 | 做编译期计算，提高效率 |
| 示例 | const int x = 5; | constexpr int x = 5; |
| 函数示例 | int f() const;（成员函数常量） | constexpr int f() { return 5; } |

`const` 强调“值不可改”，运行时或编译时都可用

`constexpr` 强调“编译期计算”，只能初始化为编译时常量，函数可做编译期计算

```cpp
#include <iostream>

constexpr int square(int n) 
{
    return n * n;
}

int main() 
{
    const int a = 10;  // a 是只读变量，但可以是运行时初始化
    int x = 5;
    const int b = x;   // 合法，只要 x 在运行时确定

    constexpr int c = 10;      // c 必须编译期确定
    // constexpr int d = x;    // 错误，x 不是编译期常量

    int val = 3;
    const int r1 = square(val);       // OK，但 val 运行时确定，square 运行时计算
    constexpr int r2 = square(5);     // OK，编译期计算

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

### Lambda 表达式
#### 定义
Lambda表达式，很多人把他叫做 **匿名内联函数** ，但是实际上他并不是一个函数，而是一个无名的非联合非聚合的类的类型。

Lambda 表达式会让代码更加的简洁，使得 C++ 支持匿名函数对象，可以在函数内定义小的函数并将其传递给其他函数。

ROS2的一些库使用了该表达式,所以需要学习一下。特别是在回调函数中常用。

```cpp
auto add = [](int a, int b)  -> int
{ 
    return a + b; 
};
std::cout << add(2, 3) << std::endl; // 输出 5
```

核心语法 `[捕获列表] (参数) -> 返回值 { 函数体 }`

*   `[]` 捕获列表：决定 lambda 如何拿外部变量

*   `(参数)` 参数列表，和普通函数一样

*   `->` 返回类型可省略（编译器推断）

*   `{}` 函数体

| Lambda 捕获规则一览（最常用的 90 % 都在这里） |
|:---|
| 语法 | 含义 | 典型场景 | 注意点 |
| [] | 不捕获任何外部变量 | 纯算法、纯回调不需要用到外围数据 | 只用参数和局部变量 |
| [=] | 按值捕获所有在 lambda 里用到的外部变量 | 把外部快照拷一份进来 | 捕获的是拷贝，修改不会影响外部 |
| [&] | 按引用捕获所有在 lambda 里用到的外部变量 | 需要回调里修改外部状态 | 外部对象要保证比 lambda 活得久 |
| [x] | 按值捕获指定变量 x | 只需用到少量变量 | 一旦拷贝，外部后续变化对 lambda 不可见 |
| [&x] | 按引用捕获指定变量 x | 只想改某个变量 | 同 [&] 的生命周期警告 |
| [x,&y] | 按值捕获指定变量 x，按引用捕获指定变量 y | 需要用到x,想改变y | 综合上方两条 |
| [this] | 捕获当前对象指针 | 类成员函数里想访问成员变量 | 其实等价于 [&] 捕获 this |
| [=, &foo] | 混合捕获：其他按值，foo 按引用 | 细粒度控制 | 仅C++17,次序：先默认再特指 |

#### 基础用法
直接把num1与num2当参数传入，此时捕获列表不用任何东西：

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;

    auto add = [](int a,int b)
    {
        return a + b;
    };

    std::cout << add(num1,num2) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image7.webp)

使用捕获列表传入变量：

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;
    int num3 = 1;

    auto add = [num1,&num2](int c)
    {
        num2 = 3;
        return num1 + num2 + c;
    };

    std::cout << add(num3) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

答案是9

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image8.webp)

接下来是讲捕获this：

在 **类的成员函数中** ，你可能希望 lambda 能访问或修改成员变量。这时，你要捕获当前对象的指针 `this`。

```cpp
#include <iostream>

class MyClass 
{
    public:
        int num2 = 2;

        void do_add() 
        {
            int num1 = 10;
            int num3 = 6;

            // 在类的成员函数中访问自己的成员变量
            auto add = [num1, this](int c) 
            {
                num2 = 3;  // 访问 this->num2
                return num1 + num2 + c;
            };

            std::cout << add(num3) << std::endl;
        }
};

int main() {
    MyClass obj;
    obj.do_add();

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

#### 高级用法
| 特性 | C++11 | C++14 | C++17 |
|:---|:---|:---|:---|
| 捕获变量 | ✅ | ✅ | ✅（支持组合、更灵活） |
| mutable | ✅ | ✅（配合 auto 更实用） | ✅ |
| 泛型参数 auto | ❌ | ✅ | ✅ |
| 捕获 *this | ❌ | ❌ | ✅（值拷贝 this） |
| constexpr λ | ❌ | ❌ | ✅ |
| 嵌套递归 lambda | 复杂 | 稍好（配合 std::function） | 更方便 |

1.  mutable

如果我就想修改按值捕获的变量怎么办？

C++11提供了mutable，可以让你在Lambda里直接修改按值捕获的变量。

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;
    int num3 = 1;

    auto add = [num1,&num2](int c)mutable
    {
        num1 = 6;
        num2 = 3;
        return num1 + num2 + c;
    };

    std::cout << add(num3) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

答案是10

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image9.webp)

2.  泛型 lambda 参数

C++14提供了泛型 lambda 参数，不再需要模板函数了，lambda 直接支持“模板参数”！

```cpp
#include <iostream>

int main(int argc,char **argv)
{

    auto add = [](auto a,auto b)
    {
        return a + b;
    };

    std::cout << add(1,7) << std::endl;
    std::cout << add(1.1,1.7) << std::endl;
    std::cout << add(std::string("hi "), std::string("vinci")) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image10.webp)

3.  捕获\*this

| 捕获方式 | 意思 | 会捕获什么 | 生命周期影响 |
|:---|:---|:---|:---|
| [this] | 捕获当前对象的指针 | 指针 this | 外部对象必须活着，否则悬空 |
| [*this] | 捕获当前对象的副本 | 整个对象（值拷贝） | 安全，跟 lambda 活得一样久 |

如果只是\[this\]，`this` 是一个指针，指向当前对象，这个 lambda 只是记住了这个地址（指针），如果这个 lambda 比对象生命周期长，就悬挂指针❌。

如果是\[\*this\]，直接把整个对象拷贝进 lambda，不依赖外部对象是否还活着，lambda比对象活得久也没关系这个 lambda 可以安全延迟执行、在线程中运行等。

`[this]` 就像是说：“我知道对象在哪，我只存地址” 🧷

`[*this]` 是：“我整个对象拷一份，我自己留着用” 📦

```cpp
#include <iostream>

class A {
public:
    int x = 100;

    auto get_this_lambda() 
    {
        return [this]() { std::cout << "this: " << x << "\n"; };
    }

    auto get_star_this_lambda() 
    {
        return [*this]() { std::cout << "*this: " << x << "\n"; };
    }
};

int main() 
{
    A a;
    auto lam1 = a.get_this_lambda();
    auto lam2 = a.get_star_this_lambda();

    lam1();  // 输出 100
    lam2();  // 输出 100

    a.x = 200;

    lam1();  // 输出 200 （引用原对象）
    lam2();  // 还是 100 （因为是副本）

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

我如果想使用\*this,还想修改副本里的数值怎么办呢？那么就得加一个mutable了。

```cpp
#include <iostream>

class MyClass 
{
    public:
        int num2 = 2;

        auto get_add_lambda() 
        {
            int num1 = 10;

            // 在类的成员函数中访问自己的成员变量
            return [num1, *this](int c)mutable
            {
                num2 = 3;  //用mutable才能修改副本的成员变量num2
                return num1 + num2 + c;
            };
        }
};

int main() 
{
    // 用 new 动态创建 obj
    MyClass* obj = new MyClass();

    // 从堆上的 obj 获取 lambda
    auto add_func = obj->get_add_lambda();

    // 手动释放 obj（原对象没了）
    delete obj;

    // lambda 里访问的是副本，安全
    std::cout << add_func(6) << std::endl;  // 输出 10 + 3 + 6 = 19

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

4.  constexpr编译期lambda

```cpp
#include <iostream>

int main() 
{
    // C++17 constexpr lambda，捕获为空
    constexpr auto square = [](int x) constexpr 
    {
        return x * x;
    };

    constexpr int val = square(5);  // 编译期计算
    std::cout << val << std::endl; // 输出 25

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image11.webp)

上面`constexpr auto square = [](int x) constexpr` 有俩`constexpr`，第一个是用来声明`square`的，第二个是用来声明Lambda的，作用不一样，而且这俩都必须要。

要在这句里实现完整的 **编译期调用 + 编译期结果存储** ，你必须要：

*   lambda 自己是 `constexpr` 的（可以编译期执行）；

*   返回值是编译期常量（用 `constexpr` 声明 `square` 变量）；

否则：

*   如果你只写 lambda 是 `constexpr`，但没有 `constexpr auto square =`，`square` 变量还是运行期的；

*   如果你只写 `constexpr auto square =`，但 lambda 不能编译期执行，那编译会报错。

### std::bind
#### 定义
`std::bind` 是 C++11 引入的一个函数适配器，可以用来 **绑定函数的参数、重排参数顺序、生成可调用对象（函数对象）** ，常用在需要把函数“提前部分绑定参数”或“适配成某种形式”的场景。

```cpp
#include <functional>

auto new_func = std::bind(函数名, 参数1, 参数2, ...);
```

参数可以是具体值，表示提前绑定；

也可以是 `_1`, `_2` 等占位符，表示“调用时再传入”；

`std::placeholders::_1`, `_2` 表示调用时提供的第一个、第二个参数…

#### 基础用法
1.  绑定普通函数的部分参数

```cpp
#include <iostream>
#include <functional>

void add(int a, int b,int c) 
{
    std::cout << a + b + c << std::endl;
}

int main() 
{   
    //把第一个参数固定为 10，第二、三个等调用时再传
    auto func1 = std::bind(add,10,std::placeholders::_1,std::placeholders::_2);
    func1(20,30);   //10+20+30

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image12.webp)

2.  绑定成员函数

注意💡：函数出现了第0个参数（也可看作第一个参数，反正就是多一个默认的参数），这个参数是对象的指针，这是因为成员函数需要一个this。不知道为何要这么做的话，请你好好学学class,先打好基础。

```cpp
#include <iostream>
#include <functional>

class MyClass
{
    public:
        int add(int a, int b,int c) 
        {
            return a+b+c;
        }
};

int main() 
{   
    MyClass obj;
    //函数出现了第0个参数，这个参数是对象的指针，这是因为成员函数需要一个this。
    //把第一个参数固定为 10，第二、三个等调用时再传
    auto func1 = std::bind(&MyClass::add,&obj,10,std::placeholders::_1,std::placeholders::_2);
    int result = func1(20,30);   //10+20+30
    std::cout << "result:" << result << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image13.webp)

3.  用于高阶函数

高阶函数其实就是把函数当作入口参数或者返回值，不懂的话，本文档也有讲，可以先去看看。（或者学完高阶函数再回来学这个）

比如你要把类的普通成员函数作为入口参数给某个函数，普通函数指针不支持，但 `std::bind` 就可以：

```cpp
#include <iostream>
#include <functional>

void DoSomething(std::function<int(int,int)> func)
{
    std::cout << func(20,30) << std::endl;
}

class MyClass
{
    public:
        int add(int a, int b, int c) 
        {
            return a+b+c;
        }
};

int main() 
{   
    MyClass obj;

    // 绑定 a = 10，b = _1，c = _2
    DoSomething(std::bind(&MyClass::add,&obj,10,std::placeholders::_1,std::placeholders::_2));
scr
    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image14.webp)

### 高阶函数
高阶函数并非现代C++的东西，他其实是C语言就拥有的东西，但是现代C++提供了`std::function` 更灵活，能接收函数指针、Lambda、函数对象等。

从这里开始，你将接触函数式编程。

定义：在 C/C++ 中，一个函数如果符合以下任意条件，就可以称为“高阶函数”：

1.  **接受函数作为参数**

2.  **返回一个函数**

#### 函数指针
##### 什么是函数指针
函数指针是指向函数的指针，它可以指向 **具有特定参数和返回值类型** 的函数。

以下是**最基础的用法**

可以把他类比于普通的变量指针来看

```cpp
#include <iostream>

int add(int a, int b) 
{
    return a + b;
}

int main(int argc,char **argv) 
{
    // 函数指针声明
    int (*func_ptr)(int, int);  // 指向返回int、接收两个int参数的函数的指针

    // 将函数地址赋给函数指针
    func_ptr = add;

    int result = func_ptr(3, 4);  // 等同于 add(3, 4)
    std::cout << result << std::endl;  // 输出 7

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image15.webp)

**把****函数指针****作为参数传给函数**

普通变量声明是`int x`,函数声明`int (*op)(int, int)`

```cpp
#include <iostream>

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}

// 高阶函数：接收函数指针作为参数
void calc(int x, int y, int (*op)(int, int)) 
{
    int result = op(x, y);
    std::cout << "Result: " << result << std::endl;
}

int main(int argc,char **argv)
{
    calc(5,3,add);
    calc(5,3,sub);

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image16.webp)

**函数指针****作为返回值**

```cpp
#include <iostream>

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}

// 高阶函数：接收函数指针作为参数
int (*choose(bool compute))(int, int) 
{
    return compute ? add : sub;
}

int main(int argc,char **argv)
{
    //下面这个auto op等同于int (*op)(int, int)  = choose(true);
    auto op = choose(true);

    std::cout << op(4, 2) << std::endl;  // 输出 6

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image17.webp)

##### 类里的函数
如果函数是**静态****成员函数** ，地址在编译期确定，可直接用普通函数指针传递。

如果不是静态成员函数，你就需要用到`std::function`了，下面先给你看用函数指针的情况。

```cpp
#include <iostream>

class MyClass 
{
public:
    static void staticCallback(int value) 
    {  
        // 静态成员函数
        std::cout << "Static callback called: " << value << std::endl;
    }
};

// 高阶函数接受函数指针参数
void doSomething(void (*callback)(int)) 
{
    callback(42);  // 触发回调
}

int main(int argc,char **argv)
{
    doSomething(MyClass::staticCallback);

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

##### 回调函数Callback
回调函数其实就是一个要被某个函数当参数传入的函数，但是他往往是因为某个事件发生而被调用的，并非直接被你从main函数里用函数调用。

**语法层面没有本质区别** ：都是把函数指针或 `std::function` 作为参数传递。

**语义和场景不同** ：

*   “普通函数当参数” → 是你要去主动调用。

*   “回调函数” → 是你把函数给别人，别人 **“事件发生”时再调用你** 。回调函数就像 **中断服务函数** ，你提前设好地址， **等着被调用** 。

| 对比项 | 普通函数参数 | 回调函数 |
|:---|:---|:---|
| 💡 本质 | 函数指针 | 函数指针 |
| 🏷️ 意图 | 你传的函数是被调用方的一部分，你主动调用它 | 你传的函数是供别人调用的，你被动等待它被调用（回调） |
| 🧠 主动/被动 | 主动调用 | 被动被调用 |
| 🔁 场景 | 自己用来调用 | 别人帮你调用，比如事件发生、完成通知等 |
| 📦 示例 | 排序函数传入比较器 | 注册鼠标点击事件处理函数、串口接收回调等 |

正常的普通函数，我们是直接在main函数直接写`doSomething(function)`，用某个函数直接调用`function`这个函数。

```cpp
int main()
{
   doSomething(function);
}

void function()
{

}
```

但回调函数是需要事件触发的，并不是自己在main里调用的。

以下均是伪代码，不能直接运行，直接举了个例子：

比如说在STM32单片机（控制组常用）中，我们在main函数里并没有直接写`doSomething(TIM_IRQHandler_Callback)`，回调函数是被事件触发被动调用的，而非我们在main函数里主动调用。

```cpp
int main() 
{
    //注册好这个定时器更新中断，每500ms中断一次（仅仅是初始化了一个定时器，并没有调用timer_callback）
    TIMER_NVIC_Init(500)；
    while(1) 
    { 
        干别的事;
    }
}

//定时器中断回调函数（早就注册过该函数的地址）
void TIM_IRQHandler_Callback() 
{
    每过500ms后，该函数被动触发！ // 系统帮你调用
}
```

比如说在ROS2中，我们在main函数里并没有直接写`doSomething(timer_callback)`，回调函数是被事件触发被动调用的，而非我们在main函数里主动调用。

```cpp
main() 
{
    //注册一个定时器事件，每500ms触发一次（仅仅是初始化了一个定时器，并没有调用timer_callback）
    timer_ = node->create_wall_timer(500ms, std::bind(&MyNode::timer_callback, this));
    rclcpp::spin(node);            // 持续跑
}

class MyNode
{
    void timer_callback() 
    {
        每过500ms后，该函数被动触发！ // 系统帮你调用
    }
}
```

#### std::function
##### 定义
他的定义式：

```cpp
template <typename Signature>
class std::function;
```

`std::function` 是一个可以存储任何“可调用对象”的类型安全的函数包装器。

你可以把它理解成一个 **通用函数指针** ，但它比传统函数指针更强大：

*   可以存储：普通函数、Lambda 表达式、函数对象（仿函数）、`std::bind` 的结果等。

*   支持复制、赋值等操作。

*   类型安全（模板参数确定函数签名）。

`std::function` 有开销（堆内存分配 + 虚函数机制）， **比直接用 lambda 慢一点** 。

如果你不需要复杂功能、只在局部使用， **prefer lambda over std::function** 。

| 特性 | 函数指针 | std::function |
|:---|:---|:---|
| 支持普通函数 | ✅ | ✅ |
| 支持 Lambda（捕获变量） | ❌ | ✅ |
| 支持类仿函数 | ❌（不能直接） | ✅ |
| 可复制赋值 | ✅（但受限） | ✅ |
| 类型安全性 | 一般 | 更强 |
| 灵活性 | 低 | 高 |

##### 基础用法
```cpp
#include <iostream>
#include <functional>

int add(int a, int b) 
{
    return a+b;
}

class MyClass 
{
public:
    int multiply(int a, int b) 
    { 
        return a * b; 
    }
};

int main() 
{   
    //传入函数
    std::function<int(int, int)> func1 = add;  // 把函数包进去
    std::cout << func1(3, 4) << std::endl;     // 调用 7

    //传入Lambda
    std::function<int(int, int)> func2 = [](int a, int b) 
    {
        return a - b;
    };
    std::cout << func2(3, 4) << std::endl;     // 调用  -1

    //传入类的普通成员变量
    MyClass obj;
    std::function<int(int, int)> func3 = std::bind(&MyClass::multiply, &obj, std::placeholders::_1, std::placeholders::_2);
    std::cout << func3(3, 4) << std::endl;    // 调用  12

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image18.webp)

### 并发编程
多线程在STM32和ROS2中是非常常用的东西，虽然STM32和ROS2中不会直接用到thread这个类，但是理解起来是差不多的，甚至ROS2里有些东西就是二次封装了C++多线程，所以我们这里直接介绍C++11的多线程。

在多线程数据共享中，STM32需要学习FreeRTOS的队列，信号量等等来解决这个问题，而ROS2中所使用的锁和C++11里的锁原理上是同一个东西，所以要重点学习多线程数据共享的部分。

由于现代C++体系太过于庞大，本章只讲非常常用的多线程操作，部分不常用的多线程操作请自学，你也可以把你认为很常用的功能补充在本文。

https://www.bilibili.com/video/BV1d841117SH

#### 线程
1.  **`std::thread(Callable &&f, Args &&args...)`**

**`Callable &&f`** 传入一个可调用对象（函数指针、函数对象、lambda 表达式、成员函数指针等）。

**`Args &&args...`** 传入 `f` 所需的参数（可变参数模板）。`std::thread` 会对这些参数进行 **值传递** ，并在新线程中执行 `f(std::forward<Args>(args)...)`。

如果线程函数没有入口参数，那么重载函数只需要第一个参数即可。

如果线程函数有入口参数，那么后面要把入口参数也写上。

2.  **`join()`**

**作用** ：等待线程执行完成，并回收线程资源。 **使用规则** ：

*   `join()` 只能在 **`joinable()` 为 `true`** 的线程上调用。

*   `join()` 之后，线程对象会变成 **不可 `joinable`** ，即 `joinable()` 返回 `false`。

`join()` 会 **阻塞主线程** ，直到 `task` 执行完毕。（如果不阻塞主线程（即main函数），主线程将会接着往下执行，如果下面直接就是return 0，主线程会直接结束整个程序，导致程序抛错）

线程执行完后，`join()` 释放线程资源，`t` 变为 **不可 `joinable`** 。

tips:阻塞主线程对于没学过STM32的同学可能比较难理解，实际上可以理解为，如果线程函数没执行完，main函数就一直运行join()，线程函数执行完毕，才会往join()下方的函数执行。

3.  **`joinable()`**

**作用** ：判断当前 `std::thread` 是否可以 `join()`。

**`joinable()` 返回 `true` 的情况**

*   线程已被创建，并且 **还没有 `join()` 或 `detach()`** 。

**`joinable()` 返回 `false` 的情况**

*   `std::thread` 对象默认构造，没有绑定任何线程。

*   线程已经 `join()` 过。

*   线程已经 `detach()` 过（变为后台线程）。

*   线程对象被 `std::move()` 走了。

下面有例子：

```cpp
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

void print_task1()
{
    cout << "Hello Task1!" << endl;
    return;
}

void print_task2(std::string msg)
{
    cout << msg << endl;
    return;
}

int main(int argc,char **argv) 
{
    std::string a = "Hello Task2!";

    std::thread t1(print_task1);
    std::thread t2(print_task2,a);

    if (t1.joinable()) 
    {
        t1.join();
    }

    if (t2.joinable()) 
    {
        t2.join();
    }

    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image19.webp)

4.  **`detach()`**

`detach()` 用于 **分离线程** ，让线程 **独立运行** ，主线程不会等待它完成。

*   **让线程独立运行** ，不受 `std::thread` 对象管理。

*   **主线程不再阻塞** ，可继续执行。

*   **线程执行完后，系统会自动回收资源** （但你无法控制具体何时回收）。

*   `detach()` 之后，线程变得不可 `joinable()`。

| 方法 | join() | detach() |
|:---|:---|:---|
| 作用 | 等待线程完成 | 让线程独立运行 |
| 是否阻塞主线程 | 是（等待线程完成） | 否（主线程继续执行） |
| 线程是否 joinable() | join() 后变 false | detach() 后变 false |
| 适用场景 | 需要等待线程完成 | 后台任务，不需要等待 |
| 风险 | 不能多次 join() | 可能访问已销毁的变量 |

```cpp
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

void print_task1()
{
    cout << "Hello Task1!" << endl;
    return;
}

int main(int argc,char **argv) 
{
    std::thread t1(print_task1);

    if (t1.joinable()) 
    {
        t1.detach();
    }

    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image20.webp)

#### 常见线程问题
1.  传递指针

```cpp
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

void print_task1(int * num)
{
    *num += 10;
    return;
}

int main(int argc,char **argv) 
{
    int a = 1;

    std::thread t1(print_task1,&a);

    if (t1.joinable()) 
    {
        t1.join();
    }
    cout << a << endl;

    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image21.webp)

2.  传递引用

```cpp
#include <functional>
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

void print_task1(int & num)
{
    num += 10;
    return;
}

int main(int argc,char **argv) 
{
    int a = 1;

    std::thread t1(print_task1,std::ref(a));

    if (t1.joinable()) 
    {
        t1.join();
    }
    cout << a << endl;

    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image22.webp)

3.  私有方法无法访问（用友元函数）

```cpp
#include <functional>
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

class C
{
    private:
        friend void function1();
        void print_task1(int & num)
        {
            num += 10;
            return;
        }

};

void function1()
{
    int a = 1;
    C obj;

    std::thread t1(&C::print_task1,&obj,std::ref(a));  //注意默认要传入类对象的地址

    if (t1.joinable()) 
    {
        t1.join();
    }
    cout << a << endl;
}

int main(int argc,char **argv) 
{
    function1();
    return 0;
}
```

#### 互斥量(解决数据共享问题)
对于没学过STM32和FreeRTOS的同学，可能不理解数据共享问题，下面举几个例子你就懂了：

1.  多个线程同时对一个变量进行写（会造成问题）

2.  一个线程对一个变量进行写的同时，另一个线程对这个变量进行读（会造成问题）

3.  多个线程同时对一个变量进行读（ **不会** 造成问题）

4.  ... ...更多情况

执行一下下方的代码：

```cpp
#include <cstdint>
#include <functional>
#include <iostream>
#include <ostream>
#include <string>
#include <thread>

using std::cout;
using std::cin;
using std::endl;

int a = 1;  // 共享变量

void print_task1(int & num)
{
    for (short i = 0; i < 1000; i++)
    {
        int temp = num;  // 读取 a 的值
        temp++;           // 增加 1
        num = temp;       // 写回 a 的值
    }
    return;
}

void print_task2(int & num)
{
    for (short i = 0; i < 1000; i++)
    {
        int temp = num;  // 读取 a 的值
        temp++;           // 增加 1
        num = temp;       // 写回 a 的值
    }
    return;
}

int main(int argc, char **argv)
{
    std::thread t1(print_task1, std::ref(a));
    std::thread t2(print_task2, std::ref(a));

    if (t1.joinable())
    {
        t1.join();
    }

    if (t2.joinable())
    {
        t2.join();
    }

    cout << a << endl;  // 打印 a 的最终值，可能不等于 2001

    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image23.webp)

会发现有几次结果不为2001，这说明发生了线程竞争。

如何解决呢？

我们可以使用mutex类对任务函数进行加锁，保证每个时刻只有一个线程对任务函数进行操作。这样就可以解决竞争问题。

1.  **手动加锁：****`std::mutex`**

`std::mutex` 是 C++11 引入的一个互斥量（mutex）类型，用于在多线程环境中保护共享资源，避免数据竞争（race conditions）。它提供了线程间的同步，确保同一时刻只有一个线程能够访问共享资源。

1.  **`mtx.lock()`** ：手动加锁操作，锁住互斥量。如果该互斥量已经被其他线程锁住，调用该函数的线程会被阻塞，直到其他线程释放锁。

2.  **`mtx.unlock()`** ：手动解锁操作，释放互斥量，允许其他线程获得锁。如果没有线程持有该锁，调用该函数是非法的，程序会崩溃。

2.  **自动加锁：****`std::lock_guard<std::mutex>`**

`std::lock_guard` 是一个模板类，它为 `std::mutex` 提供了 **RAII（资源获取即初始化）** 风格的管理。RAII 是 C++ 中的一种常见技术，意思是资源（如锁、文件句柄等）在对象生命周期开始时获取，在对象生命周期结束时释放。

**`std::lock_guard<std::mutex>` 的作用：**

*   **自动加锁和解锁** ：`std::lock_guard` 在创建时自动加锁，且在它超出**作用域**时会自动解锁，这样可以避免手动调用 `mtx.lock()` 和 `mtx.unlock()` 时的错误（如忘记解锁导致的死锁）。

*   **异常安全** ：如果在 `std::lock_guard` 的作用域内发生异常，`std::lock_guard` 会确保锁会被正确释放，从而避免死锁的发生。

*   **作用域** ：这个类对象在哪定义的，他的作用域就在哪个{}里，下面的例程的作用域就是for循环，更建议锁for循环，而不是整个函数（会让多线程更精细）。

3.  **自动加锁Plus：****`std::unique_lock<std::mutex>`**

它提供了比 `std::lock_guard<std::mutex>` 更灵活的锁管理机制，适用于那些需要在锁定期间进行复杂操作（如等待条件变量、手动解锁和重新加锁等）的场景。

它和`lock_guard`功能几乎一样，但是可以选择在作用域下用进行更加细致的加锁和解锁。（如果不需要更加精细的加锁和解锁，那么他的用法和`lock_guard`一样）（如果需要更加细致的加锁解锁，请看下方这段代码）

```cpp
void print_task1(int & num) 
{
    std::unique_lock<std::mutex> lock(mtx);  // 自动加锁
    // 在这里进行一些操作...
    num += 10;
    lock.unlock();  // 显式解锁
    // 做其他不需要锁的事情...

    lock.lock();  // 显式重新加锁
    num += 10;
    //即将超出作用域，自动解锁
}
```

4.  区别：上面这俩区别其实就和new与std::shared\_ptr区别一样，就是手动管理和自动管理的区别，当然要用自动管理更好啦！

我们加锁后，就不会出现线程竞争了。一般我们使用功能更加强大的**`std::unique_lock<std::mutex>`****。**

```cpp
#include <cstdint>
#include <functional>
#include <iostream>
#include <ostream>
#include <string>
#include <thread>
#include <mutex>  // 引入 mutex

using std::cout;
using std::cin;
using std::endl;

int a = 1;  // 共享变量
std::mutex mtx;  // 定义 mutex

void print_task1(int & num)
{
    for (short i = 0; i < 1000; i++)
    {
        std::lock_guard<std::mutex> lock(mtx);  // 自动加锁解锁,不会出现死锁现象(更推荐)
        int temp = num;  // 读取 a 的值
        temp++;           // 增加 1
        num = temp;       // 写回 a 的值
    }
    return;
}

void print_task2(int & num)
{
    for (short i = 0; i < 1000; i++)
    {
        mtx.lock();      // 手动加锁
        int temp = num;  // 读取 a 的值
        temp++;           // 增加 1
        num = temp;       // 写回 a 的值
        mtx.unlock();    // 手动解锁,如果不解锁，会造成死锁，导致后面无法访问数据
    }
    return;
}

int main(int argc, char **argv)
{
    std::thread t1(print_task1, std::ref(a));
    std::thread t2(print_task2, std::ref(a));

    if (t1.joinable())
    {
        t1.join();
    }

    if (t2.joinable())
    {
        t2.join();
    }

    cout << a << endl;  // 打印 a 的最终值，可能不等于 2001

    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image24.webp)
