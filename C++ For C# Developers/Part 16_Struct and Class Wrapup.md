# C++ For C# Developers: Part 16 – Struct and Class Wrapup

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5753),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天，我们将通过讨论一系列其他功能来总结结构和类 ：本地类、联合、重载赋值运算符和用户定义的文本。
C# 没有这些功能中的任何一个，但它可以模拟其中的一些功能。继续阅读以学习一系列新技巧！

## User-Defined Literals  用户定义的文本

C++支持创建我们自己的字面量(literals)，但有一些限制。这些字面量用于以类似于用户定义转换操作符的方式创建结构体或其他类型的实例。
</br>它们只是从字面量(literals)转换，而不是从现有对象转换。

以下是我们可以创建的 Literals 类型：

| Name                        | 	Example      |
|-----------------------------|---------------|
| Decimal literal  Decimal 文本 | 	123_suffix   |
| Octal literal  八进制字面量	      | 0123_suffix   |
| Hexadecimal literal  十六进制文本 | 	0x123_suffix |
| Binary literal  二进制文本	      | 0b123_suffix  |
| Real literal  （实数文本）        | 	0.123_suffix |
| Character literal  字符字面量    | 	'c'_suffix   |
| String literal  字符串文本       | 	"c"_suffix   |


后缀可以是任何有效的标识符。为了实现文本，我们编写一个运算符  `"" _suffix` 不属于结构体的函数：

```c++
Vector2 operator "" _v2(long double val)
{
    Vector2 vec;
    vec.X = val;
    vec.Y = val;
    return vec;
}
```

然后我们像这样调用它：

```c++
Vector2 v1 = 2.0_v2;
DebugLog(v1.X, v1.Y); // 2, 2
```

C++ 标准库保留所有不以 _ 开头的后缀供自己使用：
```c++
string greeting = "hello"s;
hours halfHour = 0.5h;
```

与其他形式的运算符重载（包括用户定义的转换运算符）一样，重要的是要认真考虑给定其简洁性的结果代码的可理解性。
由于显式声明类型，常规构造函数和成员函数可能更容易理解。

尽管如此，在某些情况下，简洁和表现力可能会派上用场。对于大量使用 `auto` 的代码库来说尤其如此：

```c++
// 用户定义的 literal 需要使用 auto 关键字进行 _less_ 输入
auto a = Vector2{2.0f};
auto b = 2.0f_v2;
 
// 用户定义的 literal 不使用 auto 关键字进行 _more_ 输入
Vector2 a{2.0f};
Vector2 b = 2.0f_v2;
```

> 读者的理解：这个就是类似于一种宏的形式，通过定义一个宏，执行一个方法

## Local Classes  本地类

本地类（或结构体）是在函数体中定义的类：

```c++
void Foo()
{
    struct Local
    {
        int32_t Val;
 
        Local(int32_t val)
            : Val(val)
        {
        }
    };
 
    Local ten{10};
    DebugLog(ten.Val); // 10
}
```

本地类在大多数方面都是常规类，但有一些限制。首先，它们的成员函数必须在类定义中定义：我们不能拆分声明和定义。

```c++
void Foo()
{
    struct Local
    {
        int32_t Val;
 
        Local(int32_t val);
    };
 
    // 编译器错误: 成员函数定义必须在类定义中
    Local::Local(int32_t val)
        : Val(val)
    {
    }
}
```

其次，它们不能有静态数据成员，但可以有静态成员函数。

```c++
void Foo()
{
    struct Local
    {
        int32_t Val;
 
        // 编译器错误: 本地类不能有静态数据成员
        static int32_t Max = 100;
 
        // 本地类可以有静态成员函数
        static int32_t GetMax()
        {
            return 100;
        }
    };
 
    DebugLog(Local::GetMax()); // 100
}
```

第三，也是最后一点，他们可以有 friends 函数，但不能声明内联 friend 函数 ：
```c++
class Classy
{
};
 
void Foo()
{
    struct Local
    {
        // 编译器错误: 本地类不能定义内联友元函数
        friend void InlineFriend()
        {
        }
 
        // 局部类可以有普通friend
        friend class Classy;
    };
}
```

与 C# 中的本地函数一样，C++ 中的本地类通常用于减少函数内部的代码重复，但放置在函数内部，因为它们对函数外部的代码没有用。
</br>当只需要 local classes 的一个实例时，看到没有名称的本地类甚至很常见。
</br>例如，此本地类删除了在玩家、敌人和 NPC 上运行的重复代码，而无需多态性：

