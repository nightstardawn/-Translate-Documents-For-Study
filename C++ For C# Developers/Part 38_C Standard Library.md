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

`limits.h`/`climits` 也有一些 `maximum` 和 `minimum` 宏，相当于 `int.MaxValue` 和 C# 中的类似值：

```c++
#include <limits.h>
 
// int 值的范围
DebugLog(INT_MIN, INT_MAX); // 可能为 -2147483648, 2147483647
 
// char值的范围
DebugLog(CHAR_MIN, CHAR_MAX); // 可能为 -128, 127
```

`inttypes.h`/`cinttypes` 标头还具有与整数相关的实用程序。
这些是必需的，因为字符串之间的转换并不像在 C# 中使用 `int.Parse` 等函数那样内置于语言中 。解析 ：

```c++
#include <inttypes.h>
 
// 将十六进制字符串解析为整型
// nullptr 表示我们不想获取到字符串末尾的指针
intmax_t i = strtoimax("f0a2", nullptr, 16);
DebugLog(i); // 61602
```

同样，float.h/cfloat 提供了一组浮点宏，类似于 C# 通过 `float.MaxValue` 等常量提供的宏：

```c++
#include <float.h>
 
// Biggest float
float f = FLT_MAX;
DebugLog(f); // 3.40282e+38
 
// 1.0与下一个更大的浮点数之间的差异
float ep = FLT_EPSILON;
DebugLog(ep); // 1.19209e-07
```

`fenv.h`/`cfenv` 为我们提供了对 CPU 如何处理浮点数的精细控制。在 C# 中没有真正的等效项：

```c++
#include <fenv.h>
 
// 清除CPU浮点异常。与C++异常不同。
feclearexcept(FE_ALL_EXCEPT);
 
// 除以零
// 使用volatile防止编译器移除此
volatile float n = 1.0f;
volatile float d = 0.0f;
volatile float q = n / d;
 
// 检查浮点异常以确定这是否是除以零或产生了不精确的结果
int divByZero = fetestexcept(FE_DIVBYZERO);
int inexact = fetestexcept(FE_INEXACT);
DebugLog(divByZero != 0); // true
DebugLog(inexact != 0); // false
 
// 清除浮点异常
feclearexcept(FE_ALL_EXCEPT);
 
// 执行一个无法精确表示商的除法
d = 10.0f;
q = n / d;
 
// 检查浮点异常
divByZero = fetestexcept(FE_DIVBYZERO);
inexact = fetestexcept(FE_INEXACT);
DebugLog(divByZero != 0); // false
DebugLog(inexact != 0); // true
```

## Strings and Arrays  字符串和数组

下一类标头处理字符串和数组 。让我们从 `string.h`/`cstring` 开始，它有很多作内置于 C# 中的 `string` 类、托管数组和 `Buffer` 中：

```c++
#include <string.h>
 
// 比较字符串：相等为0，小于为-1，大于为1
DebugLog(strcmp("hello", "hello")); // 0
DebugLog(strcmp("goodbye", "hello")); // -1
 
// 复制一个字符串
char buf[32];
strcpy(buf, "hello");
DebugLog(buf);
 
// 字符串连接
strcat(buf + 5, " world");
DebugLog(buf); // hello world
 
// 计算字符串中的字符数（其长度）
// 这将迭代直到找到NUL
DebugLog(strlen(buf)); // 11
 
// 获取字符串中字符首次出现的位置指针
DebugLog(strchr(buf, 'o')); // o world
 
// 获取字符串中字符串首次出现的位置指针
DebugLog(strstr(buf, "ll")); // llo world
 
// 获取字符串中下一个由分隔符分隔的“标记”的指针
char* next = strtok(buf, " ");
DebugLog(next); // hello
next = strtok(nullptr, ""); // null表示继续全局状态
DebugLog(next); // world
 
// 将 buf （“hel”） 的前三个字节复制到某字符串对应位置中的后面
//参数一: 目标地址
//参数二: 源地址
//参数三: 复制的字节数
memcpy(buf + 3, buf, 3);
DebugLog(buf); // helhelworld
 
// 将buf中的所有字节设置为65，并在末尾放置一个空字符
memset(buf, 65, 31);
buf[31] = 0;
DebugLog(buf); // AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

`wchar.h`/`cwchar` 等效于“宽”字符。`System.Text` 命名空间提供了对 C# 中各种字符类型的支持，该命名空间的功能与此标头中的功能类似：

```c++
#include <wchar.h>
 
wchar_t input[] = L"foo,bar,baz";
 
// 获取第一个标记，使用 'state' 来保存标记化状态
wchar_t* state;
wchar_t* token = wcstok(input, L",", &state);
DebugLog(token); // foo
 
// 获取第二个令牌
token = wcstok(nullptr, L",", &state);
DebugLog(token); // bar
 
