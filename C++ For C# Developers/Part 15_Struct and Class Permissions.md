# C++ For C# Developers: Part 15 – Struct and Class Permissions

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5728),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天，我们将介绍 C++ 中结构体的 最后一个主要主题： 我们如何控制对它们的访问。
</br>我们将讨论访问修饰符（如 `private`）、“友谊(friendship)”概念，最后介绍 `const` 的细节。

## Access Specifiers  访问说明符

与 C# 一样，C++ 中结构的成员可以由 public、protected 和 private 访问说明符更改其访问级别。
不过，它在 C++ 中的编写方式略有不同。
C++ 中的访问说明符不是像单个成员的修饰符（例如 public void Foo（） {}）那样包含在内，而是像标签一样编写（例如 public：）并应用到下一个访问说明符：

```c++
struct Player
{
// 默认的访问修饰符是public，因此TakeDamage是public
    void TakeDamage(int32_t amount)
    {
        Health -= amount;
    }
 
// 将访问修饰符更改为 private
private:
// Health is private
    int32_t Health;
 
// NumLives is private
    int32_t NumLives;
 
// 将访问修饰符改回 public
public:
// Heal is public
    void Heal(int32_t amount)
    {
        Health += amount;
    }
 
// GetExtraLife is public
    void GetExtraLife()
    {
        NumLives++;
    }
};
```

虽然在 C++ 中并不常见，但我们可以通过在每个成员之前显式添加访问说明符来使其感觉更像 C#：

```c++
struct Player
{
    public: void TakeDamage(int32_t amount)
    {
        Health -= amount;
    }
 
    private: int32_t Health;
    private: int32_t NumLives;
 
    public: void Heal(int32_t amount)
    {
        Health += amount;
    }
 
    public: void GetExtraLife()
    {
        NumLives++;
    }
};
```

`public`、`private` 和 `protected` 的含义类似于 C#：

| Access Specifier  </br>访问说明符 | Member Accessibile From </br> 成员访问权限                       |
|------------------------------|------------------------------------------------------------|
| public                       | Anywhere 任何位置                                              |
| protected                    | Only within the struct and in derived structs 仅在结构体和派生结构体中 |
| private                      | Only within the struct仅在结构体中                               |

与 C# 不同，从结构派生时也可以应用访问说明符：

```c++
struct PublicPlayer : public Player
{
};
 
struct ProtectedPlayer : protected Player
{
};
 
struct PrivatePlayer : private Player
{
};
 
// 默认的继承访问修饰符是公共的
struct DefaultPlayer : Player
{
};
```

继承访问说明符将基本结构中的成员访问级别映射到派生结构中的访问级别：

|                 | Inherit public	 | Inherit protected | 	Inherit private |
|-----------------|-----------------|-------------------|------------------|
| Base public     | 	public         | 	protected        | 	private         |
| Base private	   | private	        | private	          | private          |
| Base protected	 | protected	      | protected         | 	private         |

> 基本就是从小原则，向上兼容

这意味着 ProtectedPlayer 和 PrivatePlayer 对外部代码隐藏了 Player 的公共成员：

```c++
PublicPlayer pub{};
pub.Heal(10); // OK: Heal is public
 
ProtectedPlayer prot{};
prot.Heal(10); // Compiler error: Heal is protected
 
PrivatePlayer priv{};
priv.Heal(10); // Compiler error: Heal is private
 
DefaultPlayer def{};
def.Heal(10); // OK: Heal is public
```

当虚拟成员函数被重写时，它可能具有与它重写的成员函数不同的访问级别。
</br>在这种情况下，访问级别始终在编译时使用调用成员函数的类型来确定。
</br>这可能与对象的运行时类型不同。这意味着访问说明符不支持运行时多态性。

