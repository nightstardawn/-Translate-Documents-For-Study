# C++ For C# Developers: Part 37 – Missing Language Features

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6355),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

我们已经介绍了 C++ 语言中的所有功能！尽管如此，C# 仍具有 C++ 中缺少的一些功能。今天，我们将研究这些并探索一些替代方案来填补这些空白。

## Fixed Statements  固定语句

在 C# 的不安全上下文中，我们可以使用 `fixed` 语句来防止 GC 移动对象：

```csharp
// 不安全函数为其主体创建了一个不安全的环境
unsafe void ZeroBytes(byte[] bytes)
{
    // 防止移动数组
    fixed (byte* pBytes = bytes)
    {
        // 通过指针访问数组
        for (int i = 0; i < bytes.Length; ++i)
        {
            pBytes[i] = 0;
        }
    }
}
```

由于 C++ 没有 GC，因此我们的对象永远不会移动。因此，我们不需要固定的语句，因为我们可以简单地获取对象的地址：

```c++
struct ByteArray
{
    int32_t Length;
    uint8_t* Bytes;
};
 
void ZeroBytes(ByteArray& bytes)
{
    ByteArray* pBytes = &bytes;
    for (int i = 0; i < pBytes->Length; ++i)
    {
        pBytes->Bytes[i] = 0;
    }
}
```

不过，通常没有理由费心去拿指针。这是因为默认情况下，对象是按 value 传递的。
</br>在上面的例子中，我们在 `ZeroBytes` 中获取 `ByteArray&` 左值引用，因为只获取 `ByteArray` 会导致在调用函数时进行复制。
</br>因此，我们通常已经有一个指向对象的类似指针的引用，并且可以简单地直接使用它：

```c++
void ZeroBytes(ByteArray& bytes)
{
    for (int i = 0; i < bytes.Length; ++i)
    {
        bytes.Bytes[i] = 0;
    }
}
```

## Fixed Size Buffers  固定大小的缓冲区

`fixed` 在 C# 中的另一种含义是创建基元（`bool`、`byte`、`char`、`short`、`int`、`long`、`sbyte`、`ushort`、`uint`、`ulong`、`float` 或 `double`）的缓冲区
</br>该缓冲区直接属于类或结构 ，而不是像 `byte[]` 等数组那样创建托管引用：

```csharp
// 需要一个不安全的环境
unsafe struct FixedLengthArray
{
    // 16个整数是结构体的一部分
    // 这不是对int数组的托管引用
    public fixed int Elements[16];
}
```

C++ 不需要固定大小的缓冲区，因为它直接支持数组：

```c++
struct FixedLengthArray
{
    // 16个整数是结构体的一部分
    int32_t Elements[16];
};
```

此外，仅使用基元类型没有限制。可以使用任何类型：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
struct FixedLengthArray
{
    // 16个Vector2是结构体的一部分
    Vector2 Elements[16];
};
```

## Properties  属性

C# 结构和类支持一种称为“属性”的特殊函数，该函数给人一种用户正在引用字段而不是调用函数的错觉：

```csharp
class Player
{
    // 通常被称为“背场”
    string m_Name;
 
    // 属性名为“Name”，类型为字符串
    public string Name
    {
        // 它的"get"函数不接受任何参数，并且必须返回属性
        // type: string
        get
        {
            return m_Name;
        }
        // “set”函数隐式传递了一个属性类型（字符串）的单个参数，并且必须返回void
        set
        {
            m_Name = value;
        }
    }
}
 
Player p = new Player();
 
// 在Name属性上调用“set”，并将“Jackson”作为值参数传递
p.Name = "Jackson";
 
// 调用“get”方法并获取Name属性的返回字符串
DebugLog(p.Name);
```

当 `get` 和 `set` 函数的主体以及 “支持字段” 是微不足道的时，如上所示，可以使用自动实现的属性来告诉编译器生成此样板：

```c++
class Player
{
    public string Name { get; set; }
}
```

C++ 没有属性。相反，命名约定通常用于创建对 “get” 和 “set” 函数。下面是一个流行的命名约定：

```c++
struct Player
{
    const char* m_Name;
 
