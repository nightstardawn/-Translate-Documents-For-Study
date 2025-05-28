# C++ For C# Developers: Part 13 – Initialization

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5693),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

有了构造函数 ，我们现在可以讨论结构体和其他类型的初始化。这是一个比 C# 复杂得多的主题。继续阅读以了解许多细节！

## Explicit Constructors  显式构造函数

在开始初始化之前，我们需要多谈谈如何创建结构体对象。首先，我们编写的所有构造函数都可以选择性地声明为 `explicit`：

```c++
struct Vector2
{
    float X;
    float Y;
 
    explicit Vector2(const Vector2& other)
    {
        X = other.X;
        Y = other.Y;
    }
};
```

在 C++20 中，这可以以放在 `explicit` 关键字后面的括号中的编译时常量表达式为条件：

```c++
struct Vector2
{
    float X;
    float Y;
 
    explicit (2 > 1) Vector2(const Vector2& other)
    {
        X = other.X;
        Y = other.Y;
    }
};
```
> 简单来说就是：括号内为True就是显示构造函数，false就是转换构造函数

当构造函数是显式的时，它不再被视为“转换构造函数”。正如我们将在下面看到的，某些形式的初始化将不再允许隐式调用构造函数。

> 简单解释一下：如果一个构造函数可以接受一个参数（或有多个参数，但除第一个外都有默认值），则它被称为 **转换构造函数**。它的作用是将参数类型隐式转换为当前类类型。
> </br>看个例子：
> ```c++
> struct MyInt 
> {
>   // 转换构造函数：允许将 int 隐式转换为 MyInt 类型
>   MyInt(int value) : m_value(value) {}
>   int m_value;
> };
> 
> // 隐式转换：合法
> MyInt a = 42;  // 隐式调用 MyInt(42)
> ```
> 那么显示构造函数就很好理解了：当构造函数被标记为 `explicit` 时，它 不再参与隐式类型转换，只能用于显式初始化。
> ```c++
> class MyInt 
> {
>   // 显式构造函数：禁止隐式转换
>   explicit MyInt(int value) : m_value(value) {}
>   int m_value;
> };
> 
> // 隐式转换：非法！
> // MyInt a = 42;  // 编译错误
> 
> // 显式初始化：合法
> MyInt b(42);      // 直接调用构造函数
> MyInt c = MyInt(42); // 显式构造
> ```

## User-Defined Conversion Operators 用户定义的转换运算符

> 温馨回顾：
> C#中的User-Defined Conversion Operators 用户定义的转换运算符
> 主要是注意一下使用规则：
> 
> - 必须是 public static 方法。 
> - 使用 operator 关键字声明。 
> - 目标类型和源类型必须是不同的类型（不能为同一类型定义转换）。
> 
> </br>简单的看个例子就懂了：
> ```csharp
> public struct Celsius
> {
>     public double Value { get; }
> 
>     public Celsius(double value) => Value = value;
> 
>     // 隐式转换：double → Celsius
>     public static implicit operator Celsius(double value) => new Celsius(value);
> 
>     // 显式转换：Celsius → Fahrenheit
>     public static explicit operator Fahrenheit(Celsius celsius) => new Fahrenheit(celsius.Value * 9 / 5 + 32);
> }
> 
> public struct Fahrenheit
> {
>   public double Value { get; }
> 
>   public Fahrenheit(double value) => Value = value;
> 
>   // 显式转换：Fahrenheit → Celsius
>   public static explicit operator Celsius(Fahrenheit fahrenheit) => new Celsius((fahrenheit.Value - 32) * 5 / 9);
> }
> ```
> 使用示例：
> ```csharp
> Celsius celsius = 25.0;          // 隐式转换：double → Celsius
> Fahrenheit f = (Fahrenheit)celsius; // 显式转换：Celsius → Fahrenheit
> Celsius convertedBack = (Celsius)f; // 显式转换：Fahrenheit → Celsius
> ```

与 C# 一样，我们可以编写自己的转换运算符，从 struct 到任何其他类型：

```c++
struct Vector2
{
    float X;
    float Y;
 
    operator bool()
    {
        return X != 0 || Y != 0;
    }
};
```

此外，与 C# 一样，这些可以是显式的。

```c++
struct Vector2
{
    float X;
    float Y;
 
    explicit operator bool()
    {
        return X != 0 || Y != 0;
    }
};
```

