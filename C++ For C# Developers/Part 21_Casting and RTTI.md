# C++ For C# Developers: Part 21 – Casting and RTTI


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5845),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

现在我们已经看到了C++中类型是如何隐式转换的，我们也可以看到它们是如何通过类型转换来显式转换的。
</br>C++提供了比C#更多种类的类型转换来控制转换过程。
</br>其中之一——`dynamic_cast`——引入了运行时类型信息（RTTI）的概念，因此我们今天也会探讨这一点。

## const_cast

C# 只有一种类型转换：(DestinationType)sourceType。由于它是唯一的选择，它必须能够处理所有可能的转换原因。
</br>C++采取了不同的方法。它提供了一系列转换，我们可以从中选择以适应特定转换操作的意图。

在这个系列中，第一个转换操作是最简单的一种：const_cast。我们使用它来简单地处理一个const指针或引用，使其变为非const：

```c++
// 从指针中移除const
int x = 123;
int const * constPtr = &x;
int* nonConstPtr = const_cast<int*>(constPtr);
*nonConstPtr = 456;
DebugLog(x); // 456
 
// 从引用中移除const
int const & constRef = x;
int& nonConstRef = const_cast<int&>(constRef);
nonConstRef = 789;
DebugLog(x); // 789
 
// 可以安全地转换null
constPtr = nullptr;
int* nullPtr = const_cast<int*>(constPtr);
DebugLog(nullPtr); // null
```

请注意，函数指针和成员函数指针不能使用`const_cast`。

在前两个例子中，我们使用了`const_cast`来获取非`const`指针和引用，然后修改了它们所指向的值：x。
</br>这是完全可以接受的，因为x实际上并不是`const`。
</br>然而，如果我们修改了一个实际上是`const`的变量，那么这就是未定义的行为：

```c++
int const x = 123;
int const & constRef = x;
int& nonConstRef = const_cast<int&>(constRef);
nonConstRef = 789; // 未定义行为：修改了const变量x
DebugLog(x); // 可能是任何东西！
```

那么const_cast实际上做什么呢？它实际上只是一个编译时操作，将表达式重新分类为非const。
</br>因为CPU没有const的概念，所以不会生成CPU指令。从这个意义上说，const_cast在性能方面是“免费”的。

## reinterpret_cast

下一种类型转换是 reinterpret_cast。这是一种“免费”的类型转换，不会生成CPU指令。
</br>相反，它只是告诉编译器将一个表达式的类型“重新解释”为另一种类型。

我们只能在特定情况下使用这个。首先，只要整数足够大，可以容纳所有可能的指针值，指针就可以转换为整数，反之亦然：

```c++
// Pointer -> Integer
int x = 123;
int* p = &x;
uint64_t i = reinterpret_cast<uint64_t>(p);
DebugLog(i); // x的内存地址
 
// Integer -> Pointer
int* p2 = reinterpret_cast<int*>(i);
*p2 = 456;
DebugLog(x); // 456
```

我们还可以将 `nullptr` 重新解释为整数 `0`：

```c++
uint64_t i = reinterpret_cast<uint64_t>(nullptr);
DebugLog(i); // 0
```

更常见的是，我们可以将一种指针重新解释为另一种指针：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
struct Point2
{
    float X;
    float Y;
};
 
Point2 point{2, 4};
Point2* pPoint = &point;
Vector2* pVector = reinterpret_cast<Vector2*>(pPoint);
DebugLog(pVector->X, pVector->Y); // 2, 4
```

为了安全地使用生成的指针，我们必须采取一些预防措施。首先，CPU对各种数据类型，如浮点数，有对齐要求。目标类型的对齐要求不能比源类型的对齐要求更严格。
</br>了解我们目标CPU架构的对齐要求并确保我们负责任地使用 reinterpret_cast 是我们的责任。

C++标准指出，除了某些特定情况外，使用 reinterpret_cast 的结果是未定义的行为。
</br>第一种情况是如果它们是“相似的”。 这被定义为是同一类型、指向同一类型的指针、指向同一类成员且这些成员相似，或者大小相同（或一个具有未知大小）且元素相似的数组。
</br>以下是一些示例：

```c++
// 相似：指向同一类型的指针
int x = 123;
int* p = reinterpret_cast<int*>(&x); // int* -> int*
DebugLog(*p); // 123
 
// 相似：具有相同维度和相同类型元素的数组
int a1[3]{1, 2, 3};
int (&a)[3] = reinterpret_cast<int(&)[3]>(a1);
DebugLog(a[0], a[1], a[2]); // 1, 2, 3
 
