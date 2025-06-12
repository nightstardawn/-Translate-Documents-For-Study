# C++ For C# Developers: Part 6 – Control Flow

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5564),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

让我们继续这个系列的另一个基础话题：控制流程。
</br>维恩图在这里有很大的重叠，但C#和C++都有自己的独特特性，而且两个语言共有的某些特性在两者之间有重要的差异。继续阅读以了解细节！

## If and Else  If 和 Else

让我们从低级 `if` 语句开始，就像在 C# 中一样：

```c++
if (someBool)
{
    // ... 如果 someBool 为true则执行此操作
}
```

与`C#`不同，可以在开始处附加一个可选的初始化语句。这就像`for`循环的第一个语句，通常用于声明一个作用域限于`if`语句的变量。以下是它的典型用法：

```c++
if (ResultCode code = DoSomethingThatCouldFail(); code == FAILURE)
{
    // ... 如果DoSomethingThatCouldFail返回false，则执行此操作
}
```

`else` 部分就像 C# 一样：

## Goto and Labels  转到和标签

`goto` 语句也类似于 C# 中的语句。我们创建一个标签，然后在 `goto` 语句中命名它：

```c++
void DoLotsOfThingsThatMightFail()
{
    if (!DoThingA())
    {
        goto handleFailure;
    }
    if (!DoThingB())
    {
        goto handleFailure;
    }
    if (!DoThingC())
    {
        goto handleFailure;
    }
 
    handleFailure:
        DebugLog("Critical operation failed. Aborting program.");
        exit(1);
}
```

与在 `C#` 中一样，要 `goto` 的标签必须位于同一函数中。与 C# 不同，标签不能位于 `try` 或 `catch` 块内。

一个细微的差别是，C++中的`goto`可以跳过变量的声明，但不能跳过它们的初始化。例如：

```c++
oid Bad()
{
    goto myLabel;
    int x = 1; // 不可跳过的初始化
    myLabel:
    DebugLog(x);
}
 
void Ok()
{
    goto myLabel;
    int x; // 无初始化。可以跳过。
    myLabel:
    DebugLog(x); // 使用未初始化的变量
}
```

与使用未初始化的变量一样，这是未定义的行为，可能会导致严重错误。应注意确保在读取变量之前最终初始化变量。

## Switch  开关

C++ `switch`、`case` 和 `default` 类似于它们的 C# 对应项：

```c++
switch (someVal)
{
    case 1:
        DebugLog("someVal is one");
        break;
    case 2:
        DebugLog("someVal is two");
        break;
    case 3:
        DebugLog("someVal is three");
        break;
    default:
        DebugLog("Unhandled value");
        break;
}
```

一个区别是，不为空的 `case` 可以省略 `break` 并 “贯穿(fall through)” 到下一个 `case`。这有时被认为容易出错，但也可以减少重复。这两者是等效的：


```c++
// C#
switch (someVal)
{
    case 3:
        DoAtLeast3();
        DoAtLeast2();
        DoAtLeast1();
        break;
    case 2:
        DoAtLeast2();
        DoAtLeast1();
        break;
    case 1:
        DoAtLeast1();
        break;
}
 
// C++
switch (someVal)
{
    case 3:
        DoAtLeast3();
    case 2:
        DoAtLeast2();
    case 1:
        DoAtLeast1();
}
```

另一个区别是，在 `case` 中需要花括号才能声明变量：

```c++
switch (someVal)
{
    case 1:
    {
        int points = CalculatePoints();
        DebugLog(points);
        break;
    }
    case 2:
        DebugLog("someVal is two");
        break;
    case 3:
        DebugLog("someVal is three");
        break;
}
```

C++ `switch` 语句也支持初始化语句，与 `if` 非常相似：

```c++
switch (ResultCode code = DoSomethingThatCouldFail(); code)
{
    case FAILURE:
        DebugLog("Failed");
        break;
    case SUCCESS:
        DebugLog("Succeeded");
        break;
    default:
        DebugLog("Unhandled error code");
        DebugLog(code);
        break;
}
```

与 C# 不同，`switch` 只能用于整数和枚举类型。需要 `if` 和 `else` 的链来处理其他任何内容：

```c++
if (player == localPlayer)
{
    // .. handle the local player
}
else if (player == adminPlayer)
{
    // .. handle the admin player
}
```

C# 的模式匹配也不受支持，因此无法编写大小写 `int x`： 来匹配所有 `int` 值并将其值绑定到 `x`。
</br>也没有 `when` 子句，所以我们不能写 `case Player p when p.NumLives > 0:` .
</br>相反，我们再次在 C++ 中使用 if 和 else 来执行这些作。

同样不支持 `goto case X;`。相反，我们需要创建自己的标签并转到它：

```c++
switch (someVal)
{
    case DO_B:
        doB:
        DoB();
        break;
    case DO_A_AND_B:
        DoA();
        goto doB;
}
```

## Ternary  三元运算符

C++ 中的三元运算符也类似于 C# 版本：

```c++
int damage = hasQuadDamage ? weapon.Damage * 4 : weapon.Damage;
```

与在 C# 中一样，这等效于：

```c++
int damage;
if (hasQuadDamage)
    damage = weapon.Damage * 4;
else
    damage = weapon.Damage;
```

C++ 版本对我们可以放入 `?` 和 `：` 部分的内容要宽松得多。例如，我们可以抛出一个异常：

```c++
SaveHighScore() ? Unpause() : throw "Failed to save high score";
```

在这种情况下，表达式的类型是`非throw` 部分所具有的任何类型的类型：Unpause 的返回值。我们甚至可以把这两部分都抛出：

```c++
errorCode == FATAL ? throw FatalError() : throw RecoverableError();
```

