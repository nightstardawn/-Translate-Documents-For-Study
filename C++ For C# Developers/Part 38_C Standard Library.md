# Part 38 – C Standard Library

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6359),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天，我们将开始探索 C++ 标准库。由于 C++ 主要是 C 的超集，因此 C++ 标准库主要是 C 标准库的超集。所以我们就从那里开始吧！

## Background  背景

首先，提醒一下：C标准库非常古老。其中大部分内容至少可以追溯到30年前，甚至一些较新的部分也有大约10年的历史，并且是为了与原始设计相匹配而构建的。
</br>C语言本身也非常简单。其缺乏的特性影响了库的设计。

例如，有一些“家族”函数执行相同的功能，但针对不同的数据类型。
</br>要获取浮点数的绝对值，我们调用`fabs`（对于`double`类型），`fabsf`（对于`float`类型），以及`fabsl`（对于long double类型）。
</br>在C++中，我们只需对`abs`进行重载，使用不同的参数类型，编译器就会选择正确的函数来调用。

C++ 标准库包括许多依赖于 C++ 语言功能的更多现代设计。
</br>例如，它具有 `abs` 重载函数。C 标准库包含在 C++ 标准库中，主要是作为 C++ 保持与 C 代码高度兼容性的广泛目标的一部分。
</br>它有几个部分本身确实有用，但这些部分很少而且相距甚远。

尽管如此，30+ 年的势头是一股强大的力量，即使有更现代的替代方案可用，看到 C 标准库被使用也是极其常见的。
</br>因此，了解许多 C++ 代码库将包含一些 C 标准库用法，这一点非常重要。

我们今天不打算深入介绍 C 标准库的每一个小角落，但我们将调查它的亮点。

## General Purpose  通用

至于组成，C++标准库由头文件组成。自C++20起，模块也可用。C标准库仅以头文件的形式提供。
</br>C标准库的头文件以`.h`扩展名命名：`math.h`。这些可以直接包含到C++文件中：`#include <math.h>`。
</br>它们也被C++标准库所封装。封装的版本以c开头并去掉.h扩展名，因此我们可以`#include <cmath>`。
</br>这些封装的头文件将所有内容放置在`std`命名空间中，也可能将所有内容放置在全局命名空间中，因此`std::fabs`和`::fabs`都可以使用。

在C标准库中有一个真正通用的头文件：`stdlib.h`/`cstdlib`。与像`math.h`/`cmath`这样的更专注于数学的头文件不同，这个头文件提供了各种实用工具。
</br>其中一些基本内容包括`size_t`，这是`sizeof`运算符求值的类型，以及`NULL`，这是一个在C++11中nullptr出现之前广泛使用的空指针常量。
</br>这个头文件的广泛性质使得很难与C#进行比较，但可以大致将其视为`System`命名空间：

```c++
#include <stdlib.h>
 
// sizeof()的结果是size_t
size_t intSize = sizeof(int);
DebugLog(intSize); // Maybe 4
 
// NULL可以用作指针来表示“null”
int* ptr = NULL;
 
// 在算术中容易发生误用
int sum = NULL + NULL;
 
//nullptr不是：这是一个编译器错误
int sum2 = nullptr + nullptr;
```

在C++引入`new`和`delete`运算符用于动态内存分配之前，C代码会使用`malloc`、`calloc`、`realloc`和`free`函数。
C#中`malloc`的等价函数是`Marshal.AllocHGlobal`，`realloc`是`Marshal.ReallocHGlobal`，而`free`是`Marshal.FreeHGlobal`：

> 这里译者简单的说明一下C#中的这三个函数：
> 1. Marshal.AllocHGlobal (等价于 malloc)
>    - `IntPtr pointer = Marshal.AllocHGlobal(sizeInBytes);`
>    - 功能：从进程的非托管内存中分配指定大小的内存块
>    - 参数：内存大小（字节数），可以是 int 或 IntPtr 
>    - 返回值：IntPtr 类型的指针，指向分配的内存地址
> 2. Marshal.ReAllocHGlobal (等价于 realloc)
>    - `IntPtr newPointer = Marshal.ReAllocHGlobal(oldPointer, newSizeInBytes);`
>    - 功能：重新分配之前分配的内存块的大小
>    - 参数：旧指针 新的内存大小（字节数）
>    - 返回值：IntPtr 类型的新指针，指向重新分配的内存地址
> 3. Marshal.FreeHGlobal (等价于 free)
>    - `Marshal.FreeHGlobal(pointer);`
>    - 功能：释放之前分配的非托管内存块
>    - 参数：指向要释放的内存块的 IntPtr 指针