```c++
struct Base
{
    virtual void Foo()
    {
        DebugLog("Base Foo");
    }
 
private:
 
    virtual void Goo()
    {
        DebugLog("Base Goo");
    }
};
 
struct Derived : Base
{
private:
 
    virtual void Foo() override
    {
        DebugLog("Derived Foo");
    }
 
public:
 
    virtual void Goo() override
    {
        DebugLog("Derived Goo");
    }
};
 
// 这些调用使用了Base中的访问指定符
Base b;
b.Foo(); // "Base Foo"
//b.Goo(); // 编译器错误：Goo 是私有的
 
// 这些调用使用了派生类中的访问说明符
Derived d;
//d.Foo(); // 编译器错误：Foo 是私有的
d.Goo(); // "Derived Goo"
 
// 这些调用使用了Base中的访问指定符，即使运行时对象是派生对象
// is a Derived
Base& dRef = d;
dRef.Foo(); // "Derived Foo"
//dRef.Goo(); // 编译器错误：Goo 在 Base 中是私有的
```

使用虚拟继承时，通过派生类的最可访问路径用于确定访问级别：

```c++
struct Top
{
    int32_t X = 123;
};
 
// X 由于私有继承而私有
struct Left : private virtual Top
{
};
 
// X 由于公有继承而公有
struct Right : public virtual Top
{
};
 
// X 是 public 因为继承自 Right
// 因为这里是优先于继承自Left而私有
struct Bottom : virtual Left, virtual Right
{
};
 
Top top{};
DebugLog(top.X); // 123
 
Left left{};
//DebugLog(left.X); // 编译器错误：X 是私有的
 
Right right{};
DebugLog(right.X); // 123
 
// 访问 X 通过 Right 进行
Bottom bottom{};
DebugLog(bottom.X); // 123
```

请务必注意，访问级别可能会更改内存中结构的非静态数据成员的布局。虽然保证数据成员按顺序布局，也许它们之间有填充，但这仅适用于相同访问级别的数据成员。
</br>例如，编译器可以选择先布局所有公共数据成员，然后再布局所有私有数据成员，或者混合所有数据成员，而不管其访问级别如何：

```c++
struct Mixed
{
    private: int32_t A = 1;
    public: int32_t B = 2;
    private: int32_t C = 3;
    public: int32_t D = 4;
};
 
// 一些混合的可能的布局：
//   Ignore access level: A, B, C, D
//   Private then public: A, C, B, D
//   Public then private: B, D, A, C
```

结构体，包括上述所有示例，都使用public作为它们的默认访问级别。 这适用于它们的成员和继承。
</br>要使private成为默认，请将关键字struct替换为class：

```c++
class Player
{
    int32_t Health = 0;
};
 
Player player{};
DebugLog(player.Health); // 编译错误：健康是私有的
```

没错： 类只是具有不同默认访问级别的结构体！

它们非常兼容，我们甚至可以将它们声明为 struct 并将它们定义为 class，反之亦然：

```c++
struct Player;
class Player
{
    int32_t Health = 0;
};
 
class Weapon;
struct Weapon
{
    int32_t Damage = 0;
};
```

选择使用哪个主要取决于惯例。
- struct 关键字通常在所有或大多数成员都是公共的时使用。
- class 关键字通常在所有或大多数成员都是私有成员时使用。

就术语而言，通常只说 “类” 或 “结构” 而不是 “类和结构” ，因为这两个概念本质上是相同的。
例如，“structs can have constructors” 意味着类也可以有构造函数。本系列中所有前面关于结构的文章都同样适用于类。

## Friendship  友谊

C++ 为结构提供了一种方法，用于显式授予其成员完全访问权限，而不管访问级别如何。
为此，结构体在其定义中添加了一个 friend 声明，说明它想要授予访问权限的函数或结构体的名称：

```c++
class Player
{
    // 由于类的默认值，私有
    int64_t Id = 123;
    int32_t Points = 0;
 
    // 将 PrintId 函数设为 friend
    friend void PrintId(const Player& player);
 
    // 将 Stats 类设为 friend
    friend class Stats;
};
 
void PrintId(const Player& player)
{
    //因为PrintId是Player的frenid，所以可以访问Id和Points成员
    DebugLog(player.Id, "has", player.Points, "points");
}
 
// Stats 实际上是一个结构体而不是类，但这没关系
struct Stats
{
    static int32_t GetTotalPoints(Player* players, int32_t numPlayers)
    {
        int32_t totalPoints = 0;
        for (int32_t i = 0; i < numPlayers; ++i)
        {
            //因为Stats是Player的frenid，所以可以访问Id和Points成员
            totalPoints += players[i].Points;
        }
        return totalPoints;
    }
};
 
Player p;
PrintId(p); // 123 has 0 points
 
int32_t totalPoints = Stats::GetTotalPoints(&p, 1);
DebugLog(totalPoints); // 0
```

