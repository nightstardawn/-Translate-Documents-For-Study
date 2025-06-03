# C++ For C# Developers: Part 25 – Intro to Templates

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5935),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C#泛型（`List<T>`）在许多关键方面看起来与C++模板（`list<T>`）非常相似，但它们在许多关键方面是不同的。
</br>这是一个很大的主题，所以今天我们将从查看模板的一些最常见用途开始：将它们应用于类、函数、成员、lambda表达式和变量。


## What are Templates?  什么是模板？

模板确实如其名所示：某物的模板。当我们谈论模板时，我们说它是“某物的模板(something template)”，而不是“模板某物(template something)”。
</br>这听起来可能像是一个微不足道的区别，甚至是一个常见的错误说法，但实际上有一个重要的区别需要区分。

如果我们谈论的是函数，如果我们说“模板函数”，那么我们使用“模板”作为形容词，好像这就是函数的类型。
</br>这就是我们正确谈论“静态函数”和“成员函数”甚至“静态成员函数”的方式。这些都是形容词，用来明确我们谈论的是哪种函数。

这与模板的情况不同。我们从不编写“模板函数”，而是编写“函数模板”。这通常简称为“函数模板”。
</br>模板可以被用来创建一个函数，该函数可以像任何其他函数一样使用。
</br>这个过程被称为“实例化”。当我们创建一个类类型的对象时，我们使用的是同一个术语：对象是该类的一个实例。

模板实例化总是在编译时进行的。我们可以向模板传递参数，以控制模板如何实例化为运行时实体。这在概念上类似于调用构造函数。

C++对模板的方法与C#对泛型的方法不同。
</br>C#编译器不是在编译时实例化模板，而是生成描述泛型类型和方法的MSIL。然后，运行时在首次使用的地方实例化泛型。
</br>泛型为每个原始类型（`int`、`float`等）实例化一次，对所有引用类型（`string`、`GameObject`等）实例化一次。
</br>这种运行时实现的差异很大。
</br>例如，IL2CPP在编译时实例化泛型，但Microsoft运行时在运行时实例化它们。

考虑到这一点，让我们开始看看我们可以创建的模板类型。

## Variables  变量

也许最简单的模板形式是变量的模板。考虑定义π的情况：

```c++
constexpr float PI_FLOAT = 3.14f;
constexpr double PI_DOUBLE = 3.14;
constexpr int32_t PI_INT32 = 3;
```

这要求我们创建许多具有基本相同值的变量。我们必须为它们想出独特的名字，这给我们的代码增加了很多冗余。

现在让我们看看我们如何使用变量模板来做这件事：

```c++
template<typename T>
constexpr T PI = 3.14;
```

首先，我们从模板关键字和尖括号中的“模板参数”开始：`<typename T>`。
</br>我们将在下一篇文章中深入探讨模板参数的各种选项。现在，我们将使用简单的`typename T`。
</br>这意味着模板接受一个参数，一个名为T的“类型名”。这就像函数的参数一样。我们声明所需参数的类型以及在函数中如何引用它们：`int i`。

在模板之后，我们有模板实例化时创建的东西。在这种情况下，我们有一个名为`PI`的变量。它有权访问模板参数，在这种情况下就是`T`。
</br>在这里，我们将`T`用作变量的类型：`T PI`。就像其他变量一样，我们可以自由地将其声明为`constexpr`并初始化它：`= 3.14`。
</br>我们也可以将其声明为`*`、`&`、`const`，或者使用其他形式的初始化，如`{3.14}`。

现在我们已经定义了一个变量的模板，让我们实例化它，以便我们有一个实际的变量：

```c++
float pi = PI<float>;
DebugLog(pi); // 3.14
```

实例化涉及命名模板（PI）并为所需的模板参数提供参数。
这就像调用函数一样，只是我们使用尖括号（`<>`）而不是括号（`()`）。
我们还传递一个类型名（`float`）而不是一个对象（`3.14`）。

当编译器看到这一点时，它会查看模板（`PI`）并将我们的模板参数（`float`）与模板参数（`T`）匹配起来。
然后，它在模板实体中使用的任何地方替换这些参数。在这种情况下，`T`被替换为`float`，所以我们得到这个：

