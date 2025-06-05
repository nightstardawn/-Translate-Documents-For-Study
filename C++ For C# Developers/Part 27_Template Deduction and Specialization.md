# C++ For C# Developers: Part 27 – Template Deduction and Specialization

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5875),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

C++中的模板推导类似于C#中的泛型类型参数推导：它允许我们省略模板参数。模板特化在C#中没有等效功能，但可以根据某些参数对模板进行特殊处理。
</br>这一Part我们将探讨这些功能如何使我们的代码更加简洁且效率更高。

## Template Argument Deduction  模板参数推导

编译器必须知道所有参数以实例化模板，但这并不意味着我们必须明确地声明所有参数。
</br>就像我们可以使用`auto`变量、参数和返回值，编译器会推断它们的类型一样，编译器也可以推断模板参数。

这在某种程度上也适用于C#泛型。考虑以下示例：

```csharp
// C#
static class TypeUtils
{
    // 通用方法
    public static void PrintType<T>(T x)
    {
        DebugLog(typeof(T));
    }
}
 
// 显式指定类型参数
TypeUtils.PrintType<int>(123); // System.Int32
TypeUtils.PrintType<bool>(true); // System.Boolean
 
// 编译器推导的类型参数
TypeUtils.PrintType(123); // System.Int32
TypeUtils.PrintType(true); // System.Boolean
```

同样的，在C++中也是如此，正如我们在C#的这种直接翻译中看到的那样：

```c++
struct TypeUtils final
{
    // 成员函数模板
    template<typename T>
    static void PrintType(T x)
    {
        DebugLog(typeid(T).name());
    }
};
 
// 显式指定类型参数
TypeUtils::PrintType<int>(123); // i
TypeUtils::PrintType<bool>(true); // b
 
// 编译器推导的类型参数
TypeUtils::PrintType(123); // i
TypeUtils::PrintType(true); // b
```

C++对推导的支持比C#要先进得多。例如，可以推导出非类型模板参数：

```c++
// 模板有一个类型参数（T）和一个非类型参数（N）
template<class T, int N>
// 函数接收一个长度为N的const T元素的数组的引用
int GetLengthOfArray(const T (&t)[N])
{
    return N;
}
 
// 编译器将T推断为int类型，将N推断为3
DebugLog(GetLengthOfArray({1, 2, 3})); // 3
 
// 编译器推断T为浮点数，N为2
DebugLog(GetLengthOfArray({2.2f, 3.14f})); // 2
```

模板模板参数也可以被推断:

```c++
// 具有两个参数的模板：
// 1) T，类型参数
// 2) TContainer，模板参数
template<typename T, template<typename> typename TContainer>
void PrintLength(const TContainer<T>& container)
{
    DebugLog(container.Length);
}
 
template<typename T>
struct List
{
    int Length;
};
 
List<int> list{};
PrintLength(list); // T被推断为int，TContainer被推断为List
```

编译器还会考虑所有重载的函数，以尝试找到最佳匹配的函数：

```c++
// 模板接受一个类型参数
template<class T>
// 函数接受一个接受类型 T 并返回类型 T 的函数的指针
int CallWithDefaultAndReturn(T(*func)(T))
{
    return func({});
}
 
int AddOne(int x)
{
    DebugLog("int");
    return x + 1;
}
 
int AddOne(char x)
{
    DebugLog("char");
    return x + 1;
}
 
// CallWithDefaultAndReturn是一个重载集
// 编译器查看此函数并推断T是int:
//   int AddOne(int)
// 编译器查看此函数并无法推断T:
//   int AddOne(char)
// 由于其中一个推断成功，因此传递了它
DebugLog(CallWithDefaultAndReturn(AddOne)); // "int"  1
```

请注意，类型推导涉及一些类型转换。首先，数组“退化”为指针：

```c++
template<class T>
void ArrayOrPointer(T)
{
    DebugLog("is array?", typeid(T) == typeid(int[3]));
    DebugLog("is pointer?", typeid(T) == typeid(int*));
}
 
int arr[3];
ArrayOrPointer(arr); // is array? false, is pointer? true
```

其次，函数“退化”为函数指针：

```c++
void SomeFunction(int) {}
 
template<class T>
void FunctionOrPointer(T)
{
    DebugLog("is function?", typeid(T) == typeid(decltype(SomeFunction)));
    DebugLog("is pointer?", typeid(T) == typeid(void(*)(int)));
}
 
FunctionOrPointer(SomeFunction); // is function? false, is pointer? true
```

第三，`const`被移除了：

```c++
template<class T>
void ConstOrNonConst(T x)
{
    // 如果T是'const int'，那么这将是一个编译器错误
    x = {};
}
 
const int c = 123;
ConstOrNonConst(c); // 编译，意味着T是非const int
```

第四，`T`的引用变成了仅仅是`T`：