    const char* GetName() const
    {
        return m_Name;
    }
 
    void SetName(const char* value)
    {
        m_Name = value;
    }
};
 
Player p{};
p.SetName("Jackson");
DebugLog(p.GetName())
```

另一个流行的约定依赖于重载来消除 “Get” 和 “Set” 前缀：

```c++
struct Player
{
    const char* m_Name;
 
    const char* Name() const
    {
        return m_Name;
    }
 
    void Name(const char* value)
    {
        m_Name = value;
    }
};
 
Player p{};
p.Name("Jackson");
DebugLog(p.Name());
```

无论选择哪种约定， 都可以使用宏(Part24)来删除样板：

```c++
// Macro to create a property
#define AUTO_PROPERTY(propType, propName) \
    propType m_##propName; \
    const propType& propName() const \
    { \
        return m_##propName; \
    } \
    void propName(const propType& value) \
    { \
        m_##propName = value; \
    }
 
struct Player
{
    // Create the property
    AUTO_PROPERTY(const char*, Name)
};
 
Player p{};
p.Name("Jackson");
DebugLog(p.Name());
```

## Extern  外部

若要调用在 .NET 环境之外实现的函数，C# 可以将它们声明为 `extern`。通常，这用于调用 C 或 C++ 代码：

```csharp
using System.Runtime.InteropServices;
 
public static class WindowsApi
{
    // 此函数在Windows的User32.dll中实现
    [DllImport("User32.dll", CharSet=CharSet.Unicode)]
    public static extern int MessageBox(
        IntPtr handle, string message, string caption, int type);
}
 
// 调用外部函数
WindowsApi.MessageBox((IntPtr)0, "Hello!", "Title", 0);
```

C++ 中的 `extern` 关键字具有不同的含义：在另一个翻译单元中实现。
</br>要调用另一个 DLL 中的函数，我们使用平台的 API 加载 DLL，调用函数，然后卸载它。以下是我们如何使用 Windows API 执行此作：

```c++
// 提供DLL访问的平台API
#include <windows.h>
 
// 加载DLL
auto dll = LoadLibraryA("User32.dll");
 
// 获取MessageBoxA函数的地址（MessageBox的ASCII版本）
auto proc = GetProcAddress(dll, "MessageBoxA");
 
// 转换为适当的函数指针类型
auto mb = (int32_t(*)(void*, const char*, const char*, uint32_t))(proc);
 
// 通过函数指针调用 MessageBoxA
(*mb)(nullptr, "Hello!", "Title", 0);
 
// 卸载DLL
FreeLibrary(dll);
```

.NET 环境负责加载和卸载 `[DllImport]` 引用的 DLL，以及创建指向关联 `extern` 函数的函数指针。
</br>作为权衡，我们失去了对流程元素的控制，例如计时和错误处理。

对于多平台 C++ 代码，通常将此特定于平台的功能包装在一个抽象层中，该抽象层使用预处理器进行正确的调用。例如：

```c++
///////////////
// platform.hpp
///////////////
 
// Windows
#ifdef _WIN32
    #include <windows.h>
// Non-Windows (e.g. macOS)
#else
    // TODO
#endif
 
class Platform
{
#if _WIN32
    using MessageBoxFuncPtr = int32_t(*)(
        void*, const char*, const char*, uint32_t);
    HMODULE dll;
    MessageBoxFuncPtr mb;
#else
    // TODO
#endif
 
public:
 
    Platform()
    {
#if _WIN32
        dll = LoadLibraryA("User32.dll");
        mb = (MessageBoxFuncPtr)(GetProcAddress(dll, "MessageBoxA"));
#else
        // TODO
#endif
    }
 
    ~Platform()
    {
#if _WIN32
        FreeLibrary(dll);
#else
        // TODO
#endif
    }
 
    // 在Windows上抽象MessageBoxA的调用，在其他平台（例如macOS）上调用其他操作
    void MessageBox(const char* message, const char* title)
    {
#if _WIN32
        (*mb)(nullptr, message, title, 0);
#else
        // TODO
#endif
    }
};
 
 
///////////
// game.cpp
///////////
 
