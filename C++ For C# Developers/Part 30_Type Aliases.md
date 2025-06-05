# C++ For C# Developers: Part 28 – Variadic Templates


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6095),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C# 支持使用 `using ScoreMap = System.Collections.Generic.Dictionary<string, int>;` 指令来创建类型别名。
</br>这使得我们可以使用 `ScoreMap` 而不是冗长的 `System.Collections.Generic.Dictionary<string, int>` 或甚至 `Dictionary<string, int>`。
</br>C++ 也支持类型别名，但它们的功能远超 C#。今天我们将深入了解 C++ 提供的所有功能，以使我们的代码更加简洁和易读。

## Typedef  类型定义

在C++中创建类型别名的两种主要方法。

第一种是`typedef`，它继承自C语言。在C++代码库中仍然很常见，但文章后面我们将学习另一种本质上可以完全替代`typedef`的方法。

以这种方式创建别名，我们编写`typedef SourceType AliasName`;

```c++
// 创建一个名为“uint32”的“unsigned int”别名
typedef unsigned int uint32;
 
// 使用“uint32”别名代替“unsigned int”
constexpr uint32 ZERO = 0;
```

C#中的`using X = Y;`别名只能出现在两个地方。
- 如果它们放在.cs文件的开始处，那么它们的作用域就在该文件中。
- 如果它们放在命名空间块中，那么它们的作用域就在该块中。这意味着它们在其他命名空间块或其他文件中不可用。

C++的类型别名工作方式不同。它们可以被添加到其他类型的范围中，并且可以在文件之间使用：

```c++
////////////
// Math.h //
////////////
 
namespace Integers
{
    // 在整数命名空间中添加“uint32”别名以表示“unsigned int”
    typedef unsigned int uint32;
}
 
// 像Integers命名空间中的任何其他成员一样使用“uint32”
constexpr Integers::uint32 ZERO = 0;
 
////////////
// Game.h //
////////////
 
// 包含头文件以访问Integers命名空间和ZERO
#include "Math.h"
 
constexpr Integers::uint32 MAX_HEALTH = 100;
 
//////////////
// Game.cpp //
//////////////
 
// 包含头文件以获取Integers、ZERO和MAX_HEALTH的访问权限
#include "Game.h"
 
DebugLog(ZERO); // 0
DebugLog(MAX_HEALTH); // 100
 
// 类型别名在这里也可以使用
for (Integers::uint32 i = 0; i < 3; ++i)
{
    DebugLog(i); // 0, 1, 2
}
```

这个例子将类型别名添加到了一个命名空间中，但我们几乎可以将它们添加到任何类型的范围中。例如，我们可能只为单个函数添加一个别名：

```c++
void Foo()
{
    // 仅限于这个函数的作用域的类型别名
    typedef unsigned int uint32;
 
    for (uint32 i = 0; i < 3; ++i)
    {
        DebugLog(i); // 0, 1, 2
    }
}
```

或者甚至是一个函数内的代码块：

```c++
void Foo()
{
    {
        // 仅限于这个语句块的作用域的类型别名
        typedef unsigned int uint32;
 
        for (uint32 i = 0; i < 3; ++i)
        {
            DebugLog(i); // 0, 1, 2
        }
    }
 
    // 编译器错误：类型别名仅在上面的块中可见
    uint32 x = 0;
}
```

在类中添加类型别名也是很常见的：

```c++
struct Player
{
    // Player::HealthType现在是一个“unsigned int”的别名
    typedef unsigned int HealthType;
 
    // 在这里我们可以使用它而不需要命名空间限定符
    HealthType Health = 0;
};
 
// 我们可以通过添加命名空间限定符在类外使用它
void ApplyDamage(Player& player, Player::HealthType amount)
{
    player.Health -= amount;
}
```

这种做法特别有用，当我们认为我们以后可能会更改`Health`的类型时。
</br>我们只需简单地更新`typedef`行，将其更改为`typedef unsigned long long int HealthType`;，`Health`和`amount`的类型都会相应改变。
</br>在一个更大的项目中，这可能会让我们免于更新数百或数千个类型。

重要的是要记住，就像C#中的类型别名一样，这些`typedef`语句并不会创建新的类型。
</br>当我们使用`uint32`时，这与我们使用`unsigned int`完全相同。
</br>我们创建的别名正是这样：另一种指代同一类型的方式。

