# C++ For C# Developers: Part 22 – Lambdas

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5850),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C++和C#都有lambda表达式，但它们之间有很多不同之处。
</br>今天我们将深入探讨C++ lambda表达式的工作原理，包括它们的所有特性和与C# lambda表达式的比较与对比。
</br>继续阅读以了解所有细节！

## Basic Syntax  基本语法

在语法上，lambda 表达式在 C++ 和 C# 中的表现不同。
首先，C++ 中没有与 C# 的“表达式 lambda”等价的东西：`(arg1, arg2, ...) => expr`
C++ 只有与 C# 的“语句 lambda”等价的东西：`(arg1, arg2, ...) => { stmnt1; stmnt2; ... }`。它们最简单的形式是这样的：

```c++
[]{ DebugLog("hi"); }
```

第一部分(`[]`)是捕获列表，我们将在稍后深入探讨。第二部分(`{ ... }`)是在调用lambda时执行语句列表。

现在让我们添加一个参数列表：

```c++
[](int x, int y){ return x + y; }
```

除了捕获列表([])和省略了参数列表后面的 => 符号之外，这现在看起来就像一个C#的lambda表达式。
</br>在省略参数列表的第一种形式中，lambda 根本不接受任何参数。

请注意，与迄今为止我们所见到的所有命名函数不同，这里没有声明返回类型。
</br>返回类型是由编译器通过查看我们的返回语句的类型隐式推断出来的。
</br>这与我们在声明具有自动返回类型的函数时所见到的，或者在C#中得到的类似。

如果我们更愿意显式地声明返回类型，我们可以使用“尾随(trailing)”返回类型语法：

```c++
[](int x, int y) -> int { return x + y; }
```

就像普通函数一样，我们也可以使用自动类型化的参数：

```c++
[](auto x, auto y) -> auto { return x + y; }
[](auto x, auto y) { return x + y; } // 尾随返回类型是可选的
[](auto x, int y) { return x + y; } // 并非每个论点都必须是atuo
```

## Lambda Types  Lambda 类型

那么lambda表达式有什么类型？在C#中，我们得到一种可以转换为Action或Func<int, int, int>等委托类型的类型。
</br>在C++中，编译器生成一个无名的类。它看起来像这样：

```c++
// 为这个lambda生成的编译器类：
//   [](int x, int y) { return x + y; }
// 实际上并没有命名为LambdaClass
class LambdaClass
{
    // Lambda函数体
    // 实际上并未命名为LambdaFunction
    static int LambdaFunction(int x, int y)
    {
        return x + y;
    }
 
public:
 
    // 默认构造函数
    // 仅在没有捕获的情况下
    LambdaClass() = default;
 
    // 复制构造函数
    LambdaClass(const LambdaClass&) = default;
 
    // 移动构造函数
    LambdaClass(LambdaClass&&) = default;
 
    // 析构函数
    ~LambdaClass() = default;
 
    // 函数调用运算符
    int operator()(int x, int y) const
    {
        return LambdaFunction(x, y);
    }
 
    // 用户定义的转换函数到函数指针
    // 只有如果没有捕获
    operator decltype(&LambdaFunction)() const noexcept
    {
        return LambdaFunction;
    }
};
```

因为它只是一个普通的类，所以我们可以像使用普通类一样使用它。唯一的区别是我们不知道它的名字，所以我们必须使用auto来指定它的类型：

```c++
void Foo()
{
    // 实例化lambda类。相当于：
    //   LambdaClass lc;
    auto lc = [](int x, int y){ return x + y; };
 
    // 调用重载的函数调用操作符
    DebugLog(lc(200, 300)); // 500
 
    // 调用用户定义的转换运算符以获取函数指针
    int (*p)(int, int) = lc;
    DebugLog(p(20, 30)); // 50
 
    // 调用拷贝构造函数
    auto lc2{lc};
    DebugLog(lc2(2, 3)); // 5
 
    // 在这里调用了lc和lc2的析构函数
}
```

## Default Captures  默认捕获

