# C++ For C# Developers: Part 33 – Alignment, Assembly, and Language Linkage

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6237),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天，我们将介绍另外几个没有 C# 等效项的小功能：折叠表达式和详细的类型说明符。虽然它们很小，但它们可能非常有用

## Fold Expressions  折叠表达式

从 C++17 开始可用的 Fold 表达式允许我们将二进制运算符应用于模板参数包(Part28)中的所有参数例如，如果我们想将一些整数相加：

```c++
// 模板参数是一组整数
template<int... Vals>
// 将 + 运算符应用于 Vals
int SumOfAll = (Vals + ...);
 
// 使用四个整数在 Vals 包中实例化模板
DebugLog(SumOfAll<1, 2, 3, 4>); // 10
```

“折叠表达式”是`(Vals + ...)`这部分。
在这里需要括号，与大多数表达式不同。我们命名参数包为(`Vals`)，命名二元运算符为(`+`)，并添加`...`来表示我们想要对该包应用该运算符。

当模板实例化时，编译器将折叠表达式转换为一系列二元运算符：


```c++
// 模板参数包的扩展版本
template<int Val1, int Val2, int Val3, int Val4>
// 折叠表达式的扩展版本
int SumOfAll = Val1 + (Val2 + (Val3 + Val4));
```

这种折叠表达式被称为“一元右折叠”。这意味着最右边的参数首先应用运算符。

为了反转这个并首先将运算符应用于最左边的参数，我们使用这样的“一元左折叠”:

```c++
template<int... Vals>
int SumOfAll = (... + Vals); // 交换了“Vals” 和 “...”
 
DebugLog(SumOfAll<1, 2, 3, 4>); // 10
```

当实例化时，编译器产生等同于以下内容：

```c++
template<int Val1, int Val2, int Val3, int Val4>
int SumOfAll = ((Val1 + Val2) + Val3) + Val4;
```

当我们只是相加整数时，选择左折叠或右折叠并不重要，但与其他类型和其他运算符一起时，这肯定很重要。

如果参数包恰好为空，则只允许使用三个二元运算符。首先，我们可以使用 `&&` 来评估为真：

```c++
template<bool... Vals>
bool AndAll = (... && Vals);
 
DebugLog(AndAll<false, false>); // false
DebugLog(AndAll<false, true>); // false
DebugLog(AndAll<true, false>); // false
DebugLog(AndAll<true, true>); // true
DebugLog(AndAll<>); // true
```

其次，我们可以使用 || 来评估为假：

```c++
template<bool... Vals>
bool OrAll = (... || Vals);
 
DebugLog(OrAll<false, false>); // false
DebugLog(OrAll<false, true>); // true
DebugLog(OrAll<true, false>); // true
DebugLog(OrAll<true, true>); // true
DebugLog(OrAll<>); // false
```

第三，这无疑是使用情况中最不常见的一种，逗号操作符将评估为void()：

```c++
template<bool... Vals>
void Goo()
{
    return (... , Vals); //等同于 "return void();"
}
 
// OK
Goo();
```

现在我们已经看到了“一元”折叠表达式，让我们来看看“二元”的。为了制作这些，我们在`...`之后添加相同的二元运算符，然后是一个额外的值：

```c++
template<int... Vals>
// 然后在单一折叠之后添加运算符（+）以及额外的值（1）
int SumOfAllPlusOne = (Vals + ... + 1);
 
DebugLog(SumOfAllPlusOne<1, 2, 3, 4>); // 11
```

这里我们将一元折叠表达式`（Vals + ...）`转换为二元表达式，通过在其末尾添加 `+ 1` 实现。这将在参数包中的值之外添加另一个值。
由于这是一个“二元右折叠”，括号将首先添加到最右侧的值。

```c++
template<int Val1, int Val2, int Val3, int Val4>
int SumOfAll = 1 + (Val1 + (Val2 + (Val3 + Val4)));
```

“二进制左折叠”版本只是在左侧增加了额外的值：


```c++
template<int... Vals>
int SumOfAllPlusOne = (1 + ... + Vals);
```

当使用参数包中的四个值实例化时，它将看起来像这样：

```c++
template<int Val1, int Val2, int Val3, int Val4>
int SumOfAll = (((1 + Val1) + Val2) + Val3) + Val4;
```

无论我们编写哪种折叠表达式，我们都可以使用以下任意二元运算符：

- `+`
- `-`
- `*`
- `/`
- `%`
- `^`
- `&`
- `|`
- `=`
- `<`
- `>`
- `<<`
- `>>`
- `+=`
- `-=`
- `*=`
- `/=`
- `%=`
- `^=`
- `&=`
- `|=`
- `<<=`
- `>>=`
- `==`
- `!=`
- `<=`
- `>=`
- `&&`
- `||`
- `,`
- `.*`
- `->*`