#include "platform.hpp"
 
Platform platform{};
platform.MessageBox("Hello!", "Title");
```

使用此类预处理器指令的一种替代方法是为每个平台创建不同的文件：`platform_windows.cpp`、`platform_macos.cpp` 等。
每个 API 都包含 `Platform` 类的实现，其中包含适用于要编译的平台的代码。
然后，可以将项目配置为仅编译其中一个文件，因此不会有链接时间冲突，因为只存在一个 `Platform` 类。


## Extension Methods  扩展方法

C# 给人一种我们可以向类和结构添加方法的错觉。不过，这些并不是真正添加的，因为它们仍然是类或结构之外的静态函数。
</br>C# 只允许在它们“扩展”的类或结构的实例上调用它们：

```csharp
public static class ArrayExtensions
{
    // float[] 上的扩展方法，因为第一个参数具有 “this”
    public static float Average(this float[] array)
    {
        float sum = 0;
        foreach (float cur in array)
        {
            sum += cur;
        }
        return sum / array.Length;
    }
}
 
float[] array = { 1, 2, 3 };
 
//像调用float数组的成员方法一样调用扩展方法
DebugLog(array.Average()); // 2
 
// 或者按常规称呼它
DebugLog(ArrayExtensions.Average(array));
```

第一个版本 （`array.Average（）`） 由编译器重写为第二个版本 （ `ArrayExtensions.Average(array)` ）。
</br>扩展方法无法获得对它们包含的类或结构的任何特殊访问权限。例如，他们无法访问私有字段。

C++ 版本与第二个版本类似：我们通常在任何类之外编写一个“free function”，将类作为参数 “extend” ：

```c++
float Average(float* array, int32_t length)
{
    float sum = 0;
    for (int32_t i = 0; i < length; ++i)
    {
        sum += array[i];
    }
    return sum / length;
}
 
float array[] = { 1, 2, 3 };
DebugLog(Average(array, 3)); // 2
```

像这样的函数可以放入命名空间中，也可以放入类的静态成员函数中，但主体仍然存在：函数与它“扩展”的内容断开连接，没有对它的特殊访问权限。

## Checked Arithmetic  校验算术

C# 具有 checked 关键字，用于对算术执行运行时检查。我们可以基于每个表达式或整个块选择加入：

```csharp
public class Player
{
    public uint Health;
 
    public void TakeDamage(uint amount)
    {
        // 选择启用算术检查
        checked
        {
            // 如果发生下溢，将抛出OverflowException异常
            Health -= amount;
        }
    }
}
 
Player p = new Player{ Health = 100 };
 
// OK:Heath现在是50
p.TakeDamage(50);
 
// 溢出异常：尝试将Heath下溢至-20
p.TakeDamage(70);
```

C++ 没有内置的算术检查。相反，我们有几个选择。首先，我们可以执行自己的手动算术检查：

```c++
struct OverflowException
{
};
 
struct Player
{
    uint32_t Health;
 
    void TakeDamage(uint32_t amount)
    {
        if (amount > Health)
        {
            throw OverflowException{};
        }
        Health -= amount;
    }
};
 
Player p{ 100 };
 
// OK:Heath现在是50
p.TakeDamage(50);
 
// 溢出异常：尝试将生命值下溢至-20
p.TakeDamage(70);
```

其次，我们可以将数字类型包装在结构体中，并使用检查重载运算符。
</br>此选项与 C# 中的 `checked` 块最匹配，因为它允许我们对许多作执行检查，而无需为每个作编写任何内容：

```c++
struct CheckedUint32
{
    uint32_t Value;
 
    // 从uint32_t转换
    CheckedUint32(uint32_t value)
        : Value(value)
    {
    }
 
    // 重载减法运算符以检查下溢
    CheckedUint32 operator-(uint32_t amount)
    {
        if (amount > Value)
        {
            throw OverflowException{};
        }
        return Value - amount;
    }
 
    // 隐式转换回uint32_t
    operator uint32_t()
    {
        return Value;
    }
};
 
