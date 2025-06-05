# C++ For C# Developers: Part 28 – Variadic Templates


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6026),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

到目前为止，我们编写的所有模板都具有固定数量的参数，但C++也允许我们使用可变数量的参数。
</br>这就像C#函数中的params，但用于C++模板的参数。
</br>今天我们将深入探讨这个特性，它在C#中没有等效功能，并学习如何编写和使用具有任何数量参数的模板。

## Parameter Packs  参数包

可变模板是指具有“参数包”的模板。参数包代表零个或多个参数，就像C#函数中的`params`代表零个或多个参数的数组一样。

这是一个包含一个参数包的可变函数模板：

```c++
template<typename ...TArgs>
void LogAll(TArgs... args)
{
}
```

`TArgs`是一个参数包，因为它在（可选的）参数名`TArgs`之前有`...`。它是一个类型参数的参数包，因为它以typename开头。


要使用参数包，我们在参数名称后添加`...`：`TArgs....` 编译器将其扩展为以逗号分隔的参数列表。

让我们看看这个模板的一些实例，以了解这种展开是如何工作的：

```c++
// TArgs参数包没有零个参数
LogAll();
void LogAll() {}
 
// 一个参数传递给TArgs参数包
LogAll<int>(123);
void LogAll(int) {}
 
// 一个参数传递给TArgs参数包（带有推导）
LogAll(123);
void LogAll(int) {}
 
// 向TArgs参数包传递两个参数
LogAll(123, 3.14f);
void LogAll(int, float) {}
```

我们可以自由地混合参数包和其他模板参数：

```c++
template<typename TPrefix, typename ...TArgs>
void LogAll(TPrefix prefix, TArgs... args)
{
}
```

与C#的`params`不同，只要编译器能够推断出所有参数，参数包甚至不必是最后一个参数：


```c++
// 参数包不是最后一个参数
template<typename ...TLogParts, typename TPrefix>
void LogWithPrefix(TPrefix prefix, TLogParts... parts)
{
}
 
//编译器推断出 TPrefix 是 'float'，TLogParts 是 (int, int, int)
LogWithPrefix(3.14f, 123, 456, 789);
```

请注意，编译器永远不会用类模板推导出这一点，因此参数包必须放在末尾。

> 这里主要是区别于函数模板
> </br>函数模板更灵活：函数模板可以通过参数推导确定参数包的位置（例如使用标记类型或默认参数）：
> ```c++
> template<typename... Args, typename T>
> void func(T t, Args... args) {}  // 参数包在中间，但通过函数参数顺序解决
> ```
> 但是类模板就会出现歧义：
> ```c++
> template<typename A, typename... B, typename C>
> class MyClass {};
> ```
> 若使用 MyClass<int, double, char> 实例化：
> - A 应该是 int？
> - ...B 是 double，C 是 char？
> - 还是 ...B 是 double, char，而 C 未指定？
> </br>编译器无法确定参数包的结束位置，因此会报错。

## Pack Expansion  扩展包

既然我们已经知道如何声明模板参数包以及如何在函数参数中使用它们，让我们来看看更多使用它们的方法。
</br>一种常见的方法是将它们作为函数参数传递：

```c++
template<typename ...TArgs> // 模板参数包
void LogError(TArgs... args) // 使用参数包来声明参数
{
    DebugLog("ERROR", args...); // 将参数作为参数传递给函数
}
 
// 向函数模板传递参数
// 从参数类型推导模板参数
LogError(3.14, 123, 456, 789); // ERROR, 3.14, 123, 456, 789
 
// 编译器实例化了此函数
void LogError(double arg1, int arg2, int arg3, int arg4)
{
    DebugLog("ERROR", arg1, arg2, arg3, arg4);
}
```

在这个例子中，我们将参数直接传递为`args...`这被扩展为`arg1`、`arg2`、`arg3`、`arg4`。如果我们对参数包的名称应用一些操作，它将应用于所有参数：

```c++
template<typename ...TArgs>
void LogPointers(TArgs... args)
{
    // 对包中的每个值应用解引用
    DebugLog(*args...);
}
 
// 传递指针
float f = 3.14f;
int i1 = 123;
int i2 = 456;
LogPointers(&f, &i1, &i2); // 3.14, 123, 456
 
// 编译器实例化了此函数
void LogPointers(float* arg1, int* arg2, int* arg3)
{
    DebugLog(*arg1, *arg2, *arg3);
}
```

如果我们同时在同一个展开中命名多个参数包，它们会同时展开：

```c++
// 具有两个参数的类模板
template<typename T1, typename T2>
struct KeyValue
{
    T1 Key;
    T2 Value;
};
 
// 带有参数包的类模板
template<typename ...Types>
struct Map
{
    // ...实现
};
 
// 带有参数包的类模板
template<class ...Keys>
struct MapOf
{
    // 带参数包的成员类模板
    template<class ...Values>
    // 从Map类模板派生
    // 将KeyValue<Keys, Values>...作为模板参数传递给Map
    // 展开为(KeyValue<Keys1, Values1>, KeyValue<Keys2, Values2>等)
    struct KeyValues : Map<KeyValue<Keys, Values>...>
    {
    };
};
 
// 使用 key = (int, float)和 value = (double, bool)实例化模板
// Pairs从Map<KeyValue<int, double>, KeyValue<float, bool>>派生
MapOf<int, float>::KeyValues<double, bool> map;
 
// 编译器实例化了这个类
struct MapOf
{
    struct KeyValues : Map<KeyValue<int, double>, KeyValue<float, bool>>
    {
    };
};
```

## Where Packs Can Be Expanded  包可以扩展的位置