```c++
constexpr float PI = 3.14;
```

然后，我们将模板的使用替换为实例化模板的使用，所以我们得到这个：

```c++
float pi = PI;
DebugLog(pi); // 3.14
```

回顾一下原始示例，其中我们使用了`double`和`int32_t`版本的π，现在我们可以用`PI`模板来替换这些用法：

```c++
float pif = PI<float>;
DebugLog(pif); // 3.14
 
double pid = PI<double>;
DebugLog(pid); // 3.14
 
int32_t pii = PI<int32_t>;
DebugLog(pii); // 3
```

这个例子将 PI 变量模板实例化了三次。
</br>第一次实例化将 `float` 作为 `T` 的参数传递
</br>第二次和第三次实例化将 `double` 和 `int32_t` 传递。
</br>这导致编译器生成三个变量：

这里有两个明显的问题。

首先，我们正在用`3.14`初始化一个`int32_t`。这是可以的，因为根据初始化规则，`3.14`将被截断为`3`。
</br>如果我们不希望这种行为，我们可以使用`constexpr T PI{3.14}`，那么`PI<int32_t>`就会导致编译器错误，因为不允许`int32_t{3.14}`。

其次，显然我们有三个名为PI的变量，因此存在命名冲突。
</br>编译器介入并生成唯一的名称。它可能会将它们命名为PI_FLOAT、PI_DOUBLE和PI_INT32，或者它认为合适的任何其他名称，只要它们是唯一的。

这就是当我们重载函数时发生的相同过程：编译器为这些函数生成唯一的名称。
</br>当我们引用函数时，比如通过调用它，编译器确定我们调用的是哪一个，并将 `Foo()` 替换为 `Foo_Void()` 或它为该函数命名的任何名称。
</br>在模板中，编译器将 `PI<float>` 替换为 `PI_FLOAT`。

最后，我们可以显式实例化一个变量模板，而不需要为任何特定目的使用它：

```c++
template constexpr float PI<float>;
```

这通常用于不需要使用模板但希望确保将其编译为静态或动态库的库，以便在链接时可供用户使用。

## Functions  函数

函数模板可以被实例化以生成一个函数，就像变量模板可以被实例化以生成一个变量一样。例如，这里有一个用于返回两个参数中最大值的函数模板：

```c++
// 函数模板
template<typename T>
T Max(T a, T b)
{
    return a > b ? a : b;
}
 
// int 
int maxi = Max<int>(2, 4);
DebugLog(maxi); // 4
 
// float 
float maxf = Max<float>(2.2f, 4.4f);
DebugLog(maxf); // 4.4
 
// double 
double maxd = Max<double>(2.2, 4.4);
DebugLog(maxd); // 4.4
```

再次我们看到以`template<typename T>`开始的模板。
</br>之后，我们写了一个函数而不是变量。这个函数可以访问类型名参数T。它将T用作两个参数的类型以及返回值的类型。

然后我们看到Max模板的三种实例化：`Max<int>`、`Max<float>`和`Max<double>`。
</br>就像变量一样，编译器通过将模板参数T（`int`、`float`或`double`）替换到函数模板中T被使用的任何地方，实例化了三个函数。

```c++
int MaxInt(int a, int b)
{
    return a > b ? a : b;
}
 
float MaxFloat(float a, float b)
{
    return a > b ? a : b;
}
 
double MaxDouble(double a, double b)
{
    return a > b ? a : b;
}
```

然后，导致这种实例化的三个函数调用被替换为对实例化函数的调用：

```c++
// int 
int maxi = MaxInt(2, 4);
DebugLog(maxi); // 4
 
// float 
float maxf = MaxFloat(2.2f, 4.4f);
DebugLog(maxf); // 4.4
 
// double 
double maxd = MaxDouble(2.2, 4.4);
DebugLog(maxd); // 4.4
```

此外，就像任何模板一样，我们不仅限于原始类型。我们可以使用任何类型：

```c++
struct Vector2
{
    float X;
    float Y;
 
    bool operator>(const Vector2& other) const
    {
        return X > other.X && Y > other.Y;
    }
};
 
// Vector2 
Vector2 maxv = Max<Vector2>(Vector2{4, 6}, Vector2{2, 4});
DebugLog(maxv.X, maxv.Y); // 4, 6
```

