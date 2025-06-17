# C++ For C# Developers: Part 35 – Modules, The New Build Model

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6240),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

我们已经看到了 C++ 基于 `#include` 的传统构建模型 。今天，我们将了解 C++20 中引入的全新构建模型。
</br>这是基于 “modules” 构建的，与 C# 构建模型更相似。请继续阅读，了解如何单独使用和与 `#include` 结合使用！

## Module Basics  模块基础

新的构建系统基于一种称为“module”的新语言概念。
</br>该系统有望显著减少编译时间，包括干净和增量编译。它还承诺通过防止预处理器指令和实现细节的泄漏来显著提高封装。
</br>最后，它完全消除了在源代码中使用#include指定文件系统路径，然后通过复杂的目录查找来查找引用文件的需要。

要将翻译单元（例如 `.cpp` 文件）转换为 “module unit”，我们使用 `export module` 语句：

```c++
///////////
// math.ixx
///////////
 
export module math;
```

我们在这里做了两件事。首先，我们使用 `.ixx` 扩展名命名了模块。
</br>模块文件可以使用任何扩展名命名，也可以根本不使用扩展名命名，就像任何其他 C++ 源文件一样。
</br>这里使用 `.ixx` 扩展名仅仅是因为它是 Microsoft Visual Studio 2019 的偏好，它是最早支持模块的编译器之一。

第二行 `export` 模块 `math`; 开始一个名为 `math` 的模块。与 C++ 的其余部分一样，源文件是从上到下读取的。
</br>此语句之后的所有内容都是 `math` 模块的一部分，但之前的所有内容都不是。

目前，该模块为空，因为源文件中没有其他内容。我们来添加一些函数：

```c++
///////////
// math.ixx
///////////
 
// 在“export module”语句之前的正常函数
float Average(float x, float y)
{
    return (x + y) / 2;
}
 
// 在“export module”语句之前导出的函数
export float MagnitudeSquared(float x, float y)
{
    return x*x + y*y;
}
 
// 模块从这里开始
export module math;
 
// 在“export module”语句之后的正常函数
float Min(float x, float y)
{
    return x < y ? x : y;
}
 
// 在“export module”语句之后导出的函数
export float Max(float x, float y)
{
    return x > y ? x : y;
}
```

这里还有一些需要注意的事情。首先，我们可以在任何我们希望从模块外部使用的东西前添加`export`。
</br>这包括像这些函数、变量、类型、使用别名、模板和命名空间等。它不包括预处理指令，如宏。

模块似乎类似于命名空间，但两者截然不同。模块可以导出命名空间，而模块并不意味着命名空间。
</br>模块并不是要替换命名空间，但它们可以用于将相关功能分组在一起的类似目的。

我们可以导出任何没有内部链接的东西，例如通过声明为静态或位于未命名的命名空间中。
</br>我们的导出必须直接位于命名空间块内，位于文件顶层的任何块之外，或者位于导出块中：

```c++
// 此块中的所有内容都被导出
export
{
    float Min(float x, float y)
    {
        return x < y ? x : y;
    }
 
    // 冗余的“导出”没有效果
    export float Max(float x, float y)
    {
        return x > y ? x : y;
    }
}
```

其次，这些函数中有两个位于`export`模块`math;`语句之前。
</br>这些是“全局模块”的一部分，而不是`math`模块的一部分，就像命名空间之外的所有内容都是“全局命名空间”的一部分一样。


模块单元源文件中只能有一个模块。这是不允许的：

```c++
// First module: OK
export module math;
float Min(float x, float y)
{
    return x < y ? x : y;
}
 
// Second module: compiler error
export module util;
export bool IsNearlyZero(float val)
{
    return val < 0.0001f;
}
```

假设我们不这样做，现在让我们从另一个文件中使用这个模块：

```c++
///////////
// main.cpp
///////////
 
// 导入模块以使用
import math;
 
// OK: Max在导入的“math”模块中找到
DebugLog(Max(2, 4)); // 4
 
// 编译器错误：这些都不是“math”模块的组成部分
DebugLog(Average(2, 4));
DebugLog(MagnitudeSquared(2, 4));
DebugLog(Min(2, 4));
```

我们使用 `import` 来命名我们要使用的模块。我们可以访问该模块中标记为 `export` 的所有内容。与头文件不同，我们不指定模块单元的文件名
</br>这类似于 C# 构建系统，我们使用`using System;`。

## Partitions and Fragments  分区和片段

