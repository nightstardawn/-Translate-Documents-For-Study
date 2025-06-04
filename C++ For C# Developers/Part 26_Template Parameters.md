# C++ For C# Developers: Part 26 – Template Parameters

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5939),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

我们开始探讨C++的核心特性：模板。我们比较了它们与C#泛型之间的异同，并了解了它们在类、函数、lambda表达式甚至变量中的应用。
</br>今天，我们将利用所谓的“非类型模板参数(non-type template parameters)”和“模板模板参数(template template parameters)”的力量来编写一些非常有趣的代码。

## Type Template Parameters 类型模板参数

简介文章(Part26)中所有模板的示例都使用了一个参数：

```c++
template<typename T>
```

这通常足以创建许多模板，正如我们从C#的List<T>和NativeArray<T>泛型类等类型中看到的那样。
</br>然而，许多其他如Dictionary<TKey, TValue>需要更多的参数。添加这些参数很简单，就像添加函数参数一样：

```c++
// 接受两个参数的类模板
template<typename TKey, typename TValue>
struct Pair
{
    TKey Key;
    TValue Value;
};
 
// 指定两个参数以实例化模板
Pair<int, float> pair{123, 3.14f};
 
DebugLog(pair.Key, pair.Value); // 123, 3.14
```

也类似于函数参数，但与C#泛型中的类型参数不同，我们可以指定默认值：

```c++
// 第二个参数有默认值
template<typename TKey, typename TValue=int>
struct Pair
{
    TKey Key;
    TValue Value;
};
 
// 只需传递一个参数
// 第二个参数获取默认值：int
Pair<int> pair1{123, 456};
DebugLog("TValue is int?", typeid(pair1.Value) == typeid(int)); // true
DebugLog(pair1.Key, pair1.Value); // 123, 456
 
// 我们仍然可以传递两个参数
Pair<int, float> pair2{123, 3.14f};
DebugLog("TValue is int?", typeid(pair2.Value) == typeid(int)); // false
DebugLog(pair2.Key, pair2.Value); // 123, 3.14
```

在模板参数列表中，人们也常见使用class代替typename。这两者之间没有区别。
</br>非类类型，如int和float，完全可以与class模板参数一起使用。选择使用哪一个主要是一种风格问题：

```c++
//参数是“class”而不是“typename”，行为相同
template<class TKey, class TValue=int>
struct Pair
{
    TKey Key;
    TValue Value;
};
 
// 非类类型仍然可以用作模板参数
Pair<int> pair1{123, 456};
Pair<int, float> pair2{123, 3.14f};
```

与C#不同，即使参数有默认值，参数名称也是可选的：

```c++
// 参数名称省略，包含和不含默认值
template<class, class=int>
void DoNothing()
{
}
 
DoNothing<float>();
DoNothing<float, int>();
```

这些类型也不需要定义，只需声明即可用于实例化模板。只要模板不使用该类型，这就像它根本没有被赋予一个名字一样：

```c++
// 已声明但未定义
struct Vector2;
 
// 不使用其类型参数的模板
template<typename T>
void DoNothing()
{
}
 
// 因为实际上没有使用Vector2，所以可以
DoNothing<Vector2>();
```

当模板参数与模板外部的名称相同时，由于上下文明确指出指的是哪一个，因此不会发生冲突：

```c++
struct Vector
{
    float X = 0;
    float Y = 0;
};
 
// 模板参数与模板外部的类同名：Vector
template<typename Vector>
Vector Make()
{
    return Vector{};
}
 
// 将“int”用作名为“Vector”的类型
auto val = Make<int>();
DebugLog(val); // 0
 
// “Vector”没有引用类型参数
// 模板在这里没有被引用
auto vec = Vector{};
DebugLog(vec.X, vec.Y); // 0, 0
```

## Template Template Parameters 模板模板参数

考虑一个通过`List<T>`类模板来存储键和值的`Map`类模板：

```c++
template<typename T>
struct List
{
    //类似于C#的实现...
};
 
template<typename TKey, typename TValue>
struct Map
{
    List<TKey> Keys;
    List<TValue> Values;
};
 
Map<int, float> map;
```

`Map`模板始终使用`List`模板。如果我们想抽象出存储键和值的容器类型，以使Map更加灵活，我们可以使用“模板模板参数”。
</br>这就是我们将模板如`List`传递给模板，而不是将模板的实例化如`List<T>`作为参数传递给模板的地方：