到目前为止，我们的lambda函数始终有一个空的捕获列表：[]。
</br>在C#中，捕获总是隐式的。在C++中，我们对捕获什么以及如何捕获有更多的控制。

首先，让我们看看最像C#的捕获方式：`[&]`。这是一种“默认捕获”方式，告诉编译器“捕获lambda使用的一切引用。”
</br>下面是它的样子：

```c++
// lambda之外
int x = 123;
 
// 默认捕获模式设置为“按引用”
auto addX = [&](int val)
{
    // Lambda引用了lambda外部的“x”
    // 编译器通过引用捕获“x”：int&
    return x + val;
};
 
DebugLog(addX(1)); // 124
```

我们可以看，在捕获它之后修改x，

```c++
int x = 123;
 
// 捕获对x的引用，而不是x的副本
auto addX = [&](int val) { return x + val; };
 
// 捕获后修改x
x = 0;
 
// 调用lambda函数
// lambda函数使用对x的引用，x的值为0
DebugLog(addX(1)); // 1
```

如果我们不喜欢这种行为，我们可以将“捕获默认”切换为 `[=]`，这意味着“捕获lambda使用的所有内容作为副本。”下面是这样做的外观：

```c++
int x = 123;
 
// 捕获x的一个副本，而不是x的一个引用
auto addX = [=](int val) { return x + val; };
 
// 在捕获后修改x
// 不会修改lambda的副本
x = 0;
 
// 调用lambda函数
// lambda函数使用了x的副本，其值为123
DebugLog(addX(1)); // 124
```

虽然从C++20开始已经弃用，但重要的是要注意，[=]可以隐式捕获当前对象的引用：*this。这里有一种情况会发生：


```c++
struct CaptureThis
{
    int Val = 123;
 
    auto GetLambda()
    {
        // 默认捕获模式是“复制”
        // Lambda使用“this”，它位于lambda外部
        // “this”被复制到CaptureThis*
        return [=]{ DebugLog(this->Val); };
    }
};
 
auto GetCaptureThisLambda()
{
    // 在栈上实例化类
    CaptureThis ct{};
 
    // 获取一个捕获了指向“ct”的指针的lambda
    auto lambda = ct.GetLambda();
 
    // 返回lambda。调用“ct”的析构函数。
    return lambda;
}
 
void Foo()
{
    // 获取一个捕获了指向“ct”的指针的lambda，该指针已经调用了析构函数并被从栈中弹出
    auto lambda = GetCaptureThisLambda();
 
    // 取消对捕获的指向“ct”的指针的引用
    lambda(); // 未定义行为：可能做任何事情！
}
```

这个例子意外地创建了一个指向“this”的“悬垂”指针，但同样的事情也可能发生在任何其他指针或引用上。
</br>确保捕获的指针和引用在lambda结束之前不会结束其生命周期是很重要的！

## Individual Captures  单个捕获

接下来我们可以添加到捕获列表中的元素被称为“单个捕获”，因为它从lambda外部捕获某些具体内容。

存在几种单独捕获的形式。首先，我们可以简单地给出一个名称：

```c++
int x = 123;
 
// 通过复制单独捕获“x”
auto addX = [x](int val)
{
    // 使用“x”的副本
    return x + val;
};
 
// 捕获后修改“x”
x = 0;
 
DebugLog(addX(1)); // 124
```

如果我们想初始化捕获的副本，我们可以添加任何通常的初始化形式：

```c++
int x = 123;
 
// 单独将“x”复制到名为“a”的变量中
auto addX = [a = x](int val)
{
    // 使用通过"a"变量获得的"x"副本
    return a + val;
};
 
// Modify "x" after the capture
x = 0;
 
DebugLog(addX(1)); // 124
```

捕获的变量甚至可以与它捕获的变量同名，就像我们只使用`[x]`时一样：

```c++
[x = x](int val){ return x + val; };
```

其他初始化形式也是可用的。

```c++
[a{x}](int val){ return a + val; };
[a(x)](int val){ return a + val; };
```

同样，我们可以单独捕获引用

```c++
int x = 123;
 
// 单独通过引用捕获“x”
auto addX = [&x](int val)
{
    // 使用对“x”的引用
    return x + val;
};
 
// 捕获后修改“x”
x = 0;
 
DebugLog(addX(1)); // 1
```