从 C++20 开始，它们也可以是有条件的显式的：

```c++
struct Vector2
{
    float X;
    float Y;
 
    explicit (2 > 1) operator bool()
    {
        return X != 0 || Y != 0;
    }
};
```

没有像 C# 中那样的隐式关键字。要使一个 implicit 成为 implicit，只需不要添加 explicit。

在 C# 中，用户定义的转换运算符是`static`的，并采用与定义它们的结构类型相同的参数。在 C++ 中，它们是非静态的， `this`是隐式或显式使用的，而不是参数。

与其他重载运算符一样，它们可以被显式调用。很少看到这种情况，但这是允许的：

```c++
Vector2 v1;
v1.X = 2;
v1.Y = 4;
bool b = v1.operator bool();
```

## Initialization Types  初始化类型

C++ 将初始化分为以下类型：

- Default
- Aggregate
- Constant
- Copy
- Direct
- List
- Reference
- Value
- Zero

这些类型工作方式的规则通常遵循另一种类型工作方式的规则。这类似于一个函数调用另一个函数。它创建一个类型对另一个类型的依赖关系。这些依赖项经常在图表中形成循环，大致如下所示：

![关系图](https://s2.loli.net/2025/05/19/I38PyYOeHuxj5C7.png)

这意味着，当我们遍历初始化类型时，我们将引用我们尚未看到的其他初始化类型。请随意跳到引用的类型，或者在本文后面阅读了该类型的引用后返回重新访问该类型。

至于术语，我们经常说变量是 “X-initialized” 的，这意味着它是使用 “X” 初始化类型的规则初始化的。
</br>例如，“MyVar is direct-initialized” 表示 “MyVar 根据直接初始化类型的规则进行初始化”。

## Default Initialization  默认初始化

当声明变量时没有初始化器，则会发生默认初始化：

`T object;`

调用未提及数据成员的构造函数时，也会发生这种情况：

```c++
struct HasDataMember
{
    T Object;
    int X;
 
    HasDataMember()
        : X(123) // No mention of Object
    {
    }
};
```

如果类型 （T） 是结构，则调用其默认构造函数。如果它是一个数组，则数组的每个元素都是默认初始化的：

```c++
struct ConstructorLogs
{
    ConstructorLogs()
    {
        DebugLog("default");
    }
};
 
ConstructorLogs single; // Prints "default"
ConstructorLogs array[3]; // Prints "default", "default", "default"
```


对于所有其他类型，都不会发生任何事情。这包括原始数据类型、枚举和指针。使用这些对象之一是未定义的行为，并且可能会引起严重错误，因为编译器可以生成它想要的任何代码。

```c++
float f;
DebugLog(f); // Undefined behavior!
```

如果这些类型的变量是 const ，则不允许默认初始化，因为以后无法初始化它们：

```c++
void Foo()
{
    const float f; // 编译器错误：默认初始化器不起作用
}
```

一个例外是静态变量，包括结构体的静态数据成员和全局变量。这些是零初始化的：

```c++
const float f; // OK: 这是一个全局变量
 
struct HasStatic
{
    static float X;
};
float HasStatic::X; // OK: 这是一个静态数据成员
```

如果存在默认构造函数可以调用，因为那样会初始化变量，这也是允许的：

```c++
const HasDataMember single; // OK: 调用默认构造函数
 
struct NoDefaultConstructor
{
    NoDefaultConstructor() = delete;
};
 
const NoDefaultConstructor ndc; // 编译器错误：没有默认构造函数
```

引用（左值和右值）从不进行默认初始化。它们有自己的初始化类型，我们将在下面介绍：引用初始化。

## Copy Initialization  复制初始化

复制初始化有几种形式：
```c++
// Assignment style
T object = other;
 
// Function call
func(other)
 
// Return value
return other;
 
// Array assigned to curly braces
T array[N] = {other};
```

对于前三种形式，只涉及一个对象。调用该对象的复制构造函数， 并将 other 作为参数传入：

```c++
struct Logs
{
    Logs() = default;
 
    Logs(const Logs& logs)
    {
        DebugLog("copy");
    }
};
 
Logs Foo(Logs a)
{
    Logs b = a; // "copy" for assignment style
    return a; // "copy" for return value
}
 
Logs x;
Foo(x); // "copy" for function call
```

如果复制构造函数是显式的，则不再允许这样做：

```c++
struct Logs
{
    Logs() = default;
 
    explicit Logs(const Logs& logs)
    {
        DebugLog("copy");
    }
};
 
Logs Foo(Logs a)
{
    Logs b = a; // 编译器错误：复制构造函数是显式的
    return a; // 编译器错误：复制构造函数是显式的
}
 
Logs x;
Foo(x); // 编译器错误：复制构造函数是显式的
```

用户定义的转换运算符也可以由相同的三种形式的复制初始化来调用：

```c++
struct ConvertLogs
{
    ConvertLogs() = default;
 
    operator bool()
    {
        DebugLog("convert");
        return true;
    }
};
 
bool Foo(bool b)
{
    ConvertLogs x;
    return x; // "convert" for return value
}
 
ConvertLogs x;
bool b = x; // "convert" for assignment style
 
Foo(x); // "convert" for function call
```

然后，用户定义的 conversion 运算符的返回值（在本例中为 `bool`）用于直接初始化变量。


与复制构造函数一样，显式定义用户自定义转换运算符将禁用复制初始化，并使得所有这些“转换”行生成编译器错误，就像我们显式定义了复制构造函数一样。

对于非结构类型（如 primitives、enum 和 pointers），只需复制该值：

最后一种形式处理数组。这发生在aggregate initialization期间。

## Aggregate Initialization 聚合初始化

聚合初始化有以下形式：

```c++
// Assign curly braces
T object = { val1, val2 };
 
// No-assign curly braces
T object{ val1, val2 };
 
// Assign curly braces with "designators" (data member names)
T object = { .designator=val1, .designator=val2 };
 
// No-assign curly braces with "designators" (data member names)
T object{ .designator=val1, .designator=val2 };
 
// Parentheses
T object(val1, val2);
```

所有这些表单都适用于被视为 “聚合” 的类型 `（T）`。这包括除了使用 `= default` 的数组和结构之外，没有任何构造函数的数组和结构。

这些数组的元素和这些结构的数据成员使用给定的值进行复制初始化：`val1`、`val2` 等。这是按索引顺序完成的，从数组的第一个元素开始。对于结构，这是按照声明数据成员的顺序完成的，就像构造函数的初始值设定项列表一样。

标号从 C++20 开始可用。它们类似于 C# 的“对象初始值设定项”：`Vector2 vec = {X=2， Y=4}`;。它们必须与 struct 的数据成员的顺序相同，并且所有值都必须具有指示符。

```c++
struct Vector2
{
    float X;
    float Y;
};
 
Vector2 v1 = { 2, 4 };
DebugLog(v1.X, v1.Y); // 2, 4
 
Vector2 v2{2, 4};
DebugLog(v2.X, v2.Y); // 2, 4
 
Vector2 v3 = { .X=2, .Y=4 };
DebugLog(v3.X, v3.Y); // 2, 4
 
Vector2 v4{ .X=2, .Y=4 };
DebugLog(v4.X, v4.Y); // 2, 4
 
Vector2 v5(2, 4);
DebugLog(v5.X, v5.Y); // 2, 4
```

传递比数据成员或数组元素更多的值会导致编译错误：

```c++
Vector2 v5 = {2, 4, 6}; // 编译器错误：数据成员过多
float a1[2] = {2, 4, 6}; // 编译器错误：数据成员过多
```

然而，我们可以传递比数据成员或数组元素更少的值。剩余的数据成员使用它们的默认成员初始化器进行初始化。如果没有默认成员初始化器，它们将从空列表（`{}`）进行复制初始化。

```c++
struct DefaultedVector2
{
    float X = 1;
    float Y;
};
 
DefaultedVector2 dv1 = {2};
DebugLog(dv1.X, dv1.Y); // 2, 0
 
float a2[2] = {2};
DebugLog(a2[0], a2[1]); // 2, 0
```


如果一个数据成员是左值引用或右值引用，不传递它会导致编译错误，因为根据引用的工作方式，它永远无法在之后初始化。

```c++
struct HasRef
{
    int X;
    int& R;
};
 
HasRef hr = {123}; // 编译器错误：引用数据成员未初始化
```

字符串 aggregate-initializing arrays 有一些特殊规则：

```c++
//a1的长度为4，包含：'a'、'b'、'c'、0
char a1[4] = "abc";

//长度是可选的。这与a1相同。
char a2[] = "abc";

//花括号是可选的。这与a1相同。
char a3[] = {"abc"};

//编译器错误：数组太小，无法容纳字符串字面量的内容
char a4[1] = "abc";

//额外的数组元素将被初始化为0
//a5有长度6，包含：'a'、'b'、'c'、0、0、0
char a5[6] = "abc";
```
> a2和a3的和a1相同的原因是数组最后要包含终止符 \0 即空字符,ASCII为0

## List Initialization  列表初始化

列表初始化有两种子类型。首先，“direct list initialization” 有以下形式：

```c++
// Named variable
T object{val1, val2};
 
// Unnamed temporary variable
T{val1, val2}
 
struct MyStruct
{
    // Data member
    T member{val1, val2};
};
 
MyStruct::MyStruct()
    // Initializer list entry
    : member{val1, val2}
{
}
```

其次，还有这些形式的“拷贝列表初始化”：

```c++
// Named variable
T object = {val1, val2};
 
// Function call
func({val1, val2})
 
// Return value
return {val1, val2};
 
// Overloaded subscript operator call
object[{val1, val2}]
 
// Assignment
object = {val1, val2}
 
struct MyStruct
{
    // Data member
    T member = {val1, val2};
};
```

编译器基本上通过使用一系列相当长的 if-else 决策来选择要做什么。

首先，如果有相同类型的单个值，则它将进行复制列表初始化的复制初始化，对于直接列表初始化则进行直接初始化：

```c++
Vector2 vec;
vec.X = 2;
vec.Y = 4;
 
// 直接列表初始化直接将vec初始化到vecA
Vector2 vecA{vec};
DebugLog(vecA.X, vecA.Y); // 2, 4
 
// 复制列表初始化将vecB复制初始化为vec
Vector2 vecB = {vec};
DebugLog(vecB.X, vecB.Y); // 2, 4
```

其次，如果变量是字符数组，并且存在相同字符类型的单个值，则对变量进行aggregate-initialized：

```c++
char array[1] = {'x'}; // Aggregate-initialized
DebugLog(array[0]); // x
```

第三，如果要初始化的变量是聚合类型，则它是聚合初始化的：

```c++
Vector2 vec = {2, 4}; // Aggregate-initialized
DebugLog(vec.X, vec.Y); // 2, 4
```

第四，如果未在大括号中传递任何值，并且要初始化的变量是具有默认构造函数的结构体，则对其进行值初始化：

```c++
struct NonAggregateVec2
{
    float X;
    float Y;
 
    NonAggregateVec2()
    {
        X = 2;
        Y = 4;
    }
};
 
NonAggregateVec2 vec = {}; // Value-initialized
DebugLog(vec.X, vec.Y); // 2, 4
```

第五，如果变量的构造函数只采用标准库的 std：：initializer_list 类型，则调用该构造函数。我们还没有介绍任何 Standard Library，但这种类型的细节在这一点上并不重要。可以说，这是 C++ 等效于在 C# 中初始化集合： `List<int> list = new List<int> { 2, 4 };` 。

```c++
struct InitListVec2
{
    float X;
    float Y;
 
    InitListVec2(std::initializer_list<float> vals)
    {
        X = *vals.begin();
        Y = *(vals.begin() + 1);
    }
};
 
InitListVec2 vec = {2, 4};
DebugLog(vec.X, vec.Y); // 2, 4
```

第六，如果任何构造函数与传递的值匹配，则调用最匹配的构造函数：

```c++
struct MultiConstructorVec2
{
    float X;
    float Y;
 
    MultiConstructorVec2(float x, float y)
    {
        X = x;
        Y = y;
    }
 
    MultiConstructorVec2(double x, double y)
    {
        X = x;
        Y = y;
    }
};
 
MultiConstructorVec2 vec1 = {2.0f, 4.0f}; // Call (float, float) version
DebugLog(vec1.X, vec1.Y); // 2, 4
 
MultiConstructorVec2 vec2 = {2.0, 4.0}; // Call (double, double) version
DebugLog(vec2.X, vec2.Y); // 2, 4
```

第七种，如果变量是（有范围或无范围的） 枚举 ，并且该类型的单个值通过直接列表初始化传递，则使用该值初始化变量：

```c++
enum struct Color : uint32_t
{
    Blue = 0x0000ff
};
 
Color c = {Color::Blue};
DebugLog(c); // 255
```

第八，如果变量不是结构体，只传递一个值，并且该值不是引用，则直接初始化变量：

```c++
float f = {3.14f};
DebugLog(f); // 3.14
```

第九，如果变量不是结构体，花括号中只有一个值，并且变量不是引用或是指向单个值类型的引用，那么变量将直接初始化以进行直接列表初始化，或者以该值进行复制初始化以进行复制列表初始化：

```c++
float f = 3.14f;
 
float& r1{f}; // 直接列表初始化直接初始化
DebugLog(r1); // 3.14
 
float& r2 = {f}; // 复制列表初始化
DebugLog(r2); // 3.14
 
float r3{f}; // 直接列表初始化直接初始化
DebugLog(r3); // 3.14
 
float r4 = {f}; // 复制列表初始化
DebugLog(r4); // 3.14
```

第十，如果变量是对不同类型的引用，而不是传递的一个值，则创建对值类型的临时引用，对其进行列表初始化，并将其绑定到变量。变量必须是 `const` 才能正常工作：

```c++
float f = 3.14;
 
const int32_t& r1 = f;
DebugLog(r1); // 3
 
int32_t& r2 = f; // 编译器错误：不是常量
DebugLog(r2);
```

第十一个，也是最后一个，如果未传递任何值，则对变量进行值初始化：

```c++
float f = {};
DebugLog(f); // 0
```

最后一个需要注意的细节是，在大括号中传递的值是按顺序计算的。这与传递给函数的参数不同，后者是按照编译器确定的顺序计算的。

## Reference Initialization 引用初始化

```c++
// lvalue reference variables
T& ref = object;
T& ref = {val1, val2};
T& ref(object);
T& ref{val1, val2};
 
// rvalue reference variables
T&& ref = object;
T&& ref = {val1, val2};
T&& ref(object);
T&& ref{val1, val2};
 
// Function calls
/* Assume */ void func(T& val); /* or */ void func(T&& val);
func(object)
func({val1, val2})
 
// Return values
T& func() { T t; return t; }
T&& func() { return T(); }
 
// Constructor initializer lists
MyStruct::MyStruct()
    : lvalueRef(object)
    , rvalueRef(object)
{
}
```

如果提供了大括号，则引用将进行列表初始化：
```c++
float&& f = {3.14f};
DebugLog(f); // 3.14
```

否则，引用将遵循引用初始化规则。这些实际上是另一个 if-else 决策系列 ，但比列表初始化要短得多。

首先，对于相同类型的左值引用，引用只是绑定到传递的对象：

```c++
float f = 3.14f;
float& r = f;
DebugLog(r); // 3.14
```

当变量是左值引用，但它的类型与传递的对象不同时，如果存在用户定义的转换函数，则调用该函数，并将变量绑定到返回值：
```c++
float pi = 3.14f;
 
struct ConvertsToPi
{
    operator float&()
    {
        return pi;
    }
};
 
ConvertsToPi ctp;
float& r = ctp; // 用户定义的转换操作符被调用
DebugLog(r); // 3.14
```

在所有其他情况下，传递的表达式被评估为一个临时变量，并且引用被绑定到该变量：

```c++
float Add(float a, float b)
{
    return a + b;
}
 
// 调用函数，将返回值存储在临时变量中，将引用绑定到临时变量
float&& sum = Add(2, 4);
DebugLog(sum); // 6
```

通过引用初始化创建的临时变量，其生命周期会延长以匹配引用的生命周期。存在一些例外。
</br>首先，返回的引用总是“悬垂”的，因为它们所指向的内容在函数退出时结束其生命周期。
</br>其次，类似地，对函数参数的引用在函数退出时也会结束其生命周期。

```c++
float&& Dangling1()
{
    return 3.14f; // 临时返回在这里结束其生命周期
}
 
float& Dangling2(float x)
{
    return x; // 临时返回在这里结束其生命周期
}
 
DebugLog(Dangling1()); // 未定义行为
DebugLog(Dangling2(3.14f)); // 未定义行为
```

第三，仅当使用大括号（而不是括号）时，聚合的引用数据成员或元素的生存期才会延长：

```c++
struct HasRvalueRef
{
    float&& Ref;
};
 
// 使用了花括号。具有3.14f值的浮点数的生命周期延长。
HasRvalueRef hrr1{3.14f};
DebugLog(hrr1.Ref); // 3.14
 
// 使用了括号。值为3.14f的浮点数的生命周期未延长。
HasRvalueRef hrr2(3.14f);
DebugLog(hrr2.Ref); // 未定义行为。引用已结束其生命周期。
```

## Value Initialization  值初始化

值初始化可能如下所示：

```c++
// Variable
T object{};
 
// Temporary variable (i.e. it has no name)
T()
T{}
 
// Initialize a data member in an initializer list
MyStruct::MyStruct()
    : member1() // Parentheses version
    , member2{} // Curly braces version
{
}
```

值初始化始终遵循另一种类型的初始化。以下是它决定使用哪种类型的方式：

如果使用大括号并且变量是聚合，则会对其进行聚合初始化。

```c++
Vector2 vec{2, 4}; // Aggregate initialization
DebugLog(vec.X, vec.Y); // 2, 4
```

如果变量是一个没有默认构造函数的结构体，但它确实有一个只接受 `std：：initializer_list` 的构造函数，则该变量将使用空列表（即 `{}`）进行列表初始化。

```c++
struct InitListVec2
{
    float X;
    float Y;
 
    InitListVec2(std::initializer_list<float> vals)
    {
        int index = 0;
        float x = 0;
        float y = 0;
        for (float cur : vals)
        {
            switch (index)
            {
                case 0: x = cur; break;
                case 1: y = cur; break;
            }
        }
        X = x;
        Y = y;
    }
};
 
InitListVec2 vec{}; // List initialization (passes empty list)
DebugLog(vec.X, vec.Y); // 0, 0
```

如果变量是没有默认构造函数的结构，则默认初始化它。

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2() = delete;
};
 
