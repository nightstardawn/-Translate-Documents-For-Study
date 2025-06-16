# C++ For C# Developers: Part 10 – Struct Basics

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5601),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天，让我们继续这个系列，开始研究结构。
</br>这些函数在 C++ 中比在 C# 中强大得多，因此今天我们将从定义和初始化它们等基础知识开始。请继续阅读以开始使用！

## Declaration and Definition  声明和定义

就像函数和枚举一样，结构可以单独声明和定义：

```c++
// 定义
struct Vec3;
 
// 声明
struct Vec3
{
    float x;
    float y;
    float z;
};
```

注意结构声明和定义看起来与枚举声明和定义非常相似。
</br>我们使用`struct`关键字，给它一个名字，添加大括号来包含其内容，然后以分号结束。


我们可以创建结构体变量，就像创建原始类型和枚举类型的变量：

```c++
Vec3 vec;
```

与原始类型和枚举类型一样，这个变量未初始化。
</br>与C#相比，结构体的初始化是一个令人惊讶的复杂话题，我们将在本系列的后续部分深入探讨。
</br>现在，让我们通过单独设置结构体的每个数据成员来初始化。这是C++中对C#中字段的等效术语。它们也通常被称为“成员变量”。
</br>为此，我们使用点操作符（`.`），就像在C#中一样：

```c++
Vec3 vec;
vec.x = 1;
vec.y = 2;
vec.z = 3;
 
DebugLog(vec.x, vec.y, vec.z); // 1, 2, 3
```

我们还可以使用 `=x` 或 `{x}` 初始化结构定义中的数据成员：

```c++
struct Vec3
{
    float x = 1;
    float y{2};
    float z = 3;
};
 
Vec3 vec;
DebugLog(vec.x, vec.y, vec.z); // 1, 2, 3
```

与枚举一样，我们也可以在定义的右大括号和分号之间声明变量：

```c++
struct Vec3
{
    float x;
    float y;
    float z;
} v1, v2, v3;
```

有时在省略结构名称时使用。这个匿名结构没有我们可以输入的名称，但可以像 C# 元组 （） 一样使用它 `(string Name, int Year) t = ("Apollo 11", 1969)`; ：

```c++
// 匿名结构体带有即时变量
struct
{
    const char16_t* Name;
    int32_t Year;
} moonMission;
 
// 此类变量可以像命名结构体类型一样使用
moonMission.Name = u"Apollo 11";
moonMission.Year = 1969;
DebugLog(moonMission.Name, moonMission.Year);
```

因为匿名结构只能通过立即变量使用，所以不允许声明一个没有任何立即变量的结构：

```c++
// 编译器错误：匿名结构体至少需要一个直接变量
struct { float x; };
```

就像那些底层类型不在声明中的枚举一样，在声明后编译器不知道结构体的尺寸。
</br>需要定义来知道其尺寸，因此声明的结构体之前不能用来创建变量或定义使用结构体类型作为参数或返回值的函数：

```c++
// 定义
struct Vec3;
 
// 编译错误：在定义之前不能创建变量
Vec3 v;
 
// 编译器错误：在定义之前不能取函数参数
float GetMagnitudeSquared(Vec3 vec)
{
    return 0;
}
 
// 编译器错误：在定义之前不能返回函数的返回值
Vec3 MakeVec(float x, float y, float z)
{
    // C编译错误：在定义之前不能创建变量
    Vec3 v;
 
    // 编译器错误：在定义之前不能返回结构体
    return v;
}
```

这也意味着我们不能在 `struct` 声明后声明立即变量：

```c++
// 编译错误：在定义之前不能创建变量
struct Vec3 v1, v2, v3;
```

但是，我们可以使用指针和对结构体的引用，因为它们不依赖于其大小：

```c++
// 定义
struct Vec3;
 
// 结构指针
Vec3* p = nullptr;
 
// 左值引用
float GetMagnitudeSquared(Vec3& vec)
{
    return 0;
}
 
// 右值引用
float GetMagnitudeSquared(Vec3&& vec)
{
    return 0;
}
```

