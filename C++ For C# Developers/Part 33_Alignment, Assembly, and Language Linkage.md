# C++ For C# Developers: Part 33 – Alignment, Assembly, and Language Linkage

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6169),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天我们将探讨C++中的一些底层概念。这些是在性能至关重要且互操作性至关重要的时刻从工具箱中拿出的工具。
</br>继续阅读，了解C++的逃生机制，并精细控制内存！

## Alignof 对齐要求

让我们从一个简单的运算符开始：`alignof`。我们指定一个类型，它将评估为表示类型的对齐要求的字节数的`std::size_t`：

```c++
struct EmptyStruct
{
};
 
struct Vector3
{
    float X;
    float Y;
    float Z;
};
 
// 在x64 macOS上使用Clang编译器的示例
DebugLog(alignof(char)); // 1
DebugLog(alignof(int)); // 4
DebugLog(alignof(bool)); // 1
DebugLog(alignof(int*)); // 8
DebugLog(alignof(EmptyStruct)); // 1
DebugLog(alignof(Vector3)); // 4
DebugLog(alignof(int[100])); // 4
```

因为C++中所有类型的对齐要求在编译时已知，所以`alignof`运算符在编译时进行评估。
</br>这意味着上述代码编译后生成的机器代码与如果我们记录常量时的机器代码相同：

```c++
DebugLog(1);
DebugLog(4);
DebugLog(1);
DebugLog(8);
DebugLog(1);
DebugLog(4);
DebugLog(4);
```

请注意，当使用`alignof`与数组一起使用时，就像我们在`int[100]`中使用的那样，我们得到数组元素类型的对齐方式。
</br>这意味着我们得到4，而不是`int*`的8，尽管在C++中数组和字符串(Part7)非常相似。

## Alignas  对齐

接下来，我们有`alignas`指定符。这个指定符应用于类、数据成员和变量，以控制它们的对齐方式。
</br>除了位域(Part10)、参数或在`catch`子句(Part18)中的变量之外，支持所有类型的变量。例如，如果我们想将结构体对齐到16字节边界：

```c++
// 使用默认对齐方式
struct Vector3
{
    float X;
    float Y;
    float Z;
};
 
// 将对齐方式更改为16字节
struct alignas(16) AlignedVector3
{
    float X;
    float Y;
    float Z;
};
 
DebugLog(alignof(Vector3)); // 4
DebugLog(alignof(AlignedVector3)); // 16
```

我们不允许降低对齐要求，因为生成的代码在CPU上无法工作。如果我们尝试，将会得到编译器错误：

```c++
// 编译器错误：请求的对齐（1）低于默认值（4）
struct alignas(1) AlignedVector3
{
    float X;
    float Y;
    float Z;
};
```

同样，无效的对齐也会产生编译错误。有效的对齐取决于正在编译的CPU架构，但通常需要2的幂次方。

```c++
// 编译器错误：请求的对齐（3）无效
struct alignas(3) AlignedVector3
{
    float X;
    float Y;
    float Z;
};
```

将`0`对齐将被简单地忽略：

```c++
// OK, 但请求的对齐（0）被忽略
struct alignas(0) AlignedVector3
{
    float X;
    float Y;
    float Z;
};
 
DebugLog(alignof(AlignedVector3)); // 4
```

作为一个缩写，我们还可以使用`alignas(type)`。这相当于`alignas(alignof(type))`，当我们需要对齐与另一个类型的对齐相匹配时很有用：

```c++
struct AlignedToDouble
{
    double Double;
 
    // 每个数据成员的对齐方式与double类型相同
    alignas(double) float Float;
    alignas(double) uint16_t Short;
    alignas(double) uint8_t Byte;
};
 
// 结构占用32字节是因为对齐要求
DebugLog(sizeof(AlignedToDouble)); // 32
 
// 打印数据成员之间的距离以查看8字节对齐
AlignedToDouble atd;
DebugLog((char*)&atd.Float - (char*)&atd.Double); // 8
DebugLog((char*)&atd.Short - (char*)&atd.Double); // 16
DebugLog((char*)&atd.Byte - (char*)&atd.Double); // 24
```

虽然罕见，但如果指定了多个`alignas`，则使用最大的值：

```c++
struct Aligned
{
    // 16是最大的，所以它被用作对齐
    alignas(4) alignas(8) alignas(16) int First = 123;
 
    alignas(16) int Second = 456;
};
 
DebugLog(sizeof(Aligned)); // 32
 
Aligned a;
DebugLog((char*)&a.Second - (char*)&a.First); // 16
```

这导致`alignas`的第三种形式，其中我们传递一个模板参数包(Part28)而不是整数或类型。
</br>在这种情况下，就像我们为参数包的每个元素指定了一个`alignas`一样，因此选择了最大的值：

```c++
template<int... Alignments>
struct Aligned
{
    alignas(Alignments...) int First = 123;
    alignas(16) int Second = 456;
};
 
DebugLog(sizeof(Aligned<1, 2, 4, 8, 16>)); // 32
 
Aligned<1, 2, 4, 8, 16> a;
DebugLog((char*)&a.Second - (char*)&a.First); // 16
```