我们可以将模块的所有代码放在一个文件中，但随着我们添加的代码越来越多，这并不能很好地扩展。
</br>想象一下，所有 `System.Collections.Generic` 都放在一个文件中！C# 通过在每个文件中放置一个类（`List<T>、Dictionary<K, V>` 等）来解决此问题。
</br>C++ 以多种方式解决了这个问题。第一个称为“模块分区”，它们允许我们将代码拆分到多个文件中，同时仍然作为单个模块的一部分。

```c++
///////////////
// geometry.ixx
///////////////
 
// 指定这是“math”模块的“geometry”分区
export module math:geometry;
 
export float MagnitudeSquared(float x, float y)
{
    return x * x + y * y;
}
 
////////////
// stats.ixx
////////////
 
// 指定这是“math”模块的“stats”分区
export module math:stats;
 
export float Min(float x, float y)
{
    return x < y ? x : y;
}
 
export float Max(float x, float y)
{
    return x > y ? x : y;
}
 
export float Average(float x, float y)
{
    return (x + y) / 2;
}
 
///////////
// math.ixx
///////////
 
// 这是主要的“math”模块
export module math;
 
// 导入“stats”分区并导出它
export import :stats;
 
// 导入“几何”分区并导出它
export import :geometry;
 
///////////
// main.cpp
///////////
 
// 正常导入“math”模块
import math;
 
// 正常使用其导出的实体
DebugLog(Min(2, 4)); // 2
DebugLog(Max(2, 4)); // 4
DebugLog(Average(2, 4)); // 3
DebugLog(MagnitudeSquared(2, 4)); // 20
```

我们在这里看到分区是用 `：` 指定的。模块分区命名主模块 （`math`） 及其分区的名称 （`stats`）。
</br>主模块只使用分区的名称 （`：stats`），因为它的名称 （`math`） 已经被说明过了，不需要重复。
</br>它必须导出所有分区，以便编译器在使用模块时知道模块中可用的所有内容。

与其他标识符不同，模块名称中可能包含 `.`。这意味着我们可以改用 `math.stats` 和 `math.geometry` 作为我们的模块名称：

```c++
///////////////
// geometry.ixx
///////////////
 
// 这是一个主要的“math.geometry”模块
export module math.geometry;
 
export float MagnitudeSquared(float x, float y)
{
    return x * x + y * y;
}
 
////////////
// stats.ixx
////////////
 
// 这是一个主要的“math.stats”模块
export module math.stats;
 
export float Min(float x, float y)
{
    return x < y ? x : y;
}
 
export float Max(float x, float y)
{
    return x > y ? x : y;
}
 
export float Average(float x, float y)
{
    return (x + y) / 2;
}
 
///////////
// math.ixx
///////////
 
// 这是主要的“math”模块
export module math;
 
// 导入“math.stats”模块 然后导出
export import math.stats;
 
// 导入“math.geometry”模块 然后导出
export import math.geometry;
 
///////////
// main.cpp
///////////
 
// 正常导入“math”模块
import math;
 
// 正常使用其导出的实体
DebugLog(Min(2, 4)); // 2
DebugLog(Max(2, 4)); // 4
DebugLog(Average(2, 4)); // 3
DebugLog(MagnitudeSquared(2, 4)); // 20
```

这里的区别在于 `math.stats` 和 `math.geometry` 不是分区，它们是主模块。它们中的任何一个都可以直接使用：

```c++
// 导入“math.stats”主模块
import math.stats;
 
// 正常使用其导出的实体
DebugLog(Min(2, 4)); // 2
DebugLog(Max(2, 4)); // 4
DebugLog(Average(2, 4)); // 3
```

需要注意的是，就编译器而言，`math.stats` 和 `math.geometry` 不是“子模块”。他们只是碰巧以某种方式命名，使他们看起来是这样的。
</br>这与 C# 命名空间基本相同，因为除了命名之外，`System、System.Collections` 和 `System.Collections.Generic` 之间没有特殊关系。

最后，有一个隐式私有 “片段” ，它只能保存不可能影响模块接口的代码。此限制允许编译器在只有私有片段更改时避免重新编译使用该模块的代码：

```c++
// 主模块
export module math;
 
// 导出一些函数声明
export float Min(float x, float y);
export float Max(float x, float y);
 
// 这开始变为私有片段
module :private;
 
// 定义一些非导出函数
float Min(float x, float y)
{
    return x < y ? x : y;
}
float Max(float x, float y)
{
    return x > y ? x : y;
}
```

