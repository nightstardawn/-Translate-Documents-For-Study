# C++ For C# Developers: Part 18 – Exceptions

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5814),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

到目前为止，我们C++代码分配的所有内存要么是全局的，要么是在栈上的。对于许多在编译时不知道内存数量的情况，我们需要动态分配。
</br>今天我们将介绍`new`和`delete`的基础知识，同时也会深入探讨一些高级的C++特性，例如重载`new`和“放置”`new`。

## History and Strategy  历史与战略

让我们先回顾一下与当今C++编程仍然非常相关的历史片段。
</br>在C语言中，而不是C++，内存是通过C标准库中一系列以`alloc`结尾的函数动态分配的：

```c++
// 动态分配4字节
void* memory = malloc(4);
 
// 检查分配失败
// 对于小分配（例如4字节）不必要
// 对于大分配（例如数组）需要
if (memory != NULL)
{
    // 将其转换为指向整数的指针
    int* pInt = (int*)memory;
 
    // 读取内存
    // 这是一种未定义的行为：内存尚未初始化！
    DebugLog(*pInt);
 
    // 释放内存
    free(memory);
 
    // 写入内存
    // 这是一种未定义的行为：内存已被释放！
    *pInt = 123;
 
    // 再次释放内存
    // 这是一种未定义的行为：内存已经被释放了！
    free(memory);
}
```

上述代码中的三个错误是非常常见的错误。
</br>这种像“原始”一样使用malloc和free的方式在C++代码库中仍然很常见。
</br>然而，这实际上是一种相当低级的工作方式，在大多数C++代码库中通常是不被推荐的。
</br>这是因为很容易意外触发未定义的行为。

高级动态内存分配方法使得这些错误要么更难发生，要么根本不可能发生。
</br>例如，在C#中，由于一切都被设置为零，所以无法获取未初始化的内存，无法拥有已释放内存的引用
</br>因为只有在最后一个引用被放弃后才会释放，而且无法重复释放内存，因为这是由垃圾回收器处理的。

C++并没有像C#那样采取如此高级的方法，因为上述的C代码也是合法的C++代码。
</br>然而，它确实为大多数情况下安全比完全控制更可取的情况提供了许多高级功能。

## Allocation  分配

C++中的new运算符在概念上与在C#中使用new运算符处理类相似。
它动态分配内存，初始化它，并评估一个指针：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2()
        : X(0), Y(0)
    {
    }
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
};
 
// 1) 为Vector2分配足够的内存：sizeof(Vector2)
// 2) 调用构造函数
//    * “this”是指分配的内存
//    * 将2和4作为参数传递
//3) 计算结果为Vector2*
Vector2* pVec = new Vector2{2, 4};
 
DebugLog(pVec->X, pVec->Y); // 2, 4
```

`new`操作符结合了C代码中的多个手动步骤，这样我们就不容易忘记执行它们或意外地做错。因此，安全性在多个方面得到了提高：

- 分配的内存量是由编译器计算的，因此总是正确的
- 分配的内存总是被初始化（即通过构造函数），因此我们不能在初始化之前使用它
- 分配的内存总是被转换为正确的指针类型
- 分配失败总是被处理的（下面将进一步说明）

C# 允许我们将 `new` 与类和结构一起使用。在 C++ 中，我们可以对任何类型使用 `new`：

```c++
// 动态分配一个基本类型
int* pInt = new int{123};
DebugLog(*pInt); // 123
 
// 动态分配枚举
enum class Color { Red, Green, Blue };
Color* pColor = new Color{Color::Green};
DebugLog((int)*pColor); // 1
```

我们也可以使用`new`来分配对象的数组。就像其他数组一样，每个元素都会被初始化：

```c++
// 动态分配一个包含三个Vector2的数组
// 对每个元素调用默认构造函数
Vector2* vectors = new Vector2[3]();
 
