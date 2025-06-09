# C++ For C# Developers: Part 4 – Functions

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5553),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天的文章继续介绍通过介绍C++的构建模型，该模型与C#非常不同。
</br>我们将深入探讨预处理、编译、链接、头文件、单定义规则以及许多其他方面，了解我们的源代码是如何构建成可执行文件的。

## Compiling and Linking  编译和链接

在C#中，我们将所有源代码文件（`.cs`）编译成一个程序集，例如可执行文件（`.exe`）或库（`.dll`）。

在C++中，我们将所有的翻译单元（具有`.cpp`、`.cxx`、`.cc`、`.C`或`.c++`扩展名的源代码文件）编译成目标文件（`.obj`或`.o`），
</br>然后将它们链接在一起生成可执行文件（`app.exe`或`app`）、静态库（`.lib`或`.a`）或动态库（`.dll`或`.so`）。


如果任何源代码文件发生了变化，我们就重新编译它们以生成新的目标文件，然后使用所有未更改的目标文件运行链接器。

这个模型引发了一些问题。首先，什么是目标文件？它被称为“中间”文件，因为它既不是源代码，也不是像可执行文件那样的输出文件。
</br>C++语言标准没有说明这个文件的格式。实际上，它是一个特定编译器特定版本的特定设置下的二进制文件。如果编译器、版本或设置发生变化，所有代码都需要重新构建。

其次，静态库和动态库之间有什么区别？ 
- 动态库： 动态库与C#中的动态库非常相似。它是一个机器码库，就像可执行文件一样。它可以在运行时由可执行文件或其他动态库加载和卸载。
- 静态库： 静态库只能在编译时加载，并且永远不能卸载。因此，它更像是一个普通的对象文件

因为静态库在构建时可用，链接器会直接将它们构建到生成的可执行文件中。
</br>这意味着不需要向最终用户分发单独的动态库文件，不需要从文件系统单独打开它，也没有通过设置`LD_LIBRARY_PATH`环境变量等来覆盖其位置的可能性。

对于性能来说至关重要，静态库中的所有函数调用都是普通的函数调用。
</br>这意味着在动态库加载时，不需要通过在运行时设置的指针进行间接调用。这也意味着链接器可以执行“链接时间优化”，例如内联这些函数。

主要缺点源于需要在编译时存在静态库。这使得它们不适合加载用户创建的插件等任务。
</br>也许对于大型项目来说最重要的是，即使只是更改了一个小的源文件，也必须在每次构建时链接它们。链接时间会成比例增长，可能会阻碍快速迭代。
</br>因此，有时在开发构建中会使用动态库，而在发布构建中使用静态库。

在这个系列中，我们不会讨论如何运行编译器和链接器的具体细节。这很大程度上取决于所使用的特定编译器、操作系统和游戏引擎。
</br>通常，游戏引擎或控制台供应商会为此提供文档。另外，通常的做法是使用像Microsoft Visual Studio或Xcode这样的集成开发环境（IDE），它提供了一个“项目”抽象，用于管理源代码文件、编译器设置等。

## Header Files and the Preprocessor  头文件和预处理器

在C#中，我们添加`using`指令来引用其他文件中的代码。
C++在C++20中添加了类似的“模块(module)”系统(Part35)，我们将在本系列的后续文章中介绍。现在，我们将假装它不存在，只讨论C++的传统构建方式。

头文件（`.h`、`.hpp`、`.hxx`、`.hh`、`.H`、`.h++` 或无扩展名）是单文件中代码引用另一文件代码最常见的方式。
</br>这些文件仅仅是打算复制粘贴到另一个C++源代码文件中的C++源代码文件。复制粘贴操作由预处理器执行。

就像在C#中一样，预处理指令如`#if`在编译的主要阶段之前被评估。不需要调用一个单独的预处理程序来生成编译器接收的中间文件。预处理只是编译器的一个早期步骤。

C++使用一个名为`#include`的预处理器指令，将头文件的内容复制粘贴到另一个头文件（`.h`）或翻译单元（`.cpp`）中。下面是它的样子：

```c++
// math.h
int Add(int a, int b);
 
 
// math.cpp
#include "math.h"
int Add(int a, int b)
{
    return a + b;
}
```

`#include "math.h"` 告诉预处理器在 `math.cpp` 所在的目录中搜索名为 `math.h` 的文件。
如果找到了这样的文件，它会读取其内容并将 `#include` 指令替换为它们。否则，它会搜索它配置的“包含路径”。C++ 标准库会隐式搜索。
如果在这些位置中找不到 `math.h`，编译器将产生错误。