## Module Implementation Unit  模块实现单元

到目前为止，我们所有的模块文件都一直是“模块接口单元”，因为它们包含了`export`关键字。它们是供模块外部的代码，如我们的`main.cpp`使用的接口。

不过还有另一种模块单元：'模块实现单元'。这些旨在包含模块的实现详细信息。它们不使用`export`关键字，但包含可从模块中访问的内部代码：

```c++
///////////////
// geometry.ixx
///////////////
 
// 非导出模块分区
module math:geometry;
 
// 一个非导出函数
float MagnitudeSquared(float x, float y)
{
    return x * x + y * y;
}
 
///////////
// math.ixx
///////////
 
// 主模块
export module math;
 
// 导入模块实现分区
import :geometry;
 
// 通过声明并添加“export”关键字从模块实现部分导出函数
// 并添加“export”关键字
export float MagnitudeSquared(float x, float y);
 
// 导出更多功能
export float Magnitude(float x, float y)
{
    // 调用导入模块实现部分的函数
    float magSq = MagnitudeSquared(x, y);
    return Sqrt(magSq); // TODO: write Sqrt()
}
```

这类似于我们在头文件 （`.hpp`） 和翻译单元 （`.cpp`） 之间拆分代码的方式。
</br>在那个传统的构建系统中，我们会在头文件中添加函数的声明，并在翻译单元中添加这些函数的定义。

如果我们不需要分区，但仍想将接口与 `implementation` 分开，我们可以删除 `import` 并删除分区名称：

```c++
///////////////
// geometry.cpp
///////////////
 
// 一个非导出模块
module math;
 
// 一个非导出函数
float MagnitudeSquared(float x, float y)
{
    return x * x + y * y;
}
 
///////////
// math.ixx
///////////
 
export module math;
 
// 注意：无需导入“math”模块，因为这个模块已经存在
 
export float MagnitudeSquared(float x, float y);
 
export float Magnitude(float x, float y)
{
    float magSq = MagnitudeSquared(x, y);
    return Sqrt(magSq); // TODO: write Sqrt()
}
```

请注意，我们现在有 `geometry.cpp`，而不是 `geometry.ixx`。这是因为它不能再被导入，必须像我们在 `math.ixx` 模块单元中所做的那样隐式使用。


## Module Linkage  模块联动

在传统的构建模型中 ，有 “内部链接” 和 “外部链接” 之分。这意味着某些内容要么在翻译单元内部相同，要么在翻译单元之间外部相同。
</br>对于模块，现在有 “模块链接”。这意味着所有模块单元和模块的用户都是相同的：

```c++
///////////////////
// statsglobals.ixx
///////////////////
 
export module stats:globals;
 
// 具有“模块链接”的变量
export int NumEnemiesKilled = 0;
 
////////////
// stats.ixx
////////////
 
export module stats;
 
import :globals;
 
export void CountEnemyKilled()
{
    // 指的是与statsglobal.ixx中的相同变量
    NumEnemiesKilled++;
}
 
export int GetNumEnemiesKilled()
{
    // 指的是与statsglobal.ixx中的相同变量
    return NumEnemiesKilled;
}
 
///////////
// main.cpp
///////////
 
import stats;
 
DebugLog(GetNumEnemiesKilled()); // 0
CountEnemyKilled();
DebugLog(GetNumEnemiesKilled()); // 1
 
// 引用与statsglobal.ixx中相同的变量
DebugLog(NumEnemiesKilled); // 1
```

## Compatibility  兼容性

鉴于 C++ 已有 40+ 年的历史 ，新的构建系统必须与旧的构建系统兼容。我们有大量现有的头文件需要与模块一起使用。
</br>值得庆幸的是，C++ 提供了一个新的预处理器指令来做到这一点：

```c++
import "mylibrary.h";
// ...or...
import <mylibrary.h>;
```

尽管不以 `#` 开头，但末尾需要 `;`，但这实际上是一个预处理器指令。
</br>它与常规模块导入不同，因为它有双引号 （`“mylibrary.h”`） 或尖括号 （<`mylibrary.h>`），具体取决于所需的标题搜索规则。

该指令的作用是导出头文件中可导出的所有内容，就像我们在其源代码中添加 `export` 一样。
我们通常使用它来创建一个 “header unit”，将头文件包装在一个模块中：

```c++
////////////////
// mylibrary.ixx
////////////////
 
// 封装mylibrary.h的模块
export module mylibrary;
 
// 导出头文件中所有可导出的内容
import "mylibrary.h";
```