DebugLog(vectors[0]->X, vectors[0]->Y); // 0, 0
DebugLog(vectors[1]->X, vectors[1]->Y); // 0, 0
DebugLog(vectors[2]->X, vectors[2]->Y); // 0, 0
```

这与C#中的数组在两个方面不同。
- 首先，它不是一个托管对象，因为C++没有垃圾回收器。
- 其次，它是一个Vector2对象的数组，而不是Vector2对象的引用。它更像是一个C#结构体的数组，而不是C#类的数组：对象在内存中是顺序排列的。

## Initialization  初始化

无论类型如何，`new` 总是执行相同的步骤：分配、初始化，然后评估为指针。
</br>初始化由我们在类型名称之后放置的内容控制：无、括号或花括号。
</br>如果我们什么都不放，对象或对象数组将进行默认初始化：

```c++
// 调用类的默认构造函数
Vector2* pVec1 = new Vector2;
DebugLog(pVec1->X, pVec1->Y); // 0, 0
 
// 对原始数据类型不做任何操作
int* pInt1 = new int;
DebugLog(*pInt1); // 未定义行为：int未初始化
 
// 调用类的默认构造函数
Vector2* vectors1 = new Vector2[1];
DebugLog(vectors1[0].X, vectors1[0].Y); // 0, 0
 
// 对原始数据类型不做任何操作
int* ints1 = new int[1];
DebugLog(ints1[0]); // 未定义行为：整数未初始化
```

如果我们加上括号，单个对象就会进行直接初始化：


```c++
// 调用 (float, float) 构造函数
Vector2* pVec2 = new Vector2(2, 4);
DebugLog(pVec2->X, pVec2->Y); // 2, 4
 
// 设置为123
int* pInt2 = new int(123);
DebugLog(*pInt2); // 123
```

数组的括号必须为空。这将对数组进行聚合初始化：

```c++
// 调用类的默认构造函数
Vector2* vectors2 = new Vector2[1]();
DebugLog(vectors2[0].X, vectors2[0].Y); // 0, 0
 
// 设置为零
int* ints2 = new int[1]();
DebugLog(ints2[0]); // 0
```

花括号列表初始化单个对象：

```c++
// 调用 (float, float) 构造函数
Vector2* pVec3 = new Vector2{2, 4};
DebugLog(pVec3->X, pVec3->Y); // 2, 4
 
// 设置为123
int* pInt3 = new int{123};
DebugLog(*pInt3); // 123
```

它们可以聚合初始化数组，并且可以是非空的，例如传递参数给构造函数或将原始数据类型设置为某个值。这通常是数组和单个对象推荐的形式：

```c++
// 为每个元素调用(float, float)构造函数
Vector2* vectors3 = new Vector2[1]{2, 4};
DebugLog(vectors3[0].X, vectors3[0].Y); // 2, 4
 
// 将每个元素设置为123
int* ints3 = new int[1]{123};
DebugLog(ints3[0]); // 123
```

当内存分配失败时，例如当 there's insufficient 时，`new` 将抛出 `std：：bad_alloc` 异常：

```c++
try
{
    // 尝试进行1TB的分配
    // 如果分配失败则抛出异常
    char* big = new char[1024*1024*1024*1024];
 
    // 如果分配失败则不会执行
    big[0] = 123;
}
catch (std::bad_alloc)
{
    // 如果分配失败，将打印此信息
    DebugLog("Failed to allocate big array");
}
```

一些代码库，尤其是在游戏中，更喜欢避免异常。编译器通常提供调用 `std：：abort` 来使程序崩溃的选项，即使这在技术上违反了 C++ 标准：

```c++
// 尝试进行1TB的分配
// 如果分配失败，则调用abort()函数
char* big = new char[1024*1024*1024*1024];
 
// 如果分配失败则不会执行
big[0] = 123;
```

## Deallocation  释放

以上所有示例都会产生内存泄漏。
</br>这是因为 C++ 没有垃圾回收器来自动释放不再引用的内存。
</br>相反，我们必须在完成后释放内存。我们使用 delete 运算符来做到这一点：

```c++
Vector2* pVec = new Vector2{2, 4};
DebugLog(pVec->X, pVec->Y); // 2, 4
 
