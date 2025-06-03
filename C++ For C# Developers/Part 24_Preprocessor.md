# C++ For C# Developers: Part 24 – Preprocessor

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5902),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C# 和 C++ 有类似的预处理器指令列表，如 `#if`，但它们的特性和用法却非常不同。特别是在 C++ 中，它支持“宏”，可以替换代码。
</br>今天我们将探讨在 C++ 中可以使用预处理器做的一切，并与 C# 的预处理器进行比较。

## Conditionals  条件

就像在C#中一样，C++的预处理器在编译的早期阶段运行。
</br>这是在将文件的字节解释为字符并移除注释之后，但在对变量和函数等语言概念进行主要编译之前。因此，预处理器对源代码的理解非常有限。

它对源代码的这种有限理解进行文本替换。完成之后，得到的源代码将被编译。

在C#中，这种用法的一个常见例子是条件“指令”：`#if`、`#else`、`#elif`和`#endif`。
</br>这些指令允许在编译的预处理步骤中实现分支逻辑。C#允许对布尔预处理器符号进行逻辑操作：

```csharp
// C#
static void Assert(bool condition)
{
    #if DEBUG && (ASSERTIONS_ENABLED == true)
        if (!condition)
        {
            throw new Exception("Assertion failed");
        }
    #endif
}
```
如果 `#if` 表达式的计算结果为 `false`，则删除 `#if` 和 `#endif` 之间的代码：

这有助于我们通过删除指令和内存访问来减小生成的可执行文件大小并提高运行时性能。

一个常见的错误是假设预处理器比它实际所理解的更多关于源代码的结构。例如，我们可能认为它理解什么是标识符：

```csharp
// C#
void Foo()
{
    #if Foo
        DebugLog("Foo exists");
    #else
        DebugLog("Foo does not exist"); // 被打印出来
    #endif
}
```

C++ 对预处理器条件语句也有类似的支持。它们甚至被命名为 `#if`、`#else`、`#elif` 和`#endif`。上面的 C# 示例实际上也是有效的 C++ 代码！

这两种语言在几个小方面有所不同。首先，在C++中，#if ABC检查ABC的值，而不仅仅是它是否已定义。

```c++
// 假设ZERO的值为0
#if ZERO
    DebugLog("zero");
#else
    DebugLog("non-zero"); // 被打印出来
#endif
```

有几种方法可以避免这种情况。首先，我们可以使用预处理器的定义操作符来检查符号是否已定义，而不是检查它的值：

```c++
#if defined(ZERO) // 检测是否定义
    DebugLog("zero"); // 被打印出来
#else
    DebugLog("non-zero");
#endif
 
// 不带括号的备用版本
#if defined ZERO // 检测是否定义
    DebugLog("zero"); // 被打印出来
#else
    DebugLog("non-zero");
#endif
```

另一种方法是使用 `#ifdef` 和 `#ifndef` 而不是 `#if`：

```c++
#ifdef ZERO // 已定义ZERO。其值无关紧要。
    DebugLog("zero"); // 被打印出来
#else
    DebugLog("non-zero");
#endif
 
#ifndef ZERO // 检查是否未定义
    DebugLog("zero");
#else
    DebugLog("non-zero"); // 被打印出来
#endif
```

这些检查通常用于实现头文件保护。
</br>自C++17以来，它们也可以与`__has_include`一起使用，如果头文件存在，则其评估结果为1，如果不存在，则评估结果为0。
</br>这通常用于检查可选库是否可用或从几个等效库中选择一个。

```c++
// 如果系统通过 debug_log.h 头文件提供 DebugLog
#if __has_include(<debug_log.h>)
    // 使用系统提供的DebugLog
    #include <debug_log.h>
// 系统不提供DebugLog
#else
    // 使用C标准库中的puts定义我们自己的版本
    #include <cstdio>
    void DebugLog(const char* message)
    {
        puts(message);
    }
#endif
```