结构体与内联函数成为friend是很常见的，特别是 C++ 提供了一个定义内联函数并将其声明为friend的快捷方式：

```c++
class Player
{
    int64_t Id = 123;
    int32_t Points = 0;
 
    // 将 PrintId 函数定义为内联函数并将其设为friend
    friend void PrintId(const Player& player)
    {
        // 可以访问 Id 和 Points 成员
        DebugLog(player.Id, "has", player.Points, "points");
    }
};
```

即使 PrintId 的定义像成员函数一样出现在 Player 的定义中，它仍然是普通函数， 而不是成员函数。我们在调用它时可以看到这一点：

```c++
// 普通函数一样调用
Player p;
PrintId(p); // 123 has 0 points
 
// 成员函数一样调用
p.PrintId(); // 编译错误：玩家没有PrintId成员
```

另外，请注意，friendShip不能继承的，朋友的朋友也不是朋友：

```c++
class Player
{
    int32_t Points = 0;
    friend class Stats;
};
 
struct Stats
{
    friend class PointStats;
};
 
struct PointStats : Stats
{
    static int32_t GetTotalPoints(Player* players, int32_t numPlayers)
    {
        int32_t totalPoints = 0;
        for (int32_t i = 0; i < numPlayers; ++i)
        {
            // 编译器错误：无法访问Points
            // 因为PointStats不是Player的 friend，即使它继承自Stats并且是Stats的friend。
            // 只有Stats是friend。
            totalPoints += players[i].Points;
        }
        return totalPoints;
    }
};
```

## Const and Mutable  常量 和 Mutable

正如我们在整个系列中看到的，并且略微涉及到，C++ 中的类型可以使用 `const` 关键字进行限定。
我们可以将此限定符用于类型的任何使用：局部变量、全局变量、数据成员、函数参数、返回值、指针和引用。

从最基本的角度来看，`const` 意味着我们不能重新分配：

```c++
// x 是一个常量 int
const int32_t x = 123;
 
// 编译器错误：不能分配给常量变量
x = 456;
```

const 关键字可以放置在类型的左侧或右侧。这称为 “west const” 和 “east const” ，两者都是常用的。
在这种情况下，放置没有区别，因为两者都会导致 const 类型。

```c++
const int32_t x = 123; // "West const" version of a constant int32_t
int32_t const y = 456; // "East const" version of a constant int32_t
DebugLog(x, y); // 123, 456
```

对于稍微复杂的类型，排序更为重要。思考一下指针类型：

`const char* str` 有两种可能的含义：
1. A non-const pointer to a const char
  </br>指向 const char 的非 const 指针 
2. A const pointer to a non-const char
   </br>指向非 const char 的 const 指针

也就是说，这两者之一将是编译器错误：

```c++
//如果是情况1 那么这种情况就会编译错误 因为改变char
*str = 'H';
//如果是情况2 那么这种情况就会编译错误 因为改变指针
str = "goodbye";
```

规则是 const 修改紧邻其左侧的内容。如果左侧没有任何内容，则修改紧邻右侧的内容。
> 简单说就是先左后右的原则

因为 `const char* str` 中没有 `const` 的任何内容，所以 `const` 会立即修改其右侧的 `char`。
这意味着 `char` 是 `const` 并且指针是 `non-const`：

```c++
*str = 'H'; // 编译器错误：字符是常量
str = "goodbye"; // 好的：指针现在指向“再见”
```

使用 “east const” 消除了 “nothing to its left” 的情况，因此更容易确定什么是 const：

```c++
// 因为 'char' 在 'const' 的左边，所以 'char' 是const
char const * str = "hello";
str = "goodbye"; // OK: pointer is non-const
*str = 'H'; // Compiler error: char is const
 
// '*' 在const的左侧，因此指针是 const
char * const str = "hello";
str = "goodbye"; // Compiler error: pointer is const
*str = 'H'; // OK: char is non-const
 
// 两个 'char' 和 '*' 都位于 'const' 的左侧，因此两者都是 const
char const * const str = "hello";
str = "goodbye"; // Compiler error: pointer is const
*str = 'H';// Compiler error: char is const
```