```c++
template<class T>
void RefDetector(T x)
{
    // 如果T是一个引用，则这会将值分配给调用者的值
    // 如果T不是一个引用，则这会将值分配给局部副本
    x = 123;
}
 
int i = 42;
int& ri = i;
RefDetector(ri);
DebugLog(i); // 42
```

为了保持引用，我们必须通过添加`&`来说明我们想要一个引用：

```c++
template<class T>
void RefDetector(T& x) // <-- Added &
{
    x = 123;
}
 
int i = 42;
int& ri = i;
RefDetector(ri);
DebugLog(i); // 123
```

一个例外是当将一个左值传递给一个接受非const右值引用的函数模板时。在这种情况下，编译器会将类型推断为右值引用：

```c++
template<class T>
void Foo(T&&)
{
}
 
int i = 123; // lvalue, not lvalue reference
Foo(i); // T is int&&
Foo(123); // T is int&
```

在这些转换之后，编译器会寻找精确匹配，但也会接受一些差异。首先，`非const`可以匹配`const`，但`const`不能匹配`非const`。

```c++
template<typename T>
void TakeConstRef(const T& x)
{
}
 
template<typename T>
void TakeNonConstRef(T& x)
{
    x = 42;
}
 
// 编译器推断T为'const int&'，尽管'i1'不是const
int i1 = 123;
TakeConstRef(i1);
 
// 编译器推断 T='const int&'
const int i2 = 123;
TakeNonConstRef(i2); // 编译错误：不能分配给x
```

其次，这也适用于指针：

```c++
template<typename T>
void TakeConstRef(const T* p)
{
}
 
template<typename T>
void TakeNonConstRef(T* p)
{
    *p = 42;
}
 
// 编译器推断出T为'const int*'，尽管'i1'不是const
int i1 = 123;
TakeConstRef(&i1);
 
// 编译器推断T为'const int*'
const int i2 = 123;
TakeNonConstRef(&i2); // 编译器错误：不能赋值给 *p
```

第三，允许派生以支持多态性：

```c++
template<class T>
struct Base
{
};
 
template<class T>
struct Derived : public Base<T>
{
};
 
template<class T>
void TakeBaseRef(Base<T>&)
{
}
 
Derived<int> derived;
 
// 编译器接受 Derived<T> 用于 Base<T> 并推断 T 是 'int'
TakeBaseRef(derived);
```

> 这里其实简单总结一下就是：编译器的推断会退化一层,这之后编译器会寻找精确匹配。
> 
> | 实参类型        | 	推导出的T	    | 函数参数类型      |
> |-------------|------------|-------------|
> | int	        | int	       | int         |
> | const int   | 	int       | 	int        |
> | int*	       | int*	      | int*        |
> | const int*	 | const int* | 	const int* |
> | int&        | 	int       | 	int        |
> | const int&	 | int        | 	int        |
> 关键点：
> - 顶层const被丢弃（如const int → int)
> - 底层const保留（如const int* → const int*)
> - 引用类型被丢弃（如int& → int）

## Class Template Argument Deduction  类模板参数推导

从 C++17 开始，类模板的参数也可以推导出来：
```c++
// 类模板
template<class T>
struct Vector2
{
    T X;
    T Y;
 
    Vector2(T x, T y)
        : X{x}, Y{y}
    {
    }
};
 
// 显式类模板参数：float
Vector2<float> v1{2.0f, 4.0f};
 
// 编译器推断类模板参数为：float
Vector2 v2{2.0f, 4.0f};
 
// 也适用于 'new'
// 'v3' 是一个 Vector<float> 指针
auto v3 = new Vector2{2.0f, 4.0f};
```

为了帮助编译器推断这些参数，我们可以编写一个“推断指南”来告诉它该怎么做：

```c++
// Class template
template<class T>
struct Range
{
    // 构造函数模板
    template<class Pointer>
    Range(Pointer beg, Pointer end)
    {
    }
};
 
double arr[] = { 123, 456 };
 
// 编译器错误：无法从构造函数推断 T（类模板参数）
Range range1{&arr[0], &arr[1]};
 
// 推导指南告诉编译器如何推导类模板参数
template<class T>
Range(T* b, T* e) -> Range<T>;
 
// OK: 编译器使用推导指南来推断T是'double'
Range range2{&arr[0], &arr[1]};
```

正如我们在示例中看到的那样，推导指南的编写方式类似于具有“尾随返回语法”的函数模板。
</br>主要区别在于它们的名称是类模板的名称，而它们的“返回类型”是一个带有其参数传递的类模板。

## Specialization  特化


到目前为止，我们的模板都是按照相同的方式实例化的，无论提供的模板参数是什么。
</br>有时，当提供某些参数时，我们希望使用模板的另一个版本。这被称为模板的特化。考虑这个类模板：

