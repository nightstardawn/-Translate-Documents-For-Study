# C++ For C# Developers: Part 20 – Implicit Type Conversion

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5839),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

到目前为止，在这个系列中我们实际上已经看到了很多隐式类型转换。
</br>我们已经将整数转换为浮点数（`float f = 123`），将数组转换为指针（`int* p = a`），将基类型指针转换为派生类型指针（`D* p = &b`），等等。
</br>今天我们将把这些随意的转换汇总到一篇文章中，这篇文章将涵盖所有规则，包括用户定义的类型转换。

## When Implicit Type Conversion Happens 当隐式类型转换发生时

C#和C++都支持隐式类型转换，但存在许多语言特定的差异。在C++中，当使用更广泛的语言特性时会发生隐式转换。

首先，就像在C#中一样，当调用一个函数时，如果参数的类型与函数的参数类型不同，就会发生隐式转换：

```c++
void Foo(float x)
{
}
Foo(1); // int -> float
```

同样，在 C# 中，当返回其类型不是函数的返回类型的值时，会发生这种情况：

```c++
float Bar()
{
    return 1; // int -> float
}
```

所有布尔逻辑运算符都需要布尔操作数，因此任何非布尔值都需要转换。在C++中，这可以是隐式的，但在C#中则不行：

```c++
bool b1 = !1; // int -> bool
bool b2 = false || 1; // int -> bool
bool b3 = true && 1; // int -> bool
```

C++ 中的条件语句也是如此，但 C# 不是：

```c++
if (1) // int -> bool
{
}
 
bool b4 = 1 ? false : true; // int -> bool
```

在循环中也是如此：

```c++
while (1) // int -> bool
{
}
for (; 1; ) // int -> bool
{
}
do
{
} while (1); // int -> bool
```

switch语句需要整型数据，因此在使用其他类型时需要进行转换。两种语言都支持这一点：

```c++
switch (false) // bool -> int
{
}
```

C++的`delete`运算符仅删除类型化指针。这里有一个用户定义的转换运算符，它将一个结构体转换为int*：

```c++
struct ConvertsToIntPointer
{
    operator int*() { return nullptr; }
};
delete ConvertsToIntPointer{}; // ConvertsToIntPointer -> int*
```

最后，C++中的noexcept和explicit都可以是条件性的，并且需要布尔值：

```c++
void Baz() noexcept(1) // int -> bool
{
}
 
// C++20 and later
struct HasConditionalExplicit
{
    explicit(1) HasConditionalExplicit() {} // int -> bool
};
```

## Standard Conversions  标准转换

我们已经讨论了用户定义的转换，但这些并不是隐式地在类型之间进行转换的唯一方式。语言本身就有许多“标准”转换，这些转换不需要我们编写任何代码。

通常这些会改变类型本身，但在少数情况下，它们只是改变其分类：

```c++
int x = 123;
const int y = x; // int -> const int
                 // also, lvalue -> rvalue
```

将非const视为const总是可以的，因为这只是增加了限制。我们不能反过来，因为那样会移除const限制。

同样，我们可以从非抛出异常的函数转换为可能抛出异常的函数，但不能反过来：

```c++
void DoStuff() noexcept
{
    throw 1;
}
 
void (*pFuncNoexcept)() noexcept = &DoStuff;
void (*pFunc)() = pFuncNoexcept; // 非抛出异常函数 转化为 可抛出异常函数
void (*pFuncNoexcept2)() noexcept = pFunc; // 编译器错误
```

解决了这些问题之后，所有其他的标准转换都会改变类型。首先，我们来看函数到指针的转换。
之前的例子中，我们通过获取`DoStuff`的地址来得到它的指针，就像我们之前看到的那样，但是&是可选的，因为存在从函数到函数指针的标准转换：

```c++
void DoStuff()
{
}
 
void (*pFunc)() = DoStuff; // 函数 -> 函数指针
```

请注意，这在对非静态成员函数不适用，因为它们需要一个类实例来隐式传递this参数：

