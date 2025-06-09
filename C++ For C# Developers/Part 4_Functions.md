# C++ For C# Developers: Part 4 – Functions

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5547),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

该系列今天继续提供函数。这些显然是任何编程语言的核心，但它们与 C# 中的函数有多少不同并不明显。从编译时执行到自动返回值类型，目前有很多差异需要涵盖。

## Declaration and Definition  声明和定义

C++中的函数可以分为两部分。第一部分是函数声明，它只说明了函数的签名，而没有说明其工作原理。为此，只需在签名后添加一个分号即可：

```c++
int Add(int a, int b);
```

现在让我们来编写第二部分：函数的定义。这同样包含了函数的签名，但也包括了函数体：

```c++
int Add(int a, int b)
{
    return a + b;
}
```

那么为什么C++既有声明又有定义，而C#只有定义呢？这主要与C++的编译方式有关。
</br>我们将在下一篇文章中更深入地探讨这个问题，但到目前为止，重要的是要知道C++是从源文件的顶部到底部进行编译的。
</br>函数的定义或声明使其能够被文件中更下方的代码引用。以下是编译器所做的大致过程的近似描述：

```c++
// 从这里开始编译并阅读文件的每一行。
 
// 目前没有函数 `Add`
// 无论之后是否有 `Add` 函数都无关紧要
// 这是一个编译错误
int four = Add(1, 3);
 
// 声明 `Add` 函数
// `Add` 现在可以引用
int Add(int a, int b);
 
// 引用了编译器已知的`Add`，尽管目前还没有定义
// 没关系，定义稍后会出现
// 编译器相信定义将会稍后出现
// 如果没有，将会出现错误
int three = Add(2, 1);
 
// 定义了`Add`函数
// 程序员已经履行了定义它的承诺
// 编译器现在知道当调用`Add`时要做什么
int Add(int a, int b)
{
    return a + b;
}
```

这也是可以的：
```c++
// 定义了`Add`函数
// 现在可以引用`Add`
// 还说明了调用`Add`时应该做什么
int Add(int a, int b)
{
    return a + b;
}
 
// 指的是编译器所知的`Add`
int three = Add(2, 1);
```

因此，我们可以使用函数声明来重新排列我们的代码，即使它是从上到下编译的。
</br>我们只需在顶部声明签名，然后将主体留到后面。
</br>在下一篇文章中，我们将讨论头文件，这将会变得非常重要。目前，了解C++函数通常有声明和无声明两种形式是很重要的。

最后一个怪癖：可以在一个语句中声明多个函数，就像`int a, b`;声明了两个变量一样。这种情况很少见，通常应该避免，而应该选择单声明形式。

```c++
// 两个函数：
//   int Add(int a, int b)
//   int Sub(int a, int b)
int Add(int a, int b), Sub(int a, int b);
```

与变量一样，这两个函数共享相同的返回类型。

## Optional Argument Names and Void  可选参数名称和 void

C++中参数名称有一个奇怪的特性：它们是可选的！这个声明是完全正确的：

```c++
int Add(int, int);
```

毕竟，我们只是在告诉编译器函数的签名，参数名称对此并不重要。也许更奇怪的是，我们甚至可以省略函数定义中的参数名称！

```c++
// 一个非常糟糕的加法实现...
int Add(int a, int)
{
    return a + 1;
}
```

有时这在需要特定函数签名但实际函数体内并未使用这些参数时很有用。考虑一个打算用作事件处理器的函数：

```c++
void OnPlayerSpawned(Vector3)
{
    NumSpawns++;
}
```

这个函数并不关心玩家在哪里出生，因为它所做的只是跟踪一个统计数据。因此，我们可以出于几个原因省略参数名称。
</br>首先，它告诉读者这个参数在函数中并不重要，所以甚至不需要记住它的名字。
</br>其次，它告诉编译器不要对未使用的变量发出警告。
</br>毕竟，我们一开始就不能使用没有名字的变量。

有时在C++代码中，我们会看到一种折中的方法，即在注释中声明名称，以获得第二个好处，但不是第一个。

```c++
void OnPlayerSpawned(Vector3 /* position */)
{
    NumSpawns++;
}
```

如果函数没有任何参数，可以可选地通过在参数通常放置的位置放置`void`来明确表示这一点：
但是是否添加 `void` 纯粹是一种风格选择。