要访问指针的字段，我们可以使用 `*p` 取消引用，然后使用 `.x` 或使用简写 `p->x`。
</br>两者都与 C# 中的结构指针完全相同。对于左值或右值引用，我们只使用 `.`，因为它们本质上只是变量的别名 ，而不是指针。

```c++
// 变量
Vec3 vec;
vec.x = 1;
vec.y = 2;
vec.z = 3;
 
// 指针
Vec3* p = &vec;
p->x = 10;
p->y = 20;
(*p).z = 30; // p->z的另一种版本
 
// 左值引用
float GetMagnitudeSquared(Vec3& vec)
{
    return vec.x*vec.x + vec.y*vec.y + vec.z*vec.z;
}
 
// 右值引用
float GetMagnitudeSquared(Vec3&& vec)
{
    return vec.x*vec.x + vec.y*vec.y + vec.z*vec.z;
}
```

## Layout  布局

与在 C# 中一样，结构的数据成员在内存中分组在一起。
</br不过，它们在内存中的确切布局方式并未由 C++ 标准定义。每个编译器都将根据所编译的 CPU 体系结构等因素对数据成员进行布局。

这类似于 C# 中的默认结构布局，其行为就像 `[StructLayout(LayoutKind.Auto)]` 是显式添加的一样。
</br>C++ 中没有 `[StructLayout]` 属性，但可以使用特定于编译器的预处理器指令来获得类似级别的控制。

也就是说，编译器几乎总是以可预测的模式布置数据成员。
</br>每个 `Cookie` 都按照源代码中写入的顺序按顺序放置。
</br>根据数据类型的对齐要求在数据成员之间放置填充，这因 CPU 体系结构而异。例如：

```c++
struct Padded  // 占据 8 bytes
{
    int8_t a;  // 占据 1 byte
               // 填充 3 bytes
    int32_t b; // 咱局 4 bytes
};
```

不过，C++ 标准确实做出了一个保证：“标准布局”。这意味着，如果两个结构体以相同的数据类型序列开始，那么这些数据成员的布局将相同。
</br>这有复杂的例外，但它适用于大多数此类正常用例。这意味着我们可以安全地重新解释一些常见的 `struct` 类型：

```c++
struct Vec3
{
    float x;
    float y;
    float z;
};
 
struct Quat
{
    // 以与Vec3相同的三个浮点数开始
    float x;
    float y;
    float z;
 
    // 不属于常见。可能稍后放置在内存中的任何位置。
    float w;
};
 
// 将 Vec3 重新解释为 Quat
Vec3 vec;
Vec3* pVec = &vec;
Quat* pQuat = (Quat*)pVec;
 
// 因为类型匹配，所以可以使用前三个起始数据成员
pQuat->x = 1;
pQuat->y = 2;
pQuat->z = 3;
 
// 绝对不安全使用最后一个数据成员
// Vec3 没有第四个浮点数
// 这是一种未定义的行为，可能会损坏内存
pQuat->w = 4;
 
DebugLog(pQuat->x, pQuat->y, pQuat->z); // 1, 2, 3
DebugLog(pVec->x, pVec->y, pVec->z); // 1, 2, 3
DebugLog(vec.x, vec.y, vec.z); // 1, 2, 3
```

## Bit Fields  位域

在 C# 中，我们可以手动创建位字段 ，但 C++ 本身支持所有整数数据成员（包括 `bool`）的位字段。这允许我们指定数据成员占用多少位内存：

```c++
struct Player
{
    bool IsAlive : 1;
    uint8_t Lives : 3;
    uint8_t Team : 2;
    uint8_t WeaponID : 2;
};
```

此结构只占用一个字节的内存，因为其位域的大小之和为 8。通常它会占用 4 个字节，因为每个数据成员都会占用自己的一整个字节。

我们可以像往常一样访问这些数据成员：

