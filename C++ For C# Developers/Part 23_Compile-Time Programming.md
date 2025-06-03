# C++ For C# Developers: Part 23 – Compile-Time Programming

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5875),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正


我们编写的绝大多数代码都是在运行时执行的。今天的文章是关于另一种类型的代码，这种代码在编译时运行。C# 对此的支持非常有限。
</br>在C++中，尤其是在其较新版本中，大多数语言特性都可以在编译时使用。继续阅读，了解如何利用这一点！

## Constant Variables  常量变量

C# 有类和结构体的 `const` 字段。它们必须通过一个常量表达式立即初始化，这种表达式是在编译时评估的。
</br>它们的类型必须是原始类型，如 `int` 和 `float` 或 `string`。`const` 是隐式`static`和`Readonly`。

同样，C++有`constexpr`变量。它们必须是“字面量类型”，包括像`int`和`float`这样的原始类型，还包括满足某些条件的引用、类、字面量类型的数组或void。
</br>它们是隐式const的，但不是静态的。

```c++
struct MathConstants
{
    // 常量成员变量
    // 隐式地 `const`
    // 不是隐式地 `static`。需要添加关键字。
    static constexpr float PI = 3.14f;
};
 
// 常量全局变量
constexpr int32_t numGamesPlayed = 0;
 
void Foo()
{
    // 常量局部变量
    constexpr int32_t expansionMultiplier = 3;
 
    // 常量引用
    // 需要添加 `const` 因为 `numGamesPlayed` 是隐式常量
    constexpr int32_t 隐式地使用 `const` 元素& ngp = numGamesPlayed;
 
    // 常量数组
    // 隐式地使用 `const` 元素
    constexpr int32_t exponentialBackoffDelays[4] = { 100, 200, 400, 800 };
}
```

到目前为止，我们只使用了原始数据类型、原始数据类型的引用以及原始数据类型的数组。 类也是允许的，但有一些限制。
</br>首先，它们必须是聚合类型、lambda表达式，或者至少有一个非复制、非移动的`constexpr`构造函数。
</br>我们将在下一节中介绍constexpr函数。

```c++
struct NonLiteralType
{
    // 删除复制和移动构造函数
    NonLiteralType(const NonLiteralType&) = delete;
    NonLiteralType(const NonLiteralType&&) = delete;
private:
    // 添加一个私有非静态数据成员，这样它就不是聚合了
    int32_t Val;
};
 
// 编译器错误：不是一个“字面量类型”
constexpr NonLiteralType nlt{};
```

其次，它必须有一个 `constexpr` 析构函数。

```c++
struct NonLiteralType
{
    //析构函数不是constexpr
    NonLiteralType()
    {
    }
};
 
// 编译器错误：NonLiteralType 没有constexpr析构函数
constexpr NonLiteralType nlt{};
 
struct LiteralTypeA
{
    // 显式由编译器生成的析构函数是constexpr
    LiteralTypeA() = default;
};
 
struct LiteralTypeB
{
    // 隐式编译器生成的析构函数是constexpr
};
 
// OK
constexpr LiteralTypeA lta{};
constexpr LiteralTypeB ltb{};
```

第三，如果是一个联合体，那么至少必须有一个非静态数据成员是“字面量类型”。
</br>如果不是联合体，那么所有非静态数据成员以及其基类的所有数据成员都必须是“字面量类型。”

```c++
union NonLiteralUnion
{
    NonLiteralType nlt1;
    NonLiteralType nlt2;
};
 
// 编译器错误：联合中所有非静态数据成员都不是字面量
constexpr NonLiteralUnion nlu{};
 
struct NonLiteralStruct
{
    NonLiteralType nlt1;
    int32_t Val; // 原始类型是字面类型
};
 
// 编译器错误：结构体中并非所有非静态数据成员都是字面量
constexpr NonLiteralStruct nls{};
```

如果我们满足这些要求，我们就可以自由地创建常量类变量。这里有一个简单的聚合体：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
// 常量类实例
constexpr Vector2 ORIGIN{0, 0};
```

关于`constexpr`变量，还有一点需要注意：它们与`constinit`变量不兼容。这个C++20关键字用于要求变量必须以常量初始化：

```c++
// 需要将 `ok` 常量初始化
constinit const char* ok = "OK";
 
// 编译器错误：不能同时是constexpr和constinit
constexpr constinit const char* ok2 = "OK";
 
// 编译器错误：未使用常量初始化
constinit const char* err = rand() == 0 ? "f" : "t";
```

## Constant Functions  常量函数

C++中的函数也可以是`constexpr`。这意味着它们可以在编译时执行：

```c++
constexpr int32_t SumOfFirstN(int32_t n)
{
    int32_t sum = 0;
    for (int32_t i = 1; i <= n; ++i)
    {
        sum += i;
    }
    return sum;
}
 