// 获取第三个令牌
token = wcstok(nullptr, L",", &state);
DebugLog(token); // baz
```

`ctype.h`/`cctype` 具有仅与字符相关的函数。C 缺少 `bool` 类型意味着使用 `0` 而不是 `false`，使用`非 0` 而不是 `true`。
</br>C# 本身不使用 ASCII，因此这可以通过 ASCIIEncoding 进行近似计算：

```c++
#include <ctype.h>
 
// 检查字母字符
DebugLog(isalpha('a') != 0); // true
DebugLog(isalpha('9') != 0); // false
 
// 检查数字字符
DebugLog(isdigit('a') != 0); // false
DebugLog(isdigit('9') != 0); // true
 
//转换为大写
DebugLog(toupper('a')); // A
```

`wctype.h`/`cwctype` 等效于“宽”字符。其中很多内容都内置在 C# 的 `char` 类型中：

```c++
#include <wctype.h>
 
// 检查字母字符
DebugLog(iswalpha(L'a') != 0); // true
DebugLog(iswalpha(L'9') != 0); // false
 
// 检查数字字符
DebugLog(iswdigit(L'a') != 0); // false
DebugLog(iswdigit(L'9') != 0); // true
 
// 转换为大写
DebugLog(towupper(L'a')); // A
```

`uchar.h`/`cuchar` 具有字符转换函数。C# 的`System.Text` 命名空间中的“编码”类在 `.NET` 中提供这些转换：

```c++
// Convert to UTF-16
char input[] = "A";
char16_t output;
mbstate_t state{};
size_t len = mbrtoc16(&output, input, MB_CUR_MAX, &state);
DebugLog(len); // 1
uint8_t* outputBytes = (uint8_t*)&output;
DebugLog(outputBytes[0], outputBytes[1]); // 65, 0
```

## Language Tools  语言工具

此类别的头文件包括一系列工具，这些工具不属于 C 或 C++ 语言，但与 C 或 C++ 语言密切相关，或者将内置到其他语言中。

首先是 `stdarg.h`/`cstdarg`。此标头包含实现可变参数函数(Part4)所需的类型和宏。
这些模板在 C++ 中不常用，因为可变参数模板可用、更易于使用且类型安全。
在 C# 中，我们将使用 `params` 关键字让编译器在调用站点生成一个托管的参数数组。以下是使用 va_ 宏实现可变参数函数的方法：

```c++
#include <stdarg.h>
 
// “...”表示一个可变参数函数
void PrintLogs(int count, ...)
{
    // va_list持有状态
    va_list args;
 
    // 使用 “va_start” 宏开始获取所有参数
    va_start(args, count);
 
    for (int i = 0; i < count; ++i)
    {
        // 使用“va_arg”宏来获取下一个参数
        const char* log = va_arg(args, const char*);
        DebugLog(log);
    }
 
    // 使用“va_end”宏来停止获取参数
    va_end(args);
}
 
// 调用可变参数函数
PrintLogs(3, "foo", "bar", "baz"); // foo, bar, baz
```

接下来是包含 `assert` 宏的 `assert.h`/`cassert`。
</br>如果定义了 `NDEBUG` 预处理器符号，则会检查条件是否为 `false`，并调用 `std：：abort` 来结束程序，可能会执行其他调试步骤 例如中断交互式调试器。
</br>如果条件为 `true`，则不会发生任何事情。如果未定义 `NDEBUG`，则条件本身将从程序中剥离出来，而不是编译。
</br>在 C# 中，我们将使用 `[Conditional]` 属性来构建断言或使用现有的 `Debug.Assert`：

```c++
#include <assert.h>
 
assert(2 + 2 == 4); // OK
assert(2 + 2 == 5); // 调用 std::abort 并可能更多
```

然后我们有 `setjmp.h`/`csetjmp`，用于实现 `goto` 的高性能版本。
</br>这可以跳转到函数之外，但通过打破这些常规语言规则，可以避开用于清理本地对象的常规析构函数调用。
</br>这些在 C# 中都不可用：

```c++
#include <setjmp.h>
 
// 保存执行状态
jmp_buf buf;
 
// 使用volatile防止编译器优化掉这部分
volatile int count = 0;
 
void Goo()
{
    count++;
    DebugLog("Goo calling longjmp with", count);
 
    // 转到已保存的执行状态并将 'count' 作为 'status' 传递
    longjmp(buf, count);
}
 
void Foo()
{
    DebugLog("Foo");
 
    // 保存执行状态
    // 当调用 longjmp 时，执行将跳转到这里
    // 从 setjmp 返回的 'status'
    int status = setjmp(buf);
    DebugLog("Foo got status", status);
    if (status >= 3)
    {
        return;
    }
    DebugLog("Foo calling Goo");
    Goo();
}
```

这将打印以下内容：

```c++
Foo
Foo got status, 0
Foo calling Goo
Goo calling longjmp with, 1
Foo got status, 1
Foo calling Goo
Goo calling longjmp with, 2
Foo got status, 2
Foo calling Goo
Goo calling longjmp with, 3
Foo got status, 3
```

最后，还有 `errno.h`/`cerrno`。此标头提供 `errno` 宏，该宏包含多个 C 标准库函数使用的全局错误标志。
</br>这通常被认为是一种糟糕的错误处理方式，因为它不是线程安全的，并且调用者需要知道检查不属于函数签名的内容。
</br>它从未在 C# 中使用过，因此实际上没有等效项。
</br>不过，它在 C 标准库中被广泛使用，所以让我们看看它是如何工作的：

```c++
#include <errno.h>
 