```c++
// 三种无关的类型：没有共同的基类
struct Player
{
    int32_t Health;
};
struct Enemy
{
    int32_t Health;
};
struct Npc
{
    int32_t Health;
};
 
int32_t HealToFullIfNotDead(
    Player* players, int32_t numPlayers,
    Enemy* enemies, int32_t numEnemies,
    Npc* npcs, int32_t numNpcs)
{
    // 匿名局部类
    // 避免需要选择一个好的名称
    struct
    {
        // 不仅仅是一个封装在类中的函数
        // 还具有自己的状态来跟踪治疗
        int32_t NumHealed = 0;
 
        // 重载函数调用运算符
        // 避免需要选择一个好的名称
        int32_t operator()(int32_t health)
        {
            if (health <= 0 || health >= 100)
            {
                return health;
            }
             NumHealed++;
            return 100;
        }
    } healer;
 
    // 每个循环体都重复使用修复代码
    for (int32_t i = 0; i < numPlayers; ++i)
    {
        // 调用重载的函数调用操作符
        players[i].Health = healer(players[i].Health);
    }
    for (int32_t i = 0; i < numEnemies; ++i)
    {
        enemies[i].Health = healer(enemies[i].Health);
    }
    for (int32_t i = 0; i < numNpcs; ++i)
    {
        npcs[i].Health = healer(npcs[i].Health);
    }
 
    return healer.NumHealed;
}
 
// 一人死亡，两人受损，一人健康值满
const int32_t num = 4;
Player players[num]{{0}, {50}, {75}, {100}};
Enemy enemies[num]{{0}, {50}, {75}, {100}};
Npc npcs[num]{{0}, {50}, {75}, {100}};
int32_t numHealed = HealToFullIfNotDead(
    players, num,
    enemies, num,
    npcs, num);
 
DebugLog(numHealed); // 6
DebugLog(
    players[0].Health, players[1].Health,
    players[2].Health, players[3].Health); // 0, 100, 100, 100
DebugLog(
    enemies[0].Health, enemies[1].Health,
    enemies[2].Health, enemies[3].Health); // 0, 100, 100, 100
DebugLog(
    npcs[0].Health, npcs[1].Health,
    npcs[2].Health, npcs[3].Health); // 0, 100, 100, 100
```

## Copy and Move Assignment Operators 复制和移动赋值运算符

除了析构函数和一些构造函数，编译器还将为我们生成 copy 和 move 赋值运算符。

```c++
struct Vector2
{
    float X;
    float Y;
 
    // 编译器生成如下复制赋值运算符：
    // Vector2& operator=(const Vector2& other)
    // {
    //     X = other.X;
    //     Y = other.Y;
    //     return *this;
    // }
 
    // 编译器生成如下移动赋值运算符：
    // Vector2& operator=(const Vector2&& other)
    // {
    //     X = other.X;
    //     Y = other.Y;
    //     return *this;
    // }
};
 
void Foo()
{
    Vector2 a{2, 4};
    Vector2 b{0, 0};
    b = a; // 调用编译器生成的复制赋值运算符
    DebugLog(b.X, b.Y); // 2, 4
}
```

只要我们自己不定义赋值运算符，它就会这样做，每个非静态数据成员和基类都有一个赋值运算符，并且没有任何非静态数据成员是 const 或引用。

与构造函数和析构函数一样，我们可以使用 `= default` 和 `= delete` 来覆盖默认行为，并强制编译器生成一个或强制它不生成一个。

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2& operator=(const Vector2& other) = delete;
};
 
void Foo()
{
    Vector2 a{2, 4};
    Vector2 b{0, 0};
    b = a; // 编译器错误：复制赋值运算符已被删除
    DebugLog(b.X, b.Y); // 2, 4
}
```

## Unions 联合

我们已经看到如何使用class关键字代替struct来将默认访问级别从public更改为private。
</br>同样，C++提供了union关键字来改变struct的数据布局。
</br>而不是使struct足够大以容纳所有非静态数据成员，联合体（union）的大小正好足以容纳最大的非静态数据成员。

> 读者的理解为：
> union 这是一种类似于是一种特殊的数据类型，所有成员共享同一块内存（同一时间只用其中一个成员）

```c++
union FloatBytes
{
    float Val;
    uint8_t Bytes[4];
};
 