```c++
struct Vector2
{
    float X;
    float Y;
 
    float SqrMagnitude() const noexcept
    {
        return X*X + Y*Y;
    }
};
 
Vector2 vec{2, 4};
// 以下均为编译错误
float (*sqrMagnitude1)() = Vector2::SqrMagnitude
float (*sqrMagnitude2)() = vec.SqrMagnitude;
float (*sqrMagnitude3)(Vector2*) = Vector2::SqrMagnitude
float (*sqrMagnitude4)(Vector2*) = vec.SqrMagnitude;
```

> 读者不会记得前面是否讲过，在这里解释一下，这里函数后面 存在const的含义
> - 该函数内不允许修改成员变量（除非成员变量被声明为 mutable）。
> - 允许该函数被const对象调用(普通对象也可以调用)，侧面说明了const对象只能调用const函数

接下来，我们进行数组到指针的转换：

```c++
int arr[]{1, 2, 3};
int* pArray = arr; // int[3] -> int*
```

这被称为“数组到指针退化”，这是允许的，因为这两个概念非常相似。
</br>从语义上讲，这有点像指针是数组的“基类”。它们本质上都是指针，可以像数组一样处理（`x[123]`），但数组的大小是已知的（`sizeof(arr) == sizeof(int)*3`）。
</br>所以可以这样理解，就像是从派生类（数组更多信息）到基类（信息较少的指针）的“向上转型”。

当涉及到数字时，我们有两种广泛的分类：提升(Promotion)和转换(Conversion)。提升不会改变数字的值，但转换可能会。
</br>提升通常会增加数字的大小，因为更大的大小可以表示所有较小大小值。
</br>在C#中，当我们，例如，将`short`类型传递给一个接受`int`类型的函数时，也会发生这种情况。

这是因为所有算术运算符（例如 `x + y`）都需要 int 或更大的类型。因此，较小的原始类型将为了所有这些运算符以及我们上面看到的所有原因被提升为 int。

> 简单解释一下读者理解的这段话：
> 提升：小范围数据转大范围数据(无所谓，不影响数据本身)
> 转化：大范围数据转小范围数据(可能会产生影响，如范围不够、精度损失等)

首先，`signed char`总是提升为`int`：

```c++
signed char c = 'A';
int i = c + 1; // c 从 signed char 提升为 int
DebugLog(i); // 66 (ASCII for 'B')
```

`char`、`unsigned char`、`unsigned short` 和 `int` 的大小取决于编译器和 CPU 架构等因素。
</br>如果 `int` 可以容纳 `char`、`unsigned char`、`unsigned short` 和 `char8_t` 的全部值范围，这通常是情况，它们会被提升为 int。
</br>如果不能，它们会被提升为 `unsigned int`。

编译器还会确定`wchar_t`的大小。它以及`char16_t`和`char32_t`，都会提升为第一个足够容纳所有值范围的类型：

> 这里建议去查一下对应变量的含义，读者在这就不过多解释了

1. `int` 
2. `unsigned int`
3. `long` 
4. `unsigned long` 
5. `long long` 
6. `unsigned long long`

```c++
wchar_t c = 'A';
auto i = c + 1; // c从wchar_t提升到至少int类型
DebugLog(i); // 66 (ASCII for 'B')
```

未指定作用域的枚举，如果没有固定的底层类型，也会提升到同一类型的列表中：

```c++
enum Color // 没有基础类型
{
    Red,
    Green,
    Blue
};
 
Color c = Red;
auto i = c + 1; // c 从 'Color' 类型提升至至少 int 类型
DebugLog(i); // 1
```

如果它有一个固定的底层类型，它会被提升到那个类型，然后那个类型可以被提升：

```c++
enum Color : int // Has an underlying type
{
    Red,
    Green,
    Blue
};
 
Color c = Red;
long i = c + 1L; // c从'Color'提升为int类型，然后又提升为long类型
DebugLog(i); // 1
```

位域将被提升到可以容纳位域完整值范围的最小大小，但这是一个简短的列表：

- `int`
- `unsigned int`

```c++
struct ByteBits
{
    bool Bit0 : 1;
    bool Bit1 : 1;
    bool Bit2 : 1;
    bool Bit3 : 1;
    bool Bit4 : 1;
    bool Bit5 : 1;
    bool Bit6 : 1;
    bool Bit7 : 1;
};
 
ByteBits bb{0};
int i = bb.Bit0 + 1; // 位字段从1位提升为int类型
DebugLog(i); // 1
```