```c++
//一个非常通用的向量
template<typename T, int N>
struct Vector
{
    T Components[N];
 
    T Dot(const Vector<T, N>& other) const noexcept
    {
        T result{};
        for (int i = 0; i < N; ++i)
        {
            result += Components[i] * other.Components[i];
        }
        return result;
    }
};
 
// Usage
Vector<float, 2> v1{2, 4};
DebugLog(v1.Components[0], v1.Components[1]); // 2, 4
 
Vector<float, 2> v2{6, 8};
DebugLog(v1.Dot(v2)); // 44
```

现在让我们为一种常见用例特化`Vector`：两个浮点数组件。

```c++
// 向量模板的专门化
template<> // 不接受任何参数
struct Vector<float, 2> // 参数由特殊化提供
{
    // 专业可以包含非常不同的内容
    // 这个联合允许通过组件数组或X和Y进行访问
    union
    {
        float Components[2];
        struct
        {
            float X;
            float Y;
        };
    };
 
    float Dot(const Vector<float, 2>& other) const noexcept
    {
        // 专业版本不需要循环
        // 更易于读者理解
        // 编译器不可能失败地优化掉循环
        return X*other.X + Y*other.Y;
    }
};
 
// 我们可以使用X和Y或者Components数组来访问组件
Vector<float, 2> v1{2, 4};
DebugLog(v1.Components[0], v1.Components[1]); // 2, 4
DebugLog(v1.X, v1.Y); // 2, 4
 
// Dot 仍然有效
Vector<float, 2> v2{6, 8};
DebugLog(v1.Dot(v2)); // 44
```

可能有几个原因让我们想要为常见的向量类型和大小专门化`Vector`模板。也许我们已经检查了汇编代码，并意识到编译器没有优化掉`Dot`中的循环。
</br>也许我们想要添加`X`和`Y`数据成员的便利性，作为`Components`数组前两个元素的别名。
</br>也许我们想要使用仅适用于特定数量和特定数据类型的SIMD指令。我们将在本系列的后续部分中看到如何做到这一点。

无论我们的理由是什么，上述例子中有几个方面值得我们注意。首先，我们不仅可以特化像`T`这样的类型参数，还可以特化像`N`这样的非类型参数。

其次，我们的特化也命名为`Vector`。它不像`Vector2`那样有一个独特的名称。
</br>通常，特化是为了让模板的用户透明地使用。模板的作者通常会提供它们来优化某些用例或在某些特定情况下提供功能超集。
</br>`Vector<float, 2>`的特化可以省略`Components`数组，但那样的话，`Vector<float, 2>`就无法与其他`Vector`的实例兼容：

```c++
template<>
struct Vector<float, 2>
{
    // No Components
    float X;
    float Y;
 
    float Dot(const Vector<float, 2>& other) const noexcept
    {
        return X*other.X + Y*other.Y;
    }
};
 
Vector<float, 2> v1{2, 4};
 
// 编译器错误：Vector<float, 2> 没有Components数据成员
DebugLog(v1.Components[0], v1.Components[1]); // 2, 4
 
// OK: 向量<float, 2> 有 X 和 Y
DebugLog(v1.X, v1.Y); // 2, 4
```

话虽如此，有时不兼容性是可取的。以叉积为例。我们可能希望从二维向量的特殊化中省略这个操作，因为这个操作没有太多意义。
</br>然而，我们可能还想返回一个三维向量，比如`（0，0，1）`或`（0，0，-1）`。模板特殊化使我们能够灵活地做出这个设计选择。

最后，我们还有“部分特化”模板的选项。
</br>当我们只想特化部分模板参数，而不是像上面那样特化所有参数时，我们使用“部分特化”。例如，我们可能只想为二维向量特化，而不是为浮点数特化：

```c++
// Vector模板的部分特化
// 现在只接受一个参数：类型T
template<typename T>
// 将参数传递给主Vector模板
// 它们可以是特殊化的参数或常规参数
struct Vector<T, 2>
{
    union
    {
        // 我们仍然可以使用T，但我们也知道N是2
        T Components[2];
        struct
        {
            T X;
            T Y;
        };
    };
 
    T Dot(const Vector<T, 2>& other) const noexcept
    {
        // 循环已被移除，但我们仍然支持任何算术类型
        return X*other.X + Y*other.Y;
    }
};
 
// X和Y都可用
Vector<float, 2> v1{2, 4};
DebugLog(v1.X, v1.Y); // 2, 4
 
// 现在可以使用多种类型（float和double）了
Vector<double, 2> v2{6, 8};
DebugLog(v2.X, v2.Y); // 6, 8
```

C#和C++都支持泛型和模板中的参数推导。
</br>通常情况下，C++在复杂性和深度上更进一步。
</br>它可以推导非类型参数、模板参数以及类参数，从而使得代码更加简洁：直接使用Dictionary，而不是Dictionary<MyKeyType, MyValueType>。
</br>推导指南为我们提供了一个工具，真正推动可推导的内容，而不是满足于默认值。
