void Foo()
{
    FloatBytes fb;
    fb.Val = 3.14f;
    DebugLog(sizeof(fb)); // 4 (not 8)
 
    // 195, 245, 72, 64
    DebugLog(fb.Bytes[0], fb.Bytes[1], fb.Bytes[2], fb.Bytes[3]);
 
    fb.Bytes[0] = 0;
    fb.Bytes[1] = 0;
    fb.Bytes[2] = 0;
    fb.Bytes[3] = 0;
    DebugLog(fb.Val); // 0
}
```

因为联合体的非静态数据成员占用相同的内存空间，写入一个会影响到另一个。在上面的例子中，我们可以利用这一点来获取组成浮点数的字节，或者通过操作与之共享内存的字节数组来使用整数运算来操纵浮点数。

请注意，从技术上讲，读取除最近写入的数据成员之外的任何非静态数据成员都是未定义的行为。但是，几乎所有编译器都支持此功能，因为它是联合体的常见用法，因此很可能是安全的。

就像局部类一样，对联合体有一些限制。首先，联合体不能参与继承。这意味着它们不能有任何基类，也不能成为基类，或者拥有任何虚成员函数。

```c++
struct IGetHashCode
{
    virtual int32_t GetHashCode() = 0;
};
 
// 编译器错误：联合体不能继承
union Id : IGetHashCode
{
    int32_t Val;
    uint8_t Bytes[4];
 
    // 编译器错误：联合体不能有虚成员函数
    virtual int32_t GetHashCode() override
    {
        return Val;
    }
};
 
// 编译器错误：不能从联合体派生
struct Vec2Bytes : Id
{
};
```

其次，联合不能具有作为引用的非静态数据成员：

```c++
union IntRefs
{
    // 编译器错误：联合体不能有左值引用
    int32_t& Lvalue;
 
    // 编译器错误：联合体不能有右值引用
    int32_t&& Rvalue;
};
```

第三，如果联合体中的任何非静态数据成员具有“非平凡”(non-trivial)的复制构造函数、复制赋值运算符、移动赋值运算符或析构函数，则默认情况下，联合体的该函数版本会被删除，需要显式编写。

如果结构是显式编写的，或者任何非静态数据成员具有默认初始值设定项，或者有任何虚拟成员函数或基类，或者任何非静态数据成员或基类具有非平凡构造函数，则结构具有“非平凡”构造函数。

如果一个结构体的析构函数是显式编写的、虚拟的，或者任何非静态数据成员或基类具有非平凡的析构函数，则该结构体具有“非平凡的”析构函数。

这是很多规则，但联合包含具有这些非平凡函数的类型是相当罕见的。

通常，它们用于简单的 primitives、structs 和 arrays，如上面的例子。对于更高级的用法，我们需要记住规则：
> 读者认为这里作者说的有点乱，但总的意思是union的使用条件，下面是ai总结的
> </br>当联合（union）的成员包含 非平凡类型 时（例如 std::string、自定义类等），联合的以下函数会被隐式删除：
> - 默认构造函数
> - 拷贝/移动构造函数
> - 拷贝/移动赋值运算符
> - 析构函数
> 
> 必须 显式定义这些函数 才能使用联合。以下是触发 "非平凡性" 的条件：
> 
> 1. 非平凡构造函数（满足以下任一条件）：
>    - 成员有显式编写的构造函数
>    - 成员有默认初始值（如 int x = 42;）
>    - 成员或基类包含虚函数
>    - 成员或基类本身有非平凡构造函数
> 2. 非平凡析构函数（满足以下任一条件）：
>    - 显式编写了析构函数
>    - 析构函数是虚函数
>    - 成员或基类有非平凡析构函数

```c++
// 注意：'ctor'是'constructor'(构造函数)的常用缩写，同样地，'dtor'是'destructor'(析构函数)的常用缩写
struct NonTrivialCtor
{
    int32_t Val;
 
    NonTrivialCtor()
    {
        Val = 100;
    }
 
    // 非平凡的复制构造函数，因为它被明确编写
    NonTrivialCtor(const NonTrivialCtor& other)
    {
        Val = other.Val;
    }
};
 
// 与具有非平凡复制构造函数的非静态数据成员联合
// 默认情况下，union 的复制构造函数被删除
union HasNonTrivialCtor
{
    NonTrivialCtor Ntc;
};
 