Vector2 vec{}; // Default-initialized
DebugLog(vec.X, vec.Y); // 0, 0
```


如果编译器生成了默认构造函数，则变量将被零初始化，如果数据成员中有默认初始化器（例如 `float X = 0;`），则进行直接初始化。

```c++
struct Vector2
{
    float X = 2;
    float Y = 4;
};
 
Vector2 vec{}; // 零初始化然后直接初始化
DebugLog(vec.X, vec.Y); // 2, 4
```

如果变量是数组，则每个元素都进行值初始化。

```c++
float arr[2]{}; // 元素 值初始化
DebugLog(arr[0], arr[1]); // 0, 0
```

如果上述情况均不适用，则变量初始化为零。

```c++
float x{}; // Zero-initialized
DebugLog(x); // 0
```

## Direct initialization  直接初始化

以下是直接初始化可以采用的形式：

```c++
// Parentheses with single value
T object(val);
 
// Parentheses with multiple values
T object(val1, val2);
 
// Curly braces with single value
T object{val};
 
MyStruct::MyStruct()
    // Parentheses in initializer list
    : member(val1, val2)
{
}
```

所有这些都会查找与传递的值匹配的构造函数。如果找到一个变量，则调用最匹配的变量来初始化变量。

```c++
struct MultiConstructorVec2
{
    float X;
    float Y;
 
