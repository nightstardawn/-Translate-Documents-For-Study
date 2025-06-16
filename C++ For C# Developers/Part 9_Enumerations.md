# C++ For C# Developers: Part 9 – Enumerations

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5601),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

我们今天将继续系列讨论，探讨枚举，这是C++中又一个令人惊讶的复杂主题。
</br>实际上，我们今天要讨论的是两种紧密相关的枚举概念，所以请继续阅读，了解这两种类型的所有内容！

## Unscoped Enumerations  无范围枚举

C++中的第一种枚举类型被称为“无作用域枚举”。
</br>这是因为它们不会引入一个新的作用域来包含它们的枚举值，而是将那些枚举值引入它们所在的作用域。考虑以下示例：

```c++
enum Color
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
 
DebugLog(Red); // 0xff0000
```

这个例子展示了未指定作用域枚举的几个方面。首先，定义一个枚举与C#中的定义非常相似。
我们使用`enum`关键字，然后是枚举的名称，并将枚举值和它们的值放在花括号中，用逗号分隔。
</br>与C#不同，我们在花括号关闭后添加一个分号。

其次，我们看到`Red`、`Green`和`Blue`枚举器被放置在周围的作用域中，而不是像在C#中那样放在`Color`枚举内部。
</br>这意味着`DebugLog`行中有`Red`在作用域内，可以读取并打印出来。

其次，我们看到`Red`、`Green`和`Blue`枚举器被放置在周围的作用域中，而不是像在C#中那样放在`Color`枚举内部。
</br>这意味着`DebugLog`行中有`Red`在作用域内，可以读取并打印出来。

```c++
DebugLog(Color::Red); // 0xff0000
```

因为不需要使用名称来访问枚举值，枚举的名称本身是可选的：

```c++
enum
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
 
DebugLog(Red); // 0xff0000
```

与 C# 中一样，枚举器的值是可选的。
</br>它们甚至遵循相同的默认值规则：第一个枚举器默认为 0，后续枚举器默认为前一个枚举器的值加 1：

```c++
enum Prime
{
    One = 1,
    Two,
    Three,
    Five = 5
};
 
DebugLog(One, Two, Three, Five); // 1, 2, 3, 5
```

与 C# 不同，这些枚举器值隐式转换为整数类型：

```c++
int one = One;
DebugLog(one); // 1
```

具体而言，枚举的基础整数类型是从以下列表中选择的。选择可以容纳最大枚举器值的最小类型。

- `int`
- `unsigned int`
- `long`
- `unsigned long`
- `long long`
- `unsigned long long`

如果最大值不适合这些类型中的任何一个，编译器将生成错误。

为了进行更多控制，可以使用与 C# 中相同的语法显式指定基础整数类型：

```c++
enum Color : unsigned int
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
```

就像在 C# 中一样，我们可以将枚举器强制转换为整数。请注意，如果该整数太小而无法保存枚举器的值，则这是未定义的行为：

```c++
// OK 转换为整数
int one = (int)One;
DebugLog(one); // 1
 
// 太大以至于无法适应1个字节：未定义行为
char red = Red;
DebugLog(red); // 可能是任何东西...
```

我们还可以将整数强制转换为枚举变量，即使整数值不是枚举值之一：

```c++
// OK 转换为枚举类型变量
Prime prime = (Prime)3;
DebugLog(prime); // 3
 
// 即使不是命名的枚举器，也可以将其转换为枚举类型的变量
Prime prime = (Prime)4;
DebugLog(prime); // 4
```

我们也可以用花括号中的单个整数值来初始化它们，只要整数适合底层类型，并且底层类型已经被明确声明：

```c++
Prime prime{3};
```

请注意，我们在这里使用了 `enum name` 作为类型，就像我们在 C# 中一样。这意味着我们可以编写这样的函数：

```c++
void OutputCharacterToLedDisplay(char ch, Color color)
{
    // ...
}
```

`Color`不再允许传递任意整数：

```c++
OutputCharacterToLedDisplay('J', 0xff0000); // compiler error
 
OutputCharacterToLedDisplay('J', Red); // OK
```

就像函数一样，只要稍后定义，枚举就可以声明和引用而不必立即定义。在这种情况下，我们必须指定其底层整数类型：

```c++
// 声明枚举
enum Color : unsigned int;
 
// 使用枚举的名称
void OutputCharacterToLedDisplay(char ch, Color color)
{
    // ...
}
 
// 定义枚举
enum Color : unsigned int
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
```

声明和定义都是类似于 `int` 或 `float` 的类型，因此它们可以后跟标识符以创建变量：

```c++
// 声明
enum Color : unsigned int red, green, blue;
red = Red;
green = Green;
blue = Blue;
DebugLog(red, green, blue); // 0xff0000, 0x00ff00, 0x0000ff
 
// 定义
enum Color : unsigned int
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
} red = Red, green = Green, blue = Blue;
DebugLog(red, green, blue); // 0xff0000, 0x00ff00, 0x0000ff
```

