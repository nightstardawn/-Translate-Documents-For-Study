# C++ For C# Developers: Part 12 – Constructors and Destructors

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5675),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

到目前为止，我们已经介绍了结构体、 成员函数和重载运算符 。现在让我们谈谈它们生命周期的主要部分：构造函数和析构函数。析构函数尤其与 C# 截然不同，它代表了 C++ 的一个标志性功能，它对我们编写和设计该语言代码的方式具有广泛的影响。继续阅读以了解有关它们的所有信息！

## General Constructors  通用构造函数

首先，我们今天不打算深入讨论实际调用这些构造函数中的任何一个。初始化是一个复杂的主题，需要有自己的完整文章。因此，我们将在本周的文章中编写构造函数，并在下一章节中使用它们。

基本 C++ 构造函数与 C# 中的构造函数非常相似，语法相同且含义相同！


> 内容回顾：</br>
> C#中的构造函数
> ```csharp
> struct Vector2
> {
>   float X;
>   float Y;
>   Vector2(float x, float y)
>   {
>       X = x;
>       Y = y;
>   }
> };
> ```

cpp中的构造函数
```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2(float x, float y)
    {
        X = x;
        Y = y;
    }
};
```

任何比这个简单示例更高级的东西在语言之间都会有很大的不同。首先，与成员函数一样，我们可以通过将定义放在结构体之外来将构造函数声明从定义中拆分出来。这通常用于将声明放在头文件 `（.h）` 中，将定义放在翻译单元 `（.cpp）` 中，以减少编译时间。

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2(float x, float y);
};
 
Vector2::Vector2(float x, float y)
{
    X = x;
    Y = y;
}
```

C++ 提供了一种在函数体运行之前初始化数据成员的方法。这些称为 “初始值设定项列表(initializer lists)”。它们位于构造函数的签名和其主体之间。

```c++
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    Ray2(float originX, float originY, float directionX, float directionY)
        : Origin(originX, originY), Direction{directionX, directionY}
    {
    }
};
```


初始值设定项列表以 `:` 开头，然后列出以逗号分隔的数据成员列表。每个函数的初始化参数位于括号 `（Origin（originX， originY））` 或大括号 `（ Direction{directionX, directionY} ）` 中。顺序无关紧要，因为始终使用数据成员在 struct 中声明的顺序。

我们还可以使用初始值设定项列表来初始化基础类型。以下是 `Vector2` 的替代版本，它可以做到这一点：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
```

初始值设定项列表将覆盖数据成员的默认初始值设定项。这意味着以下版本的 Vector2 将 x 和 y 参数初始化为构造函数参数，而不是 0：

```c++
struct Vector2
{
    float X = 0;
    float Y = 0;
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
```

初始值设定项列表还可用于调用另一个构造函数，这有助于减少代码重复和“帮助程序”函数（通常称为 `Init` 或 `Setup`）。这是 `Ray2` 中将原点默认为 `（0， 0）` 的函数：

```c++
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    Ray2(float originX, float originY, float directionX, float directionY)
        : Origin(originX, originY), Direction{directionX, directionY}
    {
    }
 
    Ray2(float directionX, float directionY)
        : Ray2(0, 0, directionX, directionY)
    {
    }
};
```

如果初始值设定项列表调用另一个构造函数，则它只能调用该构造函数。它也不能初始化数据成员：

```c++
Ray2(float directionX, float directionY)
    // Compiler error: constructor call must stand alone
    // 编译器错误：构造函数调用必须独立
    : Origin(0, 0), Ray2(0, 0, directionX, directionY)
{
}
```
> 简单理解：这个 `Ray2(float directionX, float directionY)` 的构造函数被当作一个委托扔给了自己的另一个构造函数。这个构造函数(委托目标)只能有一个。
> </br>这种情况下就不能进行初始化成员变量了>

## Default Constructors  默认构造函数

“默认构造函数”没有参数。在 C# 中，结构的默认构造函数始终可用，甚至不能由我们的代码定义。C++ 确实允许我们为结构体编写默认构造函数：

> 这个说法不准确，查询[资料](https://learn.microsoft.com/zh-cn/dotnet/csharp/whats-new/csharp-version-history#c-version-10)准确来说是C#10(.net6)之前不允许定义，C#10之后就可以显示定义，会直接覆盖默认的隐式构造函数.

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2()
    {
        X = 0;
        Y = 0;
    }
};
```

在 C# 中，此构造函数始终由编译器为所有结构生成。它只是将所有字段初始化为默认值，在本例中为 0。

```csharp
// C#
Vector2 vecA = new Vector2();    // 0, 0
Vector2 vecB = default(Vector2); // 0, 0
Vector2 vecC = default;          // 0, 0
```

C++ 编译器也为我们生成了一个默认构造函数。与 C# 一样，它也会将所有字段初始化为其默认值。

C++ 结构的行为方式也与 C# 类的行为相同：如果结构定义了构造函数，则编译器不会生成默认构造函数。这意味着此版本的 `Vector2` 不会获得编译器生成的默认构造函数：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
```