// 1 + 2 + 3
DebugLog(SumOfFirstN(3)); // 6
```

因为`3`是一个编译时常量，所以对`SumOfFirstN(3)`的调用将在编译期间执行，并由返回值替换，成为：

```c++
DebugLog(6); // 6
```

如果`SumOfFirstN`的参数不是编译时常数，`SumOfFirstN`将像正常一样编译和调用：

```c++
// 从文件中读取 `n`
FILE* handle = fopen("/path/to/file", "r");
int32_t n{0};
fread(&n, sizeof(n), 1, handle);
 
// 编译时未知的参数
// 依赖于文件系统的状态
// SumOfFirstN 的调用发生在运行时
DebugLog(SumOfFirstN(n)); // 6
 
fclose(handle);
```

如果我们不想在运行时调用函数，例如为了避免意外增加可执行文件大小或在运行时执行编译时计算，C++20 允许我们用 `consteval` 替换 `constexpr`。
</br>`constexpr` 和 `consteval` 之间的唯一区别是，如果我们尝试在运行时调用 `consteval` 函数，将会得到编译器错误。

```c++
consteval int32_t SumOfFirstN(int32_t n)
{
    int32_t sum = 0;
    for (int32_t i = 1; i <= n; ++i)
    {
        sum += i;
    }
    return sum;
}
```

为了使函数成为 constexpr，它们必须满足某些条件。自从 C++11 中引入以来，这些条件在 C++14 和 C++20 中得到了极大的放宽。
</br>语言未来的版本可能会继续这一趋势。现在，我们只看看 C++20 的规则。

首先，它们的参数和返回类型必须是“字面量类型”

```c++
// 由于非constexpr析构函数，不是字面量类型
struct NonLiteral
{
    ~NonLiteral()
    {
    }
};
 
// 编译器错误：constexpr函数的参数必须是字面量类型
constexpr void Foo(NonLiteral nl)
{
}
 
// 编译器错误：constexpr函数的返回值必须是字面量类型
constexpr NonLiteral Goo()
{
    return {};
}
```

其次，具有 虚基类 或 非constexpr基类构造函数 或 析构函数的类 中不能有`constexpr`构造函数或析构函数。

```c++
struct NonLiteralBase
{
};
 
struct NonLiteralCtor : virtual NonLiteralBase
{
    // 编译器错误：构造函数不能与虚基类一起使用constexpr
    constexpr NonLiteralCtor()
    {
    }
};
 
struct NonLiteralDtor : virtual NonLiteralBase
{
    // 编译器错误：析构函数不能与虚基类一起使用constexpr
    constexpr ~NonLiteralDtor()
    {
    }
};
```

第三，函数体中不能有任何`goto`语句或标签，除了`switch`语句中的`case`和`default`之外：

```c++
constexpr int32_t GotoLoop(int32_t n)
{
    int32_t sum = 0;
    int32_t i = 1;
 
    // 编译器错误：constexpr函数不能有非case、非default标签
beginLoop:
 
    if (i > n)
    {
        // 编译器错误：constexpr函数不能有goto
        goto endLoop;
    }
    sum += i;
 
    // 编译器错误：constexpr函数不能有非case、非default标签
endLoop:
    return sum;
}
```

第四，所有局部变量必须是“字面量类型”：

```c++
constexpr void Foo()
{
    // 编译器错误：constexpr函数不能有非字面量变量
    NonLiteral nl{};
}
```

第五，函数不能有任何静态变量：

```c++
constexpr int32_t GetNextInt()
{
    // 编译器错误：constexpr函数不能有静态变量
    static int32_t next = 0;
 
    return ++next;
}
```

遵循所有这些规则的任何函数都可以是`constexpr`或`consteval`。


## Constant If  常量 If

从 C++17 开始，引入了 `if` 的新形式：`if constexpr （...）`。这些函数在编译时进行评估，无论它们是否在 `constexpr` 函数中。 然后，编译器会删除 `if`，以支持 `if` 块或 `else` 块中的代码。
</br>这对于从可执行文件中移除代码以减小其大小并消除运行时分支指令的成本非常有帮助。

例如，如果我们想在编译时设置记录的最小严重级别，而不必每次写入日志消息时都进行检查，
我们可以使用编译器定义的预处理器符号（`LOG_LEVEL`）、一个`constexpr`字符串相等函数（`IsStrEqual`）以及`if constexpr`来决定是否记录或不记录：

```c++
// 编译器根据其配置设置预处理器符号
// 这就像我们将其写入C++代码中：
#define LOG_LEVEL "WARN"
 