## Elaborated Type Specifiers  详细类型说明符

我们之前已经看到，C代码要求我们使用`struct Player`而不是仅仅使用`Player`作为`Player`结构体的类型名：

```c++
// C code
 
struct Player
{
    int Health;
    int Speed;
};
 
struct Player p; // C requires "struct" prefix
p.Health = 100;
p.Speed = 10;
DebugLog(p.Health, p.Speed); // 100, 10
```

在C++中，这通常是不必要的。
</br>然而，存在一种特殊情况，即我们有一个类和一个变量具有相同的名称。使用该名称指的是变量，因此我们不能再使用类型了：

```c++
// class
struct Player
{
};
 
// 与类名相同的变量
int Player = 123;
 
// 编译器错误：'Player' 不是一个类型
// 这是因为 'Player' 指的是变量，而不是类
Player p;
```

为了解决这个问题，我们可以使用C风格的`struct Player`来明确指出我们指的是结构体，而不是变量。
</br>这被称为“详尽类型说明符”，因为我们正在详细说明`Player`类型：

```c++
// 详细类型说明
// OK: 指的是Player结构体，而不是Player变量
struct Player p;
```

由于结构体（struct）和类（class）(Part15)非常相似，我们可以在我们的详细类型说明中互换使用它们：

```c++
// 当Player是一个“结构体”时，使用“class”进行详细类型说明
class Player p;
```

Unions(Part16)不可互换:

```c++
// union
union IntFloat
{
    int32_t Int;
    float Float;
};
 
// 具有与联合相同名称的变量
bool IntFloat = true;
 
// 编译器错误：IntFloat 不是一个类型
// 这是因为“IntFloat”指的是变量，而不是联合类型
IntFloat u;
 
// 详细类型说明
// 编译器错误：IntFloat 是一个联合体，不是一个类或结构体
class IntFloat u;
 
// 详细类型说明符
// OK：指的是IntFloat联合体，而不是IntFloat变量
union IntFloat u;
```

枚举也是它们自己的一种实体，需要用`enum`来详细说明：

```c++
// enumeration
enum DamageType
{
    Physical,
    Water,
    Fire,
    Magic,
};
 
// 与枚举具有相同名称的变量
float DamageType = 3.14f;
 
// 详细类型说明器
// 编译器错误：DamageType 是一个枚举，不是一个类或结构体
class DamageType d;
 
// 详细类型说明
// OK: 指的是DamageType枚举，而不是DamageType变量
enum DamageType d;
```

普通 `enum` 可以与有作用域的枚举一起使用，但 `enum class` 或 `enum struct` 不能与无作用域的枚举一起使用，必须与有作用域的枚举一起使用：

```c++
enum class Scoped
{
};
 
enum Unscoped
{
};
 
enum Scoped e1; // OK
enum Unscoped e2; // OK
enum class Scoped e3; // OK
enum class Unscoped e4; // 编译器错误：不能与未限定的枚举一起使用限定枚举
enum struct Scoped e5; // OK
enum struct Unscoped e6; // 编译错误：不能与未限定的枚举一起使用限定枚举
```

无论类型如何，我们也可以使用作用域解析运算符来引用其在命名空间(Part17)中的位置：

```c++
namespace Gameplay
{
    enum DamageType
    {
        Physical,
        Water,
        Fire,
        Magic,
    };
 
 
    float DamageType = 3.14f;
}
 
// 使用作用域解析运算符的详细类型说明符
enum Gameplay::DamageType d;
```

类的成员类型也是如此：

```c++
struct Gameplay
{
    // 类的成员类型
    enum DamageType
    {
        Physical,
        Water,
        Fire,
        Magic,
    };
 
    // 类的成员变量
    constexpr static float DamageType = 3.14f;
};
 
// 详细类型说明符，指代类成员类型
enum Gameplay::DamageType d;
```

## Conclusion  结论

折叠表达式为我们提供了一种将二元运算符干净地应用于模板参数包的方法。
</br>没有它们，我们就需要求助于递归实例化模板和使用特化来停止递归等替代方案。
</br>这会使代码的可读性大大降低，并且编译速度也会变慢，因为需要实例化许多模板然后再将其丢弃。
</br>我们可以选择一元或二元以及左折叠或右折叠，以便控制二元运算符如何应用于参数包的值。
</br>由于C#没有可变参数模板，因此它也没有折叠表达式。

详细类型说明符是一个小特性，它在我们有与变量同名类型的情况下提供了一个解决方案。
</br>我们可以明确地引用这些类型，以改变类型和变量之间共享名称的默认含义。
</br>这种情况很少发生，但一旦出现，这是一个很好的工具。
</br>C#不允许类型与变量同名，因此在该语言中没有与之相当的特性。