```c++
Player p;
p.IsAlive = true;
p.Lives = 5;
p.Team = 2;
p.WeaponID = 1;
 
DebugLog(p.IsAlive, p.Lives, p.Team, p.WeaponID); // true, 5, 2, 1
```

编译器将始终为编译的架构生成特定的CPU指令，并取决于诸如优化级别等设置。
</br>然而，通常情况下，指令将读取包含所需位的字节之一或多个，使用位掩码来移除读取的其他位，并将所需的位移位到数据成员类型的最低有效位。 向位字段写入的过程与此类似。

从 C++20 开始，位字段可以像其他数据成员一样在结构定义中初始化：

```c++
struct Player
{
    bool IsAlive : 1 = true;
    uint8_t Lives : 3 {5};
    uint8_t Team : 2 {2};
    uint8_t WeaponID : 2 = 1;
};
 
DebugLog(p.IsAlive, p.Lives, p.Team, p.WeaponID); // true, 5, 2, 1
```

请注意，位域的大小可能大于声明的类型：

```c++
struct SixtyFourKilobits
{
    uint8_t Val : 64*1024;
};
```

`Val` 和结构体本身的大小为 64 KB，但 Val 仍然像 8 位整数一样使用。

位域也可以是未命名的：

```c++
struct FirstLast
{
    uint8_t First : 1; // 字节的第一位
    uint8_t : 6;       // 跳过六个比特
    uint8_t Last : 1;  // 字节的最后一位
};
```

未命名的位域也可以具有零大小，这告诉编译器将下一个数据成员放在它对齐的下一个字节上：

```c++
struct FirstBitOfTwoBytes
{
    uint8_t Byte1 : 1;  // 字节的第一位
    uint8_t : 0;        // 跳到下一个字节
    uint8_t Byte2 : 1;  // 第二个字节的第一位
};
```

最后，由于位域不一定从字节的开头开始，因此我们不能获取它们的内存地址：

```c++
FirstBitOfTwoBytes x;
 
// 编译器错误：不能取位字段的地址
uint8_t* p = &x.Byte1;
```

## Static Data Members  静态数据成员

与 C# 中的静态字段一样，数据成员在 C++ 中可以是静态的：

```c++
struct Player
{
    int32_t Score;
    static int32_t HighScore;
};
```

其意义与C#相同。每个`Player`对象没有自己的`HighScore`，而是所有`Player`对象共享一个`HighScore`。
</br>因为它绑定到结构体类型，而不是结构体的实例，所以我们使用作用域解析运算符(`::`)，就像我们使用作用域枚举一样来访问数据成员：

```c++
Player::HighScore = 0;
```

我们放在 `struct` 定义里面的其实只是一个变量的声明，所以我们还是需要在 struct 之外定义它：

```c++
struct Player
{
    int32_t Score;
    static int32_t HighScore; // 定义
};
 
// 声明
int32_t Player::HighScore;
 
// 定义错误
// 这只是创建了一个新的 HighScore 变量
// 我们需要“Player::”这部分来引用声明
int32_t HighScore;
```

这也给了我们一个初始化变量的机会：

```c++
int32_t Player::HighScore = 0;
```

因为结构定义中的静态数据成员只是一个声明，所以它可以使用尚未定义的其他类型，只要它们在我们定义静态数据成员时已经定义：

```c++
// 定义
struct Vec3;
 
struct Player
{
    int32_t Health;
 
    // 声明
    static Vec3 Fastest;
};
 
// 定义
struct Vec3
{
    float x;
    float y;
    float z;
};
 
// 声明
Vec3 Player::Fastest;
```

如果静态数据成员是 `const`，我们可以内联初始化它。我们将在本系列的后面部分介绍 `const`，但现在它类似于 C# 中的 `readonly`。

```c++
struct Player
{
    int32_t Health;
    const static int32_t MaxHealth = 100;
};
```

我们仍然可以将定义放在结构体之外，但这样做是可选的。如果我们这样做，我们只能将初始化放在以下两个位置之一：