我们也可以初始化单独捕获的引用：

```c++
int x = 123;
 
// 单独通过引用捕获“x”，并将其命名为“a”
auto addX = [&a = x](int val)
{
    // 通过“a”使用对“x”的引用
    return a + val;
};
 
// 捕获后修改“x”
x = 0;
 
DebugLog(addX(1)); // 1
```

无论我们是捕获引用还是复制，我们可以使用任意表达式来初始化，而不是仅仅使用变量的名称：

```c++
auto lambda = [a = 2+2]{ DebugLog(a); };
lambda(); // 4
```

我们也有两种方式来单独捕获这个。第一种就是使用`[this]`，它通过引用来捕获：

```c++
struct CaptureThis
{
    int Val = 123;
 
    int Foo()
    {
        // 通过引用捕获“this”
        auto lambda = [this]
        {
            // 使用捕获的“this”引用
            return this->Val;
        };
 
        // 捕获后修改“Val”
        this->Val = 0;
 
        // 调用lambda函数
        // 使用对具有修改过的Val的“this”的引用
        return lambda();
    }
};
 
CaptureThis ct{};
DebugLog(ct.Foo()); // 0
```

捕获“this”的第二种方式是使用`[*this]`，它会复制类对象：

```c++
struct CaptureThis
{
    int Val = 123;
 
    int Foo()
    {
        // 通过复制来捕获“this”
        auto lambda = [*this]
        {
            // 使用捕获的“this”副本
            return this->Val;
        };
 
        // 捕获后修改“Val”
        this->Val = 0;
 
        // 调用lambda函数
        return lambda();
    }
};
 
CaptureThis ct{};
DebugLog(ct.Foo()); // 123
```

## Captured Data Members  捕获的数据成员

那么当lambda“捕获”某物时是什么意思呢？
大多数情况下，这仅仅意味着数据成员被添加到lambda的类中，并通过其构造函数进行初始化。比如说我们有一个这样的lambda：

```c++
[&m{multiply}, a{add}](float val){ return m*val + a; }
```

我们可以像这样使用 lambda：

```c++
float multiplyAndAddLoopLambda(float multiply, float add, int n)
{
    // 将“multiply”通过引用方式捕获为“m”
    // 将“add”通过复制方式捕获为“a”
    auto madd = [&m{multiply}, a{add}](float val){ return m*val + a; };
 
    float cur = 0;
    for (int i = 0; i < n; ++i)
    {
        cur = madd(cur);
    }
    return cur;
}
 
DebugLog(multiplyAndAddLoopLambda(2.0f, 1.0f, 5)); // 31
```

原因是编译器为这个lambda生成了一个类似这样的类：

```c++
// 为这个lambda生成的编译器类：
//   [&m{multiply}, a{add}](float val){ return m*val + a; }
//实际上并没有命名为LambdaClass
class LambdaClass
{
    // lambda的“捕获”
      // 顺序未指定
      // 实际上并未命名为“m”和“a”
    float& m;
    const float a;
 
public:
 
    // 构造函数
    // 初始化捕获
    LambdaClass(float& multiply, float add)
        : m{multiply}, a{add}
    {
    }
 
    // 复制构造函数
    LambdaClass(const LambdaClass&) = default;
 
    // 移动构造函数
    LambdaClass(LambdaClass&&) = default;
 
    // 析构函数
    ~LambdaClass() = default;
 
    // 函数调用运算符
    float operator()(float val) const
    {
        // Lambda 本体
        return m*val + a;
    }
};
```

请注意，默认构造函数已被替换为初始化捕获的构造函数，无论是通过引用还是复制。
如果没有捕获初始化器(`[x]或[&x]`)，则捕获将被直接初始化。
否则，它们将根据捕获初始化器(`[x{y}]`或`[x = y]`)进行复制初始化或直接初始化。数组元素将按顺序直接初始化。

