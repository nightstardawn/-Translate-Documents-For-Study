# C++ For C# Developers: Part 39 – Language Support Library


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6439),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C++的一些部分需要C++标准库的部分。我们已经简要地提到了像`std::initializer_list`和`std::typeinfo`这样的类，但今天我们将探讨更多内容。
</br>我们将看到通常会被构建到语言中或与其他特定语言特性的使用紧密相关的标准库的部分。

## Source Location  源位置

让我们从一个刚刚在C++20中添加的简单功能开始：`<source_location>`。立即我们可以看到C++标准库的命名约定与C标准库(Part38)及其C++包装器的不同。
</br>C标准库的头文件名可能会被缩写为类似于`srcloc.h`的形式。然后C++包装器的名称将是`csrcloc`。
</br>C++标准库通常更喜欢在`snake_case`中更详细地拼写名称，并且不使用任何扩展名，如`.h`等。

在 `<source_location>` 头文件中，我们看到了命名约定的延续，即 `source_location` 类。
</br>虽然并不总是存在一一对应的关系，但几乎总是使用 `snake_case`。
</br>`source_location` 类被放置在 `std` 命名空间中，因此我们通常将其称为 `std::source_location`。
</br>`std` 命名空间是为 C++ 标准库保留的。

接下来是 `std::source_location` 的实际用途。如其名称所示，它提供了一个表达源代码中位置的功能。它有拷贝和移动构造函数，但我们无法从零开始创建一个。
</br>相反，我们调用其静态成员函数 `current()` ，然后返回一个对象：

```c++
#include <source_location>
 
void Foo()
{
    std::source_location sl = std::source_location::current();
    DebugLog(sl.line()); // 42
    DebugLog(sl.column()); // 61
    DebugLog(sl.file_name()); // example.cpp
    DebugLog(sl.function_name()); // void Foo()
}
```

`file_name()` 成员函数为 `__FILE__` 宏提供了一个替代方案。同样， `line()` 替代了 `__LINE__` 。
</br>我们还获得了 `column` 和 `function_name` ，它们在标准宏形式中不存在。
</br>在 C#中， `StackTrace` 和 `StackFrame` 类大致相当于 `source_location` 。

为了避免将整个标准库引入作用域，我们可以只包含特定的类：

```c++
#include <source_location>
using std::source_location;
 
void Foo()
{
    source_location sl = source_location::current();
    DebugLog(sl.line()); // 43
    DebugLog(sl.column()); // 61
    DebugLog(sl.file_name()); // example.cpp
    DebugLog(sl.function_name()); // void Foo()
}
```

或者，我们也可以将 `using` 放在标准库被使用的地方：

```c++
#include <source_location>
 
void Foo()
{
    using namespace std;
    source_location sl = source_location::current();
    DebugLog(sl.line()); // 43
    DebugLog(sl.column()); // 61
    DebugLog(sl.file_name()); // example.cpp
    DebugLog(sl.function_name()); // void Foo()
}
```

所有这些在 C++代码库中都很常见，并且为消除大量 `std::` 杂乱提供了很好的选择。
</br>然而，有一个不好的选择应该避免：在头文件的顶层添加 `using`。
</br>因为头文件是通过 `#include` 被复制粘贴到其他文件中的，这些 `using` 语句会向所有引用它们的文件的名字查找引入标准库。
</br>当头文件 `#include` 其他头文件时，这种影响会进一步扩大：

```c++
// top.h
#include <source_location>
using namespace std; // Bad idea
 
// middlea.h
#include "top.h" // 在这里粘贴 "using namespace std;"
 
// middleb.h
#include "top.h" // 在这里粘贴 "using namespace std;"
 
// bottoma.cpp
#include "middlea.h" // 在这里粘贴 "using namespace std;"
 
// bottomb.cpp
#include "middlea.h" // 在这里粘贴 "using namespace std;"
 
// bottomc.cpp
#include "middleb.h" // 在这里粘贴 "using namespace std;"
 
// bottomd.cpp
#include "middled.h" // 在这里粘贴 "using namespace std;"
```

`using namespace std;` 在 `top.h` 中的影响已经扩散到 `#include` 它的文件： `middlea.h` 和 `middleb.h`。
</br>然后又扩散到那些依赖它们的文件： `bottoma.cpp` 、 bottomb`.cpp` 、 `bottomc.cpp` 和 `bottomd.cpp`。
</br>最好避免这种情况，以免破坏命名空间提供的模块化，而是让各个文件自行选择何时何地打破这种界限：