这里的含义是模板对其参数提出了先决条件。
</br>`Max` 模板要求存在一个 `T > T` 操作符。
</br>这 int、float 和 double 肯定满足，但我们需要编写一个重载的 `Vector2 > Vector2` 操作符，以便它能够与 `Vector2` 一起使用。
</br>如果没有这个操作符，我们会得到编译器错误：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
template <typename T>
T Max(T a, T b)
{
    // 编译器错误：
    // "二进制运算操作数无效（Vector2 和 Vector2）"
    return a > b ? a : b;
}
 
Vector2 maxv = Max<Vector2>(Vector2{4, 6}, Vector2{2, 4});
DebugLog(maxv.X, maxv.Y); // 4, 6
```

另一个选项，正如我们之前所看到的，在C++20中，我们可以隐式创建使用`auto`参数、`auto`返回类型或两者都使用的函数模板。这些被称为“简化的函数模板”：

```c++
// 简化的函数模板
auto Max(auto a, auto b)
{
    return a > b ? a : b;
}
 
// 用法相同
Vector2 maxv = Max<Vector2>(Vector2{4, 6}, Vector2{2, 4});
DebugLog(maxv.X, maxv.Y); // 4, 6
```

最后，可以这样显式实例化函数模板：

```c++
template bool IsOrthogonal<Vector2>(Vector2, Vector2);
```

## Classes  类

我们可以创建的下一类模板是类、结构体或联合体的模板。与变量和函数一样，我们以template<params>开始，然后编写一个类：

```c++
template<typename T>
struct Vector2
{
    T X;
    T Y;
 
    T Dot(const Vector2<T>& other) const
    {
        return X*other.X + Y*other.Y;
    }
};
```

注意这个类是如何使用模板参数T来代替像float这样的特定类型的。
</br>这可以在任何原本需要特定类型的地方使用，比如数据成员的类型，如X和Y，成员函数的返回类型，如Dot，或者参数，如other。

这就是我们如何实例化这个类模板来创建几种不同类型的向量：

```c++
Vector2<float> v2f{0, 1};
DebugLog(v2f.X, v2f.Y); // 0, 1
DebugLog(v2f.Dot({1, 0})); // 0
 
Vector2<double> v2d{0, 1};
DebugLog(v2d.X, v2d.Y); // 0, 1
DebugLog(v2d.Dot({1, 0})); // 0
 
Vector2<int32_t> v2i{0, 1};
DebugLog(v2i.X, v2i.Y); // 0, 1
DebugLog(v2i.Dot({1, 0})); // 0
```

编译器实例化的Vector2类如下：

```c++
struct Vector2Float
{
    float X;
    float Y;
 
    float Dot(const Vector2Float& other) const
    {
        return X*other.X + Y*other.Y;
    }
};
 
struct Vector2Double
{
    double X;
    double Y;
 
    double Dot(const Vector2Double& other) const
    {
        return X*other.X + Y*other.Y;
    }
};
 
struct Vector2Int32
{
    int32_t X;
    int32_t Y;
 
    int32_t Dot(const Vector2Int32& other) const
    {
        return X*other.X + Y*other.Y;
    }
};
```

然后，将类模板的使用替换为这些实例化类的使用

```c++
Vector2Float v2f{0, 1};
DebugLog(v2f.X, v2f.Y); // 0, 1
DebugLog(v2f.Dot({1, 0})); // 0
 
Vector2Double v2d{0, 1};
DebugLog(v2d.X, v2d.Y); // 0, 1
DebugLog(v2d.Dot({1, 0})); // 0
 
Vector2Int32 v2i{0, 1};
DebugLog(v2i.X, v2i.Y); // 0, 1
DebugLog(v2i.Dot({1, 0})); // 0
```

请注意，`Dot` 调用之所以特别紧凑且仅因为到目前为止我们看到的几个规则而有效。
</br>以 `v2f` 为例，它是一个 `Vector2<float>` 类型的对象。当我们调用 `v2f.Dot({1, 0})` 时，编译器会查看它作为 `Vector2` 模板一部分实例化的 Dot。
</br>这个 `Dot` 接受一个 `const Vector2<float>&` 参数，因此编译器将 `{1, 0}` 解释为 `Vector2<float>` 的聚合初始化。
</br>由于 `{0, 1}` 没有名称，这个 `Vector2<float>` 是一个右值。它可以传递给一个 `const lvalue` 引用，并且它的生命周期会延长到 `Dot` 返回之后。

## Members  成员

类可以包含成员函数的模板：

```c++
struct Vector2
{
    float X;
    float Y;
 