除了我们迄今为止使用的简单`typedef`语句之外，我们还可以编写一些更复杂的语句。

首先，我们可以在一个语句中创建多个别名。这类似于一次性声明多个变量：

```c++
// 创建四个类型别名：
// 1) 'Int' 代表 'int'
// 2) 'IntPointer' 代表 'int*'，即 '一个指向整数的指针'
// 3) 'FunctionPointer' 代表 'int (&)(int, int)'，即 '指向接受两个整数并返回整数的函数的引用'
// 4) 'IntArray' 代表 'int[2]'，即 '两个整数的数组'
typedef int Int, *IntPointer, (&FunctionPointer)(int, int), IntArray[2];
 
Int one = 1;
DebugLog(one); // 1
 
IntPointer p = &one;
DebugLog(*p); // 1
 
int Add(int a, int b)
{
    return a + b;
}
 
FunctionPointer add = Add;
DebugLog(add(2, 3)); // 5
 
IntArray array = { 123, 456 };
DebugLog(array[0], array[1]); // 123, 456
```

其次，有时使用`typedef`来创建结构体类型。这是从C语言继承下来的，但在C++中并不是必需的，但一些遗留代码可能仍然这样做，并且为了向后兼容，这是被支持的。
</br>这在C和C++中都是有效的：

```c++
// C code
 
// 创建两个类型别名：
// 1) 将 'Player' 用于 'struct { int Health; int Speed; }'
// 2) 将 'PlayerPointer' 用于 'Player*'，即 'Player' 的指针
typedef struct
{
    int Health;
    int Speed;
} Player, *PlayerPointer;
 
Player p;
p.Health = 100;
p.Speed = 10;
DebugLog(p.Health, p.Speed); // 100, 10
 
PlayerPointer pPlayer = &p;
DebugLog(pPlayer->Health, pPlayer->Speed); // 100, 10
```

如果没有使用`typedef`，C代码将被迫像这样在Player前加上struct：

```c++
// C code
 
struct Player
{
    int Health;
    int Speed;
};
 
struct Player p; // C语言需要“struct”前缀
p.Health = 100;
p.Speed = 10;
DebugLog(p.Health, p.Speed); // 100, 10
```

再次强调，在C++中，既不需要`struct`前缀，也不需要`typedef`的解决方案。
</br>重要的是要知道为什么这样使用`typedef`，因为这种用法在C++代码库中仍然很常见。

## Using Aliases  使用别名

自C++11以来，`typedef`不再是创建类型别名的首选方式。新的方式看起来更像是C#中的`using X = Y`；请注意，与`typedef`相比，别名和类型的顺序已经颠倒：

```c++
// 创建一个名为“uint32”的“unsigned int”别名
using uint32 = unsigned int;
 
// 使用“uint32”别名代替“unsigned int”
constexpr uint32 ZERO = 0;
```

我们只是在右侧列出类型名。这对于一些更复杂的类型来说，比`typedef`更易于阅读，因为别名名称不会与被别名的类型混合：

```c++
// int指针的别名
using IntPointer = int*;
 
// 一个接受两个整数并返回整数的函数的别名
using FunctionPointer = int (*)(int, int);
 
// 两个整型元素数组的别称
using IntArray = int[2];
```

这种语法与`typedef`完全等价。
- 两者都创建了对原始类型的别名，而不是新类型。
- 两者都可以出现在全局、命名空间、函数或函数块的作用域中。

大多数程序员认为这种形式更易读，因为它模仿了变量赋值的格式，并用`=`将别名与原始类型分开。

使用`using`关键字一次不能创建多个别名。这可能是最好的，因为那种`typedef`语法相对难以阅读且很少使用：

除了这些语法优势之外，使用还有功能上的改进：我们可以创建别名模板。
</br>考虑以下不使用别名模板的代码：

```c++
// 命名空间与类模板
namespace Math
{
    template <typename TComponent>
    struct Vector2
    {
        TComponent X;
        TComponent Y;
    };
}
 
// 另一个带有类模板的命名空间
namespace Collections
{
    template <typename TKey, typename TValue>
    struct HashMap
    {
        // ... 实现
    };
}
 
// 类型名称开始变长
Collections::HashMap<int32_t, Math::Vector2<float>> playerLocations;
Collections::HashMap<int32_t, Math::Vector2<int32_t>> playerScores;
 
// 缩短需要为每个模板实例化提供一个别名
using vec2f = Math::Vector2<float>;
using vec2i32 = Math::Vector2<int32_t>;
Collections::HashMap<int32_t, vec2f> playerLocations;
Collections::HashMap<int32_t, vec2i32> playerScores;
```

