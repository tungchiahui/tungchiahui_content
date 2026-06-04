---
title: "输入输出"
---

## 输入输出

### 数据的输入

**作用：用于从键盘获取数据**

**关键字：**cin

**语法：** `cin >> 变量 `

示例：

```cpp
int main(){

	//整型输入
	int a = 0;
	cout << "请输入整型变量：" << endl;
	cin >> a;
	cout << a << endl;

	//浮点型输入
	double d = 0;
	cout << "请输入浮点型变量：" << endl;
	cin >> d;
	cout << d << endl;

	//字符型输入
	char ch = 0;
	cout << "请输入字符型变量：" << endl;
	cin >> ch;
	cout << ch << endl;

	//字符串型输入
	string str;
	cout << "请输入字符串型变量：" << endl;
	cin >> str;
	cout << str << endl;

	//布尔类型输入
	bool flag = true;
	cout << "请输入布尔型变量：" << endl;
	cin >> flag;
	cout << flag << endl;
	system("pause");
	return EXIT_SUCCESS;
}
```

## C 风格 I/O 与 C++ 流式 I/O

1.C语言的stdio.h中的scanf和printf

**int scanf(const char \*format, ...)** 函数从标准输入流 **stdin** 读取输入，并根据提供的 **format** 来浏览输入。

**int printf(const char \*format, ...)** 函数把输出写入到标准输出流 **stdout ，并根据提供的格式产生输出。

```cpp
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