```c++
template<typename T>
struct List
{
    // 类似于C#的实现...
};
 
template<typename T>
struct FixedList
{
    //类似于C#的实现，但它是固定大小的
};
 
// 第三个参数是一个模板，而不是一个类型
// 该模板需要接受一个类型参数
template<typename TKey, typename TValue, template<typename> typename TContainer>
struct Map
{
    // 使用模板参数而不是直接使用列表
    TContainer<TKey> Keys;
    TContainer<TValue> Values;
};
 
// 将接受一个类型参数的模板List作为参数传递，不要传递模板的实例，例如List<int>
Map<int, float, List> listMap;
 
// 将FixedList作为参数传递
// 它还接受一个类型参数
Map<int, float, FixedList> fixedListMap;
```

当我们这样做时，编译器为`listMap`和`fixedListMap`实例化了这两个类：

```c++
struct MapList
{
    List<int> Keys;
    List<float> Values;
};
 
struct MapFixedList
{
    FixedList<int> Keys;
    FixedList<float> Values;
};
```

模板模板参数也可以有默认值：

```c++
template<
    typename TKey,
    typename TValue,
    template<typename> typename TKeysContainer=List,
    template<typename> typename TValuesContainer=List>
struct Map
{
    TKeysContainer<TKey> Keys;
    TValuesContainer<TValue> Values;
};
 
// TKeysContainer=List, TValuesContainer=List
Map<int, float> map1;
 
// TKeysContainer=FixedList, TValuesContainer=List
Map<int, float, FixedList> map2;
 
// TKeysContainer=FixedList, TValuesContainer=FixedList
Map<int, float, FixedList, FixedList> map3;
```

## Non-Type Template Parameters 非类型模板参数

模板参数的第三种类型被称为“非类型模板参数”。
</br>这些是编译时常量值，而不是类型或模板的名称。例如，我们可以使用这个来编写由数组数据成员支持的`FixedList`类型：

```c++
// 大小是一个“非类型模板参数”
// 需要传递一个编译时常量
template<typename T, int Size>
struct FixedList
{
    // 像使用其他任何int一样使用Size
    T Elements[Size];
 
    T& operator[](int index)
    {
        return Elements[index];
    }
 
    int GetLength() const noexcept
    {
        return Size;
    }
};
 
// Pass 3 for Size
FixedList<int, 3> list1;
list1[1] = 123;
DebugLog(list1[1]); // 123
 
// Pass 2 for Size
FixedList<float, 2> list2;
list2[0] = 3.14f;
DebugLog(list2[0]); // 3.14
```

就像“类型模板参数”和“模板模板参数”一样，在实例化模板时，编译器会在任何地方替换其值：

```c++
struct FixedListInt3
{
    int Elements[3];
 
    int& operator[](int index)
    {
        return Elements[index];
    }
 
    int GetLength() const noexcept
    {
        return 3;
    }
};
 
struct FixedListFloat2
{
    float Elements[2];
 
    float& operator[](int index)
    {
        return Elements[index];
    }
 
    int GetLength() const noexcept
    {
        return 2;
    }
};
```

非类型模板参数也可以使用默认值：

```c++
// 模板参数控制初始容量和增长因子
template<typename T, int InitialCapacity=4, int GrowthFactor=2>
class List
{
    // ...实现
};
```

我们现在可以使用这些来根据预期的使用情况调整 `List`类的性能：

```c++
// 默认值是可以接受的
List<int> list1;
 
// 开始时拥有很大的容量
List<int, 1024> list2;
 
// 不要从小规模开始，但要快速成长
List<int, 4, 10> list3;
 
// 从空开始，通过翻倍增长
List<int, 0, 2> list4;
```

我们可以传递给非类型模板参数的值类型仅限于“结构化类型”。

第一种这样的“结构化类型”是我们已经见过的：整数。
</br>我们可以使用任何大小（`short`、`long` 等），并且无论是有符号的还是无符号的（`signed`、`unsigned`、no specifier）都无关紧要。
</br>这也包括类似`char`这样的准整数以及`nullptr`的类型。

第二种类型的值是指针和左值引用：

```c++
// 接受一个指向类型T的指针和一个类型T的引用
template<typename T, const T* P, const T& R>
constexpr T Sum = *P + R;
 
// 一个常量数组和一个常量整数
constexpr int a[] = { 100 };
constexpr int b = 23;
 
// 数组 'a' '退化' 成指针
// 整数 'b' 是一个左值，因为它有一个名称：b
constexpr int sum = Sum<int, a, b>;
 
DebugLog(sum); // 123
```

