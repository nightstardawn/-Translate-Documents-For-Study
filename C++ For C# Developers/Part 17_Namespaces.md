# C++ For C# Developers: Part 17 – Namespaces

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5772),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

结构体封装完毕后，我们可以继续探讨C++的其他特性。
</br>今天，我们将探讨命名空间。我们将涵盖C#提供的基本内容，但会深入得多，并涵盖许多高级功能。继续阅读，了解所有这些内容！

命名空间在 C++ 中的高级用途与在 C# 中的作用相同。它们允许我们重用标识符并通过命名空间消除歧义。基本语法甚至看起来相同：

```c++
namespace Math
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
```

与 C# 一样，我们可以通过简单地重用名称来重新打开命名空间以添加到其中：

```c++
namespace Math
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
 
namespace Math
{
    struct Vector3
    {
        float X;
        float Y;
        float Z;
    };
}
```

我们还可以嵌套命名空间：

```c++
namespace Math
{
    namespace LinearAlgebra
    {
        struct Vector2
        {
            float X;
            float Y;
        };
    }
}
```

访问命名空间的成员略有不同。正如我们在枚举和结构中看到的那样，我们继续使用范围解析运算符 `A：：B` 而不是 C# 的点语法 `A.B`。

```c++
Math::Vector2 vec{2, 4}; // Refer to Vector2 in the Math namespace
DebugLog(vec.X, vec.Y); // 2, 4
```

要引用为全局范围隐式创建的命名空间，我们使用 `：：B` 而不是 C# 中的 `global：：B`

```c++
int32_t highScore = 0;
 
class Player
{
    int32_t numPoints;
    int32_t highScore;
 
    void ScorePoints(int32_t num)
    {
        numPoints += num;
 
        // highScore 指的是数据成员
        if (numPoints > highScore)
        {
            highScore = numPoints;
        }
 
        // “::highScore”指的是全局变量
        if (numPoints > ::highScore)
        {
            ::highScore = numPoints;
        }
    }
};
```

从 C++17 开始，我们还可以使用 scope resolution 运算符来创建嵌套命名空间：

```c++
namespace Math::LinearAlgebra
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
```

与 C# 不同，我们不仅限于将 struct 和 enum 等类型放在命名空间中。我们可以在那里放置任何我们想要的东西：

```c++
namespace Math
{
    // 变量
    const float PI = 3.14f;
 
    // 函数
    bool IsNearlyZero(float val, float threshold=0.0001f)
    {
        return abs(val) < threshold;
    }
}
```

我们还可以将声明放在命名空间内，将定义放在外面：

```c++
namespace Math
{
    // 声明
    struct Vector2;
    bool IsNearlyZero(float val, float threshold=0.0001f);
}
 
// 定义
struct Math::Vector2
{
    float X;
    float Y;
};
bool Math::IsNearlyZero(float val, float threshold)
{
    return abs(val) < threshold;
}
```

定义需要位于封闭命名空间或全局范围内：

```c++
namespace Math
{
    // 声明
    struct Vector2;
    bool IsNearlyZero(float val, float threshold=0.0001f);
}
 
// 定义
namespace Other
{
    // 编译器错误：'Other' 不是一个封装的命名空间或全局作用域
    struct Math::Vector2
    {
        float X;
        float Y;
    };
 
    // 编译器错误：'Other' 不是一个封装的命名空间或全局作用域
    bool Math::IsNearlyZero(float val, float threshold)
    {
        return abs(val) < threshold;
    }
}
```
> 读者认为"封闭命名空间"应该是指在同一个命名空间下，如下面的案例
> ```c++
> namespace Outer {
>     namespace Inner {
>         void func(); // 声明在Inner中
>     }
> 
>     // 定义在Outer（父命名空间）中
>     void Inner::func() {
>         // 实现
>     }
> }
> ```
请注意，仅包含函数的命名空间是模拟 C# 静态类的另一种方法。

## Using Directives  Using 指令

显式写出像 Math：： 这样的命名空间名称会变得乏味和冗长。与 C# 一样，C++ 具有 using 指令来缓解此问题。语法看起来类似：

```c++
// Using directive
using namespace Math;
 
// No need for Math::
Vector2 vec{2, 4};
DebugLog(vec.X, vec.Y); // 2, 4
```

与 C# 不同，using 指令必须位于文件的顶部，而 C++ 允许它们在全局范围中的任何位置、命名空间中，甚至在任何块中：

```c++
namespace MathUtils
{
    // Using directive inside a namespace
    using namespace Math;
 
    bool IsNearlyZero(Vector2 vec, float threshold=0.0001f)
    {
        return abs(vec.X) < threshold && abs(vec.Y) < threshold;
    }
}
 
void Foo()
{
    // Using directive inside a function
    using namespace Math;
 
    // No need for Math::
    Vector2 vec{2, 4};
    DebugLog(vec.X, vec.Y); // 2, 4
}
 
enum struct Op
{
    IS_NEARLY_ZERO
};
 
bool DoOp(Math::Vector2 vec, Op op)
{
    if (op == Op::IS_NEARLY_ZERO)
    {
        // Using directive inside a block
        using namespace MathUtils;
 
        return IsNearlyZero(vec);
    }
    return false;
}
```