// 1) 调用 Vector2 析构函数
// 2) 释放 pVec 指向的已分配内存
delete pVec;
 
DebugLog(pVec->X, pVec->Y); // 未定义行为：内存已被释放
 
delete pVec; // 未定义行为：内存已被释放
```

删除操作通过合并两个步骤来进一步确保安全性：先对内存内容进行解初始化，然后进行释放。
</br>然而，它并不能防止示例末尾出现的两个错误：“释放后使用”和“重复释放。”

解决这些问题的方法之一是在释放它们之后将所有指向该内存的指针设置为null：

```c++
delete pVec;
pVec = nullptr;
 
// 未定义行为：空指针解引用
DebugLog(pVec->X, pVec->Y);
 
delete pVec; // OK
```

在“释放后使用”的情况下，我们解引用空指针的行为仍然是未定义的行为。如果编译器能够确定这一点，它可以生成任何它想要的机器代码。
</br>它可能只是简单地解引用空指针导致崩溃，或者做一些奇怪的事情，比如完全删除DebugLog行。

大多数情况下，例如在代码库的某个偏远部分使用空指针时，编译器无法确定它是空的，并会假设它是一个非空指针。在这种情况下，解引用空指针会导致程序崩溃。
</br>因此，这仅仅是一个中等程度的改进，因为我们可能只能得到崩溃而不是从读取或写入已释放内存中的数据损坏。

在“双重释放”的情况下，删除null是允许的，所以这个问题现在不再存在了。

因为`Vector2*`可能是指向单个`Vector2`或`Vector2对象数组`的指针，所以存在第二种形式的`delete`，用于调用数组中所有元素的析构函数：

```c++
Vector2* pVectors1 = new Vector2[3]{Vector2{2, 4}};
// 正确调用：
// 1) 调用所有三个向量2的析构函数
// 2) 释放pVectors1指向的已分配内存
delete [] pVectors1;
 
Vector2* pVectors2 = new Vector2[3]{Vector2{2, 4}};
// 错误调用：
// 1) 在第一个向量上调用 Vector2 析构函数
// 2) 释放 pVectors2 指向的已分配内存
delete pVectors2;
```

请注意，需要调用正确的析构函数，这在继承的情况下可能会出现问题：

```c++
struct HasId
{
    int32_t Id;
 
    // 非虚析构函数
    ~HasId()
    {
    }
};
 
struct Combatant
{
    // 非虚析构函数
    ~Combatant()
    {
    }
};
 
struct Enemy : HasId, Combatant
{
    // 非虚析构函数
    ~Enemy()
    {
    }
};
 
// 分配一个敌人
Enemy* pEnemy = new Enemy();
 
// 多态性是允许的，因为 Enemy "is a" Combatant，这是由于继承的关系
Combatant* pCombatant = pEnemy;
 
// 释放一个Combatant
// 1) 调用Combatant的析构函数，而不是Enemy的析构函数
// 2) 释放由 pCombatant 指向的已分配内存
delete pCombatant;
```

这是未定义的行为，因为`pCombatant`指向的子对象可能不与分配的指针相同。为了解决这个问题，请使用虚析构函数：

```c++
struct HasId
{
    int32_t Id;
 
    virtual ~HasId()
    {
    }
};
 
struct Combatant
{
    virtual ~Combatant()
    {
    }
};
 
struct Enemy : HasId, Combatant
{
    virtual ~Enemy()
    {
    }
};
 
Enemy* pEnemy = new Enemy();
Combatant* pCombatant = pEnemy;
 
// 释放一个Combatant
// 1) 调用 Enemy 析构函数
// 2) 释放由 pEnemy 指向的已分配内存
delete pCombatant;
```

到目前为止，我们一直在使用默认的`new`和`delete`运算符。对于大多数用途来说，它们是足够的，但有时我们希望对内存分配和释放有更多的控制。
例如，我们可能想使用替代分配器来提高性能，就像Unity的`Allocator.Temp`在C#中所做的那样。为了做到这一点，我们可以重载`new`和`delete`运算符。

重载运算符可以采用多种形式，但它们应始终成对重载。这是最简单的形式：

```c++