```c++
// 选项1：在结构定义中初始化
struct Player
{
    int32_t Health;
    const static int32_t MaxHealth = 100;
};
const int32_t Player::MaxHealth;
 
// 选项2：在结构体定义外部初始化
struct Player
{
    int32_t Health;
    const static int32_t MaxHealth;
};
const int32_t Player::MaxHealth = 100;
 
// 如果在两个地方初始化，则编译器会报错
struct Player
{
    int32_t Health;
    const static int32_t MaxHealth = 100;
};
const int32_t Player::MaxHealth = 100;
```

最后，静态数据成员不能是位字段。这毫无意义，因为它们不是结构体实例的一部分，甚至不必与结构体的其他静态数据成员一起位于内存中：

```c++
struct Flags
{
    // 所有这些都是编译器错误
    static bool IsStarted : 1;
    static bool WonGame : 1;
    static bool GotHighScore : 1;
    static bool FoundSecret : 1;
    static bool PlayedMultiplayer : 1;
    static bool IsLoggedIn : 1;
    static bool RatedGame : 1;
    static bool RanBenchmark : 1;
};
```

要解决此问题，请创建一个具有非静态位字段的结构体和另一个具有第一个结构体的静态实例的结构体：

```c++
struct FlagBits
{
    bool IsStarted : 1;
    bool WonGame : 1;
    bool GotHighScore : 1;
    bool FoundSecret : 1;
    bool PlayedMultiplayer : 1;
    bool IsLoggedIn : 1;
    bool RatedGame : 1;
    bool RanBenchmark : 1;
};
 
struct Flags
{
    static FlagBits Bits;
};
 
FlagBits Flags::Bits;
 
Flags::Bits.WonGame = true;
```

## Disallowed Data Members  不允许的数据成员

C++ 禁止在结构中使用某些类型的数据成员。

首先，数据类型不允许使用 `auto`：

```c++
struct Bad
{
    // 编译器错误：即使我们内联初始化，也不允许使用auto
    auto Val = 123;
};
```

此规则的一个例外是当数据成员同时是 `static` 和 `const` 时：

```c++
struct Good
{
    // OK 因为数据成员是静态和常量的
    static const auto Val = 123;
};
```

接下来，虽然 `register` 仅对其他类型的变量不推荐使用，但对于数据成员来说，它是非法的：

```c++
struct Bad
{
    // 编译器错误：数据成员不能是寄存器变量
    register int Val = 123;
};
```

这也适用于其他存储类指定符，如`extern`：

```c++
struct Bad
{
    // 编译器错误：数据成员不能是extern变量
    extern int Val = 123;
};
```

但是，整个结构体可以用任意的存储类指定符来声明：

```c++
struct Good
{
    uint8_t Val;
};
 
register Good r;
extern Good e;
```

虽然我们上面看到，尚未定义的类型可以用于静态数据成员，但这并不适用于非静态数据成员：

```c++
struct Vec3;
 
struct Bad
{
    // 编译错误：Vec3尚未定义
    Vec3 Pos;
};
```

与其他已声明但尚未定义的类型的变量一样，我们允许有指针和引用：

```c++
struct Vec3;
 
struct Good
{
    // 可以有一个指向已声明但尚未定义的类型指针
    Vec3* PosPointer;
 
    // 允许有已声明但尚未定义的类型左值
    Vec3& PosLvalueReference;
 
    // 允许将右值赋给已声明但尚未定义的类型
    Vec3&& PosRvalueReference;
};
```

## Nested Types  嵌套类型

C++ 允许我们在结构中嵌套类型，就像在 C# 中一样。让我们从一个有作用域的枚举开始：

```c++
struct Character
{
    enum struct Type
    {
        Player,
        NonPlayer
    };
 
    Type Type;
};
 
Character c;
c.Type = Character::Type::Player;
```

请注意我们如何使用 `Character：：Type` 来引用 `Character` 中的 `Type`，然后使用 `：:P layer` 来引用 `Type` 中的枚举器。