// 比较NUL终止的字符串以实现精确相等的常量函数
constexpr bool IsStrEqual(const char* str, const char* match)
{
    for (int i = 0; ; ++i)
    {
        char s = str[i];
        char m = match[i];
        if (s)
        {
            if (m)
            {
                if (s != m)
                {
                    return false;
                }
            }
            else
            {
                return false;
            }
        }
        else
        {
            return !m;
        }
    }
    return true;
}
 
// 如果LOG_LEVEL是DEBUG，则记录调试信息
void LogDebug(const char* msg)
{
    // 在编译时将 LOG_LEVEL 与 "DEBUG" 进行比较
    // 如果为假，则从可执行文件中移除 WriteLog 调用
    if constexpr (IsStrEqual(LOG_LEVEL, "DEBUG"))
    {
        WriteLog("DEBUG", msg);
    }
}
 
// 如果LOG_LEVEL是DEBUG或WARN，则记录警告消息
void LogWarn(const char* msg)
{
    // 在编译时将 LOG_LEVEL 与 "DEBUG" 和 "WARN" 进行比较
    // 如果为假，则从可执行文件中移除 WriteLog 调用
    if constexpr (IsStrEqual(LOG_LEVEL, "DEBUG") ||
        IsStrEqual(LOG_LEVEL, "WARN"))
    {
        WriteLog("WARN", msg);
    }
}
 
// 如果LOG_LEVEL是DEBUG、WARN或ERROR，则记录警告信息
void LogError(const char* msg)
{
    // 在编译时将 LOG_LEVEL 与 "DEBUG", "WARN", 和 "ERROR" 进行比较
    // 如果为 false，则从可执行文件中移除 WriteLog 调用
    if constexpr (IsStrEqual(LOG_LEVEL, "DEBUG") ||
        IsStrEqual(LOG_LEVEL, "WARN") ||
        IsStrEqual(LOG_LEVEL, "ERROR"))
    {
        WriteLog("ERROR", msg);
    }
}
```

## Static Assertions  静态断言

与`if constexpr`类似，我们可以编写在编译时执行而不是在运行时执行的断言。
</br>例如，我们可以使用`static_assert`来确保`LOG_LEVEL`被设置为有效的值。如果不是，代码将无法编译：

```c++
static_assert(IsStrEqual(LOG_LEVEL, "DEBUG") ||
    IsStrEqual(LOG_LEVEL, "WARN") ||
    IsStrEqual(LOG_LEVEL, "ERROR"),
    "Invalid log level: " LOG_LEVEL);
```

我们还可以检查代码是否正在一个受支持的构建环境中编译：

```c++
static_assert(MSC_VER >= 1900, "Only Visual Studio 2015+ is supported");
static_assert(_WIN64, "Only 64-bit Windows is supported");
```

或者，对于断言来说，这是更传统的做法，我们可以确保我们的代码没有出错。
例如，定义一个在读取或写入文件或网络套接字时进行序列化和反序列化的结构体是很常见的。
我们可以使用`static_assert`来确保它的大小是正确的，并且我们没有通过添加数据成员或填充意外地增加了它的大小：

```c++
struct PlayerUpdatePacket
{
    int32_t PlayerId;
    float PositionX;
    float PositionY;
    float PositionZ;
    float VelocityX;
    float VelocityY;
    float VelocityZ;
    int32_t NumLives;
};
static_assert(
    sizeof(PlayerUpdatePacket) == 32,
    "PlayerUpdatePacket serializes to wrong size");
```

自C++17以来，如果我们认为错误信息没有帮助，我们可以省略它：

```c++
static_assert(sizeof(PlayerUpdatePacket) == 32);
```

## Constant Expressions  常量表达式

现在我们有了大量的编译时特性，包括 `constexpr 变量`、`constexpr 函数`、`if constexpr` 和 `static_assert`，我们需要知道何时可以使用这些特性，何时不能。
</br>显然，像 32 这样的字面量是常量表达式，而调用 rand() 来获取随机数则不是，但许多情况并不那么明显。

C++将一个表达式视为常量，只要它不符合一系列标准。自C++11以来，语言的每个版本都放宽了这些限制。语言正普遍地向编译时完全可执行的方向发展。
</br>在此之前，我们需要知道在编译时表达式中不允许什么，以便我们知道不能与所有这些编译时功能一起使用什么。让我们看看C++20的规则。

首先，不能在 `constexpr 成员函数`和 `constexpr 构造函数`之外隐式或显式地使用 this 指针：

```c++
struct Test
{
    int32_t Val{123};
 