> 这里读者的理解就是：const的东西，是不能修改的。一个是存储值的内存，一个是存储指针的内存。
>  - `str = "goodbye"` 时，指针值会变成"goodbye"所在的指针值
>  - `*str = 'H'` 时，修改的是原本指针所指向那块内存的内容，是不允许的

除了赋值之外，const 关键字还意味着我们不能修改 const 结构体的数据成员：


```c++
struct Player
{
    const int64_t Id;
 
    Player(int64_t id)
        : Id(id)
    {
    }
};
 
Player player{123};
 
player.Id = 1000;// 编译器错误：不能修改const结构的数据成员
```

对 `const T` 的引用具有类型 `const T&` 和 指向 `const T `的指针具有类型 `const T*`。
这些不能分配给类型为 `T&` 的非 const 引用和类型为 `T*` 的非 const 指针，因为这会消除它的 “const-ness”：

> 说人话就是 const的变量 对应的引用和指针 也要是const

```c++
const int32_t x = 123;
 
// 编译器错误：const int32_t& 与 int32_t& 不兼容
int32_t& xRef = x;
 
// 编译器错误：const int32_t* 与 int32_t* 不兼容
int32_t* xPtr = &x;
```

成员函数也可以通过将关键字放在函数签名之后来成为 const，类似于我们放置 override 的位置：

```c++
class Player
{
    int32_t Health = 100;
 
public:
 
    // GetHealth 是一个 const 成员函数
    int32_t GetHealth() const
    {
        return Health;
    }
};
```

结构体T的const成员函数隐式传递一个const T* this指针，而不是一个(non-const) T* this指针。
这意味着所有相同的限制都适用，并且不允许修改this的数据成员。这与C# 8.0中的readonly实例成员类似。

我们还被禁止从 struct 内部或外部在 const 结构上调用非 const 成员函数：

> 说人话就是：
> const函数只能调用const变量
> const引用和指针，只能调用const的变量和方法

```c++
class Player
{
    int32_t Health = 100;
 
public:
 
    int32_t GetHealth() const
    {
        //编译器错误：不能从const成员函数调用非const成员函数，因为'this'是一个'const Player*'
        TakeDamage(1);
 
        return Health;
    }
 
    void TakeDamage(int32_t amount)
    {
        Health -= amount;
    }
};
 
Player player{};
const Player& playerRef = player;
 
// 编译器错误：不能在const引用上调用非const的TakeDamage
playerRef.TakeDamage();
 
// 好的：GetHealth 是常量
DebugLog(playerRef.GetHealth()); // 100
```

为了取消这种限制，并允许特定的数据成员在const成员函数中被视为非const，我们使用`mutable`关键字：

```c++
class File
{
    FILE* Handle;
 
    // 可以被const成员函数修改
    mutable long Size;
 
public:
    File(const char* path)
    {
        Handle = fopen(path, "r");
        Size = -1;
    }
 
    ~File()
    {
        fclose(Handle);
    }
 
    // const function
    long GetSize() const
    {
        if (Size < 0)
        {
            long oldPos = ftell(Handle);
            fseek(Handle, 0, SEEK_END);
            Size = ftell(Handle); // OK: Size is mutable
            fseek(Handle, oldPos, SEEK_SET);
        }
        return Size;
    }
};
```

mutable 关键字通常用于缓存值，例如上述文件大小的缓存。
例如，这意味着相对昂贵的文件 I/O 作只需执行一次，而不管 GetSize 调用了多少次：

```c++
const File f{"/path/to/file"};
for (int32_t i = 0; i < 1000; ++i)
{
    // 可以在const File上调用GetSize，因为GetSize是const
    // GetSize可以更新Size，因为Size是可变的
    // 只有第一次调用才进行任何文件I/O操作：节省了999次文件I/O操作
    DebugLog(f.GetSize());
}
```

下表将 C++ 的 const 关键字与 C# 的 const 和 readonly 关键字进行了对比：