```c++
// top.h
#include <source_location>
struct SourceLocationPrinter
{
    static void Print()
    {
        // OK: 仅适用于此函数，不适用于#include的文件
        using namespace std;
 
        source_location sl = source_location::current();
        DebugLog(sl.line()); // 43
        DebugLog(sl.column()); // 61
        DebugLog(sl.file_name()); // example.cpp
        DebugLog(sl.function_name()); // void SourceLocationPrinter::Print()
    }
};
 
// middlea.h
#include "top.h" // 此处不粘贴 "using namespace std;"
 
// middleb.h
#include "top.h" // 此处不粘贴 "using namespace std;"
```

## Initializer List  初始化列表

接下来我们来看 `<initializer_list>` 。我们之前提过 `std::initializer_list` ，但现在我们将更深入地探讨。
</br>当我们使用大括号初始化列表时，这个类模板的实例会自动创建并传递给构造函数：

```c++

struct AssetLoader
{
    AssetLoader(std::initializer_list<const char*> paths)
    {
        for (const char* path : paths)
        {
            DebugLog(path);
        }
    }
};
 
AssetLoader loader = {
    "/path/to/model",
    "/path/to/texture",
    "/path/to/audioclip"
};
```

我们可以重写这部分来手动创建 `std::initializer_list<const char*>` ，但这依赖于与 `std::initializer_list` 相同的括号列表初始化
</br>因为 `std::initializer_list` 没有直接创建空实例的方法：

```c++
AssetLoader loader(std::initializer_list<const char*>{
    "/path/to/model",
    "/path/to/texture",
    "/path/to/audioclip"
});
```

正如我们在 `AssetLoader` 构造函数中所见，基于范围的 `for` 循环适用于 `std::initializer_list` 。
</br>还有一个 `size` 成员函数，但由于没有索引运算符，所以我们不能使用典型的 `for` 循环：

```c++
AssetLoader(std::initializer_list<const char*> paths)
{
    // OK: 存在一个size成员函数
    for (size_t i = 0; i < paths.size(); ++i)
    {
        // 编译器错误：没有[int]运算符
        DebugLog(paths[i]);
    }
}
```


C#的对应方式是获取一个 params 托管数组。编译器会在调用位置为我们构建这个托管数组，就像为我们构建 `std::initializer_list` 一样。

## Type Info and Index 类型信息和索引

在查看 `RTTI` 时，我们也见过一点 `<typeinfo>` 。当我们使用 `typeid` 时，我们会得到一个 `std::type_info` ，它类似于 C# `Type` 类的一个轻量级版本：

```c++
#include <typeinfo>
 
struct Vector2
{
    float X;
    float Y;
};
 
struct Vector3
{
    float X;
    float Y;
    float Z;
};
 
Vector2 v2{ 2, 4 };
Vector3 v3{ 2, 4, 6 };
 
// 所有构造函数都被删除了，但我们仍然可以获取引用
const std::type_info& ti2 = typeid(v2);
const std::type_info& ti3 = typeid(v3);
 
// 只有三个公共成员，它们都是实现特定的
DebugLog(ti2.name()); // Maybe struct Vector2
DebugLog(ti2.hash_code()); // Maybe 3282828341814375180
DebugLog(ti2.before(ti3)); // Maybe true
```

相关地， `<typeinfo>` 定义了 `bad_typeid` 类，当尝试获取指向多态类的空指针的 `typeid` 时，该类会被抛出作为异常。
在 C# 中，当我们尝试写 `nullObj.GetType()` 时，我们会得到 `NullReferenceException` 。

```c++
#include <typeinfo>
 
struct Vector2
{
    float X;
    float Y;
 
    //虚函数使这个类具有多态性
    virtual bool IsNearlyZero(float epsilonSq)
    {
        return abs(X*X + Y*Y) < epsilonSq;
    }
};
 
void Foo()
{
    Vector2* pVec = nullptr;
    try
    {
        // 尝试对一个空指针的多态类使用typeid
        DebugLog(typeid(*pVec).name());
    }
    // 抛出了这个特定的异常
    catch (const std::bad_typeid& e)
    {
        DebugLog(e.what()); // 可能尝试了一个nullptr指针的typeid！
    }
}
```

还有一个 `bad_cast` 类，在我们尝试 `dynamic_cast` 两个不相关的类型时会抛出。这相当于 C# 的 `InvalidCastException` 类：