最后，没有像C#中的`[Flags]`属性那样的特殊处理位标志。所有未限定的枚举类型的枚举值可以直接使用：
> 这里读者补充一下C#中`[Flags]`的相关知识点
> </br>首先要明确的是即使没有`[Flags]`特性，你仍然可以执行位运算。
> </br>通常，`[Flags]`枚举的成员值应该是2的幂（即1, 2, 4, 8, ...），这样每个成员代表一个独立的位,也可以使用位移运算（`<<`）来定义，以避免手动计算2的幂。
> </br>给一个例子
> ```csharp
> [Flags]
> public enum DaysOfWeek
> {
>     None    = 0,     // 0
>     Monday  = 1 << 0, // 1 (二进制: 00000001)
>     Tuesday = 1 << 1, // 2 (二进制: 00000010)
>     Wednesday=1 << 2, // 4 (二进制: 00000100)
>     Thursday=1 << 3, // 8 (二进制: 00001000)
>     Friday  = 1 << 4, // 16 (二进制: 00010000)
>     Saturday=1 << 5, // 32 (二进制: 00100000)
>     Sunday  = 1 << 6  // 64 (二进制: 01000000)
>     Weekend = Saturday | Sunday,
>     WorkDays = Monday | Tuesday | Wednesday | Thursday | Friday
> }
> ```
> 还有特别的一点，当枚举被标记为`[Flags]`时，其`ToString`方法会尝试将组合的枚举值显示为用逗号分隔的名称。例如，DaysOfWeek.Saturday | DaysOfWeek.Sunday 会被显示为 "Saturday, Sunday"。如果没有`[Flags]`特性，则只会显示数字值（如96）。
> 
> </br>总之`[Flags]`特性主要是为了改变枚举的行为（如ToString）以及增加代码的可读性，表明该枚举设计用于位域操作。

```c++
enum Channel
{
    RedOffset = 16,
    GreenOffset = 8,
    BlueOffset = 0
};
 
unsigned char GetRed(unsigned int color)
{
    return (color & Red) >> RedOffset;
}
 
DebugLog(GetRed(0x123456)); // 0x12
```

## Scoped Enumerations  作用域枚举

恰如其分地，C++ 中的另一种枚举类型称为“范围”枚举。正如预期的那样，这引入了一个包含枚举器的新范围。
</br>它们不会溢出到周围的范围中，因此需要 “scope resolution” 运算符来访问它们：

```c++
enum class Color : unsigned int
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
 
//编译错误：红色不在作用域内
auto red = Red;
 
// OK, 红色变量的类型是Color
auto red = Color::Red;
```

这里有很多共同点：枚举、枚举名、基础类型、枚举值名、枚举值、末尾的花括号和分号。
</br>唯一的不同之处在于在枚举和枚举名之间出现了`class`。这个关键字告诉编译器创建一个有作用域的枚举而不是无作用域的枚举。
</br>可以使用关键字`struct`代替，它与`class`具有相同的效果。

作用域枚举的行为基本上与无作用域枚举相同，所以我们只需讨论一些差异。

首先，枚举的类型名称是必需的。 这是因为如果枚举的类型名称没有添加到周围的作用域中，这样的枚举将非常无用。
</br>没有名称添加到作用域解析运算符（`::`）之前，将无法访问它们。

```c++
// 编译错误：没有名称
enum class : unsigned int
{
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff
};
 
// 编译器错误：无法命名枚举以访问其枚举值
auto red = ???::Red;
```

另一个区别是，作用域枚举的枚举值不会隐式转换为整数：

```c++
// 编译器错误：无法隐式转换
unsigned int red = Color::Red;
```

转换需要 Casting 才能转换：

```c++
// OK
unsigned int red = (unsigned int)Color::Red;
```

未明确说明时，底层类型的选择也要简单得多：它始终是 int：

```c++
enum class Numbers
{
    // OK: 1可以放入整数
    One = 1,
 
    // 编译器错误：太大以至于无法适应一个整型（假设整型是32位的）
    Big = 0xffffffffffffffff
};
```

由于基础类型已知为 int，因此编译器可以使用枚举类型，而无需在声明中明确声明：

```c++
// OK: 作用域枚举不需要底层类型
// 如同往常，默认的底层类型是 int
enum struct Prime;
 
// OK: 定义也不需要指定底层类型
enum struct Prime
{
    One = 1,
    Two,
    Three,
    Five = 5
};
 
// OK: 定义允许指定基础类型，只要它是int
enum struct Prime : int
{
    One = 1,
    Two,
    Three,
    Five = 5
};
```
这些实际上是两种枚举之间的全部差异。下表将它们彼此比较和对比，并与 C# 枚举进行比较和对比：

| 方面          | Example                                              | Unscoped       | Scoped  | C#                      |
|-------------|------------------------------------------------------|----------------|---------|-------------------------|
| 枚举器的初始化     | `One = 1`                                            | 自选             | 自选      | 自选                      |
| 将枚举器强制转换为整数 | `int one = (int)One;`                                | Yes            | Yes     | Yes                     |
| 将整数强制转换为枚举器 | `Prime p = (Prime)4;`                                | Yes            | Yes     | Yes                     |
| 名字          | `enum Prime {};`                                     | 自选             | 必需      | 必需                      |
| 隐式枚举器到整数的转换 | C++:`int one = Prime::One `C#:`int one = Prime::One` | Yes            | No      | No                      |
| 范围解析运算符     | C++:`Prime::One `C#:`Prime.One`                      | 自选             | 必需      | 必需                      |
| 隐式底层类型      | `enum E {};`                                         | `int`or larger | `int`   | `int`                   |
| 声明所需的基础类型   | `enum E;`                                            | Yes            | No      | N/A   (no declarations) |
| 从整数初始化      | C++:`Prime p{4} `C#:`Prime p = 4`                    | Yes            | Yes     | No                      |
| 立即变量        | `enum Prime {} p;`                                   | Yes            | Yes     | No                      |
| 使用按位运算符的要求  |                                                      | None           | Casting | None (`[Flags]`自选)      |

在 C++ 中的两种枚举中，范围枚举绝对最接近 C# 枚举。尽管如此，C++ 仍具有无作用域枚举，并且它们很常用。
</br>了解它们、范围枚举和 C# 枚举之间的区别非常重要，因为它们有许多细微的差异需要牢记。

