`bool`类型被提升为`int`，其中`false`变为0，`true`变为1（不仅仅是非零）。这在C#中是不允许的：

```c++
bool b = true;
int i = b + 1; // b 从布尔类型提升为整型，值为 1
DebugLog(i); // 2
```
`float` 类型被提升为 `double`，它适用于两种语言：

所有其他情况都是数值转换，而不是提升。与提升不同，转换过程中值可能会改变。

首先，是转换为`unsigned integer`。
</br>结果是目标类型位数的最小无符号值对2的n次幂取模，其中n是目标类型的位数。
</br>如果源类型是有符号的，则进行符号扩展或截断。如果它是无符号的，则进行零扩展或截断。
</br>在C#中不允许这样做：

```c++
int32_t si = 257;
uint8_t ui = si; // si从int32_t转换为uint8_t
                 // ui = 257 % 2^8 = 257 % 256 = 1
DebugLog(ui); // 1
```

当转换为有符号整数类型时，如果该值可以在目标类型中表示，则其值不会改变。否则，在C++20之前，该值是实现定义的。
</br>自C++20以来，C++标准要求该值必须像无符号转换一样计算：源值除以2的n次方，其中n是目标类型的位数。
</br>在C#中也不允许这样做：
```c++
uint32_t ui1 = 123;
int8_t si1 = ui1; // ui1从uint32_t转换为int8_t
                  // 由于它可以被int8_t容纳，因此值不会改变
DebugLog(si1); // 1
 
uint32_t ui2 = 257;
int8_t si2 = ui2; // ui2从uint32_t转换为int8_t
                  // 直到C++20为实施定义的值
                  // 自C++20起：
                  // si2 = 257 % 2^8 = 257 % 256 = 1
DebugLog(si2); // C++20中为1，C++17及之前版本中未知，因为是自定义的
```

我们上面看到，当布尔值必须变成整数时，它从false提升为0，从true提升为1。
</br>对于所有其他整数类型，这从技术上讲是一种转换，但它会产生相同的结果。
</br>尽管与上述转换一样没有丢失任何精度，但C#也禁止这种转换：

```c++
bool b = true;
long i = b + 1; // b被从bool转换为long，值为1
DebugLog(i); // 2
```

如果一个浮点类型需要转换为另一个浮点类型，如果这在目标类型中是可能的，则其值将被精确保留。
</br>如果不可能，并且源值在目标类型中的两个值之间，则选择其中一个。通常会选择最接近的值。 否则，转换是未定义的行为。
</br>这也被C#所禁止：

```c++
double d = 3.14f;
float f = d; // d从double转换为float
DebugLog(f); // 3.14
```

当将浮点类型转换为整数类型时，小数部分将被丢弃。
</br>如果值无法容纳，则行为未定义。 与上面不同，对于无符号整数类型不应用模数。
</br>这在C#中也不会起作用：

```c++
float f1 = 3.14f;
int8_t i1 = f1; // f1从float转换为int8_t
                // 被丢弃的小数部分（0.14）
DebugLog(i1); // 3
 
float f2 = 257.0f;
uint8_t i2 = f2; // f2被从float转换为int8_t
                 // 值无法适应。未应用取模运算
                 // 这是不确定的行为！
DebugLog(i2); // 不能打印
```

反过来，整数转换为浮点数的工作方式不同。它们可以被转换为任何浮点类型。
</br>如果浮点类型中可以精确地保留整数的值，则将保留该整数的值。否则，如果整数的值在浮点类型中的两个值之间，则选择这两个值中的一个，通常是最近的值。
</br>否则，整数值将无法适应，这是未定义的行为。 C#也允许这样做。
</br>对于bool类型，我们简单地得到0或1，就像转换为整数类型一样，但在C#中不允许这样做。
```c++
int8_t i = 123;
float f1 = i; // i从int8_t转换为float
DebugLog(f1); // 123
 
bool b = true;
float f2 = b; // b从bool转换为float
DebugLog(f2); // 1
```
在C++中，“空指针常量”是指任何值为0的整型字面量，任何值为0的常量，或者nullptr。这些都可以转换为任何指针类型。在C#中只允许null。