这个 `__has_include(<header_name>)` 检查使用与 `#include <header_name>` 相同的头文件搜索方式。
要检查 `#include "header_name"` 的头文件搜索，我们可以使用 `__has_include("header_name")`。

## Macros  宏

两种语言都允许使用 `#define` 定义预处理器符号，并使用 `#undef` 取消定义它们：

```c++
// 定义一个预处理器符号
#define ENABLE_LOGGING
 
void LogError(const char* message)
{
    // 检查预处理符号是否定义
    // 因此，调试日志保留
    #ifdef ENABLE_LOGGING
        DebugLog("ERROR", message);
    #endif
}
 
// 取消定义预处理器符号
#undef ENABLE_LOGGING
 
void LogTrace(const char* message)
{
    // 检查预处理符号是否定义
    // 它没有，所以移除了DebugLog
    #ifdef ENABLE_LOGGING
        DebugLog("TRACE", message);
    #endif
}
 
void Foo()
{
    LogError("whoops"); // 打印 "ERROR whoops"
    LogTrace("got here"); // 没有打印
}
```

C#要求`#define`和`#undef`只能出现在文件顶部，但C++允许它们出现在任何位置。

C++不仅超越了这些简单的预处理器符号定义，它还有一个完整的“宏”系统，允许进行文本替换。
</br>虽然这在“现代C++”中通常是不被鼓励的，但它的使用在特定任务中仍然非常普遍。
</br>有时当语言没有提供可行的替代方案，或者至少在编写代码时没有提供时，会使用它。无论如何，宏被广泛使用，了解它们的工作原理很重要。

首先，我们可以通过为预处理符号提供一个值来定义一个“对象-like”宏。与C#不同，该值不必是布尔值：

```c++
// 定义一个类似于对象的宏
#define LOG_LEVEL 1
#define LOG_LEVEL_ERROR 3
#define LOG_LEVEL_WARNING 2
#define LOG_LEVEL_DEBUG 1
 
void LogWarning(const char* message)
{
    // 预处理符号可以在#if表达式中使用
    #if LOG_LEVEL <= LOG_LEVEL_WARNING
        // 预处理器符号将被替换为其值
        DebugLog(LOG_LEVEL_WARNING, message);
 
        // 预处理后，上一行变为：
        DebugLog(2, message);
    #endif
}
```

我们还可以定义带参数的“函数式”宏：

```c++
// 定义一个函数式宏
#define MADD(x, y, z) x*y + z

void Foo()
{
    int32_t x = 2;
    int32_t y = 3;
    int32_t z = 4;

    // 调用函数宏
    int32_t result = MADD(x, y, z);
 
    // 预处理后，上一行变为：
    int32_t result = x*y + z;
 
    DebugLog(result); // 10
}
```

与运行时函数调用不同，调用函数式宏只是简单地执行文本替换。
</br>这很容易被忘记，尤其是在宏的命名像正常函数一样时。这可能导致错误和性能问题，因为宏被调用之前不会评估参数表达式：

```c++
// 类似于函数的宏，命名方式与普通函数相同，而不是全部大写
#define square(x) x*x
 
int32_t SumOfRandomNumbers(int32_t n)
{
    int32_t sum = 0;
    for (int32_t i = 0; i < n; ++i)
    {
        sum += rand();
    }
    return sum;
}
 
void Foo()
{
    // 调用一个非常昂贵的函数
    int32_t result = square(SumOfRandomNumbers(1000000));
 
    // 预处理后，上一行变为：
    int32_t result = SumOfRandomNumbers(1000000)*SumOfRandomNumbers(1000000);
 
    DebugLog(result); // {一些随机数字}
}
```

在正常函数调用中，SumOfRandomNumbers(1000000)会在函数被调用之前进行评估。 使用宏时，它只是文本替换，所以square最终会调用它两次。
</br>这个调用非常昂贵，因此我们遇到了性能问题。这还是一个错误，因为我们不再一定是将相同的数字乘以自己，因为两次调用可能返回不同的数字。
</br>为了更清楚地看到错误的产生，看下面这个这个宏调用：