// 与具有非平凡复制构造函数的非静态数据成员联合
// 默认情况下，union的复制构造函数被删除
union HasNonTrivialCtor2
{
    NonTrivialCtor Ntc;
 
    HasNonTrivialCtor2()
        : Ntc{}
    {
    }
 
    // 明确编写拷贝构造函数
    HasNonTrivialCtor2(const HasNonTrivialCtor2& other)
        : Ntc{other.Ntc}
    {
    }
};
 
HasNonTrivialCtor a{};
DebugLog(a.Ntc.Val);
 
// 编译器错误
// 联合体具有具有非平凡复制构造函数的非静态数据成员
// 其复制构造函数必须显式编写
HasNonTrivialCtor b{a};
DebugLog(b.Ntc.Val);
 
HasNonTrivialCtor2 c{};
 
// OK: 显式编写了拷贝构造函数
HasNonTrivialCtor2 d{c};
DebugLog(d.Ntc.Val); // 100
```

联合体也可以是“匿名”的。像结构体一样，它们可以没有名字。与结构体不同，它们也可以没有变量：

```c++
void Foo()
{
    union
    {
        int32_t Int;
        float Float;
    };
}
```

这些工会甚至比普通工会受到更多限制。它们不能有任何成员函数或静态数据成员，并且其所有数据成员都必须是公共的。
与无作用域的枚举一样，它们的成员被添加到 union 所在的任何范围内：上面例子中的 `Foo`。

```c++

void Foo()
{
    union
    {
        int32_t Int;
        float Float;
    };
 
    // 整数和浮点数被添加到Foo中，因此可以直接使用
    Float = 3.14f;
    DebugLog(Int); // 1078523331
}
```

这个特性通常被用来通过将联合体和枚举类型包装在结构体中来创建所谓的“标记联合体”

```c++
struct IntOrFloat
{
    // “tag”记录了活动成员
    enum { Int, Float } Type;
 
    // 匿名联合
    union
    {
        int32_t IntVal;
        float FloatVal;
    };
};
 
IntOrFloat iof;
 
iof.FloatVal = 3.14f; // Set value
iof.Type = IntOrFloat::Float; // Set type
 
// Read value and type
DebugLog(iof.IntVal, iof.Type); // 1078523331, Float
```

这种模式也被称为“变体”，通常在添加更多保护以确保类型和值相关联时使用：

```c++
struct TypeException
{
};
 
class IntOrFloat
{
public:
 
    enum struct Type { Int, Float };
 
    Type GetType() const
    {
        return Type;
    }
 
    void SetIntVal(int32_t val)
    {
        Type = Type::Int;
        IntVal = val;
    }
 
    int32_t GetIntVal() const
    {
        if (Type != Type::Int)
        {
            throw TypeException{};
        }
        return IntVal;
    }
 
    void SetFloatVal(float val)
    {
        Type = Type::Float;
        FloatVal = val;
    }
 
    float GetFloatVal() const
    {
        if (Type != Type::Float)
        {
            throw TypeException{};
        }
        return FloatVal;
    }
 
private:
 
    Type Type;
 
    union
    {
        int32_t IntVal;
        float FloatVal;
    };
};
 
IntOrFloat iof;
iof.SetFloatVal(3.14f); // Set value to 3.14f and type to Float
DebugLog(iof.GetFloatVal()); // 3.14
DebugLog(iof.GetIntVal()); // Throws exception: type is not Int
```

另一个常见的联合使用情况是提供一种不改变数据类型的替代访问机制。
</br>在向量、矩阵和四元数中，使用联合来提供对组件的命名字段访问或数组访问是非常常见的：
```c++
union Vector2
{
    struct
    {
        float X;
        float Y;
    };
 
    float Components[2];
};
 
 
Vector2 v;
 
// 命名字段访问
v.X = 2;
v.Y = 4;
 
// 数组访问：由于联合而具有相同的值
DebugLog(v.Components[0], v.Components[1]); // 2, 4
 
// 数组访问
v.Components[0] = 20;
v.Components[1] = 40;
 
// 命名字段访问：由于联合而具有相同的值
DebugLog(v.X, v.Y); // 20, 40
```

## Pointers to Members  指向成员的指针

最后，让我们看看如何创建指向结构成员的指针。要简单地获取指向特定结构体实例的非静态数据成员的指针，我们可以使用普通的指针语法

```c++
struct Vector2
{
    float X;
    float Y;
};
 