    template<typename T>
    bool IsNearlyZero(T threshold) const
    {
        return X < threshold && Y < threshold;
    }
};
```

尽管`Vector2`不是一个模板，但它可以包含成员函数模板。我们就像使用普通成员函数一样使用它：

```c++
Vector2 vec{0.5f, 0.5f};
 
// Float
DebugLog(vec.IsNearlyZero(0.6f)); // true
 
// Double
DebugLog(vec.IsNearlyZero(0.1)); // false
 
// Int
DebugLog(vec.IsNearlyZero(1)); // true
```

这导致编译器三次实例化`IsNearlyZero`：

```c++
struct Vector2
{
    float X;
    float Y;
 
    bool IsNearlyZeroFloat(float threshold) const
    {
        return X < threshold && Y < threshold;
    }
 
    bool IsNearlyZeroDouble(double threshold) const
    {
        return X < threshold && Y < threshold;
    }
 
    bool IsNearlyZeroInt(int threshold) const
    {
        return X < threshold && Y < threshold;
    }
};
```

然后，成员函数模板的调用被替换为对实例化函数的调用：

```c++
Vector2 vec{0.5f, 0.5f};
 
// Float
DebugLog(vec.IsNearlyZeroFloat(0.6f)); // true
 
// Double
DebugLog(vec.IsNearlyZeroDouble(0.1)); // false
 
// Int
DebugLog(vec.IsNearlyZeroInt(1)); // true
```

我们还可以为静态成员变量编写模板：

```c++
struct HealthRange
{
    template<typename T>
    constexpr static T Min = 0;
 
    template<typename T>
    constexpr static T Max = 100;
};
```

它们的使用方式与其他静态成员变量相同：

```c++
float min = HealthRange::Min<float>;
int32_t max = HealthRange::Max<int32_t>;
DebugLog(min, max); // 0, 100
```

正如预期的那样，编译器会像这样实例化这些模板：

```c++
struct HealthRange
{
    constexpr static float MinFloat = 0;
    constexpr static int32_t MaxInt = 100;
};
```

然后替换了成员变量模板的使用，用这些实例化的成员变量：
```c++
float min = HealthRange::MinFloat;
int32_t max = HealthRange::MaxInt;
DebugLog(min, max); // 0, 100
```

最后，我们可以为成员类编写模板：

```c++
struct Math
{
    template<typename T>
    struct Vector2
    {
        T X;
        T Y;
    };
};
```

然后可以像普通成员类一样使用这些类：

```c++
Math::Vector2<float> v2f{2, 4};
DebugLog(v2f.X, v2f.Y); // 2, 4
 
Math::Vector2<double> v2d{2, 4};
DebugLog(v2d.X, v2d.Y); // 2, 4
```

然后，编译器执行通常的实例化和替换：

```c++
struct Math
{
    struct Vector2Float
    {
        float X;
        float Y;
    };
 
    struct Vector2Double
    {
        double X;
        double Y;
    };
};
 
Math::Vector2Float v2f{2, 4};
DebugLog(v2f.X, v2f.Y); // 2, 4
 
Math::Vector2Double v2d{2, 4};
DebugLog(v2d.X, v2d.Y); // 2, 4
```

## Lambdas  Lambda表达式

由于lambda是编译器生成的类，它们的重载运算符`()`也可以是模板化的。在C++20之前，需要使用基于`auto`参数和返回值的“简化的函数模板”语法：

```c++
// LambdaClass运算符的“简化的函数模板”
auto madd = [](auto x, auto y, auto z) { return x*y + z; };
 
// 使用float实例化
DebugLog(madd(2.0f, 3.0f, 4.0f)); // 10
 