如果我们尝试创建此 Vector2 的实例而不提供两个 float 参数，我们将收到编译器错误：

```c++
// Compiler error: no default constructor so we need to provide two floats
// 编译器错误：没有默认构造函数，因此我们需要提供两个浮点数
Vector2 vec;
```

如果我们想找回默认构造函数，我们有两个选择。

方法一：我们可以自己定义它：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2()
    {
    }
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
```

方法二：我们可以使用 = default 告诉编译器为我们生成它：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2() = default;
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
```

我们也可以将 = default 放在结构体之外，通常在翻译单元（`.cpp` 文件中）：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2();
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
 
Vector2::Vector2() = default;
```

有时我们想做相反的事情，阻止编译器生成默认构造函数。通常我们通过编写自己的构造函数来实现这一点，但如果我们不想这样做，那么我们可以使用 `= delete`：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2() = delete;
};
```

这不能放在结构体之外：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2();
};
 
// Compiler error : Must be inside the struct
// 编译器错误 : 必须在结构体内部
Vector2::Vector2() = delete;
```

如果没有默认构造函数（由编译器生成或手动编写），则编译器也不会为具有该类型数据成员的结构生成默认构造函数：

```c++
// Compiler doesn't generate a default constructor
// because Vector2 doesn't have a default constructor
//编译器不会生成默认构造函数，因为Vector2没有默认构造函数
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
};
```

正如我们上面看到的，在为 Ray2 等类型编写构造函数时，初始化器列表特别有用。没有它们，我们会得到一个编译器错误：

```c++
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    Ray2(float originX, float originY, float directionX, float directionY)
        // 编译器错误
        // Origin和Direction没有默认构造函数
        // 需要调用 (float, float) 构造函数
        // 这需要在初始化列表中完成 
    {
        //没有 Vector2 对象来初始化
        // 它们需要在初始化列表中初始化
        Origin.X = originX;
        Origin.Y = originY;
        Origin.X = directionX;
        Origin.Y = directionY;
    }
};
```

使用初始值设定项列表，我们可以在构造函数主体运行之前调用非默认构造函数来初始化这些数据成员：

```c++
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    Ray2(float originX, float originY, float directionX, float directionY)
        : Origin(originX, originY), Direction{directionX, directionY}
    {
    }
};
```

> 这里的Vector2是定义了了[主构造函数](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/instance-constructors#primary-constructors)的


## Copy and Move Constructors 复制和移动构造函数

> 这个先简单的解释一下复制构造函数和移动构造函数:
> </br>**复制构造函数**：简单来说就是字面意思，复制创建一个新对象，是深拷贝。
> </br>**移动构造函数**：简单看成类似与C#中ref关键字的运用，但是需要转移地址所有权，主要是用于高效转移资源所有权（如文件句柄、非托管内存），避免不必要的复制。

复制构造函数是采用对相同类型结构的左值引用的构造函数。这通常是 `const` 引用。我们将在本系列的后面详细介绍 `const`，但现在可以将它视为 “只读”。

同样，移动构造函数采用对相同类型结构的右值引用。以下是 `Vector2` 中的所有四个：

```c++
struct Vector2
{
    float X;
    float Y;
 
    // Default constructor
    Vector2()
    {
        X = 0;
        Y = 0;
    }
 
    // Copy constructor
    Vector2(const Vector2& other)
    {
        X = other.X;
        Y = other.Y;
    }
 
    // Copy constructor (argument is not const)
    Vector2(Vector2& other)
    {
        X = other.X;
        Y = other.Y;
    }
 
    // Move constructor
    Vector2(const Vector2&& other)
    {
        X = other.X;
        Y = other.Y;
    }
 
    // Move constructor (argument is not const)
    Vector2(Vector2&& other)
    {
        X = other.X;
        Y = other.Y;
    }
};
```

C++ 编译器将生成一个复制构造函数，并且所有数据成员都可以进行复制构造。同样，如果我们不定义任何 copy 或 move 构造函数，编译器将生成一个 move 构造函数，并且所有数据成员都可以 move 构造。因此，编译器将在此处为 `Vector2` 和 `Ray2` 生成 copy 和 move 构造函数：

```c++
struct Vector2
{
    float X;
    float Y;
 