    MultiConstructorVec2(float x, float y)
    {
        X = x;
        Y = y;
    }
 
    MultiConstructorVec2(double x, double y)
    {
        X = x;
        Y = y;
    }
};
 
MultiConstructorVec2 vec1{2.0f, 4.0f}; // Call (float, float) version
DebugLog(vec1.X, vec1.Y); // 2, 4
 
MultiConstructorVec2 vec2{2.0, 4.0}; // Call (double, double) version
DebugLog(vec2.X, vec2.Y); // 2, 4
```

如果没有构造函数匹配，或者变量不是结构但是一个聚合，则对变量进行聚合初始化。

```c++
struct Vector2
{
    float X;
    float Y;
};
 
// 没有匹配的构造函数，但Vector2是一个聚合体
Vector2 vec{2, 4}; // Aggregate initialization
DebugLog(vec.X, vec.Y); // 2, 4
```

从 C++20 开始，变量可以是数组。在这种情况下，聚合初始化规则适用。例如，传递过多的值是编译器错误。

```c++
float a1[2]{2, 4}; // Aggregate initialization
DebugLog(a1[0], a1[1]); // 2, 4
 
float a2[2]{2, 4, 6, 8}; // Compiler error: too many values
```

有一个特定于类型的例外。如果变量是 bool 且值为 nullptr，则变量变为 false。
```c++
bool b{nullptr};
DebugLog(b); // false
```

直接初始化的括号形式的一个常见错误是在变量初始化和函数声明之间产生歧义。请考虑以下代码：

```c++
struct Enemy
{
    float X;
    float Y;
};
 