    int32_t GetVal()
    {
        // 显式使用this指针
        // 编译器错误：`this`不能在constexpr成员函数外部使用
        constexpr int32_t val = this->Val;
 
        return val;
    }
};
```

同样的情况也适用于引用捕获对象的`lambda`表达式，因为它们实际上是通过`lambda`类的`this`指针来访问的：

```c++
const int32_t outside = 123;
auto lambda = [outside]
{
    // 编译器错误：不能在lambda外部引用变量
    constexpr int32_t const* pOutside = &outside;
 
    DebugLog(*pOutside);
};
lambda();
```

其次，不允许调用`非constexpr`函数和构造函数：

```c++
struct Test
{
    int32_t Val{123};
 
    // 用户定义的转换为int32_t的操作符
    operator int32_t()
    {
        return Val;
    }
};
 
// 尝试创建一个 Test 并将其转换为 int32_t
// 编译器错误：调用非 constexpr 构造函数的 Test
constexpr int32_t val = Test{};
```

第三，我们不能调用仅声明但尚未定义的`constexpr函数`：

```c++
// 声明 constexpr 函数
constexpr int32_t Pow(int32_t x, int32_t y);
 
// 编译错误：Pow尚未定义
constexpr int32_t eight = Pow(2, 3);
 
// 定义 constexpr 函数
constexpr int32_t Pow(int32_t x, int32_t y)
{
    int32_t result = x;
    for (; y > 1; --y)
    {
        result *= x;
    }
    return result;
}
```

第四，对在表达式之前开始其生命周期的非“字面类型”类的constexpr虚函数的调用：

```c++
struct Base
{
    constexpr virtual int32_t GetVal()
    {
        return 1;
    }
};
 
struct Derived : Base
{
    constexpr virtual int32_t GetVal() override
    {
        return 123;
    }
};
 
// 类的生命周期在常量表达式之前开始
Derived d{};
 
// 编译器错误：不能在常量表达式中调用在表达式开始之前就开始其生命周期的类的constexpr虚函数
constexpr int32_t val = d.GetVal();
 
// OK：派生对象在常量表达式中开始生命周期
constexpr int32_t val2 = Derived{}.GetVal();
```

第五，触发任何形式的未定义行为。此规则为`constexpr`代码添加了一个安全网。未定义行为根本无法编译。

```c++
// 编译器错误：除以零是未定义的行为
constexpr float f = 3.14f / 0.0f;
 
// 编译器错误：有符号整数溢出是未定义的行为
constexpr int32_t i = 0x7fffffff + 1;
```

第六，如果左值不允许在常量表达式中使用，则不能将左值用作右值。例如，非const左值是不允许的，但const左值是允许的：

```c++
// i1不能在常量表达式中使用，因为它不是const
int32_t i1 = 123;
 
// i2 可以用在常量表达式中，因为它被声明为 const
const int32_t i2 = 123;
 
// i3 不能在常量表达式中使用，因为它没有用常量表达式初始化
const int32_t i3 = i1;
 
// 编译器错误：i1不能使用
constexpr int32_t i4 = i1;
 
// 编译错误：i3无法使用
constexpr int32_t i5 = i3;
 
// Ok：可以使用i2
constexpr int32_t i6 = i2;
```

引用不是允许的解决方案，因为它们只是对象的同义词：

```c++
struct HasVal
{
    int32_t Val{123};
};
 
HasVal hv{};
constexpr HasVal const& rhv{hv};
constexpr int32_t ri{rhv.Val};
 
constexpr HasVal hv2{};
constexpr HasVal const& rhv2{hv2};
constexpr int32_t ri2{rhv2.Val};
```

第七，只能使用联合体中的活动成员。访问非活动成员是一种未定义的行为，在运行时代码中通常被编译器允许，但在编译时代码中是不允许的。

```c++
union IntFloat
{
    int32_t Int;
    float Float;
 
    constexpr IntFloat()
    {
        // 将 Int 设置为活动成员
        Int = 123;
    }
};
 
 
// 调用使 Int 激活的 constexpr 构造函数
constexpr IntFloat u{};
 
// OK：可以使用 Int，因为它是最活跃的成员
constexpr int32_t i = u.Int;
 
// 编译器错误：不能使用Float，因为它不是活动成员
constexpr float f = u.Float;
```

也不允许使用其活动成员`mutable`的联合的编译器生成的 `copy` 或 `move` 构造函数或赋值运算符：

```c++
union IntFloat
{
    mutable int32_t Int;
    float Float;
 