    // Compiler generates copy constructor:
    // Vector2(const Vector2& other)
    //     : X(other.X), Y(other.Y)
    // {
    // }
 
    // Compiler generates move constructor:
    // Vector2(const Vector2&& other)
    //     : X(other.X), Y(other.Y)
    // {
    // }
};
 
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    // Compiler generates copy constructor:
    // Ray2(const Ray2& other)
    //     : Origin(other.Origin), Direction(other.Direction)
    // {
    // }
 
    // Compiler generates move constructor:
    // Ray2(const Ray2&& other)
    //     : Origin(other.Origin), Direction(other.Direction)
    // {
    // }
};
```

这些编译器生成的 copy 和 move 构造函数的参数是 `const` （如果有可供调用的 `const` 复制和移动） 构造函数，如果没有 const 则为 `non-const` （非 const）。

与默认构造函数一样，我们可以使用 `= default` 来告诉编译器生成 copy 和 move 构造函数：
```c++
struct Vector2
{
    float X;
    float Y;
 
    // 结构体内部
    Vector2(const Vector2& other) = default;
};
 
struct Ray2
{
    Vector2 Origin;
    Vector2 Direction;
 
    Ray2(Ray2&& other);
};
 
// 结构体外部
// 显式默认的移动构造函数不能接受const
Ray2::Ray2(Ray2&& other) = default;
```

我们还可以使用 `=delete` 来禁用编译器生成的 copy 和 move 构造函数：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2(const Vector2& other) = delete;
    Vector2(const Vector2&& other) = delete;
};
```

> 这里读者刚刚开始看的时候，有点懵，给个例子就清晰了
> </br>先是复制构造函数：
> ```c++
> #include <iostream>
> #include <cstring>
>
> class String {
> private:
>   char* data;
>   size_t length;
>
> public:
>   // 普通构造函数
>   String(const char* str = "") {
>   length = strlen(str);
>   data = new char[length + 1];
>   strcpy(data, str);
>   }
>   
>      // 复制构造函数（深拷贝）
>      String(const String& other) {
>          length = other.length;
>          data = new char[length + 1];
>          strcpy(data, other.data);
>          std::cout << "Copy Constructor Called\n";
>      }
>   
>      // 析构函数
>      ~String() {
>          delete[] data;
>      }
>   
>      void print() {
>          std::cout << data << std::endl;
>      }
>   };
>   
>   int main() {
>   String s1("Hello");
>   String s2 = s1; // 调用复制构造函数
>   
>      s1.print(); // 输出 "Hello"
>      s2.print(); // 输出 "Hello"
>      return 0;
>   }
>```
> 
> </br>输出
> ```
> Copy Constructor Called
> Hello
> Hello
> ```
> 
> 简单解释一下：复制构造函数确保 s2 独立拥有 s1 数据的副本。 
> 如果没有深拷贝，两个对象的 data 指针会指向同一内存，导致重复释放
> 
> 然后是移动构造函数：
> ```c++
> #include <iostream>
> #include <cstring>
> #include <utility> // 用于 std::move
>
> class String {
> private:
>   char* data;
>   size_t length;
>
> public:
>   // 普通构造函数
>   String(const char* str = "") {
>   length = strlen(str);
>   data = new char[length + 1];
>   strcpy(data, str);
>   }
>   
>      // 复制构造函数
>      String(const String& other) { /* 同上 */ }
>   
>      // 移动构造函数（C++11 引入）
>      String(String&& other) noexcept {
>          // 直接“窃取”资源
>          data = other.data;
>          length = other.length;
>   
>          // 将原对象的指针置空，避免重复释放
>          other.data = nullptr;
>          other.length = 0;
>   
>          std::cout << "Move Constructor Called\n";
>      }
>   
>      // 析构函数
>      ~String() {
>          if (data) delete[] data;
>      }
>   
>      void print() {
>          if (data) std::cout << data << std::endl;
>          else std::cout << "(Empty)\n";
>      }
>   };
>   
>   int main() {
>   String s1("Hello");
>   
>      // 移动语义：将 s1 的资源转移给 s2
>      String s2 = std::move(s1);
>   
>      s1.print(); // 输出 "(Empty)"
>      s2.print(); // 输出 "Hello"
>      return 0;
>   }
> 
>```
> 输出：
> ```c++
> Move Constructor Called
> (Empty)
> Hello
>```
> 简单解释一下：移动构造函数将 s1 的资源直接转移给 s2，无需深拷贝。 原对象 s1 的 data 被置空，不再拥有资源。