与C#不同，使用指令是可传递的。
</br>在上面的代码中，`MathUtils`命名空间有`using namespace Math`。
</br>这意味着任何`using namespace MathUtils`都隐式地包含了`using namespace Math`：

```c++
// 隐式包含 MathUtils 的 '使用命名空间 Math'
using namespace MathUtils;
 
// 无需使用Math::，因为使用了transitive using指令
Vector2 vec{2, 4};
 
// 无需MathUtils::
DebugLog(IsNearlyZero(vec)); // false
```

即使在`using`指令之后添加到命名空间中的成员也会被递归包含：

```c++
namespace Math
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
 
namespace MathUtils
{
    using namespace Math;
 
    bool IsNearlyZero(Vector2 vec, float threshold=0.0001f)
    {
        return abs(vec.X) < threshold && abs(vec.Y) < threshold;
    }
}
 
namespace Math
{
    struct Vector3
    {
        float X;
        float Y;
        float Z;
    };
}
 
void Foo()
{
    // 隐式包含MathUtils的"使用命名空间Math"
    // 即使在 'using namespace Math' 之后，也包含了 Vector3
    using namespace MathUtils;
 
    Vector3 vec{2, 4, 6};
    DebugLog(vec.X, vec.Y, vec.Z); // 2, 4, 6
}
```

请注意，通常认为在头文件的全局作用域中放置`using`指令是一种不好的做法，因为它将整个命名空间以及任何间接使用的命名空间强加给所有包含它的文件。

## Inline Namespaces 内联命名空间

C++ 命名空间可以是内联的：

```c++
inline namespace Math
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
```

这与紧随非内联命名空间之后的`using`指令类似：

```c++
namespace Math
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
using namespace Math;
```

因此，我们可以在没有范围解析运算符或显式 `using` 指令的情况下使用命名空间的成员：

```c++
Vector2 vec{2, 4};
DebugLog(vec.X, vec.Y); // 2, 4
```

从 C++20 开始，在定义嵌套命名空间时，我们可以在每个名称之前添加关键字 inline，但第一个名称除外：

```c++
// Math 是一个非内联命名空间
// LinearAlgebra 是嵌套在 Math 中的内联命名空间
namespace Math::inline LinearAlgebra
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
 
// 仍需要使用 Math::，因为 Math 不是一个内联命名空间
Math::Vector2 vec{2, 4};
DebugLog(vec.X, vec.Y); // 2, 4
```

这为内联命名空间的一般用例增加了便利。
</br>通常，人们希望将功能分组在一起，同时也提供部分功能。例如，C++标准库提供了字符串和小时类型的用户定义字面量，如下所示：

```c++
namespace std::inline literals::inline string_literals
{
    std::string operator""s(const char* chars, size_t len)
    {
        // 函数体
    }
}
namespace std:inline literals::inline chrono_literals
{
    std::chrono::hours operator""h(long double val)
    {
        // 函数体
    }
}
 
void UseAllLiterals()
{
    // 间接使用字符串字面量和时间字面量
    // 无需指定每个
    using namespace std::literals;
 
    std::string greeting = "hello"s;
    std::chrono::hours halfHour = 0.5h;
}
 
void UseJustStringLiterals()
{
    // 仅使用字符串字面量
    using namespace std::literals::string_literals;
 
    std::string greeting = "hello"s;
    std::chrono::hours halfHour = 0.5h; // 编译器错误
}
 
void UseJustChronoLiterals()
{
    // 仅使用chrono_literals
    using namespace std::literals::chrono_literals;
 
    std::string greeting = "hello"_s; // 编译器错误
    std::chrono::hours halfHour = 0.5h;
}
```

> 这个作者给的例子，有一点点小复杂，这里读者给两个个内联最直观的例子，看完这再看上面就简单了
> 
> </br>第一个
> ```c++
> namespace Library {
>     inline namespace Version1 { // 内联命名空间
>         void func() { std::cout << "Version1\n"; }
>     }
>     namespace Version2 {        // 普通命名空间
>         void func() { std::cout << "Version2\n"; }
>     }
> }
> 
> int main() {
>     Library::func();        // 直接访问 Version1::func（内联）
>     Library::Version2::func(); // 显式访问 Version2::func
> }
> ```
> </br>第二个
> ```c++
> namespace A {
>     inline namespace B {
>         inline namespace C { // 多层内联
>             void func() {}
>         }
>     }
> }
> // 所有层级均可直接访问
> A::func(); 
> ```

## Unnamed Namespaces  未命名命名空间

命名空间可以没有名称。就像内联命名空间一样，这些命名空间后面有一个隐式的 using 指令：

```c++
namespace
{
    struct Vector2
    {
        float X;
        float Y;
    };
}
 
// 由编译器隐式添加
// UNNAMED 仅是编译器为命名空间提供的占位符
using namespace UNNAMED;
 
// 可以使用未命名的命名空间中的成员
Vector2 vec{2, 4};
DebugLog(vec.X, vec.Y); // 2, 4
```