// 使用int实例化
DebugLog(madd(2, 3, 4)); // 10
```

编译器生成的lambda类将看起来像这样：

```c++
struct Madd
{
    // 简化的函数模板
    auto operator()(auto x, auto y, auto z) const
    {
        return x*y + z;
    }
};
```

然后，对其 operator（） 的两次调用会导致缩写函数模板实例化为重载集：

```c++
struct Madd
{
    float operator()(float x, float y, float z) const
    {
        return x*y + z;
    }
 
    int operator()(int x, int y, int z) const
    {
        return x*y + z;
    }
};
```

编译器然后将lambda语法替换为这些类的实例化和对它们的`operator()`的调用：

```c++
Madd madd{};
 
// Call (float, float, float) overload of operator()
DebugLog(madd(2.0f, 3.0f, 4.0f)); // 10
 
// Call (int, int, int) overload of operator()
DebugLog(madd(2, 3, 4)); // 10
```

从C++20开始，我们可以使用正常的、非缩写的模板语法风格。编译器使用这个版本的代码生成的代码与使用“缩写”语法的代码完全相同：

```c++
// 显式模板的Lambda和“尾随返回类型”
auto madd = []<typename T>(T x, T y, T z) -> T { return x*y + z; };
 
// float
DebugLog(madd(2.0f, 3.0f, 4.0f)); // 10
 
// int
DebugLog(madd(2, 3, 4)); // 10
```

## C# Equivalency  C# 等效性

正如我们所见，C++在泛型编程方面的做法与C#大不相同。 它的模板不仅可以应用于类、结构体和成员函数，还可以应用于lambda表达式、变量、函数、成员变量和成员函数。
</br>除了lambda表达式之外，所有这些都可以通过将它们包装在静态类中来在C#中模拟：

```csharp
// C#
public static class Wrapper<T>
{
    // 变量
    public static readonly T Default = default(T);
 
    // 函数
    public static string SafeToString(T obj)
    {
        return object.ReferenceEquals(obj, null) ? "" : obj.ToString();
    }
}
 
// 变量
DebugLog(Wrapper<float>.Default); // 0
 
// 函数
DebugLog(Wrapper<Player>.SafeToString(new Player())); // "Player 1"
DebugLog(Wrapper<Player>.SafeToString(null)); // ""
```

当我们考虑我们可以对类型参数做什么时，主要的区别就出现了。默认情况下，C#中的类型参数被处理成`System.Object`/`object`。
</br>这意味着它几乎没有任何超出上面我们使用的`ToString()`和`default(T)`表达式等基本功能的功能。

我们可以使用where约束来添加限制，以启用更多功能，但这非常有限。
</br>有一些约束，如`where T : new()`允许我们调用默认构造函数或`where T : class`允许我们使用null
</br>但大多数情况下，我们使用`where T : ISomeInterface`或`where T : SomeBaseClass`来启用对接口或基类中虚拟函数的调用。

让我们尝试将上述 `Vector2` 示例之一从 C++ 移植到 C#：

```c++
template<typename T>
struct Vector2
{
    T X;
    T Y;
 
    T Dot(const Vector2<T>& other) const
    {
        return X*other.X + Y*other.Y;
    }
};
```

首先，语法的直接翻译如下：

```csharp
// C#
public struct Vector2<T>
{
    public T X;
    public T Y;
 
    public T Dot(in Vector2<T> other)
    {
        return X*other.X + Y*other.Y;
    }
}
```

除了需要将`Dot`设置为非const，因为在C#中不支持这一点，没有太多变化。
</br>问题是现在我们在`X*other.X`、`Y*other.Y`以及它们之间的`+`运算符上遇到了编译器错误：


编译器将`T`视为对象，因此`object*object`和`object+object`都是编译错误。我们只使用像`float`和`double`这样的类型，这些类型支持`*`和`+`运算符，但这并不重要。
</br>编译器坚持认为所有可能的`T`类型都必须支持`T*T`和`T+T`。由于像`Player`这样的类型不支持这一点，所以我们得到了编译错误。

因此，我们需要添加一个`where`约束来限制`T`只属于支持`*`和`+`操作符的类型子集。
</br>查看[选项列表](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)，我们没有看到类似`where T : T*T`或`where T : T+T`的东西。
</br>我们唯一的选项是避免使用`*`和`+`，而是调用实现接口或基类中的名为`Multiply`和`Add`的虚函数，因为我们可以为这些函数编写`where`约束。
</br>下面是这样一个接口：

```csharp
// C#
public interface IArithmetic<T>
{
    T Multiply(T a, T b);
    T Add(T a, T b);
}
```

这里有一个为浮点数实现它的结构体：

```csharp
// C#
public struct FloatArithmetic : IArithmetic<float>
{
    public float Multiply(float a, float b)
    {
        return a * b;
    }
 