// 我们需要 std::size_t 类型
#include <cstddef>
 
struct Vector2
{
    float X;
    float Y;
 
    void* operator new(std::size_t count)
    {
        return malloc(sizeof(Vector2));
    }
 
    void operator delete(void* ptr)
    {
        free(ptr);
    }
};
 
// 调用 Vector2 中的重载 new 运算符
Vector2* pVec = new Vector2{2, 4};
 
DebugLog(pVec->X, pVec->Y); // 2, 4
 
// 调用 Vector2 中的重载 delete 运算符
delete pVec;
```

数组版本单独重载：

```c++
struct Vector2
{
    float X;
    float Y;
 
    void* operator new[](std::size_t count)
    {
        return malloc(sizeof(Vector2)*count);
    }
 
    void operator delete[](void* ptr)
    {
        free(ptr);
    }
};
 
Vector2* pVecs = new Vector2[1];
delete [] pVecs;
```

重载的运算符，包括`new`，可以接受任何参数。我们将它们放在`new`关键字和要分配的类型之间：

```c++
struct Vector2
{
    float X;
    float Y;
 
    // 重载接受（float, float）参数的新运算符
    void* operator new(std::size_t count, float x, float y)
    {
        // 仅用于演示目的的注释
        // 正常代码只需使用构造函数
        Vector2* pVec = (Vector2*)malloc(sizeof(Vector2)*count);
        pVec->X = x;
        pVec->Y = y;
        return pVec;
    }
 
    // 重载正常的 delete 运算符
    void operator delete(void* memory, std::size_t count)
    {
        free(memory);
    }
 
    // 重载与 new 操作符相对应的 delete 操作符
    // 接受（float, float）参数
    void operator delete(void* memory, std::size_t count, float x, float y)
    {
        // 将调用转发到常规删除操作符
        Vector2::operator delete(memory, count);
    }
};
 
// 调用重载的（float, float）new运算符
Vector2* pVec = new (2, 4) Vector2;
 
DebugLog(pVec->X, pVec->Y); // 2, 4
 
// 调用正常 delete 操作符
delete pVec;
```

出现了一种约定，即使用`void*`作为第二个参数来表示“放置新对象”。在这种情况下，不会分配内存，对象只是使用由该`void*`指向的内存。

```c++
struct Vector2
{
    float X;
    float Y;
 
    // 重载“placement new”运算符
    // 标记为“noexcept”因为没有可能抛出异常
    void* operator new(std::size_t count, void* place) noexcept
    {
        // 不要分配。只需返回给定的内存地址。
        return place;
    }
};
 
// 分配我们自己的内存来存储 Vector2
// 我们可以使用全局变量、栈或者任何其他东西
char buf[sizeof(Vector2)];
 
// 调用“placement new”运算符
// 将 Vector2 放入 buf
Vector2* pVec = new (buf) Vector2{2, 4};
DebugLog(pVec->X, pVec->Y); // 2, 4
 
// 注意：因为没有实际分配内存，所以没有进行“删除”操作
```

像其他重载运算符一样，我们也可以在 class 外部重载来处理多个类型。例如，以下是所有类型的 “placement new”（新版面）：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
void* operator new(std::size_t count, void* place) noexcept
{
    return place;
}
 
char buf[sizeof(Vector2)];
Vector2* pVec = new (buf) Vector2{2, 4};
DebugLog(pVec->X, pVec->Y); // 2, 4
 
float* pFloat = new (buf) float{3.14f};
DebugLog(*pFloat); // 3.14
```

## Owning Types  拥有类型

到目前为止，我们已经克服了许多可能在使用像`malloc`和`free`这样的低级动态分配函数时犯的错误。
</br>即便如此，在“现代C++”（即C++11及更新的版本）代码库中，通常不推荐“裸露”地使用`new`和`delete`。这是因为我们仍然容易受到常见错误的影响：