```c++
#include <typeinfo>
 
struct Vector2
{
    float X;
    float Y;
 
    virtual bool IsNearlyZero(float epsilonSq)
    {
        return abs(X*X + Y*Y) < epsilonSq;
    }
};
 
struct Vector3
{
    float X;
    float Y;
    float Z;
 
    virtual bool IsNearlyZero(float epsilonSq)
    {
        return abs(X*X + Y*Y + Z*Z) < epsilonSq;
    }
};
 
void Foo()
{
    Vector3 vec3{};
    try
    {
        Vector2& vec2 = dynamic_cast<Vector2&>(vec3);
    }
    catch (const std::bad_cast& e)
    {
        DebugLog(e.what()); // Maybe "Bad dynamic_cast!"!
    }
}
```

`<typeindex>` 头文件提供了 `std::type_index` 类，而不是一个整数，它包装了我们在上面看到的 `std::type_info` 。
</br>这个类提供了一些重载的运算符，因此我们可以以多种方式比较它们，而不仅仅是通过 `before` 成员函数：

```c++
#include <typeindex>
 
struct Vector2
{
    float X;
    float Y;
};
 
struct Vector3
{
    float X;
    float Y;
    float Z;
};
 
Vector2 v2{ 2, 4 };
Vector3 v3{ 2, 4, 6 };
 
// 向构造函数传递一个 std::type_info 对象
const std::type_index ti2{ typeid(v2) };
const std::type_index ti3{ typeid(v3) };
 
// 一些来自 std::type_info 的成员函数被继承
DebugLog(ti2.name()); // Maybe struct Vector2
DebugLog(ti2.hash_code()); // Maybe 3282828341814375180
 
// 提供了重载运算符以进行比较
DebugLog(ti2 == ti3); // false
DebugLog(ti2 < ti3); // Maybe true
DebugLog(ti2 > ti3); // Maybe false
```

C# `Type` 类不能直接比较，所以我们通常会比较类似其完全限定名称字符串这样的内容。


## Compare  比较

C++20 引入了三向比较运算符： `x <=> y` 。这允许我们重载一个运算符，声明我们的类如何与另一个类进行比较。
</br>我们需要返回一个支持所有单个比较运算符的对象： `<` 、 `<=` 、 `>` 、 `>=` 、 `==` 和 `!=` 。
</br>与其定义自己的类来实现这一点，标准库通过 `<compare>` 头文件提供了一些内置类。
</br>例如，我们可以通过其静态数据成员之一返回一个 `std::strong_ordering` ：

```c++
#include <compare>
 
struct Integer
{
    int Value;
 
    std::strong_ordering operator<=>(const Integer& other) const
    {
        // 确定关系一次
        return Value < other.Value ?
            std::strong_ordering::less :
            Value > other.Value ?
                std::strong_ordering::greater :
                std::strong_ordering::equal;
    }
};
 
Integer one{ 1 };
Integer two{ 2 };
std::strong_ordering oneVsTwo = one <=> two;
 
// 所有单个比较运算符都受支持
DebugLog(oneVsTwo < 0); // true
DebugLog(oneVsTwo <= 0); // true
DebugLog(oneVsTwo > 0); // false
DebugLog(oneVsTwo >= 0); // false
DebugLog(oneVsTwo == 0); // false
DebugLog(oneVsTwo != 0); // true
```

对于较弱比较结果，有类似的类： `std::weak_ordering` 和 `std::partial_ordering` 。
</br>还有辅助函数，可以在这些比较类中的任何一个上调用所有这些运算符，因此我们可以这样写：

```c++
DebugLog(std::is_lt(oneVsTwo)); // true
DebugLog(std::is_lteq(oneVsTwo)); // true
DebugLog(std::is_gt(oneVsTwo)); // false
DebugLog(std::is_gteq(oneVsTwo)); // false
DebugLog(std::is_eq(oneVsTwo)); // false
DebugLog(std::is_neq(oneVsTwo)); // true
```

提供了辅助函数来获取这些排序类对象，即使是从原始类型获取：

```c++
std::strong_ordering so = std::strong_order(1, 2);
std::weak_ordering wo = std::weak_order(1, 2);
std::partial_ordering po = std::partial_order(1, 2);
std::strong_ordering sof = std::compare_strong_order_fallback(1, 2);
std::weak_ordering wof = std::compare_weak_order_fallback(1, 2);
std::partial_ordering pof = std::compare_partial_order_fallback(1, 2);
```

在 C# 中没有 `<=>` 运算符的对应物，因此这个头文件也没有对应的等价物。

## Concepts  概念

C++20 中另一个具有库支持的特性是概念(Part29)。有许多预定义的概念可供我们立即使用，并且我们可以扩展它们。这里有一些例子:

```c++
#include <concepts>
 
template <typename T1, typename T2>
requires std::same_as<T1, T2>
bool SameAs;
 
template <typename T>
requires std::integral<T>
bool Integral;
 
template <typename T>
requires std::default_initializable<T>
bool DefaultInitializable;
 
SameAs<int, int>; // OK
SameAs<int, float>; // Compiler error
 
Integral<int>; // OK
Integral<float>; // Compiler error
 
struct NoDefaultCtor { NoDefaultCtor() = delete; };
DefaultInitializable<int>; // OK
DefaultInitializable<NoDefaultCtor>; // Compiler error
```

还有更多适用于不同需求的选项： `derived_from` 、 `destructible` 、 `equality_comparable` 、 `copyable` 、 `invocable` ，等等。
</br>这些都没有 C#的对应项，因为 C#泛型约束是不可定制的。

## Coroutine  协程

最后获得库支持的 C++20 特性是协程(Part36)。 `<coroutine>` 头文件提供了我们在实现自定义协程“返回对象”时已经见过的所需 `std::coroutine_handle` 类。
它还提供了 `std::suspend_never` 和 `std::suspend_always` ，因此我们不必像之前那样编写自己的版本。
以下是使用 `std::suspend_never` 而不是我们的自定义 `NeverSuspend` 类时，我们简单的协程示例的样子：

```c++
#include <coroutine>
 
struct ReturnObj
{
    ReturnObj()
    {
        DebugLog("ReturnObj ctor");
    }
 
    ~ReturnObj()
    {
        DebugLog("ReturnObj dtor");
    }
 
    struct promise_type
    {
        promise_type()
        {
            DebugLog("promise_type ctor");
        }
 
        ~promise_type()
        {
            DebugLog("promise_type dtor");
        }
 
        ReturnObj get_return_object()
        {
            DebugLog("promise_type::get_return_object");
            return ReturnObj{};
        }
 
        std::suspend_never initial_suspend()
        {
            DebugLog("promise_type::initial_suspend");
            return std::suspend_never{};
        }
 
        void return_void()
        {
            DebugLog("promise_type::return_void");
        }
 
        std::suspend_never final_suspend()
        {
            DebugLog("promise_type::final_suspend");
            return std::suspend_never{};
        }
 
        void unhandled_exception()
        {
            DebugLog("promise_type unhandled_exception");
        }
    };
};
 
ReturnObj SimpleCoroutine()
{
    DebugLog("Start of coroutine");
    co_return;
    DebugLog("End of coroutine");
}
 
void Foo()
{
    DebugLog("Calling coroutine");
    ReturnObj ret = SimpleCoroutine();
    DebugLog("Done");
}
```

这里打印出的是什么：

```c++
Calling coroutine
promise_type ctor
promise_type::get_return_object
ReturnObj ctor
promise_type::initial_suspend
Start of coroutine
promise_type::return_void
promise_type::final_suspend
promise_type dtor
Done
ReturnObj dtor
```

还有一个三无操作协程功能： `std::noop_coroutine` 、 `std::noop_coroutine_promise` 和 `std::noop_coroutine_handle` 。
</br>这些实现了 `void noop() {}` 函数的协程等效。 
</br>`noop_coroutine` 是协程，它返回一个 `noop_coroutine_handle`，其“承诺”是一个 `noop_coroutine_promise` 。

C# 没有为其迭代函数提供这种级别的自定义，但我们可以实现 `IEnumerable` 、 `IEnumerable<T>` 、 `IEnumerator` 和 `IEnumerator<T>` 来对迭代过程进行一定控制。这些接口及其方法与这个头文件提供了最接近的类比。

## Version  版本

正如我们在查看预处理器的过程中所见，存在一个名为 `<version>` 的头文件，其中包含大量宏，我们可以使用这些宏来检查语言和标准库中各种功能是否可用。
</br>例如，我们可以检查今天我们看到的某些标准库功能：

```c++
#include <version>
 
// 这些根据标准库是否具有这些功能打印“true”或“false”
DebugLog("Standard Library concepts?", __cplusplus >= __cpp_lib_concepts);
DebugLog("source_location?", __cplusplus >= __cpp_lib_source_location);
```

C# 拥有一些标准化的预处理符号，包括 `DEBUG` 和 `TRACE` ，但其套件远不如 C++ 全面。
每个 .NET 实现，如 Unity 和 .NET Core，都可能定义自己的额外符号，例如 `UNITY_2020_2_OR_NEWER` ，而这些版本号通常与可用的语言和库功能相关。

## Type Traits  类型特征