> 这里简单的补充一下使用#include <> 和 #include ""
> - #include <>
>   </br>只会在系统目录中查找头文件
> - #include ""
>   </br>先在当前源文件所在目录查找，再去到系统目录中查找

之后，`math.cpp` 如下所示：

```c++
int Add(int a, int b);
int Add(int a, int b)
{
    return a + b;
}
```

第一个Add是一个函数声明，第二个是函数定义。由于签名匹配，编译器知道我们正在定义先前的声明。

```c++
// user.cpp
#include "math.h"
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
```

这例子展示了`user.cpp`如何添加相同的`#include`

```c++
int Add(int a, int b);
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
```

现在，编译器将遇到 `Add` 的声明，并且可以接受 `AddThree` 调用它，即使还没有 `Add` 的定义。它只是在输出 （`user.obj`） 的对象文件中记下 `Add` 是一个未满足的依赖项。

当链接器执行时，它会读取`user.obj`和`math.obj`。`math.obj`包含`Add`的定义，而`user.obj`包含`AddThree`的定义。在那个时刻，链接器真正需要`Add`的定义，因此它使用了在`math.obj`中找到的定义。

常见的一个`#include`的替代版本：

```c++
#include <math.h>
```

这个版本旨在仅搜索C++标准库以及其他由编译器提供的头文件。
</br>例如，Microsoft Visual Studio允许使用`#include <windows.h>`来调用`Windows`操作系统。这对于区分应用代码库和由编译器提供的文件名是有用的。
</br>想象一下这个程序：

```c++
#include "math.h"
bool IsNearlyZero(float val)
{
    return fabsf(val) < 0.000001f;
}
```

`fabsf` 是 `C` 标准库中的一个函数，用于取浮点数的绝对值。当预处理程序以引号版本的 `#include` 运行时，它会找到我们的 `math.h`，所以我们得到这个：

```c++
int Add(int a, int b);
 
bool IsNearlyZero(float val)
{
    return fabsf(val) < 0.000001f;
}
```

然后编译器找不到 `fabsf`，所以它出错了。相反，我们应该使用 `#include` 的尖括号版本，因为我们正在寻找编译器提供的 `math.h`：

```c++
#include <math.h>
bool IsNearlyZero(float val)
{
    return fabsf(val) < 0.000001f;
}
```

这会产生我们想要的：

```c++
float fabsf(float arg);
// ...以及许多许多更多的数学函数声明...
 
bool IsNearlyZero(float val)
{
    return fabsf(val) < 0.000001f;
}
```

另请注意，我们可以在 `#include` 中指定与目录结构相对应的路径：

```c++
#include "utils/math.h"
#include <nlohmann/json.hpp>
```

最后，虽然它很深奥，而且通常最好避免，但没有什么能阻止我们使用 `#include` 来拉取非头文件。只要结果是合法的 C++，我们就可以 `#include` 任何文件。
</br>有时 `#include` 甚至被放置在函数的中间以填充其主体的一部分！

## ODR and Include Guards  ODR 和 包含保护

C++ 具有所谓的“一个定义规则(one definition rule)”，通常缩写为 ODR。
</br>这表示翻译单元中可能只有一个定义。这包括变量和函数，随着代码库的增长，这会给我们带来一些问题。想象一下，我们扩展了数学库并在其上添加了一个向量数学库：

```c++
// math.h
int Add(int a, int b);
float PI = 3.14f;
 
 
// vector.h
#include "math.h"
float Dot(float aX, float aY, float bX, float bY);
 
 
// user.cpp
#include "math.h"
#include "vector.h"
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

这里我们看到`vector.h`使用`#include`来包含`math.h`。同时，`user.cpp`也使用`#include`来包含`vector.h`和`math.h`。
</br>这是一种良好的做法，因为它避免了对 `math.h` 的隐式依赖关系，如果更改 `vector.h` 以删除 `#include “math.h”`，该依赖关系就会中断。
</br>尽管如此，我们即将看到这带来了一个问题。让我们看看预处理器替换 `#include “math.h”` 指令后 `user.cpp`：

```c++
int Add(int a, int b);
float PI = 3.14f;
#include "vector.h"
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

现在编译器替换了`#include "vector.h"`

```c++
int Add(int a, int b);
float PI = 3.14f;
#include "math.h"
float Dot(float aX, float aY, float bX, float bY);
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

最后，它替换了从它复制进来的`vector.h`内容中的`#include "math.h"`：