// 不同类型的指针：不相似
float* pFloat = reinterpret_cast<float*>(p);
DebugLog(*pFloat); // 未定义行为
```

因为未定义行为可以由编译器按其选择的任何方式实现，所以许多编译器的限制比C++标准要求的要宽松。
</br>特别是，将`int*`转换为`float*`的最后一个例子通常被编译器允许，作为一种类似于我们之前在联合中看到的“类型欺骗”方式。

如果类型不是“相似”的，那么我们还有两个机会来避免未定义行为。首先，如果一种类型是相同类型的有符号或无符号版本：

```c++
// int* -> unsigned int*
int x = 123;
unsigned int* p = reinterpret_cast<unsigned int*>(&x);
DebugLog(*p); // 123
```

其次，如果我们将其重新解释为char、unsigned char，或者在C++17及以后版本中，std::byte 。
</br>这些操作是特别允许的，这样我们就可以查看对象的字节表示，例如用于磁盘或网络的序列化：

```c++
// 打印Vector2的字节
Vector2 vec{2, 4};
char* p = reinterpret_cast<char*>(&vec);
DebugLog(p[0], p[1], p[2], p[3], p[4], p[5], p[6], p[7]);
```

## static_cast

接下来，我们有了第一个可以生成CPU指令的类型转换：`static_cast`。编译器会检查一系列条件以决定`static_cast`应该做什么。
首先检查的是是否存在从源类型到目标类型的隐式转换序列，或者目标类型是否可以直接从源类型初始化。
如果这两种情况中的任何一种成立，`static_cast`的行为就像我们写了`DestType tempVariable(sourceType)`:

```c++
struct File
{
    FILE* handle;
 
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
 
File reader{"/path/to/file", "r"};
FILE* handle = static_cast<FILE*>(reader); // 隐式转换
```

接下来，它会检查我们是否是从基类指针或引用向下转换为（非虚）派生类指针或引用

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
 
Vector3 vec;
vec.X = 2;
vec.Y = 4;
vec.Z = 6;
Vector2& refVec2 = vec; // 从Vector3&到Vector2&的隐式转换
Vector3& refVec3 = static_cast<Vector3&>(refVec2); // Downcast
DebugLog(refVec3.X, refVec3.Y, refVec3.Z); // 2, 4, 6
```

我们还可以使用`static_cast`到`void`来显式地丢弃一个值。这有时被用来消除“未使用变量”的编译器警告：

```c++
Vector2 vec{2, 4};
static_cast<void>(vec); // 丢弃抛出的异常结果
```

如果存在从目标类型到源类型的标准转换，`static_cast`将会反转它。
但它不会反转任何左值到右值的转换、数组到指针的退化、函数到指针的转换、函数指针转换或布尔值转换。

```c++
int i = 123;
float f = static_cast<int>(i); // 撤销标准转换：整型 -> 浮点型
DebugLog(f); // 123
```

我们还可以使用 `static_cast` 显式执行一些隐式转换：左值到右值、数组到指针的衰减和函数到指针：

```c++ 
// 左值转换为右值
int i = 123;
int i2 = static_cast<int&&>(i);
DebugLog(i2); // 123
 
// 数组到指针退化
int a[3]{1, 2, 3};
int* p = static_cast<int*>(a);
DebugLog(p[0], p[1], p[2]); // 1, 2, 3
 
// 函数到指针的转换
void SayHello()
{
    DebugLog("hello");
}

void (*pFunc)() = static_cast<void(*)()>(SayHello);
pFunc(); // hello
```

使用 static_cast 可以将范围枚举转换为整数或浮点类型。自 C++20 以来，这就像枚举的底层类型到目标类型的隐式转换。
</br>在此之前，转换为 bool 的处理方式不同，因为只有 0 会变成 false，而其他所有值都会变成 true。

```c++
enum class Color
{
    Red,
    Green,
    Blue
};
 
Color green{Color::Green};
 
// Scoped enum -> int
int i = static_cast<int>(green);
DebugLog(i); // 1
 
// Scoped enum -> float
float f = static_cast<float>(green);
DebugLog(f); // 1
```

我们也可以反过来：整数和浮点类型可以被`static_cast`转换为有作用域或无作用域的枚举。我们还可以在枚举类型之间进行转换：

```c++
// Integer -> enum
int i = 1;
FgColor g1 = static_cast<FgColor>(i);
DebugLog(g1); // Green
 
// Floating point -> enum
float f = 1;
FgColor g2 = static_cast<FgColor>(f);
DebugLog(g2); // Green
 
// Cast between enum types
FgColor g3{FgColor::Green};
BgColor g4 = static_cast<BgColor>(g3);
DebugLog(g4); // Green
```

如果枚举的底层类型不是固定的，并且要转换到枚举的值超出了其范围，则这种行为是未定义的。
</br>如果它是固定的，结果就像转换到其底层类型一样。浮点值首先转换为底层类型。

我们还可以使用 `static_cast` 从指向派生类中成员的指针向上转换为指向基类中成员的指针：

```c++
float Vector3::* p1 = &Vector3::X;
float Vector2::* p2 = static_cast<float Vector2::*>(p1);
Vector3 vec;
vec.X = 2;
vec.Y = 4;
vec.Z = 6;
DebugLog(vec.*p1, vec.*p2); // 2, 2
```

最后，`static_cast`可以像`reinterpret_cast`一样用于将`void*`转换为任何其他指针类型。关于对齐和类型相似性的注意事项同样适用。

## C-Style Cast and Function-Style Cast

“C-style”的转换在C和C#中看起来都像C中的转换：(DestinationType)sourceType。
</br>在C++中，它与C#的行为相当不同。
</br>在C++中，它主要是一个简写，用于满足以下顺序的第一个“命名”转换：

1. `const_cast<DestinationType>(sourceType)`
2. `static_cast<DestinationType>(sourceType)`
   </br>具有更宽松的限制：
   </br>派生类、派生类成员的指针和引用 -> 基类、基类成员的指针和引用 
3. `static_cast (with more leniency)` then `const_cast`
4. `reinterpret_cast<DestinationType>(sourceType)`
5. `reinterpret_cast` then `const_cast`

```c++
// Uses const_cast (#1)
int const i1 = 123;
int i2 = (int)i1;
DebugLog(i2); // 123
 
// Uses static_cast (#2)
Vector2 vec{2, 4};
Vector3* pVec = (Vector3*)&vec;
DebugLog(pVec->X, pVec->Y); // 2, 4 (undefined behavior to use Z!)
 
// Uses static_cast then const_cast (#3)
Vector2 const * pConstVec = &vec;
Vector3* pVec3 = (Vector3*)pConstVec;
DebugLog(pVec3->X, pVec3->Y); // 2, 4 (undefined behavior to use Z!)
 
// Uses reinterpret_cast (#4)
float* f1 = (float*)&i2;
DebugLog(*f1); // 1.7236e-43
 
// Uses reinterpret_cast then const_cast (#5)
float* f2 = (float*)&i1;
DebugLog(*f2); // 1.7236e-43
```

函数风格的转换与C风格转换类似。它看起来像函数调用，甚至要求类型只有一个单词：`int`而不是`unsigned int`。
</br>注意不要将其误认为是函数调用或类初始化。

```c++
int i = 123;
float f = float(i);
DebugLog(f); // 123
```
> 合着写这么多最后又绕回来了(笑)，那读者先总结一下
> 
> | 特性   | const_cast | static_cast              | reinterpret_cast |
> |------|------------|--------------------------|------------------|
> | 主要作用 | 移除const    | 兼容性转换</br>(参考上一篇文章的转换规则) | 指针与整数、不同类型指针之间的转 |
> | 注意事项 | 自己看上面文章    | 自己看上面文章                  | 自己看上面文章          |
 
## dynamic_cast

到目前为止，我们所看到的所有转换都是“静态的”。
</br>这意味着它们的操作方式是在编译时确定的，并且不依赖于被转换表达式的运行时值。
</br>例如，考虑这个向下转换：

```c++
void PrintZ(Vector2& vec)
{
    // Downcast
    Vector3& refVec3 = reinterpret_cast<Vector3&>(vec);
 
    // 如果vec不是真正的Vector3，则会出现未定义行为
    DebugLog(refVec3.X, refVec3.Y, refVec3.Z);
}
```

记住，reinterpret_cast不会生成CPU指令。
</br>这意味着编译器不会生成任何检查vec是否真的是Vector3的CPU指令。
</br>如果是，这段代码运行良好。如果不是，读取Z将会读取内存中Vector2之后的四个字节。这几乎肯定不是一个有效的Z值，当我们以这种方式使用它时，将会在程序逻辑中引起严重的错误。
</br>这同样是未定义的行为，因此编译器可能会生成一些令人惊讶的CPU指令，比如完全跳过读取和打印Z的操作。

为了解决这个问题，C++有一个称为`dynamic_cast`的“安全”转换。它的工作方式与C#的唯一转换非常相似。

与static_cast类似，会进行一系列检查以决定CPU应该做什么。首先，我们可以进行同一类型的转换或添加const：

```c++
// 转换为相同类型
Vector2 v{2, 4};
Vector2& r1 = v;
Vector2& r2 = dynamic_cast<Vector2&>(r1);
DebugLog(r2.X, r2.Y); // 2, 4
 
// 转换为添加const
Vector2 const & r3 = dynamic_cast<Vector2 const &>(r1);
DebugLog(r3.X, r3.Y); // 2, 4
```

其次，如果值是null，则结果也是null：

```c++
Vector2* p1 = nullptr;
Vector2* p2 = dynamic_cast<Vector2*>(p1);
DebugLog(p2); // 0
```

第三，我们可以从派生类的指针或引用向上转换为基类的指针或引用：

```c++
Vector3 vec;
vec.X = 2;
vec.Y = 4;
vec.Z = 6;
Vector3& r3 = vec;
Vector2& r2 = dynamic_cast<Vector2&>(r3);
DebugLog(r2.X, r2.Y); // 2, 4
```

第四，我们可以将具有至少一个虚函数的类的指针转换为`void*`，这样我们就会得到指向该指针所指向的最派生对象的指针：

```c++
struct Combatant
{
    virtual ~Combatant()
    {
    }
};
 
struct Player : Combatant
{
    int32_t Id;
};
 
Player player;
player.Id = 123;
Combatant* p = &player;
void* pv = dynamic_cast<void*>(p); // 向下转型为最派生类：Player*
Player* p2 = reinterpret_cast<Player*>(pv);
DebugLog(p2->Id); // 123
```

最后，我们有了dynamic_cast的主要用途： 从基类指针或引用到派生类指针或引用的向下转型。
这会生成CPU指令来检查要转型的表达式所指向或引用的对象。
如果该对象确实是目标类型的基类，并且目标类型只有一个基类的子对象（在非虚继承的情况下可能不是这样），那么转型就会成功，得到派生类的指针或引用：

```c++
Player player;
player.Id = 123;
Combatant* p = &player;
Player* p2 = dynamic_cast<Player*>(p); // Downcast
DebugLog(p2->Id); // 123
```

这也可以用来从一个基类向另一个基类执行“侧向转换”：

```c++
struct RangedWeapon
{
    float Range;
 
    virtual ~RangedWeapon()
    {
    }
};
 
struct MagicWeapon
{
    enum { FireType, WaterType, ArcaneType } Type;
};
 
struct Staff : RangedWeapon, MagicWeapon
{
    const char* Name;
};
 
Staff staff;
staff.Name = "Staff of Freezing";
staff.Range = 10.0f;
staff.Type = MagicWeapon::WaterType;
 
Staff& staffRef = staff;
RangedWeapon& rangedRef = staffRef; // 隐式类型提升
MagicWeapon& magicRef = dynamic_cast<MagicWeapon&>(rangedRef); // Sidecast
DebugLog(magicRef.Type); // 1
```

如果向下转型或侧转型都不成功，转型将失败。
</br>当指针被转型时，转型结果将评估为目标类型的空指针。如果引用被转型，将抛出 `std::bad_cast` 异常：

```c++
struct Combatant
{
    virtual ~Combatant()
    {
    }
};
 
struct Player : Combatant
{
    int32_t Id;
};
 
struct Enemy : Combatant
{
    int32_t Id;
};
 
// 创建Combatant 作为基类
Combatant combatant;
Combatant* pc = &combatant;
Combatant& rc = combatant;
 
// 转化失败，因为 Combatant 不是一个 Player ，返回空 
Player* pp = dynamic_cast<Player*>(pc);
DebugLog(pp); // 0
 
try
{
    // 转化失败，因为 Combatant 不是一个 Player ， std::bad_cast 抛出.
    Player& rp = dynamic_cast<Player&>(rc);
    DebugLog(rp.Id); // 永远不会调用
}
catch (std::bad_cast const &)
{
    DebugLog("cast failed"); // 打印"cast failed"
}
```

> 这里转化失败的原因：可以理解为这个combatant就是基类，和Player八竿子打不着
> </br>之前可以转化成功是因为之前转化的变量是类似于父类装子类的情况

请注意，在构造函数中使用`dynamic_cast`是未定义的行为，除非目标类型是相同的类类型或基类类型。我们将在下一节中看到原因。

## Run-Time Type Information

为了实现`dynamic_cast`，编译器必须生成所谓的运行时类型信息（RTTI）。
这个信息的具体格式是编译器特定的，但编译器将生成数据，以便在运行时由`dynamic_cast`使用，以确定特定对象的类型。

由于 `dynamic_cast` 仅适用于至少具有一个 `virtual function` 的类型，因此它可以利用对象的 virtual function table 或 “vtable”。
这是编译器为类的所有虚拟函数生成的函数指针数组。将为继承层次结构中的每个类生成一个表。
指向表的指针（称为“虚拟表指针”或“vpointer”）将添加为层次结构中所有类的数据成员，并在构造期间进行初始化。

因此，这个虚表指针也可以用来识别对象的类，因为每个类都有一个虚函数表。继承层次结构因此可以概念上表示为一个虚表指针的树，其实现细节因编译器而异。

因为所有这些RTTI数据都会增加可执行文件的大小，所以许多编译器允许禁用它。这也禁用了`dynamic_cast`，因为它依赖于RTTI。

## typeid  类型 ID

RTTI还有另一个用途：typeid运算符。它用于获取类型信息，类似于C#中的typeof或GetType。
</br>操作数可以是静态命名的，类似于C#中的typeof，或者动态的，类似于C#中的GetType，根据对象的值查找类型。
</br>使用此功能需要C++标准库中的<typeinfo>头文件。

```c++
// 基于类型的静态使用
std::type_info const & ti1{typeid(Combatant)};
 
// 基于变量的动态使用虚拟函数
Enemy enemy;
std::type_info const & ti2{typeid(enemy)};
 
// 基于无虚函数的变量的动态使用
// 等同于静态使用：typeid(int)
int i = 123;
std::type_info const & ti3{typeid(i)};
```

它具有几个有用成员的const std::type_info

- `operator==(const std::type_info&)` and `operator!=(const std::type_info&)` 用于比较type
- `std::size_t hash_code()` 获取给定类型的哈希值
- `const char* name()` 获取类型的字符串名称

当使用`typeid`对一个空指针进行操作时，会抛出`bad_typeid`异常：

```c++
Enemy* pe = nullptr;
try
{
    // 不取消引用null，而是尝试获取pe指向的类型信息
    std::type_info const & ti{typeid(*pe)};
 
    // 不会打印
    DebugLog(ti.name());
}
catch (std::bad_typeid const &)
{
    DebugLog("bad typeid call"); // 打印bad typeid call
}
```

另一个是，name成员函数不返回任何特定的字符串。这个字符串通常是一些编译器特定的代码，可能或可能不包含源代码中的类型名称：

```c++
// 这些都会因编译器的不同而有所差异
DebugLog(typeid(int).name()); // i
DebugLog(typeid(long).name()); // l
DebugLog(typeid(Enemy).name()); // 5Enemy
```
还有一个问题是，即使是相同类型的std::type_info，一个调用得到的对象可能与另一个调用得到的对象不同。 应该使用hash_code成员函数来代替：

```c++
DebugLog(&typeid(int) == &typeid(int)); // 可能为true
DebugLog(typeid(int).hash_code() == typeid(int).hash_code()); // 永远为true
```

## Conclusion  结论

在比较这两种语言时，C++提供了许多选项，而C#则只提供少数几个。
</br>在类型转换方面，C++提供了多种命名的、C风格的和函数风格的类型转换，用于特定目的
</br>而C#基本上只提供`dynamic_cast`。

当适当地使用时，这可以使许多转换“免费”，因为不会生成CPU指令，也不会增加可执行文件的大小。
</br>当不恰当地使用时，未定义的行为可能会导致严重的错误，例如崩溃和数据损坏。作为程序员，了解转换规则并明智地选择适合我们任务的适当转换取决于我们。
</br>即使在C#中抛出异常的情况下，粗心转换的后果也确实要求我们在任何语言和转换类型中都要谨慎行事。

> 最后总结一下这四种转化方式：
>
> | 特性   | const_cast | static_cast              | reinterpret_cast | dynamic_cast |
> |------|------------|--------------------------|------------------|--------------|
> | 主要作用 | 移除const    | 兼容性转换</br>(参考上一篇文章的转换规则) | 指针与整数、不同类型指针之间的转 | 多态类型转化       |
> | 注意事项 | 自己看上面文章    | 自己看上面文章                  | 自己看上面文章          | 自己看上面文章      |
> | 使用场景 | 移除const    | 基础类型转换、向上转型              | 底层编程、硬件操作        | 向下转型、交叉转换    |
