```c++
int* p1 = 0; // 值为0的整数常量被转换为int*
DebugLog(p1); // 0
 
int* p2 = nullptr; // nullptr 转换为 int*
DebugLog(p2); // 0
```

与我们在数组到指针转换中看到的“退化”类似，去掉noexcept和添加const，所有指针类型都转换为void*，因为它是一个“指向任何东西的指针”。
这在两种语言中都是允许的：

```c++
int x = 123;
int* pi = &x;
void* pv = pi; // int* 转换为 void*
DebugLog(pv == pi); // true
```

正如我们在讨论继承时已经看到的，派生类指针可以转换为基类指针。结果是它指向派生类中基类子对象。
</br>同样，C#也允许通过类引用进行这种转换。

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
 
Vector3 vec{};
vec.X = 1;
vec.Y = 2;
vec.Z = 3;
Vector3* pVec3 = &vec;
Vector2* pVec2 = pVec3; // 将Vector3*转换为Vector2*
DebugLog(pVec2->X, pVec2->Y); // 1, 2
```

基类非静态成员的指针也可以转换为派生类非静态成员的指针：

```c++
float Vector2::* pVec2X = &Vector2::X;
float Vector3::* pVec3X = pVec2X; // 指向基类成员的指针被转换为指向派生类成员的指针
Vector3 vec{};
vec.X = 1;
vec.Y = 2;
vec.Z = 3;
Vector3* pVec3 = &vec;
Vector2* pVec2 = pVec3;
DebugLog((*pVec2).*pVec2X, (*pVec3).*pVec3X); // 1, 1
```

请注意，这在虚继承中是不允许的：
```c++
struct Vector2
{
    float X;
    float Y;
};
 
struct Vector3 : virtual Vector2 // Virtual inheritance
{
    float Z;
};
 
float Vector2::* pVec2X = &Vector2::X;
float Vector3::* pVec3X = pVec2X; // 编译器错误
```

最后，所有整数类型、浮点类型、无作用域枚举、指针和成员指针都可以转换为布尔值。
</br>零和空指针变为false，其余都变为true。
</br>C#不允许这些操作。

```c++
int i = 123;
bool b1 = i; // int is converted to bool
DebugLog(b1); // true
 
float f = 3.14f;
bool b2 = f; // float is converted to bool
DebugLog(b2); // true
 
Color c = Red;
bool b3 = c; // Color is converted to bool
DebugLog(b3); // false
 
int* p = nullptr;
bool b4 = p; // int* is converted to bool
DebugLog(b4); // false
 
float Vector2::* pVec2X = &Vector2::X;
bool b5 = pVec2X; // Pointer to member is converted to bool
DebugLog(b5); // true
```

## Conversion Sequences  转换序列

既然我们已经了解了提升和转换，让我们看看它们是如何按顺序进行以改变类型的。
</br>首先，C++有一个“标准转换序列”，它包括以下步骤，这些步骤大部分不适用于C#。

1) 零个或一个从左值到右值的转换，数组到指针的退化，以及函数到指针的转换
2) 零个或一个数字的提升或转换
3) 零个或一个函数指针的转换，包括非抛出到可能抛出（仅允许在C++17及以后版本中）
4) 零个或一个非const到const的转换

C++ 还有一个“隐式转换序列”，包括以下步骤：

1) 零个或一个标准转换序列
2) 零个或一个用户定义的转换
3) 零个或一个标准转换序列

当我们向类的构造函数或用户定义的转换函数传递参数时，我们只能使用标准转换序列。这排除了调用用户定义转换操作符的可能性：

```c++
truct MyClass
{
    MyClass(const int32_t)
    {
    }
};
 
uint8_t i1{123};
MyClass mc{i1}; // 1) 1) 左值转换为右值
                // 2) 从 uin8_t 升级到 uint32_t
                // 3) N/A
                // 4) uint8_t 转换为 'const uint8_t'
 