到目前为止，我们已经看到包被扩展到函数参数、函数参数和模板参数中。还有许多其他地方可以进行扩展。首先，当使用括号进行初始化时：

```c++
struct Pixel
{
    int X;
    int Y;
 
    Pixel(int x, int y)
        : X(x), Y(y)
    {
    }
};
 
// 函数模板接受一个int参数包
template<int ...Components>
Pixel MakePixel()
{
    // 展开到括号初始化
    return Pixel(Components...);
};
 
Pixel pixel = MakePixel<2, 4>();
DebugLog(pixel.X, pixel.Y); // 2, 4
```

或者使用花括号初始化：

```c++
// 函数模板接受一个int参数包
template<int ...Components>
Pixel MakePixel()
{
    // 展开到大括号初始化
    return Pixel{Components...};
};
```

其次，我们可以将类型参数包展开为非类型参数包：

```c++
// 带有类型参数包的类模板
template<typename... Types>
struct TypedPrinter
{
    // 带有非类型参数的函数模板
    // 由类型包的扩展形成
    template<Types... Values>
    static void Print()
    {
        // 扩展非类型参数包
        DebugLog(Values...);
    }
};
 
// 使用类型和非类型参数实例化模板
TypedPrinter<char, int>::Print<'c', 123>(); // c, 123
 
// 编译器错误：'c' 不是一个bool
TypedPrinter<bool, int>::Print<'c', 123>();
```

第三，一个类可以通过扩展类型包从零个或多个基类继承：

```c++
struct VitalityComponent
{
    int Health;
    int Armor;
};
 
struct WeaponComponent
{
    float Range;
    int Damage;
};
 
struct SpeedComponent
{
    float Speed;
};
 
template<class... TComponents>
// 扩展一组基类
class GameEntity : public TComponents...
{
};
 
// turret是一个从VitalityComponent and WeaponComponent派生出来的类
GameEntity<VitalityComponent, WeaponComponent> turret;
turret.Health = 100;
turret.Armor = 200;
turret.Range = 10;
turret.Damage = 15;
 
//civilian是一个从VitalityComponent and SpeedComponent派生出来的类
GameEntity<VitalityComponent, SpeedComponent> civilian;
civilian.Health = 100;
civilian.Armor = 200;
civilian.Speed = 2;
```

第四，可以通过打包扩展形成`lambda`捕获列表：

```c++
template<class ...Args>
void Print(Args... args)
{
    // 将 'args' 包扩展到 lambda 捕获列表中
    auto lambda = [args...] { DebugLog(args...); };
    lambda();
}
 
Print(123, 456, 789); // 123, 456, 789
```

第五，`sizeof`运算符有一个接受参数包的变体。它计算包中元素的数量，而不管它们的尺寸如何：

```c++
// 求和的一般形式
// 仅作声明，因为它从未真正实例化
template<typename ...TValues>
int Sum(TValues... values);
 
// 当至少有一个值时的专业化
template<typename TFirstValue, typename ...TValues>
int Sum(TFirstValue firstValue, TValues... values)
{
    // 将包扩展为递归调用
    return firstValue + Sum(values...);
}
 
// 当没有值时的专业化
template<>
int Sum()
{
    return 0;
}
 
template<typename ...TValues>
int Average(TValues... values)
{
    // 将包展开为Sum调用
    // 使用sizeof...来计算包中参数的数量
    return Sum(values...) / sizeof...(TValues);
}
 
DebugLog(Average(10, 20)); // 15
```

在这个例子中，编译器实例化了这些模板：

```c++
// 为Sum(10, 20)实例化
int Sum2(int firstValue, int value)
{
    // 将包扩展为递归调用
    return firstValue + Sum1(value);
}
 
// 为Sum(20)实例化
int Sum1(int firstValue)
{
    return firstValue + Sum0();
}
 
// 实例化用于求和
int Sum0()
{
    return 0;
}
 
int Average(int value1, int value2)
{
    return Sum2(value1, value2) / 2;
}
 
DebugLog(Average(10, 20)); // 15
```

编译器优化几乎总是将其归结为一个常量：

```c++
DebugLog(15); // 15
```

或者对于不是编译时常量的参数`x`和`y`：

```c++
DebugLog((x + y) / 2);
```

> 这里读者刚刚看的时候一脸蒙蔽，原作者突然冒出来这两句话，读者查了一下资料，简单解释一下
> </br>`DebugLog(15); // 15`这种直接使用常量的情况：编译器可能会被直接优化成为直接操作该常量的汇编指令
> </br>`DebugLog((x + y) / 2);`这种情况：编译器会生成对应的 算术指令（ADD + DIV）， 计算结果在运行时确定，消耗额外的CPU周期进行计算。

## Conclusion  结论

可变参数模板使我们能够根据任意数量的参数编写模板。这使我们免去了反复编写几乎相同的模板版本的需要。
</br>例如，C# 有 `Action<T>`、`Action<T1,T2>`、`Action<T1,T2,T3>`，一直到 `Action<T1,T2,T3,T4,T5,T6,T7,T8,T9,T10,T11,T12,T13,T14,T15,T16>`！
</br>同样的巨大重复也应用于其 Func 对应版本：`Func<T1,T2,T3,T4,T5,T6,T7,T8,T9,T10,T11,T12,T13,T14,T15,T16,TResult>`。
</br>编写起来如此痛苦，以至于我们通常都不愿意这么做，或者编写一个代码生成器来输出所有这些冗余的 C# 代码。
</br>我们从未得到一个可以接受任意数量参数的解决方案，而只是现在足够多的参数数量。