```c++
void Foo()
{
    int32_t i = 1;
    int32_t result = square(++i);
 
    // 预处理后，上一行变为：
    int32_t result = ++i*++i;
 
    DebugLog(result, i); // 6, 3
}
```

再次，宏调用之前不会评估参数 (`++i`)，而是每次宏引用参数时都会重复。这意味着在乘法 (`*`) 产生结果 `2*3=6` 并将 `i` 设置为 `3` 之前，`i` 从 `1` 增加到 `2`，然后再次增加到 `3`。
如果这是一个正常的普通函数调用，我们预计会是 2*2=4，并且 i 的值在之后应该是 2。这些潜在的错误是宏被劝阻使用的一个原因。

函数式宏可以访问几个特殊操作符：`#` 和 `##`。`#` 操作符将参数用引号括起来以创建一个字符串字面量：

```c++
// 将 msg 用引号括起来以创建 "msg"
#define LOG_TIMESTAMPED(msg) DebugLog(GetTimestamp(), #msg);
 
void Foo()
{
    // 无需引号。'hello'变成'hello'。
    LOG_TIMESTAMPED(hello) // {timestamp} hello
 
    // 额外的引号被添加，并且现有的引号被转义："hello"
    LOG_TIMESTAMPED("hello") // {timestamp} "hello"
}
```

`##`运算符用于连接两个符号，这些符号可以是参数：

```c++
// 每一行将一些文本（例如 m_）与 name 的值连接起来
// 使用反斜杠来创建多行宏
#define PROP(type, name) \
    private: type m_##name; \
    public: type Get##name() const { return m_##name; } \
    public: void Set##name(const type & val) { m_##name = val; }
 
struct Vector2
{
    PROP(float, X)
    PROP(float, Y)
 
    // 这些宏调用被替换为：
 
    private: float m_X;
    public: float GetX() const { return m_X; }
    public: void SetX(const float & val) { m_X = val; }
    private: float m_Y;
    public: float GetY() const { return m_Y; }
    public: void SetY(const float & val) { m_Y = val; }
};
 
void Foo()
{
    Vector2 vec;
    vec.SetX(2);
    vec.SetY(4);
    DebugLog(vec.GetX(), vec.GetY()); // 2, 4
}
```

宏也可以使用`...`来接受可变数量的参数，类似于函数。使用`__VA_ARGS__`来访问参数：

```c++
#define LOG_TIMESTAMPED(level, ...) DebugLog(level, GetTimestamp(), __VA_ARGS__);
 
void Foo()
{
    LOG_TIMESTAMPED("DEBUG", "hello", "world") // DEBUG {timestamp} hello world
 
    // 这个宏调用被替换为：
 
    DebugLog("DEBUG", GetTimestamp(), "hello", "world");
}
```

在C++20中，也提供了`__VA_OPT__(x)`。
</br>如果`__VA_ARGS__`为空，则替换为无。
</br>如果`__VA_ARGS__`不为空，则替换为x。
</br>这可以用来使像`LOG_TIMESTAMPED`这样的宏中的参数可选：

```c++
// __VA_OPT__(,) 仅在 __VA_ARGS__ 不为空时添加逗号，这意味着调用者传递了一些日志消息
#define LOG_TIMESTAMPED(...) DebugLog(GetTimestamp() __VA_OPT__(,) __VA_ARGS__);
 
void Foo()
{
    LOG_TIMESTAMPED() // {timestamp}
    LOG_TIMESTAMPED("hello", "world") // {timestamp} hello world
 
    // 这些宏调用被替换为：
 
    DebugLog(GetTimestamp()  );
 
    DebugLog(GetTimestamp() , "hello", "world");
}
```

没有 `__VA_OPT__`，我们就不知道宏是否应该放一个逗号，因为我们不知道是否还有任何参数要传递给它。