```c++
int Add(int a, int b);
float PI = 3.14f;
int Add(int a, int b);
float PI = 3.14f;
float Dot(float aX, float aY, float bX, float bY);
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

多次声明`Add`函数是可以的，因为它们不是定义，所以不会违反ODR。编译器会简单地忽略重复的声明。

另一方面，PI 的定义绝对是一个定义。同一个变量名有两个定义违反了 ODR，因此我们会得到编译器错误。

为了解决这个问题，我们在头文件中添加了一个称为“包含保护(include guard)”的东西。
</br>这可以有两种基本形式，但两者都使用了预处理器。
</br>这是在math.h中的第一种形式：

```c++
#if (!defined MATH_H)
#define MATH_H
 
int Add(int a, int b);
float PI = 3.14f;
 
#endif
```

这使用了`#if`、`#define`和`#endif`指令，这些指令与C#中的对应指令类似。在这种情况下，唯一的真正区别是C++中使用`!defined MATH_H`，而不是C#中的`!MATH_H`。

这种方法的另一种变体是使用仅适用于C++的`#ifndef MATH_H`，作为`#if (!defined MATH_H)`的一种简写：

```c++
#ifndef MATH_H
#define MATH_H
 
int Add(int a, int b);
float PI = 3.14f;
 
#endif
```

无论哪种情况，我们都会选择一种命名约定，并将其应用于文件名，以生成文件的唯一标识符。这包括许多流行的形式，例如这些：

```c++
math_h
MATH_H
MATH_H_
MYGAME_MATH_H
```

为了避免需要想出独特的名称，所有常见的编译器都提供了非标准的`#pragma once`指令：

```c++
#pragma once
 
int Add(int a, int b);
float PI = 3.14f;
```

无论选择哪种形式，让我们看看这是如何帮助避免ODR违规的。以下是所有`#include`指令解析后`user.cpp`的样子：（为了清晰，增加了缩进）

```c++
#ifndef MATH_H
#define MATH_H
 
    int Add(int a, int b);
    float PI = 3.14f;
 
#endif
 
#ifndef VECTOR_H
#define VECTOR_H
 
    #ifndef MATH_H
    #define MATH_H
 
        int Add(int a, int b);
        float PI = 3.14f;
 
    #endif
 
    float Dot(float aX, float aY, float bX, float bY);
 
#endif
 
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

在第一行 （`#ifndef MATH_H`） 上，预处理器发现 `MATH_H` 未定义，因此它会保留所有代码，直到 `#endif`。这包括一个 `#define MATH_H`，所以现在它被定义出来了。

同样，`#ifndef VECTOR_H` 成功并允许定义 `VECTOR_H`。但是，嵌套 `#ifndef MATH_H` 失败，因为 `MATH_H` 现已定义。在匹配的 `#endif` 之前的所有内容都被剥离。

最后，我们得到了这个结果：

```c++
int Add(int a, int b);
float PI = 3.14f;
 
float Dot(float aX, float aY, float bX, float bY);
 
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
```

## Inline  内嵌

即使ODR编译器错误已经修复，我们仍然存在问题：链接错误。原因是`vector.cpp`的翻译单元也包含`PI`的一个副本。以下是原始代码的样式：

```c++
#include "vector.h"
 
float Dot(float aX, float aY, float bX, float bY)
{
    return Add(aX*bX, aY+bY);
}
```

这是在预处理器解析 `#include` 指令之后：

```c++
#ifndef VECTOR_H
#define VECTOR_H
 
    #ifndef MATH_H
    #define MATH_H
 
        int Add(int a, int b);
        float PI = 3.14f;
 
    #endif
 
    float Dot(float aX, float aY, float bX, float bY);
 
#endif
 
float Dot(float aX, float aY, float bX, float bY)
{
    return Add(aX*bX, aY+bY);
}
```

请记住，每个翻译单元都是单独编译的。 在此翻译单元中，`MATH_H` 和 `VECTOR_H` 未像在 `user.cpp` 翻译单元中那样使用 `#define` 进行设置。
所以两个包含保护都成功了，我们得到这个：

```c++
int Add(int a, int b);
float PI = 3.14f;
 
float Dot(float aX, float aY, float bX, float bY);
 
float Dot(float aX, float aY, float bX, float bY)
{
    return Add(aX*bX, aY+bY);
}
```

这对于编译此翻译单元非常有用，因为没有违反 ODR 的重复定义。编译将成功，但链接将失败。

链接错误的原因是，默认情况下，在链接时我们也不能有 `PI` 的重复定义。如果我们想这样做，我们需要将内联关键字添加到 `PI` 中，以告诉编译器应该允许多个定义。
</br>这将导致以下这些翻译单元：