## Assembly  汇编

C++ 允许我们嵌入汇编代码。这称为 “内联汇编(inline assembly)”，其含义与编译器和被编译的 CPU 高度特定。
</br>C++ 语言标准所说的只是我们编写 `asm（“源代码”）`， 其余的由编译器决定。
</br>例如，下面是一些内联程序集，它在 x86 上从 20 中减去 5，由 Clang 在 macOS 上编译

```c++
int difference = 0;
asm(
    "movl $20, %%eax;" // Put 20 in the eax register
    "movl $5, %%ebx;" // Put 5 in the ebx register
    "subl %%ebx, %%eax ":"=a"(difference)); // difference = eax - ebx
DebugLog(difference); // 15
```

此外，编译器特定的还包括汇编代码与周围代码的交互方式。在这种情况下，Clang 允许我们编写 `:='a'(difference)` 来引用`asm`语句内部的差值局部变量。

每个编译器都会对内联汇编代码施加自己的限制。这包括是否使用Intel或AT&T汇编语法，C++代码如何与内联汇编交互，以及当然支持的CPU架构指令集。

所有这些不一致性导致大多数内联汇编的使用被摒弃，转而采用所谓的“内联函数”。
</br>这些函数被替换为单个CPU指令。它们几乎总是以该CPU指令命名，接受该CPU指令操作的参数，并计算为CPU指令的结果。
</br>这其中的含义有很多变化，但这是在C++程序中嵌入汇编的一种更简单、更自然的方式：

```c++
// x86 SSE intrinsics
#include <xmmintrin.h>
 
// Component-wise addition of four floats in two arrays into a third array
void Add4(const float* a, const float* b, float* c)
{
    // Load a's four floats from memory into a 128-bit register
    __m128 reg1 = _mm_load_ps(a);
 
    // Load b's four floats from memory into a 128-bit register
    const auto reg2 = _mm_load_ps(b);
 
    // Add corresponding floats of a and b into the first 128-bit register
    reg1 = _mm_add_ps(reg1, reg2);
 
    // Store the result register into c's memory
    _mm_store_ps(c, reg1);
}
 
float a[] = { 1, 1, 1, 1 };
float b[] = { 1, 2, 3, 4 };
float c[] = { 9, 9, 9, 9 };
 
Add4(a, b, c);
 
DebugLog(a[0], a[1], a[2], a[3]); // 1, 1, 1, 1 (unmodified)
DebugLog(b[0], b[1], b[2], b[3]); // 1, 2, 3, 4 (unmodified)
DebugLog(c[0], c[1], c[2], c[3]); // 2, 3, 4, 5 (sum)
```

这种方法有几个优点。不需要指定特定的寄存器名称，因为编译器的寄存器分配器只需执行其正常工作。
</br>我们可以使用正常的C++约定，如参数、返回值、const变量，甚至auto类型。这些变量是强类型的，这意味着我们可以得到我们习惯的编译器错误检查：

```c++
// Compiler error: too many arguments
__m128 reg1 = _mm_load_ps(a, b);
 
// Compiler error: return value is __m128, not bool
bool reg2 = _mm_load_ps(b);
```

> 这里读者没学过汇编，代码内的注释就不修改了

## Language Linkage  语言链接

当链接器将目标文件链接在一起时，它们遵循相同的约定是很重要的。
</br>通常情况下，这不会成为问题，因为我们链接的是由相同语言（C++）的源代码编译的目标文件，这些源代码是由相同版本的同一种编译器使用相同的编译器设置编译的。

在其他情况下，我们希望链接不同编译方式的代码。一个常见的场景是将C++和C代码链接在一起，例如当C++代码使用C库或反之亦然时。
</br>在这种情况下，由于语言有不同的目标文件约定，这会导致它们冲突。

以C++中重载函数的情况为例。
</br>C不支持这些，因此C的目标文件简单地按源代码中的名称命名函数。
</br>C++需要消除歧义，因此它“混淆”名称以使它们独特。
</br>即使只有一个函数重载，它也会这样做：

```c++
////////////////////
// library.h (C++)
////////////////////
 
int Madd(int a, int b, int c);
 
////////////////////
// library.cpp (C++)
////////////////////
 
#include "library.h"
 
// 编译成名为Maddiii_i的对象文件
// 仅为例子名称。实际名称基本上是不可预测的。
int Madd(int a, int b, int c)
{
    return a*b + c;
}
 
////////////////////
// main.c (C)
////////////////////
 
#include "library.h"
 
void Foo()
{
    // OK: library.h 声明了一个Madd函数，该函数接收三个int类型的参数并返回一个int类型的值
    int result = Madd(2, 4, 6);
 
    // 打印结果
    printf("%d\n", result);
}
```

`library.cpp` 和 `main.c` 都可以编译，但接受 `library.o` 和 `main.o` 的链接器无法将它们链接在一起。
问题是 `main.o` 正在尝试查找名为 `Madd` 的函数，但是没有找到。虽然有一个名为`Maddiii_i` 的函数，但这不算数，因为只有完全匹配的名称才会被匹配。