在这个编译器生成的lambda类中，另一个变化是移除了用户定义的转换为函数指针的转换操作符。
这是因为普通的函数指针无法访问所需的`this`指针，以获取它完成工作所需捕获的内容。这就像我们试图编写这样：

```c++
float LambdaFunction(float val)
{
    // 编译错误：没有“m”
    // 编译错误：没有“a”
    return m*val + a;
}
```

由于我们可能需要控制放置在lambda类数据成员上的修饰符，
我们可以在lambda中添加如`mutable`和`noexcept`等关键字，它们也会被添加到数据成员上：

```c++
int x = 1;
 
// 编译器错误
// LambdaClass::operator() 是 const 的，而 LambdaClass::x 不是可变的
auto lambda1 = [x](){ x = 2; };
 
// OK: LambdaClass::x是可变的
auto lambda2 = [x]() mutable { x = 2; };
```

当我们使用上面的lambda时，编译器生成了类似于以下代码的lambda类：

```c++
float multiplyAndAddLoopClass(float multiply, float add, int n)
{
    // 将“multiply”和“add”变量作为“madd”的数据成员捕获
    LambdaClass madd{multiply, add};
 
    float cur = 0;
    for (int i = 0; i < n; ++i)
    {
        cur = madd(cur);
    }
    return cur;
}
 
DebugLog(multiplyAndAddLoopClass(2.0f, 1.0f, 5)); // 31
```
## Capture Rules  捕获规则

关于如何使用捕获，有一些语言规则。首先，如果默认的捕获模式是按引用，则单个捕获也不能是按引用：

```c++
int x = 123;
 
// 编译器错误：当默认捕获模式为引用时，无法单独通过引用捕获
auto lambda = [&, &x]{ DebugLog(x); };
```

其次，如果默认捕获模式是复制，那么所有单独的捕获都必须是通过引用、this或*this：

```c++
//编译器错误：当默认捕获模式为复制时，无法单独通过复制进行捕获
auto lambda1 = [=, =x]{ DebugLog(x); };
 
auto lambda2 = [=, &x]{ DebugLog(x); }; // OK
auto lambda3 = [=, this]{ DebugLog(this->Val); }; // OK
auto lambda4 = [=, *this]{ DebugLog(this->Val); }; // OK
```

第三，我们只能捕获一个名称或this一次：

```c++
int x = 123;
 
// 编译器错误：不能两次按名称捕获
auto lambda1 = [x, x]{ DebugLog(x); };
 
// 编译器错误：在初始化时不能两次按名称捕获
auto lambda2 = [x, x=x]{ DebugLog(x); };
 
// 编译器错误：混合捕获模式不能两次捕获
auto lambda3 = [x, &x]{ DebugLog(x); };
 
// 编译错误：不能两次捕获“this”
auto lambda4 = [this, this]{ DebugLog(this->Val); };
 
// 编译器错误：不能两次捕获“this”（混合捕获模式）
auto lambda5 = [this, *this]{ DebugLog(this->Val); };
```

第四，如果lambda不在一个块中或类默认数据成员初始化器中，它不能使用默认捕获或也不能在没有初始值设定项的情况下具有单个捕获：

```c++
//全局范围...
 
//编译器错误：此处不能使用默认捕获
auto lambda1 = [=]{ DebugLog("hi"); };
auto lambda2 = [&]{ DebugLog("hi"); };
 
// 编译器错误：在这里不能使用未初始化的捕获
auto lambda3 = [x]{ DebugLog(x); };
auto lambda4 = [&x]{ DebugLog(x); };
```

第五，类成员只能通过初始化器单独捕获：

```c++
class Test
{
    int Val = 123;
 
    void Foo()
    {
        // 编译器错误：成员必须用初始化器捕获
        auto lambda1 = [Val]{ DebugLog(Val); };
 
        auto lambda2 = [Val=Val]{ DebugLog(Val); }; // OK
        auto lambda3 = [&Val=Val]{ DebugLog(Val); }; // OK
    }
};
```

第六，同样地，类成员永远不会被默认捕获模式捕获。只有这个（this）被捕获，成员从这个指针访问。