struct C
{
};
 
struct B
{
    operator C()
    {
        return C{};
    }
};
 
struct A
{
    operator B()
    {
        return B{};
    }
};
 
// 编译错误：此处不允许用户定义的转换运算符
C c = A{};
```

否则，我们可以使用隐式转换序列。这里有一个自动关闭文件但转换为FILE*以便与C++标准库中的各种函数一起使用的类，例如写入文件的fwrite：

```c++
class File
{
    FILE* handle;
 
public:
 
    File(const char* path, const char* mode)
    {
        handle = fopen(path, mode);
    }
 
    ~File()
    {
        fclose(handle);
    }
 
    operator FILE*()
    {
        return handle;
    }
};
 
void Foo()
{
    File writer{"/path/to/file", "w"};
    char msg[] = "hello";
 
    // fwrite 的参数:
    //   std::size_t fwrite(
    //     const void* buffer,
    //     std::size_t size,
    //     std::size_t count,
    //     std::FILE* stream);
 
    // 最后一个参数隐式地从File转换为FILE*
    fwrite(msg, sizeof(msg), 1, writer);
 
    // 注意：在此处调用文件析构函数以关闭文件
}
```

## Overflows  溢出

整数运算可能会导致“溢出”，即结果无法适应整数类型。
《/br》C++没有C#的checked和unchecked上下文。相反，它根据数学运算是有符号还是无符号来以不同的方式处理溢出。

对于有符号数学，溢出是未定义的行为:

```c++
int32_t a = 0x7fffffff;
int32_t b = a + 1; // 溢出。未定义行为！
DebugLog(b); // 可能是任何东西！
```

这并不像看起来那样灾难性。
</br>编译器通常会只是生成一个加法指令，并且溢出会根据CPU架构的溢出规则来处理。
</br>只有在编译器可以证明会发生有符号整数溢出的情况下，比如这个例子，才可能会生成意外的CPU指令。它还可能会生成编译器警告，以引起程序员的注意。
</br>通常在C#和C++中，结果最终会相同，但从技术上讲并不一定如此。无符号数学更加宽容。溢出只是执行模2n的操作，其中n是整数类型的位数。

无符号数学更加宽容。溢出操作只是简单地执行模2^n，其中n是整数类型的位数：

```c++
uint8_t a = 255;
uint8_t b = a + 1; // Overflow. b = (255 + 1) % 256 = 0.
DebugLog(b); // 0
```

## Arithmetic  算术

我们已经看到了由于算术而导致的很多提升和转换，但到目前为止只涵盖了简单的案例。
</br>还有许多更多规则用于确定哪些操作数会被提升或转换，以及应该对什么“公共类型”进行算术运算。

首先，C++20 废弃了浮点数类型与枚举类型或枚举类型与其他枚举类型的混合。这些在 C# 中从未被允许。

```c++
enum Color
{
    Red,
    Green,
    Blue
};
 
// 已弃用：混合枚举和浮点数
auto a = Red + 3.14f;
 
enum RangeType
{
    Melee,
    Distance
};
 
// 已弃用：混合枚举类型
auto b = Red + Melee;
```

整数首先提升。然后，对于除了位移运算符之外的所有二元运算符，会发生一系列特定的类型变化。
首先，如果任一操作数是`long double`，则另一个操作数转换为`long double`。对于`double` 和 `float`也是同样的情况。C#的行为本质上相同。

```c++
int i = 123;
long double ld = 3.14;
long double sum1 = ld + i; // i 被从 int 转换为 'long double'
DebugLog(sum1); // 126.14
 
double d = 3.14;
double sum2 = d + i; // i 被从 int 转换为 'double'
DebugLog(sum2); // 126.14
 