最后，今天我们介绍的是 `<type_traits>` ，它用于编译时编程(Part23)。这个头文件早于 C++20 中的概念，因此其中很多内容以非概念形式存在重叠。
</br>例如，我们有各种 `constexpr` 变量模板，用于检查类型是否满足特定条件。
</br>这些可以作为类模板的 `static` 成员变量，也可以作为命名空间作用域的变量模板使用：

```c++
#include <type_traits>
 
// 使用类模板的静态值数据成员
static_assert(std::is_integral<int>::value); // OK
static_assert(std::is_integral<float>::value); // Compiler error
 
// 使用变量模板
static_assert(std::is_integral_v<int>); // OK
static_assert(std::is_integral_v<float>); // Compiler error
```

这里有很多这样的工具，它们可以检查类型的几乎所有特性。这里有一些更高级的工具：

```c++
#include <type_traits>
 
struct Vector2
{
    float X;
    float Y;
};
 
struct Player
{
    int Score;
 
    Player(const Player& other)
    {
        Score = other.Score;
    }
};
 
static_assert(std::is_bounded_array_v<int[3]>); // OK
static_assert(std::is_bounded_array_v<int[]>); // Compiler error
 
static_assert(std::is_trivially_copyable_v<Vector2>); // OK
static_assert(std::is_trivially_copyable_v<Player>); // Compiler error
```

除了类型检查之外，还有多种用于查询类型的工具：

```c++
#include <type_traits>
 
DebugLog(std::rank_v<int[10]>); // 1
DebugLog(std::rank_v<int[10][20]>); // 2
DebugLog(std::extent_v<int[10][20], 0>); // 10
DebugLog(std::extent_v<int[10][20], 1>); // 20
DebugLog(std::alignment_of_v<float>); // Maybe 4
DebugLog(std::alignment_of_v<double>); // Maybe 8
```

我们还可以获取类型的修改版本：

```c++
#include <type_traits>
 
//我们知道T是一个指针（例如int*），但我们不知道它指向什么（例如int），可以使用std::remove_pointer_t来获取它
template <typename T>
auto Dereference(T ptr) -> std::remove_pointer_t<T>
{
    return *ptr;
}
 
int x = 123;
int* p = &x;
int result = Dereference(p);
DebugLog(result); // 123
```

有一个特别有用的函数是 `std::underlying_type` ，可用于实现安全的枚举之间“转换”函数：

```c++
#include <type_traits>
 
// 从整数到枚举的“转换”
template <typename TEnum, typename TInt>
TEnum FromInteger(TInt i)
{
    // 确保模板参数是枚举和整数
    static_assert(std::is_enum_v<TEnum>);
    static_assert(std::is_integral_v<TInt>);
 
    // 使用 type_traits 中的 is_same_v 来确保 TInt 是 TEnum 的底层类型
    static_assert(std::is_same_v<std::underlying_type_t<TEnum>, TInt>);
 
    // 执行转换
    return static_cast<TEnum>(i);
}
 
// 将枚举类型转换为整数
template <typename TEnum>
auto ToInteger(TEnum e) -> std::underlying_type_t<TEnum>
{
    // 确保模板参数是枚举类型
    static_assert(std::is_enum_v<TEnum>);
 
    // 执行转换
    return static_cast<std::underlying_type_t<TEnum>>(e);
}
 
enum class Color : uint64_t
{
    Red,
    Green,
    Blue
};
 
Color c = Color::Green;
DebugLog(c); // Green
 
// 从枚举类型转换为整数
auto i = ToInteger(c);
DebugLog(i); // 1
 
// 从整数转换为枚举类型
Color c2 = FromInteger<Color>(i);
DebugLog(c2); // Green
```
这些“转换”函数在运行时不会产生任何开销，因为所有的检查都在编译时进行。
</br>然而，它们确实增加了安全性，因为如果我们不小心尝试使用错误大小的类型，将会得到编译器诊断信息：

部分功能在 C#中通过 `Type` 类及其相关的反射类：`FieldInfo`、 `PropertyInfo`等实现。与 C++不同，这些功能在运行时执行，而它们的 C++对应功能在编译时执行。

## Conclusion  结论

C++的某些部分依赖于标准库。我们需要使用 `std::initializer_list` 来处理括号初始化列表，并且需要使用 `std::coroutine_handle` 来实现协程返回对象。
</br>这类似于 C#将.NET API 的部分内容封装到语言中： `Type` 、 `System.Single` 等。

今天我们看到了很多这些准语言特性，以及一些通用的语言支持功能，比如 `source_location` 和许多预定义的概念。这些是语言和库的基础元素，同时也预示了标准库设计方面的未来发展方向。
