    constexpr IntFloat()
    {
        Int = 123;
    }
};
 
constexpr IntFloat u{};
constexpr IntFloat u2{u};
```

第八，从void*转换为任何其他指针类型：

```c++
constexpr int32_t i = 123;
constexpr void const* pv = &i;
 
// 编译器错误：常量表达式不能转换void*
constexpr int32_t const* pi = static_cast<int32_t const*>(pv);
```

第九，任何使用 `reinterpret_cast`：

```c++
constexpr int32_t i = 123;
 
// 编译器错误：常量表达式不能使用 reinterpret_cast 即使是转换到同一类型
constexpr int32_t i2 = reinterpret_cast<int32_t>(i);
```

第十，修改在常量表达式中,开始其生命周期的非字面量类型的对象：

```c++
constexpr int Foo(int val)
{
    // val在这里开始其生命周期，在常量表达式开始之前
 
    // 编译器错误：不能修改在之前开始生命周期的对象
    constexpr int ret = ++val;
 
    return ret;
}
```

第十一个，`new` 和 `delete`，除非分配的内存被 `constant` 表达式删除：

```c++
// 编译器错误：在常量表达式中未删除由 `new` 分配的内存
constexpr int32_t* pi = new int;
 
constexpr int32_t GetInt()
{
    int32_t* p = new int32_t;
    *p = 123;
    int32_t ret = *p;
    delete p;
    return ret;
}
 
// OK：GetInt()的调用删除了它使用new运算符分配的内存
constexpr int32_t i = GetInt();
```

第十二，比较指针在技术上属于“未指定行为”，不允许在常量表达式中使用：

```c++
int32_t i = 123;
int32_t* pi1 = &i;
int32_t* pi2 = &i;
 
// 编译器错误：不能在常量表达式中比较指针
constexpr bool b = pi1 < pi2;
```

第十三，虽然我们可以捕获异常，但我们不能抛出它们：

```c++
constexpr void Explode()
{
    // 编译器错误：不能在常量表达式中抛出异常
    throw "boom";
}
```

`dynamic_cast` 和 `typeid` 同样不允许抛出异常：

```c++
struct Combatant
{
    virtual int32_t GetMaxHealth() = 0;
};
 
struct Enemy : Combatant
{
    virtual int32_t GetMaxHealth() override
    {
        return 100;
    }
};
 
struct TutorialEnemy : Combatant
{
    virtual int32_t GetMaxHealth() override
    {
        return 10;
    }
};
 
Enemy e;
Combatant& c{e};
 
// 编译器错误：如果dynamic_cast会抛出异常，则不能在常量表达式中调用它
constexpr TutorialEnemy& te{dynamic_cast<TutorialEnemy&>(c)};
 
Enemy* p = nullptr;
 
// 编译器错误：如果typeid调用将抛出异常，则不能在常量表达式中调用typeid
constexpr auto name = typeid(p).name();
```

第十四，最后，使用`va_start`、`va_arg`和`va_end`的变长函数，这些函数通常也不推荐使用：

```c++
constexpr void DebugLogAll(int count, ...) 
{
    va_list args;
 
    // 编译器错误：不能在常量表达式中使用va_start
    va_start(args, count);
 
    for (int i = 0; i < count; ++i)
    {
        const char* msg = va_arg(args, const char*);
        DebugLog(msg);
    }
 
    va_end(args);
}
```

## Conclusion  结论

C++提供了一组强大的编译时编程选项，包括`constexpr变量`、`constexpr函数`、`if constexpr`和`static_assert`。
</br>每种语言的新版本都使得越来越多的常规运行时特性在编译时可用。
</br>尽管仍然存在许多限制，但其中许多是为了一些晦涩的特性，如可变参数函数，或者为了明确防止诸如禁止未定义行为之类的错误。

语言的一般方向是最终在编译时完全可用，因此无需使用另一种语言或编写另一个程序来生成源代码。结果是统一的构建过程和编译时与运行时代码的去重。
</br>通过简单地添加`constexpr`关键字，我们通常可以将运行时计算移动到编译时，并通过使用这些预先计算的结果来提高运行时性能。

相比之下，C#对编译时编程的支持仅限于在const和默认函数参数中使用内置运算符对基本类型和字符串进行操作。
</br>这些功能自第一版（2002年）以来就已经可用，并且自那时以来没有添加或宣布更多功能。
</br>编译时编程几乎总是通过外部工具和构建步骤来完成，这些步骤生成.cs或.dll文件。这似乎代表了两种语言在哲学上的差异。

随着本系列的深入，我们将介绍更多C++的编译时编程选项：模板“元编程”和预处理器宏。