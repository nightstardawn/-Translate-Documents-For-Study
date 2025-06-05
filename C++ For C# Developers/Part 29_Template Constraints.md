# C++ For C# Developers: Part 29 – Template Constraints


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6058),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C#中的约束(where)使我们的泛型能够做更多的事情。
C++也有约束，它们使我们能够编写更具有表现力和更高效的代码。今天我们将看到如何向我们的模板添加一些约束来实现这些目标。

## Constraints  约束

C#有11种特定的`where`[约束](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)，我们可以将其应用于泛型类型参数。这些约束包括如where T : new()这样的约束，表示T有一个不带参数的公共构造函数。
</br>相比之下，C++为我们提供了构建自己约束的工具，这些工具是基于编译时表达式(Part23)的。

到目前为止，在C++中，我们所有的模板都没有约束。然而，我们仍然能够以许多方式使用这些参数。
</br>这两种语言之间的区别在于，C#的默认做法是将泛型参数视为最小公倍数类型：`System.Object`/`object`。
</br>而C++的默认做法是光谱的另一端：模板参数可以以任何编译的方式使用。

两种语言都允许我们设置约束，将参数的要求更多地移向光谱的另一端。
</br>这意味着在C#中，约束使得泛型的类型参数更加具体，因此我们可以使用更具体的功能，比如调用无参数的构造函数。
</br>我们将看到C++模板约束如何使模板参数不那么具体，以实现更好的重载解析并提供更友好的编译器错误信息。所有这些都是在C++20中添加的，并且正在所有主要编译器中[提供](https://en.cppreference.com/w/cpp/compiler_support.html)。

## Requires Clauses  Requires 子句

虽然C#使用关键字`where`来添加约束，但C++使用`requires`。让我们直接跳到函数模板中添加一个约束：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
// 变量模板
// 默认值是false
template <typename T>
constexpr bool IsVector2 = false;
 
// 变量模板的专用化
// 将值更改为true以指定特定类型
template <>
constexpr bool IsVector2<Vector2> = true;
 
// 函数模板
template <typename TVector, typename TComponent>
// 需要子句
// 编译时表达式评估为 bool
// 可以在这里使用模板参数
requires IsVector2<TVector>
// 函数
TComponent Dot(TVector a, TVector b)
{
    return a.X*b.X + a.Y*b.Y;
}
 
// OK
Vector2 vecA{2, 4};
Vector2 vecB{2, 4};
DebugLog(Dot<Vector2, float>(vecA, vecB));
 
// 编译器错误：
// 候选模板被忽略：约束未满足
// [with TVector = int, TComponent = int]
// TComponent Dot(TVector a, TVector b)
//            ^
// test.cpp:60:10: note: 因为 'IsVector2<int>' 评估为 false
// 需要 IsVector2<TVector>
DebugLog(Dot<int, int>(2, 4));
```

我们是在`requires`关键字之后指定任何编译时表达式,不是像在C#中使用的语言指定的`where`约束。
在这种情况下，我们使用了一个默认为`false`的变量模板，然后将其特化以将特定的`Vector2`类型选择为`true`。

这些编译时表达式可以访问模板参数。与其他编译时表达式一样，它们允许任意复杂。例如，考虑这个没有`requires`子句的变量模板：


```c++
// 变量模板
template <typename T, int N>
// 递归到下一个更低的值
constexpr T SumUpToN = N + SumUpToN<T, N-1>;
 
// 0停止递归的专门化
template <typename T>
constexpr T SumUpToN<T, 0> = 0;
 
// OK
DebugLog(SumUpToN<float, 3>); // 6
 
// 编译错误：
// 
// test.cpp:44:28: fatal error: recursive template instantiation exceeded
// maximum depth of 1024
// constexpr T SumUpToN = N + SumUpToN<T, N-1>;
//                            ^
// test.cpp:44:28: note: in instantiation of variable template specialization
// 'SumUpToN<float, -1025>' requested here
// test.cpp:44:28: note: in instantiation of variable template specialization
// 'SumUpToN<float, -1024>' requested here
// test.cpp:44:28: note: in instantiation of variable template specialization
// 'SumUpToN<float, -1023>' requested here
// test.cpp:44:28: note: in instantiation of variable template specialization
// 'SumUpToN<float, -1022>' requested here
// test.cpp:44:28: note: in instantiation of variable template specialization
// 'SumUpToN<float, -1021>' requested here
// 
// ... many more lines of errors
DebugLog(SumUpToN<float, -1>);
```

为了停止无限递归，我们可以添加一个`requires`子句：

```c++
template <typename T, int N>
// 仅约束为正值
requires (N >= 0)
constexpr T SumUpToN = N + SumUpToN<T, N-1>;
 
template <typename T>
constexpr T SumUpToN<T, 0> = 0;
 
// OK
DebugLog(SumUpToN<float, 3>); // 6
 
// 编译错误：
// 
// test.cpp:54:14: error: constraints not satisfied for variable template
// 'SumUpToN' [with T = float, N = -1]
//     DebugLog(SumUpToN<float, -1>);
//              ^~~~~~~~~~~~~~~~~~~
// test.cpp:42:11: note: because '-1 >= 0' (-1 >= 0) evaluated to false
// requires (N >= 0)
DebugLog(SumUpToN<float, -1>);
```

我们已经通过`requires`约束成功阻止了无限递归的开始。
</br>编译器不需要实例化成千上万的模板，也不需要打印出成千上万的错误信息供我们解析。相反，我们只得到一条可读的消息，告诉我们约束未满足。

## Concepts  概念

到目前为止，`requires` 一直是一个相当有限的工具。这是因为这不是使用它的主要方式。
</br>相反，我们通常使用 requires 来定义一个“概念(Concepts)”。一个概念应该是对类型类别的一种语义描述。
</br>例如，我们可能会定义一个 `Number` 概念为一个支持各种数值运算的类型：

```c++
template <typename T>
concept Number = requires(T t) {
    t + t;
    t - t;
    t * t;
    t / t;
    -t;
    +t;
    --t;
    ++t;
    t--;
    t++;
};
```

这是对`requires`的新用法。

首先，我们添加了一个类似于函数的参数列表。 这给我们提供了一个名为`t`的参数，其类型与模板参数`T`相匹配。

其次，我们随后使用该参数在一系列语句中，类似于函数的行为。如果这些语句能够编译，则约束条件得到满足。

最后，我们使用`concept`关键字保存由`requires`创建的约束。这让我们可以在需要使用它的时候给约束命名。

所有`concept`都是接受类型参数的模板，因为它们的目的是对类型进行分类。

现在让我们来使用这个`concept`：

```c++
// 具有两个类型参数的函数模板（后者已默认）
template <typename TVal, typename TThreshold=TVal>
// 需要子句名称概念
requires Number<TVal> && Number<TThreshold>
// 函数
bool IsNearlyZero(TVal val, TThreshold threshold)
{
    return (val < 0 ? -val : val) < threshold;
}
 
// 所有这些都可以，因为double、float和int都满足Number
DebugLog(IsNearlyZero(0.0, 0.1)); // true
DebugLog(IsNearlyZero(0.2f, 0.1f)); // false
DebugLog(IsNearlyZero(2, 1)); // true
 
struct Player{};
 
// 编译错误：玩家不满足Number约束
DebugLog(IsNearlyZero(Player{}, Player{}));
```

这种使用`requires`的方式不是直接评估为`bool`，而是命名了`Number`概念并将其传递给它：`Number<TVal>`和`Number<TThreshold>`。
`requires`子句仍然可以使用`&&`和`||`运算符进行二进制逻辑运算，以组合`concept`或检查`bool`。

我们可以使用几种不同的语法。首先，我们可以在参数列表之后放置`requires`子句：

```c++
template <typename TVal, typename TThreshold=TVal>
bool IsNearlyZero(TVal val, TThreshold threshold)
    requires Number<TVal> && Number<TThreshold>
{
    return (val < 0 ? -val : val) < threshold;
}
```

如果我们有一个简单的`requires`子句，它仅仅命名了一个单一的概念，这是非常典型的，我们可以用概念的名字来替换`typename`：

```c++
// 普通concept版本（仍然使用typename）
template <typename T>
bool IsNearlyZero(T val, T threshold)
    requires Number<T>
{
    return (val < 0 ? -val : val) < threshold;
}
 
// 将typename替换为concept
template <Number T>
bool IsNearlyZero(T val, T threshold)
{
    return (val < 0 ? -val : val) < threshold;
}
```

如果我们使用`auto`参数来创建“简化的函数模板”，那么我们就不会使用模板。相反，我们只需在`auto`之前放置概念名称，以要求参数满足该模板：

```c++
bool IsNearlyZero(Number auto val, Number auto threshold)
{
    return (val < 0 ? -val : val) < threshold;
}
```

无论我们选择哪种语法，用法总是与上面相同，因为这些语法都是等价的。对语法的偏好主要取决于我们需要灵活性的程度和风格。

因为这个形式的`requires`定义了一个`concept`，而`concept`是根据另一种形式的`requires`来命名的，所以我们有时使用`requires requires`来定义一个临时的概念：

```c++
template <typename T>
// 临时concept
requires requires(T t) {
    t + t;
    t - t;
    t * t;
    t / t;
    -t;
    +t;
    --t;
    ++t;
    t--;
    t++;
}
bool IsNearlyZero(T val, T threshold)
{
    return (val < 0 ? -val : val) < threshold;
}
```

请注意，这个Number概念相当不完整，仅用于示例目的。
</br>C++标准库中有许多设计良好的概念，例如`std::integral`和`std::floating_point`，这些概念适合用于生产代码。

## Combining Concepts  组合概念

许多`concept`是根据其他`concept`来定义的。就像我们在`requires`子句中使用`&&`一样，我们也可以在定义`concept`时这样做：

```c++
// 用另一个concept和临时concept来定义一个concept
template <typename T>
concept Integer = Number<T> && requires(T t) {
    t << t;
    t <<= t;
    t >> t;
    t >>= t;
    t % t;
};
```

我们也可以在我们的concepts定义中使用concepts。这些被称为“嵌套概念(Combining Concepts)”：

```c++
template <typename T>
concept Vector2 = requires(T t) {
    // 从concept内部使用一个concept
    // 这要求t.X的类型满足数字约束
    Number<decltype(t.X)>;
 
    // 也要求Y是一个数字
    Number<decltype(t.Y)>;
};
 
struct Vector2f
{
    float X;
    float Y;
};
 
bool IsOrthogonal(Vector2 auto a, Vector2 auto b)
{
    return (a.X*b.X + a.Y*b.Y) == 0;
}
 
Vector2f a{0, 1};
Vector2f b{1, 0};
Vector2f c{1, 1};
DebugLog(IsOrthogonal(a, b)); // true
DebugLog(IsOrthogonal(a, c)); // false
```

或者，我们可以使用“复合要求(compound requirements)”来隐式地将表达式求值得到的类型作为第一个参数传递给`concept`。下面是`Vector2`概念如何使用这个方法的示例：

```c++
template <typename T>
concept Vector2 = requires(T t) {
    // t.X必须评估为满足数字约束的类型
    {t.X} -> Number;
 
    {t.Y} -> Number;
};
```

## Overload Resolution  重载解决

我们已经看到了如何使用专业化(Part27)来编写模板的定制版本。这通常适用于特定类型，如`float`，但对于整个类型类别进行特化则很困难。
这就是`concept`发挥作用的地方。在调用重载函数时，编译器会查看`concept`以找到受约束最严格的函数：

```c++
// 动态数组类的定义不完整
struct List
{
    int Length;
    int* Array;
 
    int* GetBegin()
    {
        return Array;
    }
 
    int& operator[](int i)
    {
        return Array[i];
    }
};
 
// 链表类定义不完整
struct LinkedList
{
    struct Node
    {
        int Value;
        Node* Next;
    };
 
    Node* Head;
 
    Node* GetBegin()
    {
        return Head;
    }
};
 
// 定义可迭代类型的概念
template <typename T>
concept Iterable = requires(T t) {
    t.GetBegin();
};
 
// 一个定义可以索引的类型的概念
// 这比仅仅可迭代的约束更严格
template <typename T>
concept Indexable = Iterable<T> && requires(T t) {
    t[0]; // 可以从索引中读取
    t[0] = 0; // 可以写入索引
};
 
// 可索引的重载简单索引：O(1)
int GetAtIndex(Indexable auto collection, int index)
{
    return collection[index];
}
 
// 可迭代版本必须遍历列表：O(N)
int GetAtIndex(Iterable auto collection, int index)
{
    auto cur = collection.GetBegin();
    for (int i = 0; i < index; ++i)
    {
        cur = cur->Next;
    }
    return cur->Value;
}
 
// 重载解析调用可索引版本
List list;
int a = GetAtIndex(list, 1000);
 
// 重载解析调用可迭代的版本
LinkedList linkedList;
int b = GetAtIndex(linkedList, 1000);
```

## C# Equivalency  C# 等效性

现在我们知道了C++中约束的工作原理，让我们看看如何近似C#提供的11个[`where`约束](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)。
</br>为此，我们将使用C++标准库中`<concepts>`头文件中的一些预定义`concept`和`<type_traits>`中的变量模板，而不是编写我们自己的版本。

| C# Constraint          | 	C++ Concept (approximation)                                                       |
|------------------------|------------------------------------------------------------------------------------|
| `where T : struct`	    | `template <class T> concept C = std::is_class_v<T>;`                               |
| `where T : class`	     | `template <class T> concept C1 = !Nullable<T> && std::is_class_v<T>`               |
| `where T : class?`	    | `template <class T> concept C = std::is_class_v<T>;`                               |
| `where T : notnull`	   | `template <class T> concept C = !Nullable<T>;`                                     |
| `where T : unmanaged`  | 	N/A. All C++ types are unmanaged.                                                 |
| `where T : new()`	     | `std::default_initializable<T>`                                                    |
| `where T : BaseClass`  | 	`template <class T> concept C = !Nullable<T> && std::derived_from<T, BaseClass>;` |
| `where T : BaseClass?` | 	`std::derived_from<T, BaseClass>`                                                 |
| `where T : Interface`  | 	`template <class T> concept C = !Nullable<T> && std::derived_from<T, BaseClass>;` |
| `where T : Interface?` | 	`std::derived_from<T, BaseClass>`                                                 |
| `where T : U`	         | `std::derived_from<T, U>`                                                          |


在上面的表中，`std::derived_from` 和 `std::default_initializable` 是`concept`，而 `std::is_class_v` 是一个bool变量模板。
`Nullable` 不在 C++ 标准库中，但它可能看起来像这样：

```c++
template <class T> concept Nullable =
    // 可以将其赋值为nullptr
    requires(T t) { t = nullptr; } &&
    // 具有用户定义的转换为nullptr或是指针的转换操作符
    (requires(T t) { t.operator decltype(nullptr)(); } || std::is_pointer_v<T>);
```

此概念以及其他一些概念是其 C# 等效概念的近似值。C++ 没有 C#“可为 null 的上下文”和其他细微语言差异的精确匹配项。请随意调整这些概念以适应预期用途。

## Conclusion  结论

C++中由`requires`和`concept`提供的约束与C#中由`where`提供的约束发挥着类似的作用。
在比较这两种语言时，C++版本本质上是一个C#功能的超集。
虽然一些概念是通过C++标准库中的`<concepts>`头文件提供的，但我们也被赋予了编写自己概念的工具体现，就像上面用`Nullable`和其他概念所做的那样。

我们创建的约束使我们能够限制我们的模板可以工作在什么上，向模板的用户表达这种意图，生成更易于阅读的编译器错误，甚至可以选择最优的重载函数。

这与C#的约束不同，它使我们的泛型能够使用类型参数的更多功能。
因为C#只提供了11个基本约束，而且我们无法创建自己的约束，所以我们经常不得不做出权衡，比如由于在接口上调用函数而造成性能损失，由于装箱到引用类型而创建垃圾，或者[为了编写泛型代码而跳过许多步骤](https://www.jacksondunstan.com/articles/5520)。

> 原作者在另外一篇文章中有以及[Type-Agnostic Generic Algorithms](https://www.jacksondunstan.com/articles/5520)
> </br>等c++完结了在回来阅读


