当我们这样做时，表达式的类型是 `void`。当然，例外是它们自己的控制流类别，我们将在本系列后面更深入地介绍。

还有更多规则来确定三元表达式的类型，但通常我们只在 `？` 和 `：` 部分使用相同的类型，就像我们对 `damage` 示例所做的那样。在这种最典型的情况下，三元表达式的类型与任一部分相同。

## While, Do-While, Break, and Continue


`while` 和 `do-while` 循环本质上与 C# 中的完全相同：

```c++
while (NotAtTarget())
{
    MoveTowardTarget();
}
 
do
{
    MoveTowardTarget()
} while (NotAtTarget());
```

`break` 和 `continue` 的工作方式也相同：

```c++
int index = 0;
int winnerIndex = -1;
while (index < numPlayers)
{
    // 已故玩家不能成为赢家
    // 使用 `continue` 跳过循环体的其余部分
    if (GetPlayer(index).Health <= 0)
    {
        index++;
        continue;
    }
 
    // 如果他们至少有100分，就找到了赢家
    // 不需要继续搜索，所以使用 `break` 来结束循环
    if (GetPlayer(index).Points >= 100)
    {
        winnerIndex = index;
        break;
    }
}
if (winnerIndex < 0)
{
    DebugLog("no winner yet");
}
else
{
    DebugLog("Player", index, "won");
}
```

## For  For循环

常规的三部分`for`循环在C#中也基本上是一样的：

```c++
for (int i = 0; i < numBullets; ++i)
{
    SpawnBullet();
}
```

C++ 有一个 `for` 的变体，它取代了 C# 中的 `foreach`。它称为“基于范围的 `for` 循环”，用冒号表示：

```c++
int totalScore = 0;
for (int score : scores)
{
    totalScore += score;
}
```

它甚至支持一个可选的初始化语句，就像我们看到的 `if` 一样：

```c++
int totalScore = 0;
for (int index = 0; int score : scores)
{
    DebugLog("Score at index", index, "is", score);
    totalScore += score;
    index++;
}
```
编译器实质上将基于范围的 `for` 循环转换为常规的 `for` 循环，如下所示：

```c++
int totalScore = 0;
{
    int index = 0;
    auto&& range = scores;
    auto cur = begin(range); // or range.begin()
    auto theEnd = end(range); // or range.end()
    for ( ; cur != theEnd; ++cur)
    {
        int score = *cur;
        DebugLog("Score at index", index, "is", score);
        totalScore += score;
        index++;
    }
}
```

我们很快就会介绍指针和引用(Part8)，但就目前而言，`auto&& range = scores` 本质上是 `scores` 的同义词，称为 `range`， 而 `*cur` 则采用 `cur` 指针指向的值。

必须有采用任何类型 `scores` 的 `begin` 和 `end` 函数，否则 `scores` 必须具有名为 `begin` 和 `end` 的方法，这些方法不带参数。
</br>如果编译器找不到任何一组 `begin` 和 `end` 函数，则会出现编译器错误。无论它们位于何处，这些函数还需要返回一个类型，该类型可以比较不等式 （`cur ！= end`）、预递增 （`++cur`） 和取消引用 （`*cur`），否则将出现编译器错误。

正如我们将在整个系列中看到的那样，有许多类型符合这一标准，许多用户创建的类型也被设计得符合这一标准。

## C#-Exclusive Operators  C# 独占运算符

C# 的某些控制流运算符在 C++ 中根本不存在。首先，没有 `？？` 或 `？？=` 运算符。三元运算符 or `if` 通常用于其位置：

```c++
// `??`操作符的替代方案
int* scores = m_Scores ? m_Scores : new Scores();
 
// `??=`运算符的替代方案
if (!scores) scores = new Scores();
```

请注意，在C#中，`nullptr`与`null`等价，它只是一个符合任何类型指针但不符合整数的空值。


## Return  返回

恰如其分地，我们今天以 `return` 结束。典型的版本就像在 C# 中一样：

```c++
int CalculateScore(int numKills, int numDeaths)
{
    return numKills*10 - numDeaths*2;
}
```

还有一个替代版本，其中大括号用于或多或少地将参数传递给返回类型的构造函数：
```c++
CircleStats GetCircleInfo(float radius)
{
    return { 2*PI*radius, PI*radius*radius };
}
```

在本系列的后续部分，我们将进一步探讨构造函数(Part13)。
</br>目前，C++语言对于像`CircleStats`这样的返回对象有一个重要的保证：复制省略。
</br>这意味着如果花括号中的值是“纯”的，就像这些简单的常量和原始数据类型一样，那么`CircleStats`对象将在调用位置初始化。
</br>这意味着`CircleStats`不会在`GetCircleInfo`函数内部分配在栈上，而是在`GetCircleInfo`返回时复制到调用位置。
</br>这有助于我们在复制返回值时避免昂贵的复制，尤其是当涉及复制大量数据，如大数组时。

## Conclusion  结论

许多控制流机制在 C++ 和 C# 之间共享。
</br>我们仍然有 `if`、`else`、`？：`、`switch`、`goto`、`while`、`do`、`for`、`foreach/“基于范围的 for”`、`break`、`continue` 和 `return`。

C# 还具有 `？？`、`？？=`、`？.` 和 `？[]`，
</br>但 C++ 还对 `if`、`switch` 和`基于范围的 for 循环`、返回值复制省略以及 `？：`、`goto` 和 `switch` 具有更大的灵活性。

这些差异导致我们使用这两种语言编写代码的方式不同。
</br>例如，我们需要 `begin` 和 `end` 函数或方法，以便为 C++ 中的类型启用`基于范围的 for 循环`。
</br>如果我们编写 C#，我们通常会实现 `IEnumerator<T>` 接口。