// 向 sqrt（来自 math.h）传递一个无效的参数
float root = sqrt(-1.0f);
 
// 它返回NaN
DebugLog(root); // NaN
 
// 它通过将errno设置为EDOM（域外）来指示此错误
DebugLog(errno); // Maybe 33
 
// 检查这是否是所设置的
DebugLog(errno == EDOM); // true
```

## System Integration  系统集成

最后一类头文件涉及我们运行程序的系统。让我们从 `time.h`/`ctime` 开始，它类似于 C# 中 `DateTime` 的基本版本：

```c++
#include <time.h>
 
// 获取时间，既在返回值中，也在我们传递的指针中
time_t t1{};
time_t t2 = time(&t1);
DebugLog(t1, t2); // Maybe 1612052060, 1612052060
 
// 获取程序使用的CPU时间量
// 与任何特定时间（如UNIX纪元）无关
clock_t c1 = clock();
 
// 做些我们想要基准测试的昂贵事情
volatile float f = 123456;
for (int i = 0; i < 1000000; ++i)
{
    f = sqrtf(f);
}
 
// 再次检查clock
clock_t c2 = clock();
double secs = ((double)(c2) - c1) / CLOCKS_PER_SEC;
DebugLog("Took", secs, "seconds"); // 可能：耗时0.011秒
```

我们还有 `signal.h`/`csignal` 来处理 OS 信号。这使我们能够处理诸如作系统终止之类的信号，并自己发出这些信号。
</br>这通常不会使用 C# 完成，因为我们的程序运行的 .NET 环境会处理此类信号：

```c++
#include <signal.h>
 
signal(SIGTERM, [](int val){DebugLog("terminated with", val); });
raise(SIGTERM); // 可能：以15结束
```

许多 C 标准库函数使用全局 “locale” 设置来确定它们的工作方式。
</br>`locale.h`/`clocale` 头文件具有更改此设置的功能。它类似于 C# 中特定于线程的 `CultureInfo`：

```c++
#include <locale.h>
 
// 将所有内容的区域设置更改为日语
// 这是全局的：不是线程安全的
setlocale(LC_ALL, "ja_JP.UTF-8");
 
// 获取全局区域设置
lconv* lc = localeconv();
DebugLog(lc->currency_symbol); // Â¥
```

最后，我们将以在 C 语言中启用 “Hello， world！”的标头结束：`stdio.h`/`cstdio`。这
</br>类似于 C# 中的 `Console`。还有文件系统访问，类似于 C# 中的 `File` 方法：

```c++
#include <stdio.h>
 
// 输出一个格式化的字符串到标准输出
  // 第一个字符串是“格式字符串”，包含值占位符：%s %d
  // 后续的值必须与占位符的类型匹配
  // 这是一个可变参数函数
printf("%s %d\n", "Hello, world!", 123); // Hello, world! 123
 
// 从标准输入读取一个值
// 使用相同的“格式字符串”来接受不同类型
int val;
int numValsRead = scanf("%d", &val);
DebugLog(numValsRead); // {如果用户输入了数字，则为1，否则为0}
if (numValsRead == 1)
{
    DebugLog(val); // {用户输入的数字}
}
 
// 用户输入的数字
FILE* file = fopen("/path/to/myfile.dat", "r");
fseek(file, 0, SEEK_END);
long len = ftell(file);
fclose(file);
DebugLog(len); // {文件中的字节数}
 
// 删除文件
int deleted = remove("/path/to/deleteme.dat");
DebugLog(deleted == 0); // 如果文件已被删除则为真
 
// 重命名文件
int renamed = rename("/path/to/oldname.dat", "/path/to/newname.dat");
DebugLog(renamed == 0); // 如果文件已被重命名
```

## Conclusion  结论

C 标准库非常古老，但仍然非常常用。由于如此古老并且基于功能要弱得多的 C，它的很多设计还有很多不足之处。
`rand` 和 `strtok` 等函数以及 `errno` 等宏使用的全局状态不是线程安全的，很难理解如何正确使用。
使用特殊的 `int` 值，即使不一致，而不是更结构化的输出（如异常和枚举）同样难以使用。

无论我们对 C 标准库的设计有任何抱怨，我们仍然需要知道如何使用它。
C++ 标准库为我们今天在这里看到的许多内容提供了替代方案，但情况并非总是如此。
当然，我们可以将 `<random>` 换成 `rand`，将 `<chrono>` 换成`time` ，将 `<filesystem>` 换成 `remove`，但 `assert` 和 `stdint.h` 仍然是实现这些功能领域的最现代的标准化方法。




