```c++
// user.cpp
int Add(int a, int b);
inline float PI = 3.14f;
 
float Dot(float aX, float aY, float bX, float bY);
 
int AddThree(int a, int b, int c)
{
    return Add(a, Add(b, c));
}
bool IsOrthogonal(float aX, float aY, float bX, float bY)
{
    return Dot(aX, aY, bX, bY) == 0.0f;
}
 
 
// vector.cpp
int Add(int a, int b);
inline float PI = 3.14f;
 
float Dot(float aX, float aY, float bX, float bY);
 
float Dot(float aX, float aY, float bX, float bY)
{
    return Add(aX*bX, aY+bY);
}
```

看起来很奇怪，`inline` 关键字被应用于变量。
</br>历史原因是因为它最初是给编译器的一个提示，让它内联函数，但就像 `register` 关键字一样，这并不是强制性的，几乎总是被忽略。
</br>现在它意味着“允许有多个定义”，因此现在可以应用于变量和函数。

例如，只要它是内联的，我们就可以将函数定义添加到 `math.h` 中：


```c++
inline int Sub(int a, int b)
{
    return a - b;
}
```

不过，这通常是避免的，因为对函数的任何更改都需要直接或间接地重新编译包含它的所有翻译单元，这在大型代码库中可能需要相当长的时间。

## Linkage  联动

最后，对于今天，C++ 有了 “linkage” 的概念。默认情况下，`PI` 等变量具有外部链接。这意味着它可以被其他翻译单元引用。
</br>例如，假设我们向 `math.cpp` 添加了一个变量：

```c++
float SQRT2 = 1.4f;
```

现在假设我们想从 `user.cpp` 中引用它。`#include “math.h”` 不起作用，因为 `SQRT2` 在 `math.cpp` 中，而不是在 `math.h` 中。
</br>我们仍然可以使用 `extern` 关键字来引用它：

```c++
extern float SQRT2;
 
float GetDiagonalOfSquare(float widthOrHeight)
{
    return SQRT2 * widthOrHeight;
}
```

这类似于函数声明，因为我们告诉编译器信任我们，并假装存在一个名为 `SQRT2` 的浮点数 。
</br>因此，当它编译 `user.cpp` 时，它会在 `user.obj` 对象文件中注明我们还没有满足 `SQRT2` 的依赖项。
</br>当编译器编译 `math.cpp` 时，它会记下有一个名为 `SQRT2` 的浮点数可用于链接。

稍后，链接器将运行并读取 `user.obj` 以及所有其他对象文件（包括 `math.obj`）。
</br>在处理 `user.obj` 时，它会从编译器读取该注释，指出缺少 `SQRT2` 的定义，并浏览其他目标文件以查找它。
</br>它在 `math.obj` 中找到一个注释，指出有一个名为 `SQRT2` 的浮点数，因此链接器使 `GetDiagonalOfSquare` 引用该变量。

快速提示：`extern` 关键字也可以应用于 `math.cpp` 中，但由于外部链接是默认的，所以这没有效果。不过，下面是它的样子：

```c++
extern float SQRT2 = 1.4f;
```

防止这种行为的一种方法是将`static`关键字添加到`SQRT2`中。
</br>这将改变链接为“内部(internal)”，并防止编译器在`math.obj`中添加注释，表示名为`SQRT2`的浮点变量可用于链接。

现在，如果我们尝试链接 `user.obj` 和 `math.obj`，链接器在任何对象文件中都找不到 `SQRT2` 的任何可用定义，因此会产生错误。

`extern` 和 `static` 也可以与函数一起使用。例如：
```c++
// math.cpp
int Sub(int a, int b)
{
    return a - b;
}
static int Mul(int a, int b)
{
    return a * b;
}
 
 
// user.cpp
extern int Sub(int a, int b);
 
int SubThree(int a, int b, int c)
{
    return Sub(Sub(a, b), c);
}
 
extern int Mul(int a, int b); // 编译器错误：Mul 是 `静态` 的
```

## Conclusion  结论

今天我们看到了C++在构建源代码方面非常不同的方法。‘编译后链接’的方法与头文件相结合，对ODR（链接时解析规则）、链接和包含保护器产生了连锁反应。
</br>在系列的后续部分，我们将探讨C++20的模块系统，该系统解决了许多这些问题，并导致构建模型更加类似于C#。
</br>即使有模块，头文件仍然非常相关。关于ODR和链接还有很多细节需要深入探讨，但我们将随着介绍更多语言概念，如模板和线程局部变量，逐步覆盖这些内容。