第三种类似的是：成员指针。

```c++
// 一个具有两个整型数据成员的类
struct Player
{
    int Health = 100;
    int Armor = 50;
};
 
// 获取类中整型数据成员的函数模板
// 接受类的类型以及其int数据成员的指针
template<typename TCombatant, int TCombatant::* pStat>
constexpr int GetStat(const TCombatant& combatant)
{
    return combatant.*pStat;
}
 
// 通过函数模板和成员指针获取两个整型数据成员
Player player;
DebugLog(GetStat<Player, &Player::Health>(player)); // 100
DebugLog(GetStat<Player, &Player::Armor>(player)); // 50
```

从C++20开始，有另外两种类型。首先，是像`float`和`double`这样的浮点类型：

```c++
template<float MinValue, float MaxValue>
float Clamp(float value)
{
    return x > MaxValue ? MaxValue : x < MinValue ? MinValue : value;
}
 
DebugLog(Clamp<0, 100>(50)); // 50
DebugLog(Clamp<0, 100>(150)); // 100
DebugLog(Clamp<0, 100>(-50)); // 0
```

其次，我们之前在编写编译时代码(Part23)时看到的“字面量类型”：

```c++
// As a simple aggregate, this is a "literal type"
struct Pixel
{
    int X;
    int Y;
};
 
// Template taking a "literal type"
template<Pixel P>
bool IsTopLeft()
{
    return P.X == 0 && P.Y == 0;
}
 
// Passing a "literal type" as a template argument
DebugLog(IsTopLeft<Pixel{2, 4}>()); // false
DebugLog(IsTopLeft<Pixel{0, 0}>()); // true
```

无论哪种语言版本，我们传递给模板参数的表达式类型都有一些额外的限制。
</br>首先，我们不能传递指向子对象（如基类和数组元素）的指针或引用：

```c++
template<const int& X>
constexpr int ValOfTemplateParam = X;
 
constexpr int a[] = { 100 };
 
//编译器错误：不能将a的子对象作为非类型模板参数引用
constexpr int val = ValOfTemplateParam<a[0]>;
```

临时对象也无法传递：

```c++
// 编译器错误：不能传递临时对象
constexpr int val = ValOfTemplateParam<123>;
```

也无法传递字符串字面量:

```c++
template<const char* str>
void Print()
{
    DebugLog("Letter:", *str);
}
 
constexpr char c = 'A';
 
// 编译错误：无法传递字符串字面量
Print<"hi">();
 
Print<&c>(); // Letter: A
```

最后，`typeid`返回的任何类型都不能作为模板参数传递：

```c++
template<decltype(typeid(char)) tid>
void PrintTypeName()
{
    DebugLog(tid.name());
}
 
// 编译器错误：无法传递typeid评估的结果
PrintTypeName<typeid(char)>();
```

虽然它们并不是严格禁止的，但重要的是要知道模板参数中的数组会被隐式转换为指针。这可能会带来一些严重的后果：

```c++
// 数组参数自动转换为指针
template<const int X[]>
constexpr void PrintSizeOfArray()
{
    // 错误：打印的是指针的大小，而不是数组的大小
    DebugLog(sizeof(X));
}
 
constexpr int32_t arr[3] = { 100, 200, 300 };
 
// Bug
PrintSizeOfArray<arr>(); // 8 (on 64-bit CPUs)
 
// OK
DebugLog(sizeof(arr)); // 12
```

## Ambiguity  多义性

存在一些情况，模板参数可能显得模糊不清。与运算符优先级类似，有一些明确的规则可以确定编译器如何消除模板参数的歧义。
</br>这些情况通常不会出现，因为程序员在命名时做出了良好的选择，但了解这些规则对于能够弄清楚编译器在边缘情况下的行为是很重要的。

> 这里比较难理解：
> </br>简单来说，就是因为取名的时候，刚刚好模板名 和 某一个class、struct ，刚刚好重复了
> </br>下面是c++处理的解决方法

第一种情况发生在成员模板在类外部声明时，其参数名称与它所属的类中的一个成员名称相同。在这种情况下，类中的成员被用作模板参数，而不是模板参数本身：