现在考虑如果我们能够访问别名模板：

```c++
// 类型别名的模板
// 接收两个类型参数：TKey 和 TValue
template <typename TKey, typename TValue>
using map = Collections::HashMap<TKey, TValue>; // Can use parameters in alias
 
template <typename TComponent>
using vec2 = Math::Vector2<TComponent>;
 
// 像对待任何其他模板一样将参数传递给别名
map<int32_t, vec2<float>> playerLocations;
map<int32_t, vec2<int32_t>> playerLocations;
 
// 我们仍然可以创建非模板类型别名以获得更具体
using vec2f = vec2<float>;
using vec2i32 = vec2<int32_t>;
map<int32_t, vec2f> playerLocations;
map<int32_t, vec2i32> playerScores;
 
// 甚至更加具体……
using LocationMap = map<int32_t, vec2f>;
using ScoreMap = map<int32_t, vec2i32>;
LocationMap playerLocations;
ScoreMap playerScores;
```

别名模板为我们提供了一个工具，可以在不强制别名具体类型的情况下保留一些类型参数。
</br>这些模板可以被重复使用，就像我们使用`map`和`vec`一样，而不是重复别名。随着类型的复杂性和泛化程度的提高，这变得越来越有用。


这些别名模板继承了其他类型模板的所有功能，例如函数和类模板。例如，我们可以使用非类型参数：

```c++
// 固定长度数组的类模板
template <typename TElement, int N>
struct FixedList
{
    int Length = N;
    TElement Elements[N];
 
    TElement& operator[](int index)
    {
        return Elements[index];
    }
};
 
// 接受非类型参数的别名模板：int N
template <int N>
using ByteArray = FixedList<unsigned char, N>;
 
// 将一个非类型参数传递给别名模板：<3>
ByteArray<3> bytes;
bytes[0] = 10;
bytes[1] = 20;
bytes[2] = 30;
DebugLog(bytes.Length, bytes[0], bytes[1], bytes[2]); // 3, 10, 20, 30
```

## Permissions  权限

最后，简单地说，类型别名还有最后一个用途。正如我们之前(Part15)看到的，类权限如`private`可以用来防止类外部的代码使用某些成员。这也适用于类创建的类型：

```c++
class Outer
{
    // 成员类型为私有，是“类”的默认值
    struct Inner
    {
        int Val = 123;
    };
};
 
// 编译器错误：内部类是私有的
Outer::Inner inner;
```

可以使用类型别名来避免这种限制。这是因为编译器只检查正在使用的类型的权限。如果该类型是另一个类型的别名，则别名类型的权限无关紧要且被忽略：

```c++
class Outer
{
    // 成员类型仍然是私有的
    struct Inner
    {
        int Val = 123;
    };
 
public:
 
    // 类型别名是公开的
    using InnerAlias = Inner;
};
 
// OK: 使用InnerAlias的权限级别，而不是Inner
Outer::InnerAlias inner;
```

通常我们一开始就会指定所需的权限级别。在例如使用第三方库的情况下，我们无法更改原始的权限级别。可以使用这个解决方案来获取所需的访问权限。

## Conclusion  结论

C++中的类型别名远远超出了C#中的对应物。它们不仅限于单个源代码文件或命名空间块。相反，我们可以在头文件中作为全局变量、在命名空间中以及作为类成员来声明它们。
</br>我们甚至在函数或函数块中声明简短的名字，以避免大量的类型冗余，尤其是在使用像`HashMap<TKey, TValue>`这样的泛型代码时。
</br>这些别名可以一次性创建并在整个项目中共享，而不仅仅是单个文件中。


别名模板更进一步，允许我们创建不解析为具体类型的别名。
</br>这些别名可以防止大量代码重复，并为介于非常通用的`HashMap<TKey, TValue>`和非常具体的`LocationMa`p之间的中间步骤，如映射，命名。
</br>它们继承了其他C++模板的权力，包括非类型参数(Part26)和可变数量(Part28)的参数的能力。