struct Player
{
    uint32_t Health;
 
    void TakeDamage(uint32_t amount)
    {
        // 将Heath放入包装结构中以检查其算术运算符
        Health = CheckedUint32{ Health } - amount;
    }
};
```

或者我们可以创建执行检查的函数。这与 C# 中仅适用于一个作的 `checked` 表达式非常匹配：

```c++
uint32_t CheckedSubtraction(uint32_t a, uint32_t b)
{
    if (b > a)
    {
        throw OverflowException{};
    }
    return a - b;
}
 
struct Player
{
    uint32_t Health;
 
    void TakeDamage(uint32_t amount)
    {
        Health = CheckedSubtraction(Health, amount);
    }
};
```

最后一种方法由 [Boost Checked Arithmetic](https://www.boost.org/doc/libs/master/libs/safe_numerics/doc/html/checked_arithmetic.html) 等库采用。

`unchecked` 关键字在 C++ 中不存在，因为没有要禁用的 `checked` 算术。

## Nameof

C# 的 `nameof` 运算符获取变量、类型或成员的字符串名称：

```csharp
Player p = new Player();
DebugLog(nameof(p)); // p
```

C++ 没有内置此功能，但有一个[库](https://github.com/Neargye/nameof)可用于提供 NAMEOF 宏以实现类似功能：

```c++
Player p{};
DebugLog(NAMEOF(p)); // p
```

与 C# 运算符一样，它支持变量、类型和成员。此外，它还支持宏、枚举 “flag” 值，并在编译时和运行时运行。

## Decimal  十进制

C# 具有内置的十进制类型，用于财务计算和其他需要表示小数位而不进行任何舍入的情况：

```csharp
float f = 1.0f;
for (int i = 0; i < 10; ++i)
{
    f -= 0.1f;
    DebugLog(f);
}
```

这将打印不准确的值，因为浮点数如果不四舍五入就无法表示这些值：

```csharp
0.9
0.8
0.6999999
0.5999999
0.4999999
0.3999999
0.2999999
0.1999999
0.09999993
-7.450581E-08
```

如果我们使用 `decimal`，则避免四舍五入：

```cs
decimal d = 1.0m;
for (int i = 0; i < 10; ++i)
{
    d -= 0.1m;
    DebugLog(d);
}
```

```csharp
0.9
0.8
0.7
0.6
0.5
0.4
0.3
0.2
0.1
0.0
```

C++ 没有内置的 `decimal` 类型，但 [GMP](https://gmplib.org/) 和 [decimal_for_cpp](https://github.com/vpiotr/decimal_for_cpp) 等库会创建此类类型。例如，在后一个库中，我们可以这样写：

```c++
#include "decimal.h"
using namespace dec;
 
decimal<1> d{ 1.0 };
for (int i = 0; i < 10; ++i)
{
    d -= decimal<1>{ 0.1 };
    DebugLog(d);
}
```

这将打印出我们期望的结果：

```c++
0.9
0.8
0.7
0.6
0.5
0.4
0.3
0.2
0.1
0.0
```

## Reflection  反射

C# 在它编译到的二进制文件中隐式存储有关程序结构的大量信息。
</br>然后，C# 代码可以在运行时访问此信息，以便通过“反射”方法（如 GetType）进行查询，该方法返回 Type 等类。

```csharp
public class Player
{
    public string Name;
    public uint Health;
}
 
Player p = new Player{Name="Jackson", Health=100};
Type type = p.GetType();
foreach (FieldInfo fi in type.GetFields())
{
    DebugLog(fi.Name + ": " + fi.GetValue(p));
}
```

```csharp
Name: Jackson
Health: 100
```

C++ 存储的唯一此类信息是 RTTI(part21) 支持 `dynamic_cast` 和 `typeid` 的数据。
它是 C# 中可用内容的一个非常小的子集，因为即使是完整的类型名称通常也不会保留在 `typeid` 中，并且 `dynamic_cast` 仅支持具有虚拟函数的类。

所以如果我们想存储这些信息，我们需要自己存储它。我们可以通过实现自己的反射系统来手动执行此作：

```c++
// 我们反射系统支持的不同类型
enum class Type
{
    None,
    ConstCharPointer,
    Uint32,
};
 