| Factor 因素                                    | 	C++ const	           | C# const                   | 	C# readonly                                                  |
|----------------------------------------------|-----------------------|----------------------------|---------------------------------------------------------------|
| Types  </br>类型                               | 	Any	                 | Numbers, strings, booleans | 	Any                                                          |
| Applicability  </br>适用性                      | 	Everywhere           | 	Fields, local variables	  | Fields, references                                            |
| Can assign to field </br>可以分配给字段             | 	Only mutable         | 	N/A	                      | No                                                            |
| Can set field of field </br>可设置字段的字段         | 	Only mutable         | 	N/A                       | 	Structs no, Classes yes                                      |
| Can set field of reference </br>可以设置参考域      | 	Only mutable         | 	N/A                       | 	Yes, but doesn't change referenced structs </br>是，但不更改引用的结构  |
| Can call function of field </br>可以调用字段的函数    | 	Only const function	 | N/A                        | 	Yes, but doesn't change field of structs </br>是的，但不会更改结构体的字段 |
| Can call function of reference </br>可以调用引用函数 | 	Only const function	 | N/A                        | 	Yes, but doesn't change referenced structs </br>是，但不更改引用的结构  |

通常，C++ 的 `const` 比 C# 的 `readonly` 提供更一致和彻底的不变性，后者是最接近的等效项。

## C# Equivalency  C# 等效性

C# 比 C++ 多三个访问级别：

- internal
- protected internal
- private protected

| Access Specifier </br>访问说明符 | 	Member Accessibile From </br>访问权限                                                        |
|-----------------------------|-------------------------------------------------------------------------------------------|
| internal	                   | Within same assembly  </br>在同一程序集中                                                        |
| protected internal          | 	Within same assembly or in derived classes </br>在同一程序集中或在派生类中                            |
| private protected           | 	Within same assembly or in derived classes within same assembly </br>在同一程序集中或同一程序集内的派生类中 |

C++ 没有像 .NET DLL 那样的“程序集”概念，因此这些都不适用。但是，可以在 C++ 中模拟它们。

一种解决方案采用 `friend` 类的组合，并保留一些库的头文件来模拟 internal：

```c++
// Player.h：随库分发（例如在'public'目录中）
class Player
{
    // 通过特定的结构体授予“内部”访问权限
    friend struct PlayerInternal;
 
    // 内部数据成员现在是私有的
    int64_t Id;
 
public:
 
    // Public API
    int64_t GetId() const
    {
        return Id;
    }
};
 
// PlayerInternal.h：不随库分发（例如在'internal'目录中）
#include "Player.h"
struct PlayerInternal
{
    // friend结构体的函数提供对“内部”数据成员的访问
    static int64_t& GetId(Player& player)
    {
        return player.Id;
    }
};
 
// Library.cpp：库内部编写的代码
#include "Player.h"
#include "PlayerInternal.h"
Player player{};
PlayerInternal::GetId(player) = 123; // OK
DebugLog(player.GetId()); // OK
 
// User.cpp：由库用户编写的代码
#include "Player.h"
Player player{};
PlayerInternal::GetId(player) = 123; // 编译器错误：未定义的PlayerInternal
DebugLog(player.GetId()); // OK
```

可以创建此方法的变体来模拟 `protected internal` 和 `private protected`。

## Conclusion  结论

同样，这两种语言有很多重叠，但也有很多不同之处。
</br>它们都具有含义大致相同的 public、protected 和 private access 说明符。
</br>C# 还具有 internal、protected internal 和 private protected，C++ 需要模拟它们。
</br>C++ 具有继承访问说明符、friend 函数和 friend 类，而 C# 没有。

C++ const 的作用与 C# readonly 和 const 类似，但适用范围比两者都广泛得多。
</br>允许在函数参数和返回值以及任意变量、指针和引用上使用它是一项对 C++ 代码的设计方式有重大影响的功能。
</br>许多 C++ 代码库将尽可能多的变量、引用和函数设为 const，以强制编译器错误的不变性。mutable 关键字提供了灵活性，例如在实现缓存时。

语言之间的最后一个巨大区别是 C++ 中的结构和类是相同的。
</br>唯一的区别是使用的关键字和默认访问级别。
</br>在 C# 中，结构和类截然不同。
</br>例如，C# 结构不支持继承或默认构造函数，并且没有 readonly 类类型或非托管类。
</br>C++ 结构和类几乎提供了 C# 结构、类和接口的组合功能集，并用于构建其他语言功能，如委托。