    public float Add(float a, float b)
    {
        return a + b;
    }
}
```

现在我们可以将 `FloatArithmetic` 传递给 `Vector2`，以便它调用 `IArithmetic.Multiply` 和 `IArithmetic.Add`，而不是使用内置的 `*` 和 `+` 运算符：

```csharp
// C#
public struct Vector2<T, TArithmetic>
    where TArithmetic : IArithmetic<T>
{
    public T X;
    public T Y;
    private TArithmetic Arithmetic;
 
    public Vector2(T x, T y, TArithmetic arithmetic)
    {
        X = x;
        Y = y;
        Arithmetic = arithmetic;
    }
 
    public T Dot(Vector2<T, TArithmetic> other)
    {
        T xProduct = Arithmetic.Multiply(X, other.X);
        T yProduct = Arithmetic.Multiply(Y, other.Y);
        return Arithmetic.Add(xProduct, yProduct);
    }
}
```

以下是使用此方法的示例：

```csharp
// C#
var vecA = new Vector2<float, FloatArithmetic>(1, 0, default);
var vecB = new Vector2<float, FloatArithmetic>(0, 1, default);
DebugLog(vecA.Dot(vecB)); // 0
```

虽然这种设计可行，但它引发了一些问题。
</br>首先，我们在 `IArithmetic`、`FloatArithmetic` 中有很多样板代码，泛型中多了额外的类型参数（`<T, TArithmetic>` 而不是仅仅 `<T>`），以及构造函数中多了一个额外的算术参数。
</br>这影响了生产力和可读性，但至少这不是编译器生成的可执行文件中翻译成的问题。

第二个问题是，由于我们的`Vector2`包含了算术字段，其大小已经增加。这是一个指向`IArithmetic`的托管引用。在64位CPU上，它至少占用8个字节。
</br>由于`X`和`Y`各自占用4个字节，`Vector2`的大小翻倍了。这将影响内存使用，也许更重要的是，缓存利用率，因为现在只有一半的向量可以放入缓存行。

第三个问题是，`FloatArithmetic`需要从结构体“装箱”到受管理的`IArithmetic`引用。这将创建垃圾，供垃圾收集器稍后收集。
</br>在上面的例子中，每次调用`Vector2`构造函数时都会发生这种情况。这种延迟的性能成本可能会导致帧跳变或其他问题。

为了避免装箱，我们可以从结构体切换到类，并共享一个全局实例：

```csharp
// C#
public class FloatArithmetic : IArithmetic<float>
{
    public static readonly FloatArithmetic Default = new FloatArithmetic();
 
    public float Multiply(float a, float b)
    {
        return a * b;
    }
 
    public float Add(float a, float b)
    {
        return a + b;
    }
}
 
var vecA = new Vector2<float, FloatArithmetic>(1, 0, FloatArithmetic.Default);
var vecB = new Vector2<float, FloatArithmetic>(0, 1, FloatArithmetic.Default);
DebugLog(vecA.Dot(vecB)); // 0
```

这又提出了另一个性能问题：我们读取`FloatArithmetic.Default`时可能会从“冷”内存中读取，即不在CPU缓存中的内存。

第四个也是最后一个问题是，`Dot` 中的 `Arithmetic.Multiply` 和 `Arithmetic.Add` 调用是虚函数调用，因为接口中的所有函数都是隐式虚函数。
</br>在某些情况下，编译器能够确定 `TArithmetic` 是 `FloatArithmetic`，并将调用“去虚化”。
</br>在许多其他情况下，我们将在每个 `Dot` 上承受三个虚函数调用的[运行时开销](https://www.jacksondunstan.com/articles/5490)。

另一种方法是使用一个实现了乘法和加法接口的类来包装浮点值：

```csharp
// C#
public interface INumeric<T>
{
    INumeric<T> Create(T val);
    T Value { get; set; }
    INumeric<T> Multiply(T val);
    INumeric<T> Add(T val);
}
 