为了解决这个问题，C++提供了一种方法来告诉编译器代码应该使用与C相同的语言链接规则：

```c++
////////////////////
// library.h (C++)
////////////////////
 
// 此块中的所有内容应使用C的链接规则进行编译
extern "C"
{
    int Madd(int a, int b, int c);
}
 
////////////////////
// library.cpp (C++)
////////////////////
 
#include "library.h"
 
//定义需要与它们的声明语言链接相匹配
extern "C"
{
    // 编译成名为Madd的对象文件
    // 未混淆成Maddiii_i
    int Madd(int a, int b, int c)
    {
        return a*b + c;
    }
}
```

现在`Madd`没有名称混淆，链接器可以找到它并生成一个可工作的可执行文件。

对于已经切换到C语言链接的代码，有一些特殊的规则适用。

首先，类成员始终具有C++链接，无论是否指定了C链接。

其次，因为C不支持函数重载，任何同名函数都被认为是同一个函数。这意味着当我们尝试创建重载时，通常会得到编译器错误，因为我们在重新定义同一个函数。

第三，同样地，不同命名空间中具有相同名称的变量会被编译器视为同一个变量。这是因为C不支持命名空间。尝试重新定义这些变量时，我们通常会得到相同的编译器错误。

第四点，同样地，即使变量和函数在不同的命名空间中，它们也不能有相同的名称。所有这些规则都源于C语言要求每个实体都必须有唯一名称的要求。

如果只有一个实体需要更改其语言链接，则可以省略花括号，就像它们在单语句`if`块中是可选的一样。然而，这并不会像其他花括号那样创建一个块作用域：

```c++
extern "C" int Madd(int a, int b, int c);
 
extern "C" int Madd(int a, int b, int c)
{
    return a*b + c;
}
```

C++语言保证支持两种语言的链接：C和C++。编译器可以自由地实现支持更多语言。默认的语言链接当然是C++，但如果需要，也可以明确指定：

```c++
extern "C++" int Madd(int a, int b, int c);
 
extern "C++" int Madd(int a, int b, int c)
{
    return a*b + c;
}
```

链接规则可以嵌套。在这种情况下，使用最内层的链接。

```c++
// 将链接更改为C
extern "C"
{
    // 将链接更改为C++
    extern "C++" int Madd(int a, int b, int c);
}
 
// 链接是C++
extern "C++" int Madd(int a, int b, int c)
{
    return a*b + c;
}
```

最后，一个常见的做法是使用预处理器检查(Part24) `__cplusplus`，以确定代码是否被编译为 C++。
</br>作为回应，使用 C++ 语言链接。这允许代码被编译为 C++ 库，可以与 C 代码链接。
</br>当没有 C++ 编译器时，代码也可以直接编译为 C。这种方法要求代码只使用两种语言都合法的子集：

```c++
////////////////////
// library.h
////////////////////
 
// 如果以C++编译，则此为定义
#ifdef __cplusplus
    // 创建一个名为EXPORTED的宏，用于设置C语言链接
    #define EXPORTED extern "C"
// 如果编译为C（由于不是C++，所以假设如此）
#else
    // 创建一个名为EXPORTED的空宏
    #define EXPORTED
#endif
 
// 在开头添加 EXPORTED
// 对于C++，这会将语言链接设置为C
// 对于C，这不起作用
EXPORTED int Madd(int a, int b, int c);
 
////////////////////
// library.c
////////////////////
 
#include "library.h"
 
// 无论语言如何，编译成名为Madd的对象文件
EXPORTED int Madd(int a, int b, int c)
{
    return a*b + c;
}
```

## Conclusion  结论

除了C++为我们提供的众多底层控制之外，这些特性还为我们提供了更多的控制。
</br>我们可以查询和设置各种数据类型和变量的对齐方式，以优化特定CPU架构的要求，从而以多种方式提高性能。
</br>C#对结构体字段布局提供了一些控制，但与C++中的`alignas`相比，这只是一个功能远为有限的工具。

利用对对齐的控制的一种方法是通过编写内联汇编，这是C++的另一个特性，或者通过使用CPU特定的内省。
</br>这些特性结合在一起，可以精确控制实际执行的机器代码，而无需编写整个程序使用汇编语言。
</br>从.NET Core 3.0开始，C#开始提供内省，首先是针对[x86](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86?view=net-5.0)的，以及[Unity 的特定于 Burst 的内部函数](https://docs.unity3d.com/Packages/com.unity.burst@1.5/manual/docs/CSharpLanguageSupport_BurstIntrinsics.html)。

C++还允许与它的前辈C保持高度兼容。尽管C++拥有更多功能，但通过设置语言链接模式并遵循一些特殊规则，C++代码可以轻松与C代码集成。
</br>这使得我们的C++库可以在C以及遵循C链接规则的环境中使用。
</br>其中有很多这样的环境，包括C#、Rust、Python和JavaScript（通过Node.js）的语言绑定。对于C#来说，它的P/Invoke语言绑定系统也允许与C链接模型进行互操作。