- 忘记调用 `delete`，导致内存泄漏
- 重复调用`delete`，这是未定义的行为，可能会导致崩溃
- 在调用`delete`之后使用已分配的内存，这是未定义的行为，很可能会导致损坏

为了缓解这些问题，通常将`new`和`delete`运算符封装在一个称为“拥有类型”的类中。这使我们能够访问构造函数和析构函数，以便更安全地分配和释放内存。
</br>C++标准库为此目的提供了几个通用类型，我们将在系列后续内容中介绍。
</br>现在，让我们构建一个简单的“拥有类型”，它拥有一个float数组：

```c++
class FloatArray
{
    int32_t length;
    float* floats;
 
public:
 
    FloatArray(int32_t length)
        : length{length}
        , floats{new float[length]{0}}
    {
    }
 
    float& operator[](int32_t index)
    {
        if (index < 0 || index >= length)
        {
            throw IndexOutOfBounds{};
        }
        return floats[index];
    }
 
    virtual ~FloatArray()
    {
        delete [] floats;
        floats = nullptr;
    }
 
    struct IndexOutOfBounds {};
};
 
try
{
    FloatArray floats{3};
    floats[0] = 3.14f;
 
    //索引超出范围
    // 抛出异常
    // 调用 FloatArray 析构函数
    DebugLog(floats[-1]); // 3.14
}
catch (FloatArray::IndexOutOfBounds)
{
    DebugLog("whoops"); // 被打印出来
}
```

在这里，我们可以看到我们已经将new或delete运算符封装到了FloatArray类中。
</br>代码库的大部分只是这个类的用户，它永远不需要编写new或delete运算符。
</br>尽管如此，它解决了上述三个问题：

- 我们不能忘记调用 delete，因为析构函数会这样做，即使抛出异常也是如此
- 我们不能调用 delete 两次，因为析构函数会为我们执行此作
- 调用 delete 后，我们不能使用内存，因为我们没有用于调用成员函数的变量

通过使用类，我们还可以防止其他常见错误：

- 构造函数始终初始化数组的元素，以避免在写入之前读取它们时出现 undefined 行为
- 重载数组下标 `[]` 运算符执行边界检查以避免内存损坏

然而，这仍然是一个“拥有类型”的糟糕实现，因为它容易受到各种其他问题的困扰。例如，编译器生成的一个复制构造函数会复制指针，导致双重释放：

```c++
void Foo()
{
    FloatArray f1{3};
    FloatArray f2{f1}; // Copies floats and length
 
    // 1) 调用f1的析构函数，该函数删除分配的内存
    // 2) 调用f2的析构函数，再次删除分配的内存：崩溃
}
```

与其创建像`FloatArray`这样的自定义拥有类型，更常见的是使用平台库类，如C++标准库中的`std::vector`或 Unreal 中的`TArray`
对于其他拥有类型，如C++标准库中的“智能指针”`std::unique_ptr`和`std::shared_ptr`，情况也是如此，它们是对单个对象的引用。

## Conclusion  结论

C#通过要求垃圾回收提供了非常高级的内存管理。
</br>为了避免它，我们被迫使用“不安全”的代码，并且必须放弃许多语言特性，包括类、接口和委托。
</br>这种情况与Unity的Burst编译器相同，它强制使用HPC#子集。

C++ 提供了一整套选项。
- 使用malloc和free进行低级控制，创建自己的分配函数，
- 使用原始的new和delete，全局或按类型重载new和delete，向new和delete传递额外参数
- 使用“placement new”在特定地址分配，甚至编写“拥有类型”以避免几乎所有手动分配和释放代码。

这里功能强大，复杂性也相当高，但我们在向高级或低级内存管理策略迁移的过程中，任何时候都不必放弃任何语言特性。
我们将在系列后续部分介绍一些（非常常用的）高级技术，届时我们将涵盖C++标准库。