因为这些命名空间没有名称，所以无法使用作用域解析运算符（`A::B`）显式地引用它们的成员，也无法在`using`指令中命名它们。

特别地，未命名命名空间的所有成员，包括嵌套命名空间，都具有内部链接，就像静态全局变量一样。

```c++
// other.cpp
int32_t Global; // 外部链接
namespace
{
    int32_t InNamespace; // 内部链接
}
 
// test.cpp
extern int32_t Global; // OK: 具有外部链接
extern int32_t InNamespace; // 链接错误：存在内部链接
                            // 在这里无法命名命名空间
void Foo()
{
    Global = 123;
    InNamespace = 456;
}
```

> 简单来来说，未命名空间中的成员，限制在了当前编译单元中(当前源文件中)

## Using Declarations  使用声明

除了 `using` 指令，C++ 还有 `using` 声明：

```c++
namespace Math
{
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
}
 
// Use just Vector2, not Vector3
using Math::Vector2;
 
Vector2 vec2{2, 4}; // OK
Vector3 vec3a{2, 4, 6}; // 编译器错误
Math::Vector3 vec3b{2, 4, 6}; // OK
```

与变量声明一样，我们可以在单个 using 声明中命名多个命名空间成员：

```c++
namespace Game
{
    class Player;
    class Enemy;
}
 
void Foo()
{
    // Use Vector2 and Player, not Vector3 or Enemy
    using Math::Vector2, Game::Player;
 
    Vector2 vec2{2, 4}; // OK
    Vector3 vec3{2, 4}; // 编译器错误
    Player* player; // OK
    Enemy* enemy; // 编译器错误
}
```

与使用指令不同，使用声明实际上会将指定的命名空间成员添加到它们声明的块中。这意味着可能会出现一些冲突：

```c++
namespace Stats
{
    int32_t score;
}
 
namespace Game
{
    struct Player
    {
        int32_t Score;
    };
}
 
bool HasHighScore(Game::Player* player)
{
    using Stats::score;
 
    int32_t score = player->Score; // 编译器错误：已声明score
    return score > score; // 对 score 的模糊引用
}
```

在一种情况下，可以有多个具有相同名称的标识符：函数重载。使用多个引用同名函数的 using 声明，我们可以在函数中创建一个重载集：

```c++
namespace Game
{
    struct Player
    {
        int32_t Health;
    };
}
 
namespace Damage
{
    struct Weapon
    {
        int32_t Damage;
    };
 
    void Use(Weapon& weapon, Game::Player& player)
    {
        player.Health -= weapon.Damage;
    }
}
 
namespace Healing
{
    struct Potion
    {
        int32_t HealAmount;
    };
 
    void Use(Potion& potion, Game::Player& player)
    {
        player.Health += potion.HealAmount;
    }
}
 
void DamageThenHeal(
    Game::Player& player, Damage::Weapon& weapon, Healing::Potion& potion)
{
    using Damage::Use; // 现在有一个 Use 函数
    using Healing::Use; // 现在有两个 Use 函数：一个重载集
 
    Use(weapon, player); //调用 Use(Weapon&, Player&) 重载
    Use(potion, player); // 调用 Use(Potion&, Player&) 重载
}
 
Game::Player player{100};
Damage::Weapon weapon{20};
Healing::Potion potion{10};
DamageThenHeal(player, weapon, potion);
DebugLog(player.Health); // 90
```

最后一个命名空间功能很简单：别名。我们可以使用这些来缩短较长的、通常是嵌套的命名空间名称：

```c++
namespace Math
{
    namespace LinearAlgebra
    {
        struct Vector2
        {
            float X;
            float Y;
        };
    }
}
 
// mla 是 Math::LinearAlgebra 的别名
namespace mla = Math::LinearAlgebra;
 
mla::Vector2 vec2{2, 4};
DebugLog(vec2.X, vec2.Y); // 2, 4
```

这与在 C# 中使用 `N1 = N2` 非常接近。主要区别在于它可以放置在全局范围、命名空间或任何其他块中。它不需要放在文件的顶部，并且仅适用于该文件。

## Conclusion  结论

C++命名空间在很多方面与C#命名空间非常相似。正如通常情况，它们构成了C#功能的粗略超集。
</br>我们拥有诸如内联命名空间、using声明和无名命名空间等高级功能。
</br>我们还可以将变量和函数放入其中，而不仅仅是类型定义。我们甚至可以将using声明和指令放置在文件的几乎任何地方，而不仅仅是顶部。

然而，随着功能的增强，复杂性也随之增加。我们需要避免过度宽泛的using指令，注意传递性using指令，并使用using声明来解决标识符冲突。

命名空间几乎在每个 C++ 代码库中都使用，就像它们几乎在每个 C# 代码库中一样。我们现在知道了如何使用它们的规则，因此我们离有效地编写 C++ 又近了一步！