```c++
class Test
{
    int Val = 123;
 
    void Foo()
    {
        // 默认捕获模式未捕获成员
        // 只有“this”被捕获
        auto lambda1 = [=]{ DebugLog(Val); };
        auto lambda2 = [&]{ DebugLog(Val); };
    }
};
```

第七，默认参数中的lambda无法捕获任何内容：

```c++
// 编译器错误：默认参数中的lambda不能有捕获
void Foo(int val = ([=]{ return 2 + 2; })())
{
    DebugLog(val);
}
```

第八，匿名联合体成员不能被捕获：

```c++
union
{
    int32_t intVal;
    float floatVal;
};
intVal = 123;
 
// 编译器错误：无法捕获匿名联合体成员
auto lambda = [intVal]{ DebugLog(intVal); };
```

第九点，最后，如果一个嵌套的lambda捕获了被嵌套的lambda捕获的某个东西，嵌套捕获会以两种情况之一进行转换。
</br>第一种情况是如果嵌套的lambda通过复制捕获了某个东西。
</br>在这种情况下，嵌套的lambda捕获的是外部lambda类的数据成员，而不是最初捕获的内容。

```c++
void Foo()
{
    int x = 1;
    auto outerLambda = [x]() mutable
    {
        DebugLog("outer", x);
        x = 2;
        auto innerLambda = [x]
        {
            DebugLog("inner", x);
        };
        innerLambda();
    };
    x = 3;
    outerLambda(); // outer 1 inner 2
}
```

第二种情况是如果嵌套的lambda通过引用捕获了某些内容。在这种情况下，嵌套的lambda捕获了原始变量或this：

```c++
void Foo()
{
    int x = 1;
    auto outerLambda = [&x]() mutable
    {
        DebugLog("outer", x);
        x = 2;
        auto innerLambda = [&x]
        {
            DebugLog("inner", x);
        };
        innerLambda();
    };
    x = 3;
    outerLambda(); // outer 3 inner 2
}
```

## IILE 立即调用的Lambda表达式

在C++中，一个常见的惯用用法，如上例中默认函数参数所示，被称为立即调用的Lambda表达式。
我们可以在各种情况下使用这些表达式来绕过各种语言规则。例如，许多C++程序员努力将所有可以const的变量都设置为const。
然而，如果初始化const变量的值需要多个语句，那么可能需要移除const。例如：

```c++
Command command;
switch (byteVal)
{
    case 0:
        command = Command::Clear;
        break;
    case 1:
        command = Command::Restart;
        break;
    case 2:
        command = Command::Enable;
        break;
    default:
        DebugLog("Unknown command: ", byteVal);
        command = Command::NoOp;
}
```
尽管我们可能只在switch语句中初始化它，之后从未设置过，但我们无法将command变成一个const变量。
</br>我们本可以将switch转换成一系列的条件运算符，但那样的话，我们就无法在默认情况下打印错误信息了：

```c++
const Command command = byteVal == 0 ?
    Command::Clear :
    byteVal == Command::Restart ?
        Command::Enable :
        Command::NoOp;
}
 
// 不必要的分支指令：我们已经在上面确定它是NoOp
if (command == Command::NoOp)
{
    DebugLog("Unknown command: ", byteVal);
}
```

为了解决这个问题，我们可以使用一个IILE（Immediate If Else Lambda）来包装switch语句。
</br>为此，我们需要在lambda表达式周围加上括号，然后在后面再加上括号以立即调用它：

```c++
const Command command = ([byteVal]{
                                    switch (byteVal)
                                    {
                                        case 0: return Command::Clear;
                                        case 1: return Command::Restart;
                                        case 2: return Command::Enable;
                                        default:
                                            DebugLog("Unknown command: ", byteVal);
                                            return Command::NoOp;
                                    }
                                   }
                         )();
```

编译器随后将创建一个在语句结束时被销毁的lambda类实例。
</br>构造函数和析构函数的开销将被优化掉，从而使得IILE（即时局部变量表达式）以及它所启用的const变得“免费”。


## C# Equivalency  C# 等效性
到目前为止，我们已经稍微比较了C++ lambda表达式和C# lambda表达式，但让我们更深入地看看。
</br>首先，我们注意到C++只支持“语句lambda”。我们不能像这样编写一个C#的“表达式lambda：