## Destructors 析构函数

C# 类可以具有析构器，通常称为析构函数。C# 结构体不能，但 C++ 结构可以。

与构造函数不同，这两种语言之间的构造函数相当相似，而C++的析构函数则极为不同。这些差异对C++代码的设计和编写方式产生了巨大影响。

从语法上讲，C++ 析构函数看起来与 C# 类终结器/析构函数相同：我们只在结构名称前加上一个 `~`，不接受任何参数。

```c++
struct File
{
    FILE* handle;
 
    // 构造函数
    File(const char* path)
    {
        // fopen() opens a file
        handle = fopen(path, "r");
    }
 
    // 析构函数
    ~File()
    {
        // fclose() closes the file
        fclose(handle);
    }
};
```

我们还可以将定义放在结构体之外：

```c++
struct File
{
    FILE* handle;
 
    // 构造函数
    File(const char* path)
    {
        // fopen() opens a file
        handle = fopen(path, "r");
    }
 
    // 析构函数 申明
    ~File();
};
 
// 析构函数 定义
File::~File()
{
        // fclose() closes the file
        fclose(handle);
}
```

析构函数通常以隐式方式调用，但也可以显式调用：

```c++
File file("myfile.txt");
file.~File(); // 调用析构函数
```

C#的析构器和C++的析构函数的基本目的是相同的：当对象消失时进行一些清理。在C#中，对象在垃圾回收后才会消失。[析构器](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/finalizers)被调用的[时机](https://ericlippert.com/2015/05/18/when-everything-you-know-is-wrong-part-one/)，如果被调用的话，非常复杂，非确定性的，并且是多线程的。

> 作者在这里提供提供了微软官网的析构器英文文档，我在这提供[中文](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/finalizers)。
> 这个析构器的调用时机的博客，我有空在学习完之后翻译（画饼）

在 C++ 中，对象的析构函数在其生命周期结束时被简单地调用：

```c++
void OpenCloseFile()
{
    File file("myfile.txt");
    DebugLog("file opened");
    // 编译器生成：file.~File();
}
```

cpp保证无论如何都会调用析构函数。考虑一个异常，我们将在本系列后面更深入地介绍该异常，但其作用类似于 C# 异常：

```c++
void OpenCloseFile()
{
    File file("myfile.txt");
    if (file.handle == nullptr)
    {
        DebugLog("file filed to open");
        // 编译器生成：file.~File();
        throw IOException();
    }
    DebugLog("file opened");
    // 编译器生成：file.~File();
}
```

无论`file`如何超出范围，都会首先调用其析构函数

即使是基于运行时计算的 `goto` 也无法绕过析构函数： 

```c++
void Foo()
{
    label:
    File file("myfile.txt");
    if (RollRandomNumber() == 3)
    {
        // 编译器生成：file.~File();
        return;
    }
    shouldReturn = true;
    // 编译器生成：file.~File();
    goto label;
}
```

为了简要了解这如何影响 C++ 代码的设计，让我们向 `File` 添加一个 `GetSize` 成员函数，以便它可以执行一些有用的作。我们还添加一些基于异常的错误处理：

```c++
struct File
{
    FILE* handle;
 
    File(const char* path)
    {
        handle = fopen(path, "r");
        if (handle == nullptr)
        {
            throw IOException();
        }
    }
 
        long GetSize()
    {
        // === 步骤 1：保存当前文件指针位置 ===
        // - ftell 返回当前文件指针相对于文件开头的偏移量（字节单位）
        // - 返回 -1 表示错误（例如未打开文件、流不支持寻址等）
        long oldPos = ftell(handle);
        if (oldPos == -1)
        {
            throw IOException(); // 无法获取当前位置，抛出异常
        }
    
        // === 步骤 2：将文件指针移动到文件末尾 ===
        // - fseek 的第三个参数 SEEK_END 表示基准位置是文件末尾
        // - 偏移量 0 表示移动到文件末尾的位置
        // - 返回值 0 表示成功，非 0 表示失败（例如不可寻址的流）
        int fseekRet = fseek(handle, 0, SEEK_END);
        if (fseekRet != 0)
        {
            throw IOException(); // 无法移动指针到末尾，抛出异常
        }
    
        // === 步骤 3：获取文件大小 ===
        // - 此时文件指针在文件末尾，ftell 返回指针距离文件开头的字节数,即文件总字节数
        long size = ftell(handle);
        if (size == -1)
        {
            throw IOException(); // 无法获取文件大小，抛出异常
        }
    
        // === 步骤 4：恢复原始文件指针位置 ===
        // - 将文件指针移回步骤 1 保存的位置（oldPos）
        // - SEEK_SET 表示基准位置是文件开头
        fseekRet = fseek(handle, oldPos, SEEK_SET);
        if (fseekRet != 0)
        {
            throw IOException(); // 无法恢复指针位置，抛出异常
        }
    
        // === 最终返回 ===
        // - 返回文件大小（单位：字节）
        // - 注意：文件指针已恢复到调用此方法前的位置，不影响后续操作
        return size;
    }
 
    ~File()
    {
        fclose(handle);
    }
};
```