## Built-in Macros and Feature-Testing 内置宏和功能测试

就像C#预定义了`DEBUG`和`TRACE`预处理器符号一样，C++也预定义了一些类似对象的宏：

| 宏                                  | 值                                                                                                         | 含义                        |
|------------------------------------|-----------------------------------------------------------------------------------------------------------|---------------------------|
| `__cplusplus`                      | 199711L (C++98 and C++03)</br>201103L (C++11)</br>201402L (C++14)</br>201703L (C++17)</br>202002L (C++20) | C++ 语言版本                  |
| `__STDC_HOSTED__`                  | 1如果有操作系统，0如果没有                                                                                            |                           |
| `__FILE__`                         | "mycode.cpp"                                                                                              | 当前文件名称                    |
| `__LINE__`                         | 44                                                                                                        | 当前行号                      |
| `__DATE__`                         | "2020 10 26"                                                                                              | 编译代码的日期                   |
| `__TIME__`                         | "02:00:00"                                                                                                | 代码编译的时间                   |
| `__STDCPP_DEFAULT_NEW_ALIGNMENT__` | 8                                                                                                         | 默认的`new`对齐。仅限于C++17及以上版本。 |


自C++20以来，在`<version>`头文件中提供了一大堆“功能测试”宏。
</br>这些宏都是对象式的，它们的值是语言或标准库功能被添加到C++的日期。目的是将它们与`__cplusplus`进行比较，以确定是否支持该功能。
</br>如果列出的的话就太多了，但以下展示了其中几个的实际应用：

```c++
void Foo()
{
    if (__cplusplus >= __cpp_char8_t)
    {
        DebugLog("char8_t is supported in the language");
    }
    else
    {
        DebugLog("char8_t is NOT supported in the language");
    }
 
    if (__cplusplus >= __cpp_lib_byte)
    {
        DebugLog("std::byte is supported in the Standard Library");
    }
    else
    {
        DebugLog("std::byte is NOT supported in the Standard Library");
    }
}
```