```csharp
// C#
(int x, int y) => x + y
```

这个例子还显示了另一个区别：C#的lambda参数总是显式类型化的。C++的lambda参数可以是auto，以支持多种参数类型：

```c++
auto lambda = [](auto x, auto y){ return x + y; };
 
// int 参数
DebugLog(lambda(2, 3)); // 5
 
// float 参数
DebugLog(lambda(3.14f, 2.0f)); // 5.14
```

同样，C++的返回类型可以是auto，实际上当没有使用像-> float这样的尾随返回类型时，这是默认的。
</br>C#的lambda表达式必须始终有隐式的返回类型。为了强制实现，通常在lambda体内部使用强制类型转换。

```csharp
// C#
(float x, float y) => { return (int)(x + y); };
```

另一方面，当将lambda表达式存储在变量中时，C#比C++更明确，因为不能使用var。

```csharp
// C#
Func<int, int, int> f1 = (int x, int y) => { return x + y;}; // OK
var f2 = (int x, int y) => { return x + y;}; // Compiler error
```

C++ allows for `auto`:
```c++
auto lambda = [](int x, int y){ return x + y; };
```

C#的lambda表达式支持丢弃参数：

```csharp
// C#
Func<int, int, int> f = (int x, int _) => { return x; }; // 丢弃 y
```

C++可以通过省略名称，类似于_，或者将参数转换为void来实现这一点：

```c++
// 省略参数名称
auto lambda1 = [](int x, int){ return x; };
 
// 将参数转换为空
auto lambda2 = [](int x, int y){ static_cast<void>(y); return x; };
```

C# 有静态lambda表达式来防止捕获局部变量或非静态字段。这是C++的默认行为。
</br>在C++中，捕获是默认可选的，并且可以通过默认和单独的捕获进行选择：

```c++
int x = 123;
 
// 捕获不到任何内容
 // 编译错误：无法访问x
auto lambda1 = []{ DebugLog(x); };
 
//通过复制隐式捕获
auto lambda2 = [=]{ DebugLog(x); };
 
// 隐式通过引用捕获
auto lambda3 = [&]{ DebugLog(x); };
 
// 通过复制显式捕获
auto lambda4 = [x]{ DebugLog(x); };
 
// 显式地通过引用捕获
auto lambda5 = [&x]{ DebugLog(x); };
```

C#禁止捕获`in`、`ref`和`out`变量。C++的引用和指针，与C#最相似，可以以多种方式自由捕获。

C# 支持**异步**lambda表达式，就像它支持其他类型的函数一样。C++ 没有内置的异步和await系统，因此这些功能不支持。

最后，并且最重要的是，C++中的lambda表达式与C#中的委托不同。C++没有托管类型的概念，也没有任何内置构造，可以像委托那样在运行时进行垃圾回收和支持多监听器。

相反，C++的lambda表达式只是普通的C++类。它们有构造函数、赋值运算符、析构函数、重载运算符和用户定义的转换运算符。

因此，它们的行为更像其他C++类对象，而不是像受管理的、垃圾回收的C#类。

## Conclusion  结论

两种语言中的Lambda都扮演着相似的角色：提供匿名函数。
</br>除了C#中的异步Lambda之外，C++版本的Lambda提供了更广泛的功能集。两种语言的方法有所不同，因为C#通过使Lambda成为托管委托来权衡安全性的利弊。
</br>C++采用低，或通常是零开销的方法，使用常规类，但代价是可能出现悬垂指针和引用等可能的错误。

> 在原作者的评论区有一个补充：
> </br>在 C# 中，lambda 表达式也会转换为类。可以在此处查看编译器生成的代码：
> https://sharplab.io/#v2:CYLg1APgAgTAjAWAFBQMwAJboMLoN7LpGYZQAs6AsgBQCU+hxTjTRUArADwCWAdgC4AadHyEiBAPnQAzOOgC86aqPQAPYSoCe9eVLyYA7GvRh0mgNwBfcy3SXkloA===