float f = 3.14f;
double sum3 = f + i; // i 被从 int 转换为 'float'
DebugLog(sum3); // 126.14
```

对于有符号和无符号整数，我们需要考虑所涉及类型的“转换等级”：

1. `bool` 
2. `signed char`, `unsigned char`, and `char` 
3. `short` and `unsigned short` 
4. `int` and `unsigned int` 
5. `long` and `unsigned long` 
6. `long long` and `unsigned long long`

`char8_t`、`char16_t`、`char32_t`和`wchar_t`的转换等级与它们的底层类型相同，这取决于编译器、操作系统和CPU架构等因素。

考虑到这一点，如果两个操作数都是有符号或无符号的，则将转换等级较低的操作数转换为转换等级较高的操作数的类型。C#的行为方式相同。

```c++
unsigned char uc = 'A'; // 转换等级 = 2
unsigned long ul = 1; // 转换等级 = 5
 
// uc的转换等级较低，因此它被转换为无符号长整型
unsigned long sum = uc + ul;
DebugLog(sum); // 66 (ASCII for 'B')
```

另一种情况，一个操作数是有符号的，另一个是无符号的。
</br>C# 不允许这样做，但C++可以。
</br>在这种情况下，如果无符号操作数的转换等级大于或等于有符号操作数的转换等级，则将有符号操作数转换为无符号操作数的类型：

```c++
short s = 123; // 转换等级 = 3
unsigned long ul = 1; // 转换等级 = 5
 
// s的转换等级较低，因此它被转换为无符号长整型
unsigned long sum = s + ul;
DebugLog(sum); // 124
```

如果情况不是这样，但有符号类型可以表示无符号类型的所有值，则无符号操作数转换为有符号操作数的类型：

```c++
long l = 123; // 转换等级 = 5
unsigned short us = 1; // 转换等级 = 3
 
// l的转换等级更高，而long可以表示所有'unsigned short'的值，因此us被转换为long
long sum = l + us;
DebugLog(sum); // 124
```

如果情况也不是这样，那么两个操作数都会转换为有符号类型的无符号对应类型：

```c++
long l = 123; // 转换等级 = 5
unsigned int ui = 1; // 转换等级 = 4
 
// 假设int和long都是4字节（例如Windows系统）
// l的转换等级更高，但long无法表示所有'unsigned int'的值，因此l从long转换为unsigned long，而ui从unsigned int转换为unsigned long
unsigned long sum = l + ui;
DebugLog(sum); // 124
```

## Narrowing Conversions  缩小转换范围

到目前为止，我们已经看到类型要么保持大小不变，要么变得更大。
</br>有时，类型会变得更小或者以其他方式失去精度。这些被称为“缩窄”转换，并且它们是我们上面看到的隐式转换的一个子集。
</br>C#从不允许这些转换，但C++可以。
</br>例如，从浮点数到整数的转换会通过截断小数部分来失去精度：
```c++
float f = 3.14f;
int i = f;
DebugLog(i); // 3
```

从更大的浮点类型转换为更小的类型也可能丢失精度：

```c++
long double ld1 = 3.14;
double d1 = ld1; // 'long double' -> double
DebugLog(d1); // 3.14
 
long double ld2 = 3.14;
float f1 = ld2; // 'long double' -> float
DebugLog(f1); // 3.14
 
double d2 = 3.14;
float f2 = d2; // double -> float
DebugLog(d2); // 3.14
```

如果浮点类型不能完全表示整数，则从整数或枚举转换为浮点类型也可能会丢失精度：

```c++
uint64_t i = 0xffffffffffffffff;
float f = i; // uint64_t -> float
DebugLog(f); // 1.84467e+19
uint64_t i2 = f;
DebugLog(i == i2); // false
```

这些缩窄转换可以通过我们在这里看到的复制初始化允许，但由聚合和列表初始化禁止：

```c++
// 编译器错误：使用缩窄初始化聚合
int i1{3.14f};
IntHolder i2{3.14f};
int i3 = {3.14f};
 
// 编译器错误：列表初始化类型缩窄
int i4[] = { 3.14f, 3.14f };
 
// OK: 复制初始化类型缩窄
int i5 = 3.14f; // copy
int i6(3.14f); // copy
```

避免类型缩小是更喜欢使用花括号初始化的另一个原因。

## Conclusion  结论

这标志着对C++隐式类型转换系统的深入研究结束。它比C#要宽松得多。这使我们的代码更加简洁，但也使我们容易受到许多潜在错误的影响。
</br>因此，了解从数值提升到转换等级再到溢出处理的全部规则非常重要。