// 反射值
struct Value
{
    // 值的类型
    Type Type;
 
    // 值指针
    void* ValuePtr;
};
 
// 将“接口”实现为使类支持反射
struct IReflectable
{
    using MemberName = const char*;
 
    // 获取类字段的名称
    virtual const MemberName* GetFieldNames() = 0;
 
    // 获取类实例的字段值
    virtual Value GetFieldValue(MemberName* name) = 0;
};
 
// 玩家支持反射
class Player : IReflectable
{
    // 字段的名称。在类初始化之后。
    static const char* const FieldNames[3];
 
public:
 
    const char* Name;
    uint32_t Health;
 
    virtual const MemberName* GetFieldNames() override
    {
        return FieldNames;
    }
 
    virtual Value GetFieldValue(MemberName* name) override
    {
        // strcmp是一个标准库函数，当字符串相等时返回0
        if (!strcmp(name, "Name"))
        {
            return { Type::ConstCharPointer, &Name };
        }
        else if (!strcmp(name, "Health"))
        {
            return { Type::Uint32, &Health };
        }
        return { Type::None, nullptr };
    }
};
const char* const Player::FieldNames[3]{ "Name", "Health", nullptr };
 
Player p;
p.Name = "Jackson";
p.Health = 100;
 
auto fieldNames = p.GetFieldNames();
for (int32_t i = 0; fieldNames[i]; ++i)
{
    auto fieldName = fieldNames[i];
    auto fieldValue = p.GetFieldValue(fieldName);
    switch (fieldValue.Type)
    {
    case Type::ConstCharPointer:
        DebugLog(fieldName, ": ", *(const char**)fieldValue.ValuePtr);
        break;
    case Type::Uint32:
        DebugLog(fieldName, ": ", *(uint32_t*)fieldValue.ValuePtr);
        break;
    }
}
```

这将打印相同的日志：

```c++
Name: Jackson
Health: 100
```

手动添加所有这些内容非常乏味，并且随着代码的更改会产生维护问题。因此，有许多反射库可供 C++ 删除大量样板：

- [Boost PFR](https://www.boost.org/doc/libs/1_75_0/doc/html/boost_pfr.html) 提供基本反射
- [Magic Enum](https://github.com/Neargye/magic_enum) 仅支持枚举
- [RTTR](https://github.com/rttrorg/rttr) 具有更完整的反射功能

例如，在 RTTR 中，我们可以这样写：

```c++
#include <rttr/registration>
using namespace rttr;
 
class Player
{
    const char* Name;
    uint32_t Health;
};
 
RTTR_REGISTRATION
{
    registration::class_<Player>("Player")
         .property("Name", &Player::Name)
         .property("Health", &Player::Health);
}
 
Player p;
p.Name = "Jackson";
p.Health = 100;
 
type t = type::get<Player>();
for (auto& prop : t.get_properties())
{
    DebugLog(prop.get_name(), ": ", prop.get_value(p));
}
```

## Conclusion  结论

这两种语言都不是另一种语言的子集。在本系列的几乎每篇文章中，我们都看到了各种语言功能的 C++ 版本比 C# 等效版本更大、更强大。
</br>今天，我们看到了相反的情况：C# 具有 C++ 所没有的几个功能。

我们还了解了如何至少在需要时在 C++ 中近似该功能。有时，就像固定语句和缓冲区一样，C++ 中不需要这样的功能，我们可以简单地停止使用 C# 功能。

其他时候，与扩展方法和属性一样，没有直接的等效项，我们需要调整我们的设计以适应 C++ 规范，例如使用自由函数和“GetX”函数。

然后，在某些情况下，可以使用库在 C++ 语言之上实现类似的功能。decimal、nameof 和 reflection 就是这种情况。C++ 提供的强大、相对低级的工具使此类库的高效实现成为可能。

最后，还有一些缺失的 C# 功能，其替代方案专门依赖于标准库。我们将在本系列的后面看到这些替代方案。






