public class FloatNumeric : INumeric<float>
{
    public float Value { get; set; }
 
    public FloatNumeric(float val)
    {
        Value = val;
    }
 
    public INumeric<float> Create(float val)
    {
        return new FloatNumeric(val);
    }
 
    public INumeric<float> Multiply(float val)
    {
        return Create(Value * val);
    }
 
    public INumeric<float> Add(float val)
    {
        return Create(Value + val);
    }
}
```

现在`Vector2`可以持有`INumeric`字段并调用其虚拟函数：

```csharp
// C#
public struct Vector2<T, TNumeric>
    where TNumeric : INumeric<T>
{
    public TNumeric X;
    public TNumeric Y;
 
    public Vector2(TNumeric x, TNumeric y)
    {
        X = x;
        Y = y;
    }
 
    public T Dot(Vector2<T, TNumeric> other)
    {
        INumeric<T> xProduct = X.Multiply(other.X.Value);
        INumeric<T> yProduct = Y.Multiply(other.Y.Value);
        INumeric<T> sum = xProduct.Add(yProduct.Value);
        return sum.Value;
    }
}
 
var vecA = new Vector2<float, FloatNumeric>(
    new FloatNumeric(1),
    new FloatNumeric(0)
);
var vecB = new Vector2<float, FloatNumeric>(
    new FloatNumeric(0),
    new FloatNumeric(1)
);
DebugLog(vecA.Dot(vecB)); // 0
```

这和之前的方法有相同的问题：为每个创建的FloatNumeric产生垃圾，调用Add和Multiply的虚拟函数，多余的类型参数，样板代码等。
</br>在这种情况下，许多C#程序员会放弃，并像C++编译器那样手动“实例化”Vector2模板：

```csharp
// C#
public struct Vector2Float
{
    public float X;
    public float Y;
 
    public float Dot(in Vector2Float other)
    {
        return X*other.X + Y*other.Y;
    }
}
 
public struct Vector2Double
{
    public double X;
    public double Y;
 
    public double Dot(in Vector2Double other)
    {
        return X*other.X + Y*other.Y;
    }
}
 
public struct Vector2Int
{
    public int X;
    public int Y;
 
    public int Dot(in Vector2Int other)
    {
        return X*other.X + Y*other.Y;
    }
}
 
var vecA = new Vector2Float{X=1, Y=0};
var vecB = new Vector2Float{X=0, Y=1};
DebugLog(vecA.Dot(vecB)); // 0
```

这是高效的，但现在却遭受了代码重复的常见问题：
</br>需要更改多个副本，副本不同步时的错误等。为了解决这个问题，我们可能会转向使用某种模板来生成`.cs`文件的[代码生成工具](https://www.jacksondunstan.com/articles/4959)。
</br>这可能在更早的构建步骤中运行，但它不会集成到主代码库中，可能需要额外的语言，仍然需要唯一的命名，以及各种其他问题。

C++避免了代码重复、外部工具、虚函数调用、冷内存读取、装箱和垃圾回收、需要接口及其样板实现、额外的类型参数以及where约束。
</br>相反，当T*T或T+T是语法错误时，它简单地产生编译器错误。

## Conclusion  结论

我们刚刚触及了模板的表面，就已经发现它们比C#泛型强大得多。我们可以轻松编写像Dot这样的代码，它同时具有高效、易读和泛型的特点。
</br>C#在处理甚至这样的简单示例时都会遇到困难，并且通常需要我们牺牲一个或多个这些品质。

下周的文章将继续探讨模板，通过研究更高级的功能，如模板“特化”和默认参数。敬请期待！

> 这里有作者的额外两篇文章：等C++完结之后再做阅读
> </br>[代码生成工具](https://www.jacksondunstan.com/articles/4959)
> </br>[运行时开销](https://www.jacksondunstan.com/articles/5490)