```c++
// 具有一个类型参数的类模板：T
template<class T>
struct MyClass
{
    // 成员类
    struct Thing
    {
    };
 
    // 成员函数声明，而非定义
    int GetSizeOfThing();
};
 
// 类外成员函数定义
// 使用 'Thing' 代替类类型参数名 'T'
// 'Thing' 与成员类 'Thing' 名称相同
template<class Thing>
int MyClass<Thing>::GetSizeOfThing()
{
    // 'Thing' 指的是成员类，而不是类型参数
    return sizeof(Thing);
}
 
// 使用T=double实例化类模板
MyClass<double> mc{};
 
// 在MyClass<double>上调用成员函数
// 返回成员类的尺寸：空结构体为1
DebugLog(mc.GetSizeOfThing()); // 1, not 8
```

第二种情况发生在类模板的成员在类模板外部定义时。具体来说，这仅在参数的名称与类所在的命名空间中成员的名称相同时发生。
</br>在这种情况下，我们得到相反的结果：类型参数被用作命名空间成员的替代。

```c++
namespace MyNamespace
{
    // 命名空间中的类成员
    class Thing
    {
    };
 
    // 具有一个类型参数的类模板：T
    template<class T>
    struct MyClass
    {
        // 成员函数声明，而非定义
        int GetSizeOfThing(T thing);
    };
}
 
// 类外成员函数定义
// 使用 'Thing' 而不是 'T' 作为类的类型参数名称
// 'Thing' 与命名空间成员类 'Thing' 名称相同
// 'Thing' 被用作函数参数的类型
template<class Thing>
int MyNamespace::MyClass<Thing>::GetSizeOfThing(Thing thing)
{
    // “Thing”指的是类型参数，而不是命名空间成员
    return sizeof(Thing);
}
 
// 使用T=double实例化类模板
MyNamespace::MyClass<double> mc{};
 
// 在MyClass<double>上调用成员函数
// 返回类型参数的大小：对于double为8
DebugLog(mc.GetSizeOfThing({})); // 8, not 1
```

第三种情况是当一个类模板的参数与它的基类中的一个成员具有相同的名称时。在这种情况下，歧义将指向基类的成员：

```c++
struct BaseClass
{
    struct Thing
    {
    };
};
 
// 具有一个类型参数的类模板：Thing
// 'Thing' 与基类的成员类 'Thing' 名称相同
template<class Thing>
struct DerivedClass : BaseClass
{
    // 'Thing' 指的是基类的成员类，而不是类型参数
    int Size = sizeof(Thing);
};
 
// 使用 Thing=double 实例化类模板
DerivedClass<double> dc;
 
// 看看初始化'Size'时'Thing'有多大
// 它是BaseClass::Thing的大小：对于空结构体为1
DebugLog(dc.Size); // 1, not 8
```

与前两种情况不同，这种情况在C#中也是可能的。与C++不同，'Thing'指的是类型参数，而不是基类成员：

```csharp
// C#
public class BaseClass
{
    public struct Thing
    {
    };
};
 
// 具有一个类型参数的泛型类：Thing
// 'Thing' 与基类的成员类 'Thing' 名称相同
public class DerivedClass<Thing> : BaseClass
{
    // 'Thing' 指的是类型参数，而不是基类成员类
    public Type ThingType = typeof(Thing);
};
 
// 使用 Thing=double 实例化泛型类
DerivedClass<double> dc = new DerivedClass<double>();
 
// 使用 Thing=double 实例化泛型类
DebugLog(dc.ThingType); // System.Double
```

## Conclusion  结论

C#泛型支持类型参数，但不支持C++模板提供的非类型参数和模板参数。即便如此，C++的类型参数包括额外的功能，例如支持默认参数和省略参数名称。

模板参数通过使用像`Container`这样的模板作为变量而不是像`List<T>`这样的特定模板，允许编写更通用的代码。
</br>非类型模板参数允许传递编译时常量表达式，因此我们可以在模板中使用值而不是类型。
</br>这使我们能够创建具有静态大小的类模板，如`FixedList<T>`，以避免在不需要时进行动态分配和动态调整大小的成本，或者在我们需要时调整`List<T>`的分配策略。

关于模板还有很多内容值得我们探索。
</br>下一个Part我们将深入探讨模板参数推导，它允许我们编写 `Foo(3.14f)` 而不是 `Foo<float>(3.14f)`
</br>以及模板特化，我们可以编写模板的定制版本（`Vector<2>`），而不是依赖于通用版本（`Vector<int NumDimensions>`）。敬请期待！