struct Vector2
{
    float X;
    float Y;
 
    Vector2() = default;
 
    Vector2(Enemy enemy)
    {
        X = enemy.X;
        Y = enemy.Y;
    }
};
 
Vector2 defaultEnemySpawnPoint(Enemy());
```

最后一行是模棱两可的。命名使我们认为它是一个名为 `defaultEnemySpawnPoint` 的 `Vector2` 类型的变量，该变量正在使用值初始化的临时 `Enemy` 变量进行直接初始化。

另一种解读该行的方法是，它声明一个名为 `defaultEnemySpawnPoint` 的函数，该函数返回一个 `Vector2`，并采用一个未命名的指针指向一个不带参数并返回 `Enemy` 的函数。在该替代读取中，我们可以编写如下代码：

```c++
// 满足函数指针类型的函数定义
Enemy cb()
{
    return {};
}
 
// 上述声明的定义，无论是有意还是无意
Vector2 defaultEnemySpawnPoint(Enemy())
{
    return {};
}
 
// 可以用 'cb' 作为函数指针参数来调用
defaultEnemySpawnPoint(cb);
```

当出现这种歧义时，编译器始终选择函数声明。这意味着上面的代码是有效的，实际上是可以工作的，但如果我们试图像变量一样使用`defaultEnemySpawnPoint`，而它实际上是一个函数，那么我们会得到错误：

```c++
//编译器错误：defaultEnemySpawnPoint 是一个函数
//函数没有 X 或 Y 数据成员可以获取
DebugLog(defaultEnemySpawnPoint.X, defaultEnemySpawnPoint.Y);
```

值得庆幸的是，只需使用直接初始化的大括号形式就很容易解决歧义，因为函数指针语法不使用大括号：
```c++
Vector2 defaultEnemySpawnPoint(Enemy{});
DebugLog(defaultEnemySpawnPoint.X, defaultEnemySpawnPoint.Y); // 0, 0
```

## Constant Initialization  常量初始化

常量初始化只有两种形式：

```c++
T& ref = constantExpression;
T object = constantExpression;
```

这两个都只适用于变量既是`const`又是`static`的情况，例如全局变量和静态结构体数据成员。否则，变量将被零初始化。

```c++
struct Player
{
    static const int32_t MaxHealth;
 