还要注意我们如何同时拥有 `Type` 枚举和 `Type` 数据成员。这两者由用于访问结构体内容的运算符消除歧义：

```c++

Character c;
Character* p = &c;
Character& r = c;
 
// "." 操作符表示“访问数据成员”
auto t = c.Type;
t = r.Type;
 
// "->" 运算符表示“解除指针引用然后访问数据成员”
t = p->Type;
 
// "::" 运算符表示“获取与类型相关的某个内容”
Character::Type t2;

```

如果数据成员是静态的，并且与嵌套类型具有相同的名称，则会出现歧义：

```c++
struct Character
{
    enum struct Type
    {
        Player,
        NonPlayer
    };
 
    static Type Type;
};
 
// 编译器错误：Character::Type 是模糊的
// 它可能是作用域枚举或静态数据成员
Character::Type Character::Type = Character::Type::Player;
```

我们还可以嵌套无作用域的枚举：

```c++
struct Character
{
    enum Type
    {
        Player,
        NonPlayer
    };
 
    Type Type;
} c;
 
// 可选地指定未限定的枚举类型名称
c.Type = Character::Type::Player;
 
// 或者不指定它
// 枚举器被添加到周围的作用域：结构体
c.Type = Character::Player;
```

最后，我们可以在 `struct` 中嵌套 structs。与枚举一样，这可用于将它们置于上下文中，例如清理上面的 Flags 示例：


```c++
struct Flags
{
    struct FlagBits
    {
        bool IsStarted : 1;
        bool WonGame : 1;
        bool GotHighScore : 1;
        bool FoundSecret : 1;
        bool PlayedMultiplayer : 1;
        bool IsLoggedIn : 1;
        bool RatedGame : 1;
        bool RanBenchmark : 1;
    };
 
    static FlagBits Bits;
};
 
Flags::FlagBits Flags::Bits;
```

我们可以将其与匿名结构结合使用，以消除一些冗长。如果我们这样做，我们需要使用 `decltype` 来声明静态变量的类型，当我们在 `struct` 之外定义它时，因为我们没有给它一个显式的名称：

```c++
struct Flags
{
    // 未命名的结构体带有位字段
    // 数据成员Bits是静态的
    static struct
    {
        bool IsStarted : 1;
        bool WonGame : 1;
        bool GotHighScore : 1;
        bool FoundSecret : 1;
        bool PlayedMultiplayer : 1;
        bool IsLoggedIn : 1;
        bool RatedGame : 1;
        bool RanBenchmark : 1;
    } Bits;
};
 
// 未命名的结构体没有我们可以直接键入的名称，可以使用decltype来引用其类型
decltype(Flags::Bits) Flags::Bits;
 
Flags::Bits.WonGame = true;
```

当然，我们可以继续将结构体无限嵌套在其他结构体中，但通常最好将其保持在两到三个级别，并避免诉诸于这样的东西：
```c++
struct S1
{
    struct S2
    {
        struct S3
        {
            struct S4
            {
                struct S5
                {
                    uint8_t Val;
                };
            };
        };
    };
};
 
S1::S2::S3::S4::S5 s;
s.Val = 123;
```

## Conclusion  结论

我们只是触及了 C++ 结构的皮毛，它们已经具有比 C# 结构更高级的功能：


| 特性          | 示例代码                                       |
|-------------|--------------------------------------------|
| 分离声明与定义     | `struct S; struct S{};`                    |
| 内联数据成员初始化   | `struct S {int X=1, int Y=2};`             |
| 位域          | `struct S {bool a:1; bool :6; bool b:1;};` |
| 立即变量        | `struct S {} s;`                           |
| 匿名结构体       | `struct {float X; float Y;} pos2;`         |
| 结构体引用       | `struct S {} s; S& lr = s; S&& rr = S();`  |
| 自动数据成员类型    | `struct S {static const auto X=1;};`       |
| 嵌套类型与数据成员同名 | `struct S {enum E{}; E E;};`               |