Vector2 v{2, 4};
float* p = &v.X; // p指向a的X数据成员
```
> 这之后的内容据说实际项目中很少用，所以看得懂看看，了解一下即可
 
然而，我们也可以获取到结构体实例中任何非静态数据成员的指针：

```c++
float Vector2::* p = &Vector2::X; //  p指向Vector2的X数据成员
```

要取消引用这样的指针，我们需要一个 struct 的实例，它指向其 data member：
```c++
float Vector2::* p = &Vector2::X;
Vector2 v{2, 4};
 
// 解除特定结构的指针引用
DebugLog(v.*p); // 2
```
这些指针不能相互转换，但只要基类不是虚的，就可以允许多态

```c++
struct Vector2
{
    float X;
    float Y;
};
 
struct Vector3 : Vector2
{
    float Z;
};
 
float Vector2::* p = &Vector2::X;
Vector2 v{2, 4};
 
float* p2 = p; // 编译器错误：不兼容
 
float f = 3.14f;
float Vector2::* pf = &f; // 编译器错误：不兼容
 
float Vector3::* p3 = p; // OK: Vector3 继承自 Vector2
DebugLog(v.*p3); // 2
```

当创建一个指向成员的指针，而这个成员本身也是一个指针时，语法会变得稍微复杂一些。幸运的是，这种情况很少见：

```c++
struct Float
{
    float Val;
};
 
struct PtrToFloat
{
    float Float::* Ptr;
};
 
// 指向由PtrToFloat中的Ptr指向的Float中的Val的指针
float Float::* PtrToFloat::* p1 = &PtrToFloat::Ptr;
 
Float f{3.14f};
PtrToFloat ptf{&Float::Val};
 
float Float::* pf = ptf.*p1; // 取消第一层间接引用
float floatVal = f.*pf; // 取消二级间接引用
DebugLog(floatVal); // 3.14
 
// 同时取消两次间接引用
DebugLog(f.*(ptf.*p1)); // 3.14
```

也可以获取指向成员函数的指针。语法类似于数据成员指针和普通函数指针的组合：

```c++
struct Player
{
    int32_t Health;
};
 
struct PlayerOps
{
    Player& Target;
 
    PlayerOps(Player& target)
        : Target(target)
    {
    }
 
    void Damage(int32_t amount)
    {
        Target.Health -= amount;
    }
 
    void Heal(int32_t amount)
    {
        Target.Health += amount;
    }
};
 
// 指向PlayerOps的非静态成员函数的指针
// 接受一个int32_t类型的参数并返回void·
void (PlayerOps::* op)(int32_t) = &PlayerOps::Damage;
 
Player player{100};
PlayerOps ops(player);
 
// 通过指针调用Damage函数
(ops.*op)(20);
DebugLog(player.Health); // 80
 
// 重新分配到另一个兼容的函数
op = &PlayerOps::Heal;
 
// 通过指针调用Heal函数
(ops.*op)(10);
DebugLog(player.Health); // 90
```

## Conclusion  结论

今天，我们看到了 C# 中不可用的大量其他类功能。用户定义的文本可以使代码同时更具表现力和更简洁。它最好谨慎用于非常稳定的核心类型，例如 Standard Library 的 string。

本地类具有许多与 C# 中的本地函数相同的好处，但更进一步，允许几乎完整的类功能，包括数据成员、构造函数、析构函数和重载运算符。

复制和移动赋值运算符允许我们使用熟悉的 x = y 语法轻松复制和移动类，而不是通常命名为 Clone 或 Copy 的实用函数。编译器甚至会为我们生成它们，从而节省大量样板和如果该样板与类的更改不同步时出错的可能性。

联合允许节省内存、对 float 等类型后面的位和字节进行高级作，以及替代访问样式的便利性。它们可以在 C# 中部分模拟 ，但 C++ 中的本机支持更方便，并且提供更高级的功能。

指向成员的指针允许我们将它们限制为专门指向类的成员，并且不将该访问权限绑定到类的任何特定实例。由于同时支持数据成员和成员函数，我们有一个工具，可以在运行时确定要使用的数据或要调用的函数，而无需像 C# 的委托这样的重量级语言功能。
</br>这可用于设置模式（例如， 伤害模式与修复模式）、GUI 回调（例如，单击处理程序）或各种其他情况。