此 `import` 指令与 `#include` 和 import with a module 之间有几个关键区别。
首先，与 `#include` 相反，在 `import` 指令之前定义的预处理器符号对导入的头文件不可见：

```c++
//////////////
// mylibrary.h
//////////////
 
int ReadVersion()
{
    int version = ReadTextFileAsInteger("version.txt");
 
    #if ENABLE_LOGGING
        DebugLog("Version: ", version);
    #endif
 
    return version;
}
 
///////////
// main.cpp
///////////
 
#include "mylibrary.h"
int version = ReadVersion(); // 没有打印
 
// ...相当于...
 
int ReadVersion()
{
    int version = ReadTextFileAsInteger("version.txt");
 
    #if ENABLE_LOGGING // 注意：未定义
        DebugLog("Version: ", version);
    #endif
 
    return version;
}
int version = ReadVersion();
 
/////////////////
// mainlogged.cpp
/////////////////
 
// 在#include之前定义一个预处理符号
#define ENABLE_LOGGING 1
 
#include "mylibrary.h"
int version = ReadVersion(); // 有打印
 
// ...相当于...
 
#define ENABLE_LOGGING 1
 
int ReadVersion()
{
    int version = ReadTextFileAsInteger("version.txt");
 
    #if ENABLE_LOGGING // 注意：已定义
        DebugLog("Version: ", version);
    #endif
 
    return version;
}
 
int version = ReadVersion();
```

C++ 提供了一种解决此限制的工具。我们可以在命名的 module 之前使用 `module`;，并在这两个语句之间放置预处理器指令。
</br>这里的所有内容都将成为 “global module” 的一部分，可以从模块内部访问：

```c++
///////////////
// metadata.ixx
///////////////
 
// 没有模块名称表示“全局模块”
module;
 
// 在#include之前定义一个预处理符号
// 此部分只允许使用预处理符号
#define ENABLE_LOGGING 1
 
// 使用#include代替导入指令
#include "mylibrary.h"
 
// 我们的命名模块
export module metadata;
 
// 从头文件中导出函数
export int ReadVersion();
 
///////////
// main.cpp
///////////
 
// 正常使用该模块
import metadata;
DebugLog(ReadVersion()); // 6
```

`import` 指令和使用 module 的 `import` 之间的第二个区别是导出头文件中的预处理器宏：

```c++
///////////////
// legacymath.h
///////////////
 
// 在头文件中定义的宏
#define PI 3.14
 
///////////
// math.ixx
///////////
 
export module math;
 
// 导入指令暴露了PI宏
import "legacymath.h";
 
export double GetCircumference(double radius)
{
    // 导入指令中的宏是可以使用的
    return 2.0 * PI * radius;
}
 
///////////
// main.cpp
///////////
 
import math;
 
// OK
DebugLog(GetCircumference(10.0));
 
// 编译器错误：从导入指令导入的宏没有被导出
DebugLog(PI);
```

请注意 `PI` 宏如何可用于使用 `import` 指令的标头单元，但不能用于该模块的用户。这可以防止宏在整个程序中传递 “泄漏”。

## Conclusion  结论

C++20 的新模块构建系统比其自己的旧头文件和 `#include` 更类似于 C#。在 C++ 术语中，C# 在某种程度上将命名空间和模块混合在一起。
</br>我们编写命名空间的名称（ 使用 `Math;`）以便访问其内容。
</br>C++ 将这两个功能分开。我们可以编写 `import math;` 而 `math` 不是一个命名空间。我们可以在模块之上对命名空间进行分层，甚至可以导出它们。

C# 支持通过在每个文件中添加命名空间的一个成员来将代码拆分到多个文件中。
</br>在 C++ 中也可以实现同样的作，但我们也可以通过在单个文件中添加多个成员并从实现中拆分接口来更进一步。
</br>分区和片段是灵活的工具，允许我们将大型模块细分到许多源文件中。

作为最近才标准化的 C++20 功能，在撰写本文时，模块并不常用。
</br>然而，它们注定最终会成为占主导地位的构建系统，并将对头文件的许多改进带到绝大多数代码库中。
</br>同时，我们有诸如新的 `import “header.h”` 指令之类的工具，并可以访问 `global` 模块来简化过渡。
</br>使用模块的新代码可以使用这些工具将遗留代码打包到模块中，就像从一开始就以这种方式编写一样。 旧代码可以继续使用头文件。