我们可以使用它来获取文件的大小，如下所示：

```c++
long GetTotalSize()
{
    File fileA("myfileA.txt");
    File fileB("myfileB.txt");
    long sizeA = fileA.GetSize();
    long sizeB = fileA.GetSize();
    long totalSize = sizeA + sizeB;
    return totalSize;
}
```

编译器为此生成多个析构函数调用。要查看它们，让我们看看构造函数生成的伪代码版本：

```c++
long GetTotalSize()
{
    File fileA("myfileA.txt");
 
    try
    {
        File fileB("myfileB.txt");
        try
        {
            long sizeA = fileA.GetSize();
            long sizeB = fileA.GetSize();
            long totalSize = sizeA + sizeB;
            fileB.~File();
            fileA.~File();
            return totalSize;
        }
        catch (...) // 捕获所有类型的异常
        {
            fileB.~File();
            throw; // 将异常重新抛出到外部的捕获块
        }
    }
    catch (...) // 捕获所有类型的异常
    {
        fileA.~File();
        throw; // 重新抛出异常
    }
}
```
在这个展开的视图中，我们看到编译器在 `fileA` 或 `fileB` 可能结束其生命周期的每个可能位置生成析构函数调用。我们不可能忘记调用析构函数，因为编译器会彻底地为我们添加所有析构函数调用。我们知道，根据设计，两个文件句柄都不会泄漏。

析构函数的另一个方面在这里也可见：它们在对象上的调用顺序与调用构造函数的顺序相反。因为我们首先声明了 `fileA`，然后声明了 `fileB`，所以构造函数顺序是 `fileA` 然后 `fileB`，析构函数顺序是 `fileB` 然后 `fileA`。

```c++
struct TwoFiles
{
    File FileA;
    File FileB;
};
 
void Foo()
{
    // 如果我们写这么一段...
    TwoFiles tf;
 
    // 编译器生成构造函数调用：A 然后 B
    // 伪代码：不能真正直接调用构造函数
    tf.FileA();
    tf.FileB();
 
    // 然后析构函数调用：B 然后 A
    tf.~FileB();
    tf.~FileA();
}
```

这就解释了为什么我们不能更改初始值设定项列表中数据成员的顺序：无论构造函数做什么，编译器都需要能够生成析构函数调用的相反顺序。

最后，编译器隐式生成析构函数：

```c++
struct TwoFiles
{
    File FileA;
    File FileB;
 
    // 编译器生成的析构函数
    ~TwoFiles()
    {
        FileB.~File();
        FileA.~File();
    }
};
```

我们可以使用 `= default` 来显式告诉它这样做：

```c++
// 结构体内
struct TwoFiles
{
    File FileA;
    File FileB;
 
    ~TwoFiles() = default;
};
//-------------------- 
// 结构体外
struct TwoFiles
{
    File FileA;
    File FileB;
 
    ~TwoFiles();
};
TwoFiles::~TwoFiles() = default
```

我们可以阻止编译器使用 `= delete` 生成一个：
```c++
struct TwoFiles
{
    File FileA;
    File FileB;
 
    ~TwoFiles() = delete;
};
```

只要我们没有编写析构函数，编译器就会生成析构函数，并且所有数据成员都可以析构。

## 总结

在最基本的情况下，C# 和 C++ 中的构造函数是相同的。不过，这两种语言很快就离开了编译器隐式或显式生成的 default、copy 和 move 构造函数，支持编写自定义默认构造函数、严格的初始化排序和初始值设定项列表。

析构函数与C#的终结器/析构函数截然不同。它们在对象的生命周期结束时立即被调用，而不是在对象释放后很长时间的另一个线程上，或者可能根本不会被调用。这种模式与C#的`using（IDisposable）`类似，但不需要添加`using`部分，也不会忘记它。它们还严格按与构造相反的顺序进行销毁，并为我们提供了生成或不生成析构函数的选项。