```c++
// 分配1 KB未初始化的内存
// 失败时返回null
// 内存是无类型的，因此需要转换才能读取或写入
void* memory = malloc(1024);
 
// 在初始化之前读取它是未定义的行为
int firstInt = ((int*)memory)[0];
 
// 释放内存。不这样做会导致内存泄漏。
free(memory);
 
// 分配并初始化为全零的1KB内存：256 x 4字节
memory = calloc(256, 4);
 
// 重新分配之前分配的内存以获取更多或更少的内存
memory = realloc(memory, 2048);
 
// 也需要释放calloc和realloc分配的内存
free(memory);
```

有一些函数可以从字符串中解析数字，类似于 `int.Parse` 、 `float.Parse` 等：

```c++
// 解析一个双精度浮点数
double d = atof("3.14");
DebugLog(d); // 3.14
 
// 解析一个整数
int i = atoi("123");
DebugLog(i); // 123
 
// 解析一个浮点数并在字符串中获取其结束指针
const char* floatStr = "2.2 123.456";
char* pEnd;
float f = strtof(floatStr, &pEnd);
DebugLog(f); // 2.2
 
//使用结束指针来解析更多
f = strtof(pEnd, &pEnd);
DebugLog(f); // 123.456
```

提供了一些通用算法，类似于C#的`Array`类以及`Random`和`Math`：

```c++
// 初始化全局随机数生成器
// 这不是线程安全的
srand(123);
 
// 使用全局随机数生成器生成一个随机数
int r = rand();
DebugLog(r); // 可能为 440
 
// 比较整数的指针
auto compare = [](const void* a, const void* b) {
    return *(int*)a - *(int*)b;
};
 
// 对一个数组进行排序
int a[] = { 4, 2, 1, 3 };
qsort(a, 4, sizeof(int), compare);
DebugLog(a[0], a[1], a[2], a[3]); // 1, 2, 3, 4
 
// 在数组中二分查找2
int valToFind = 2;
int* pVal = (int*)bsearch(&valToFind, a, 4, sizeof(int), compare);
int index = pVal - a;
DebugLog(index); // 1
 
// 取绝对值
DebugLog(abs(-10)); // 10
 
// 除法并获取余数
// stdlib.h/cstdlib 也提供了 div_t 结构体类型
div_t d = div(11, 3);
DebugLog(d.quot, d.rem); // 3, 2
```

最后，还有一些与作系统相关的功能：

```c++
// 运行系统命令
int exitCode = system("ping example.com");
DebugLog(exitCode); // 0 if successful
 
//获取一个环境变量
char* path = getenv("PATH");
DebugLog(path); // 可执行文件路径
 
//注册一个函数（在这种情况下是lambda函数）以在程序退出时调用
atexit([]{ DebugLog("Exiting..."); });
 
//明确地使用退出码退出程序
exit(1); // Exiting...
```

## Math and Numbers  数学和数字

C标准库中的下一个类别头文件与数学相关。我们在整个系列中看到的一个是`stdint.h`/`cstdint`，它通过`typedef`提供整数类型。
在C#中，基本类型如`int`有保证的大小，但这个头文件做得更多，还定义了满足特定要求的类型：

```c++
#include <stdint.h>
 
int32_t i32; // 始终为有符号32位
int_fast32_t if32; // 至少32位的最快有符号整数类型
intptr_t ip; // 可以存储指针的有符号整数
int_least32_t il; // 最小的至少32位的有符号整数
intmax_t imax; // 最大的可用有符号整数
 
//32位整数的范围
DebugLog(INT32_MIN, INT32_MAX); // -2147483648, 2147483647
 
// 最大的size_t
DebugLog(SIZE_MAX); // 可能为 18446744073709551615
```

`stddef.h`/`cstddef` 中也有一些类型。其中一些是满足特定要求的更多类型。不同寻常的是，此标头的 `cstddef` 版本中还有特定于 C++ 的类型：

```c++
#include <cstddef>
 
// C and C++ types
std::max_align_t ma; // 使用最大对齐方式的类型
std::ptrdiff_t pd; // 足够大以容纳两个指针的相减结果
 
// C++-specific types
std::nullptr_t np = nullptr; // The type of nullptr
std::byte b; // An "enum class" version of a single byte
```






