C++ 标准对 <version> 头文件的定义中提供了[完整列表](https://eel.is/c++draft/version.syn#header:%3Cversion%3E)。

## Miscellaneous Directives 其他指令

预定义的 `__FILE__` 和` __LINE__` 值可以被另一个预处理器指令 `#line` 覆盖。这与 C# 中的用法类似，但`default`和`hidden`是不允许的：

```c++
void Foo()
{
    DebugLog(__FILE__, __LINE__); // main.cpp, 38
#line 100
    DebugLog(__FILE__, __LINE__); // main.cpp, 100
#line 200 "custom.cpp"
    DebugLog(__FILE__, __LINE__); // custom.cpp, 200
}
```

可以使用 `#error` 指令使编译器产生错误：

```c++
#ifndef _MSC_VER
    #error Only Visual Studio is supported
#endif
```

`#pragma` 用于允许编译器提供自己的预处理器指令，就像在 C# 中一样：

```c++
// mathutils.h
 
// 编译器特定的头文件保护替代方案
#pragma once
 
float SqrMagnitude(const Vector2& vec)
{
    return vec.X*vec.X + vec.Y*vec.Y;
}
```

可以使用 `_Pragma("expr")` 来代替 `#pragma expr`。它们具有相同的效果：

```c++
_Pragma("once")
```

C#中的`#region`和`#endregion`在C++中不受支持，但像Visual Studio这样的编译器可以通过`#pragma`来实现：

```c++
#pragma region Math
 
float SqrMagnitude(const Vector2& vec);
float Dot(const Vector2& a, const Vector2& b);
 
#pragma endregion Math
```

## Usage and Alternatives  用法和替代方案

C++的每个新版本都使得预处理器的使用变得不那么必要。例如，C++11引入了`constexpr`变量，这消除了许多使用对象宏的原因：

```c++
// C++11 之前
#define PI 3.14f
 
// C++11 之后
constexpr float PI = 3.14f;
```

这使得`PI`成为一个实际的对象，因此它具有类型（`float`），可以获取其地址（`&PI`），并且可以像其他对象一样使用，而不是作为文本替换的浮点字面量。
</br>在`struct类型`、`lambda类`和其他非原始类型的情况下，这种好处变得更加显著，因为这些情况下实际上不可能为通用目的创建宏：

```c++
// 在C++11之前
// 这在许多上下文中不可用，例如Foo(EXPONENTIAL_BACKOFF_TIMES)
#define EXPONENTIAL_BACKOFF_TIMES { 1000, 2000, 4000, 8000, 16000 }
 
// AC++11之后
// 这就像任何数组对象一样工作：
constexpr int32_t ExponentialBackoffTimes[] = { 1000, 2000, 4000, 8000, 16000 };
```

同样，`constexpr`和`consteval`函数大大减少了函数宏的需求：

```c++
constexpr int32_t Square(int32_t x)
{
    return x * x;
}
 
void Foo()
{
    int32_t i = 1;
    int32_t result = Square(++i);
    DebugLog(result); // 4
}
```

这些行为更像常规函数而不是文本替换。我们跳过了宏可能引起的所有错误和性能问题，但保留了编译时评估。
</br>我们甚至可以在C++20中使用consteval强制进行编译时评估。我们获得了强类型，因此Square("Foo")是一个错误。
</br>我们可以使用该函数在运行时，而不仅仅是编译时。它的行为就像任何其他函数一样：我们可以取函数指针，我们可以创建成员函数，等等。

然而，当我们在没有原始文本替换的情况下无法表达某些内容时，宏提供了一种逃生机制。
</br>上面提到的`PROP`宏示例生成了具有访问修饰符的成员。否则根本无法做到这一点。
</br>那个例子可能不是最好的，但其他例子确实如此。一个经典的例子是断言宏：

```c++
//当启用断言时，定义 ASSERT 为一个宏，该宏测试一个布尔值，当其为假时记录并终止程序。
#ifdef ENABLE_ASSERTS
    #define ASSERT(x) \
        if (!(x)) \
        { \
            DebugLog("assertion failed"); \
            std::terminate(); \
        }
// 当断言被禁用时，assert什么也不做
#else
    #define ASSERT(x)
#endif
 
bool IsSorted(const float* vals, int32_t length)
{
    for (int32_t i = 1; i < length; ++i)
    {
        if (vals[i] < vals[i-1])
        {
            return false;
        }
    }
    return true;
}
 
float GetMedian(const float* vals, int32_t length)
{
    ASSERT(vals != nullptr);
    ASSERT(length > 0);
    ASSERT(IsSorted(vals, length));
    if ((length & 1) == 1)
    {
        return vals[length / 2]; // odd
    }
    float a = vals[length / 2 - 1];
    float b = vals[length / 2];
    return (a + b) / 2;
}
 
void Foo()
{
    float oddVals[] = { 1, 3, 3, 6, 7, 8, 9 };
    DebugLog(GetMedian(oddVals, 7));
 
    float evenVals[] = { 1, 2, 3, 4, 5, 6, 8, 9 };
    DebugLog(GetMedian(evenVals, 8));
 
    DebugLog(GetMedian(nullptr, 1));
 
    float emptyVals[] = {};
    DebugLog(GetMedian(emptyVals, 0));
 
    float notSortedVals[] = { 3, 2, 1 };
    DebugLog(GetMedian(notSortedVals, 3));
}
```

当启用断言时，调用 ASSERT 执行以下替换：

```c++
ASSERT(IsSorted(vals, length));
 
// 变为:
 
if (!(IsSorted(vals, length)))
{
    DebugLog("assertion failed");
    std::terminate();
}
```

当禁用时，包括传递的参数在内的所有内容都将被删除：

```c++
ASSERT(IsSorted(vals, length));
 
// 变为:


```

现在想象一下，如果我们使用了一个 `constexpr` 函数而不是宏：

```c++
#ifdef ENABLE_ASSERTS
    constexpr void ASSERT(bool x)
    {
        if (!x)
        {
            DebugLog("assertion failed");
            std::terminate();
        }
    }
#else
    constexpr void ASSERT(bool x)
    {
    }
#endif
```

当断言被禁用时，我们得到一个空的`constexpr`函数：

```c++
constexpr void ASSERT(bool x)
{
}
```

但是当我们调用 `ASSERT` 时，即使函数本身什么都不做，参数仍然需要被评估：

```c++
ASSERT(IsSorted(vals, length));
 
// 等同于:
 
bool x = IsSorted(vals, length);
Assert(x); // 什么也不做
```

编译器可能能够确定对IsSorted的调用没有副作用，可以安全地移除。
</br>在许多情况下，它无法做出这种判断，并且仍然会执行昂贵的IsSorted调用。我们不希望这种情况发生，因此我们使用宏。

宏也可以用来实现C#泛型或C++模板的原始形式，我们将在系列中很快介绍这一点：

```c++
// Vector2类的“通用”/“模板”
#define DEFINE_VECTOR2(name, type) \
    struct name \
    { \
        type X; \
        type Y; \
    };
 
// 调用宏以生成 Vector2 类
DEFINE_VECTOR2(Vector2f, float);
DEFINE_VECTOR2(Vector2d, double);
 
// 函数的“通用”/“模板”
#define DEFINE_MADD(type) \
    type Madd(type x, type y, type z) \
    { \
        return x*y + z; \
    }
 
// 调用宏以生成Madd函数
DEFINE_MADD(float);
DEFINE_MADD(int32_t);
 
void Foo()
{
    // 使用生成的Vector2类
    // 使用sizeof来显示它们具有不同的组件大小
    Vector2f v2f{2, 4};
    DebugLog(sizeof(v2f), v2f.X, v2f.Y); // 8, 2, 4
 
    Vector2d v2d{20, 40};
    DebugLog(sizeof(v2d), v2d.X, v2d.Y); // 16, 20, 40
 
    // 使用生成的Madd函数
    // 通过返回值上的typeid来显示它们是重载
    float xf{2}, yf{3}, zf{4};
    auto maddf{Madd(xf, yf, zf)};
    DebugLog(typeid(maddf) == typeid(float)); // true
    DebugLog(typeid(maddf) == typeid(int32_t)); // false
 
    int32_t xi{2}, yi{3}, zi{4};
    auto maddi{Madd(xi, yi, zi)};
    DebugLog(typeid(maddi) == typeid(float)); // false
    DebugLog(typeid(maddi) == typeid(int32_t)); // true
}
```

这种代码生成形式在缺乏C++模板的C代码库中很常见。当模板可用时，例如在所有版本的C++中，由于许多原因，它们是首选选项。
</br>其中一个原因是能够“重载”类名，这样我们就可以只使用`Vector2`，而不是想出像`Vector2f`和`Vector2d`这样的尴尬的唯一名称。

另一个原因是，通常不需要为每个类和函数中所需类型的每一种排列都列出通常很大的`DEFINE_X`宏调用列表。
</br>当有几个“类型参数”时，这真的变得难以控制。相反，编译器根据我们对类或函数的使用生成所有排列，所以我们不需要明确维护这样的列表。

在系列中稍后讨论模板时，我们还会涉及到更多原因。

## Conclusion  结论

这两种语言在预处理器的使用上有很多重叠。它们在编译的同一阶段运行，并且具有许多同名指令和相同的功能。

主要的差异点在于`#include`，这是在C++20之前构建模型的一个关键部分，以及由`#define`创建的宏。
</br>函数式宏代表了一种编译时编程的另一种形式，它在预处理期间运行，而不是`constexpr`在主要编译期间运行。
</br>它们也是泛型或模板的另一种形式。虽然随着时间的推移，它们的必要性已经降低，但它们对于某些任务仍然是必不可少的，对于其他任务来说也很方便。