```c++
uint64_t GetCurrentTime(void);
```

## Automatic Return Types  自动返回类型

C++ 变量可以使用 `auto` 类型声明，类似于 C# 中的 `var`。在 C++ 中，函数返回类型也可以声明为 `auto`：

```c++
auto Add(int a, int b)
{
    return a + b;
}
```

就像变量一样，编译器会确定返回类型应该是什么。在这种情况下，它只是`int`，因为当我们相加两个`int`值时，我们得到的就是`int`。

如果我们将 `auto` 放在函数名称之前，我们还可以在 argument list 之后指定返回类型：

```c++
auto Add(int a, int b) -> int
{
    return a + b;
}
```

在这种情况下，我们明确声明了返回类型。编译器不会自动确定它，尽管我们仍然需要在函数名前添加`auto`。
</br>当返回类型非常复杂时，这种替代语法有时很有用。在系列后面的例子中，当我们处理函数指针和模板时，我们将看到一些示例。

> 这里原作者似乎忘记了这码事，我在这里补充一下
> </br>下面的例子需要看完Template(Part26)内容在回过头来看
> </br>这里主要的作用是在一些特定的场景下，使得更加直观
> 
> </br>例如第一个例子: 返回类型依赖于参数类型
> ```c++
> template <typename T, typename U>
> auto Add(T a, U b) -> decltype(a + b) {
>     return a + b;  // 返回类型由 a+b 的结果类型决定
> }
> ```
> 
> 再来看第二个例子: 简化复杂返回类型声明
> ```c++
> // 传统写法：返回函数指针
> int (*GetFunc())(int, int) { ... }
> 
> // 尾置写法更清晰
> auto GetFunc() -> int (*)(int, int) { ... }
> ```

## Default Arguments  默认参数

在C#中，一旦某个参数指定了默认值，它之后的所有参数都必须有默认值。
</br>然而，在C++中情况略有不同，因为声明和定义是分开的。如果一个函数既有声明又有定义，默认参数将指定在声明中：

```c++
// 函数声明指定默认参数值
void SpawnPlayer(Vector3 position, float speed=0.0f);
 
// 函数定义省略了它们
void SpawnPlayer(Vector3 position, float speed)
{
    // ...
}
```

## Variadic Functions  可变参数函数

就像在C#中，函数可以接受可变数量的参数。
但在C++中，这却有着完全不同的实现方式。它并没有像`register`那样被弃用，但通常认为使用这个特性也是一种不好的做法。
尽管如此，让我们看看它是如何实现的：

```c++
// `...` 放在所有正常参数之后
// 它意味着“0个或多个任意类型的参数”
void PrintLog(LogLevel level, ...)
{
    // ...
}
```

该函数随后应调用 `va_start`、`va_arg` 和 `va_end` 来获取参数。这相当不安全，并且接口非常笨拙，这也是为什么通常不建议使用该功能的原因。
</br>有几种替代方案更受欢迎，但许多都是将在系列后续部分讨论的更高级功能。
</br>现在，让我们讨论一个简单的：重载。

## Overloading  重载

与在 C# 中一样，函数可以重载，因为可能存在多个具有相同名称的函数。在调用该函数时，编译器会确定实际上应该调用哪些同名函数。

```c++
// 根据玩家的ID获取玩家的分数
int GetPlayerScore(int playerId);
 
// 获取本地玩家的分数
int GetPlayerScore();
 
// 获取给定位置玩家的得分
int GetPlayerScore(Vector3 position);
```

这些函数根据参数类型和参数数量而有所不同。现在我们可以这样编写代码：

```c++
score = GetPlayerScore(myPlayerId);
score = GetPlayerScore();
score = GetPlayerScore(myPosition);
```

在这种情况下，编译器将按照我们声明的顺序生成对三个函数的调用。

## Ref, Out, and In Arguments  Ref、Out 和 In 参数

在C#中，可以使用`ref`、`in`和`out`关键字来声明参数。
</br>这些关键字中的每一个都将参数更改为指向传递值的指针。在C++中，这些关键字不存在。相反，我们使用一些替代方案：