    int32_t Health;
};
 
// 常量初始化数据成员
const int32_t Player::MaxHealth = 100;
 
// 常量初始化全局引用
const int32_t& defaultHealth = Player::MaxHealth;
```

此初始化发生在所有其他初始化之前，因此在其他类型的初始化期间从这些变量中读取数据是安全的。如果其他初始化出现在常量初始化之前，则情况也是如此：

```c++
struct Player
{
    static const int32_t MaxHealth;
 
    int32_t Health;
};
 
// 2) 聚合初始化
Player localPlayer{Player::MaxHealth};
 
// 1) 常量初始化
const int32_t Player::MaxHealth = 100;
const int32_t& defaultHealth = Player::MaxHealth;
 
// 3) 普通代码，不是初始化
DebugLog(localPlayer.Health); // 100
```

## Zero Initialization  零初始化

最后，我们的初始化为零。与所有其他类型的不同，它没有任何显式形式。相反，正如我们上面看到的，其他类型的初始化可能会导致零初始化：

```c++
//静态变量不是常量初始化的
// 零初始化仍然发生在其他类型初始化之前
static T object;

// 在非结构类型的价值初始化期间
// 包括结构数据成员和数组元素
T();
T t = {}; 
T{};

// 当从一个过短的字符串字面量初始化数组时
// 剩余的元素被零初始化
char array[N] = ";"
```

零初始化将结构的基元和所有填充位设置为 0。它不对引用执行任何作。

## 总结

正如我们现在所看到的，初始化在 C++ 中是一个比在 C# 中复杂得多的主题。主要原因是 C++ 提供了更多的功能。支持默认构造函数、临时变量、数组、引用、函数指针、const、字符串文字等需要相当多的语法。

尽管如此，对于我们尚未介绍的语言功能，这里还是省略了相当多的细节：继承、lambda 等。我们将在本系列的其余部分介绍这些内容。