```c++
// `ref`的替代方案
// 使用一个左值引用，它类似于非空指针
void MovePlayer(Player& player, Vector3 offset)
{
    player.position += offset;
}
 
// `in`的替代方案
// 使用常量左值引用
// `const`表示它不能被更改
void PrintPlayerName(const Player& player)
{
    DebugLog(player.name);
}
 
// `out` 的替代方案
// 只需使用返回值
ReallyBigMatrix ComputeMatrix()
{
    ReallyBigMatrix matrix;
    // 具体计算matrix
    return matrix
}
 
// 另一个 `out` 的替代方案
// 使用左值引用参数
void ComputeMatrix(ReallyBigMatrix& mat1, ReallyBigMatrix& mat2)
{
    mat1 = /* math for mat1 */;
    mat2 = /* math for mat2 */;
}
 
// 另一个 `out` 的替代方案
// 将输出打包到返回值中
tuple<ReallyBigMatrix, ReallyBigMatrix> ComputeMatrix()
{
    return make_tuple(/* math for mat1 */, /* math for mat2 */);
}
```

在`ref`的情况下，C++函数通常使用左值引用。如果需要可空引用，这在C#的`ref`参数中是不允许的，可以使用指针（`Player*`）代替。

对于`in`参数，通常使用`const lvalue`引用。我们还没有介绍`const`，但简单来说，它不允许更改变量。
如果尝试写入`player.score = 0`;，将会导致编译器错误。
这在大体上与C#中的`in`参数发生的情况相似。同样，如果参数有时需要为`null`，可以使用指针（`const Player* player`）。

`out`参数通常只是通过返回它们来编写的。如果需要多个返回值，有几个主要选项。
首先，可以采用左值引用参数。这个的缺点是它们在被分配之前可以被读取，因此可能会被意外地用作输入。调用者也不清楚他们是在提供输入、输出还是两者的参数。
第二个是更受欢迎的选项：将所有输出打包到一个结构体中并返回。我们还没有讨论结构体或模板，它们类似于C#泛型，但标准库的`tuple`类型和`make_tuple`辅助函数在这里被展示为C#元组的替代品。

## Static Variables  静态变量

函数内的局部变量可以被声明为静态。类似于C#中类和结构体的静态字段，这意味着该变量只有一个实例。静态的C++局部变量在整个函数调用过程中只有一个实例：

```c++
int GetNextId()
{
    static int id = 0;
    id++;
    return id;
}
 
GetNextId(); // 1
GetNextId(); // 2
GetNextId(); // 3
```

在这个例子中，我们有一个静态的局部变量`id`。在整个调用`GetNextId`的过程中，`id`只有一个。这就像`id`是一个全局变量，但只能在其声明的函数内部引用。
</br>这可以非常方便，但也可能让人感到惊讶，就像C#中的静态字段一样。

## Constexpr  常量

今天最后，函数可以被标记为`constexpr`。这意味着函数可以在编译时进行评估。例如：

```c++
constexpr int GetSquareOfSumUpTo(int n)
{
    int sum = 0;
    for (int i = 0; i < n; ++i)
    {
        sum += i;
    }
    return sum * sum;
}
```

此函数可以在编译时进行评估，以生成一个常量：

```c++
DebugLog(GetSquareOfSumUpTo(5000));
 
// 相当于...
 
DebugLog(1020530960)
```

该函数也可以在运行时进行评估，例如当其参数依赖于运行时值时：

```c++
int n = file.ReadInt();
DebugLog(GetSquareOfSumUpTo(n));
```

这意味着普通的C++可以用于编译时和运行时的工作。通常不需要在另一种语言中运行脚本以生成C++文件。程序已经包含的类型和功能可以通过这种机制在编译时使用。

尽管在`constexpr`函数中可以实现一些功能，但仍然存在一些限制。
</br>自从C++11引入它们以来，每个版本都放宽了这些限制。
</br>然而，即使在C++20中，一些功能如静态局部变量和`goto`也是不允许的。

> Part23 有对 `constexpr` 这个关键字有具体的讲解。

## Conclusion  结论

从广义上讲，C# 和 C++ 之间的函数非常相似。不过，存在许多差异。这些差异涵盖了语法怪癖，比如 `ref` 参数的声明方式，一直到完全不同的功能，比如编译时函数执行和静态局部变量。
</br>随着本系列的进展，我们将了解更多类型的函数，包括成员函数（“methods”）和 lambdas(Part22)！