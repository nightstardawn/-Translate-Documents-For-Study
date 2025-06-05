# C++ For C# Developers: Part 28 – Variadic Templates


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6130),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

这两种语言都有解构`（var (x, y) = vec;）`和属性(或者叫特性)`（[MyAttribute]）`。
</br>C++在多个方面与C#不同，所以今天我们将探讨这些差异，并学习如何利用这些语言特性。

> 这里解释一下解构：
> </br>允许你将对象（特别是复合对象）的多个属性值一次性提取到独立的变量中。
> </br>用的比较多的应该就是元组了
> </br>原作者在下面给了自定义的结构例子，我这里给几个我们之前熟悉的例子
> 
> ```csharp
> // 1. 元组解构
> var person = (Name: "Alice", Age: 30);
> var (name, age) = person;
> Console.WriteLine($"{name} is {age} years old"); // Alice is 30 years old
> 
> // 2. 数组解构
> int[] numbers = { 1, 2, 3 };
> var (first, second, third) = numbers;
> Console.WriteLine(second); // 2
> 
> // 3. 忽略某些值（使用 _）
> var (_, _, last) = numbers;
> Console.WriteLine(last); // 3
> ```

## Structured Bindings  结构化绑定

在C#中，“解构”是指将结构体或类的字段提取到单独的变量中的过程。它看起来像这样：

```csharp
// C#
 
// 我们想要解构的类型
struct Vector2
{
    public float X;
    public float Y;
 
    // 创建一个Deconstruct方法，该方法接受一个用于分解每个变量的'out'参数，并返回void
    public void Deconstruct(out float x, out float y)
    {
        x = X;
        y = Y;
    }
}
 
// 实例化可解构的类型
var vec = new Vector2{X=2, Y=4};
 
// 解构。隐式调用 vec.Deconstruct(out x, out y)。
// x 是 vec.X 的副本，y 是 vec.Y 的副本
var (x, y) = vec;
 
DebugLog(x, y); // 2, 4
```

在C++术语中，我们不是“解构(deconstruct)”，而是“创建结构化绑定(create structured bindings)”。以下是等效代码：

```c++
// 我们想要为其创建结构化绑定的类型
struct Vector2
{
    float X;
    float Y;
};
 
// 实例化该类型
Vector2 vec{2, 4};
 
// 创建结构化绑定。x 是 vec.X 的副本，y 是 vec.Y 的副本。
auto [x, y] = vec;
 
DebugLog(x, y); // 2, 4
```

到目前为止，这两种语言在本质上是一样的，除了两个变化。

首先，C++ 使用方括号 `[x, y]` 而不是圆括号 `(x, y)`。

其次，C++ 不需要我们编写一个 Deconstruct 函数。相反，编译器简单地使用 `Vector2` 字段的声明顺序，以便 `x` 与 `X` 对齐，`y` 与 `Y` 对齐。

这反映了初始化的过程，其中 `Vector2{2, 4}` 按声明顺序初始化数据成员：首先是 X，然后是 Y。

这可以通过“元组类似”类型进行自定义，但在C#中需要更多的工作：

```c++
struct Vector2
{
    float X;
    float Y;
 
    // 获取一个const向量的数据成员
    template<std::size_t Index>
    const float& get() const
    {
        // 断言只有两个有效的索引
        static_assert(Index == 0 || Index == 1);
 
        // 根据索引返回正确的项
        if constexpr(Index == 0)
        {
            return X;
        }
        return Y;
    }
 
    // 获取非const向量的数据成员
    template <std::size_t Index>
    float& get()
    {
        // 将变量转换为const，以便我们可以调用此函数的const重载版本，以避免代码重复
        const Vector2& constThis = const_cast<const Vector2&>(*this);
 
        // 调用此函数的const重载版本
        // 返回数据成员的const引用
        const float& constComponent = constThis.get<Index>();
 
        // 将数据成员转换为非const
        // 这是因为我们知道这个向量不是const
        float& nonConstComponent = const_cast<float&>(constComponent);
 
        // 返回非const数据成员引用
        return nonConstComponent;
    }
};
 
// 将 tuple_size 类模板特化为从 integral_constant 继承
// 传递 2，因为 Vector2 总是有 2 个组件
template<>
struct std::tuple_size<Vector2> : std::integral_constant<std::size_t, 2>
{
};
 
// 将元组元素结构体专门化，以指示Vector2的索引0的类型为'float'
template<>
struct std::tuple_element<0, Vector2>
{
    // 创建一个名为 'type' 的成员，它是 'float' 的别名
    using type = float;
};
 
// 同样适用于索引1
template<>
struct std::tuple_element<1, Vector2>
{
    using type = float;
};
 
// 用法相同
Vector2 vec{2, 4};
auto [x, y] = vec;
DebugLog(x, y);
```

上述代码的结果是，`Vector2` 现在成为一个“tuple-like”类型，可以与 C++ 标准库中的结构化绑定和多个泛型算法一起使用。

我们可以为最后一种可以创建结构化绑定的对象是数组。在C#中默认不允许这样做：

```c++
// 创建一个包含两个整型元素的数组
int arr[] = { 2, 4 };
 
// 创建结构化绑定。x 是 arr[0] 的副本，y 是 arr[1] 的副本。
auto [x, y] = arr;
 
DebugLog(x, y); // 2, 4
```

需要注意的是，我们一直在创建的结构化绑定必须使用`auto`自动类型化。即使我们知道类型，我们也不能使用它：

```c++
int arr[] = { 2, 4 };
 
// 编译器错误：此处必须使用 'auto' 类型
int [x, y] = arr;
```

我们可以做的是使用`auto&`创建结构化绑定引用：

```c++
int arr[] = { 2, 4 };
 
// 创建arr[0]和arr[1]的副本
auto [xc, yc] = arr;
 
// 创建对arr[0]和arr[1]的引用
auto& [xr, yr] = arr;
 
// 修改数组元素
arr[0] = 20;
arr[1] = 40;
 
DebugLog(xc, yc); // 2, 4
DebugLog(xr, yr); // 20, 40
```

同样，这也适用于通过`auto&&`的右值引用：

```c++
Vector2 Add(Vector2 a, Vector2 b)
{
    return Vector2{a.X+b.X, a.Y+b.Y};//这里返回的是右值
}
 
// 编译器错误：Add函数的返回值不是左值，因此不能取左值引用
auto& [x, y] = Add(Vector2{2, 4}, Vector2{6, 8});
 
// OK
auto&& [x, y] = Add(Vector2{2, 4}, Vector2{6, 8});
DebugLog(x, y); // 8, 12
```

两种引用形式以及副本也可以是`const`：

```c++
Vector2 vec{2, 4};
 
// 常量复制
const auto [xc, yc] = vec;
 
// 常量左值引用
const auto& [xr, yr] = vec;
 
// 常量左值引用
const auto&& [xrr, yrr] = Add(Vector2{2, 4}, Vector2{6, 8});
```

在创建结构化绑定时，我们还可以使用其他形式的初始化(13)。到目前为止，我们已经使用 `=` 来复制初始化我们的变量，但我们也可以使用 `{}` 和 `()` 来直接初始化。

```c++
Vector2 vec{2, 4};
 
// 初始化副本
auto [x1, y1] = vec;
 
// 使用花括号直接初始化
auto [x2, y2]{vec};
 
// 使用括号直接初始化
auto [x3, y3](vec);
```

最后，C# 支持使用下划线`（_）`的形式忽略一些解构变量。
</br>这在 C++ 中是不支持的，因此我们需要明确命名并忽略它们以避免编译器警告：

```c++
Vector2 vec{2, 4};
 
auto [x, y] = vec;
static_cast<void>(x); // 明确忽略x的一种方法
(void)x; // 忽略x的另一种方法
 
// Use only y
DebugLog(y); // 4
```

## Attributes  属性/特性

C# 属性将一些元数据与实体（如类、方法或参数）关联起来。这些元数据可以通过两种方式之一使用。
</br>首先，编译器可以查询属性，如 `[MethodImplAttribute(MethodImplOptions.AggressiveInlining)]`，以修改编译过程。
</br>其次，我们的 C# 代码可以通过反射在运行时查询属性，以各种自定义方式使用它。


由于C++不支持反射，它对特性的使用不涵盖运行时用例。
</br>然而，它确实涵盖了编译时用例。因为这个功能是由编译器实现的而不是我们实现的，所以我们不能创建自定义特性。
</br>相反，我们使用两组内置的特性。首先，有几个特性是由C++语言定义的：

| Attribute                 | 	Version | 	Meaning                                                                                                            |
|---------------------------|----------|---------------------------------------------------------------------------------------------------------------------|
| `[[noreturn]]`	           | C++11    | 	This function will never return </br>此函数永远不会返回                                                                     |
| `[[carries_dependency]]`	 | C++11    | 	Unnecessary memory fence instructions for this can be removed in some situations </br>在某些情况下，可以删除不必要的内存围栏指令        |
| `[[deprecated]]`	         | C++14	   | This is deprecated</br>   这已被弃用                                                                                     |
| `[[deprecated("why")]]`   | 	C++14   | 	This is deprecated for a specific reason</br> 这是出于特定原因而被弃用的                                                        |
| `[[fallthrough]]`	        | C++17    | 	This case in a switch intentionally falls through to the next case </br>switch 中的此 case 有意落入下一个 case               |
| `[[nodiscard]]`	          | C++17    | 	A compiler warning should be generated if this is ignored </br>如果忽略此警告，则应生成编译器警告                                   |
| `[[nodiscard("msg")]]`	   | C++20    | 	A compiler warning with a particular message should be generated if this is ignored </br> 如果忽略此消息，则应生成带有特定消息的编译器警告 |
| `[[maybe_unused]]`	       | C++17    | 	No compiler warning should be generated for not using this</br> 不使用此 API 不应生成编译器警告                                 |
| `[[likely]]`	             | C++20    | 	This branch is likely to be taken </br>这个分支很可能被占领                                                                  |
| `[[unlikely]]`	           | C++20    | 	This branch is unlikely to be taken</br> 此分支不太可能被占用                                                                |
| `[[no_unique_address]]`	  | C++20    | 	This non-static data member doesn’t need to have a unique memory address </br>此非静态数据成员不需要具有唯一的内存地址                 |

这些有两个方面需要注意。
</br>首先，并且显然，C++属性使用两个方括号(`[[X]]`)而不是C#中的一个方括号(`[X]`)。
</br>其次，所有属性都是为了两个目的之一：控制编译器警告和优化生成的代码。

第二组属性是针对编译器的，不属于C++标准的一部分。
</br>例如，`Clang`为了各种目的拥有大量这样的属性。如果指定了这些属性，而编译器不支持它们，那么它们就会被简单地忽略。

现在让我们看看一些使用它们的代码：

```c++
class File
{
    FILE* handle = nullptr;
 
public:
 
    ~File()
    {
        if (handle)
        {
            ::fclose(handle);
        }
    }
 
    // 如果忽略返回值，则生成编译器警告
    [[nodiscard]] bool Close()
    {
        if (!handle)
        {
            return true;
        }
        return ::fclose(handle) == 0;
    }
 
    // 如果忽略返回值，则生成编译器警告
    [[nodiscard]] bool Open(const char* path, const char* mode)
    {
        if (!handle)
        {
            // 没有编译器警告，因为使用了返回值
            if (!Close())
            {
                return false;
            }
        }
        handle = ::fopen(path, mode);
        return handle != nullptr;
    }
};
 
File file{};
 
// 编译器警告：返回值被忽略
file.Open("/path/to/file", "r");
 
// 编译器警告：未使用变量
bool success = file.Open("/path/to/file", "r");
 
// 没有编译器警告：抑制未使用的变量
[[maybe_unused]] bool success = file.Open("/path/to/file", "r");
 
// 没有编译器警告，因为使用了返回值
if (!file.Open("/path/to/file", "r"))
{
    DebugLog("Failed to open file");
}
```

要同时使用多个属性，就像同时声明多个变量时一样，添加逗号：

```c++
[[deprecated("Wasn't very good. Use Hash2() instead."), nodiscard]]
uint32_t Hash(const char* bytes, std::size_t size)
{
    uint32_t hash = 0;
    for (std::size_t i = 0; i < size; ++i)
    {
        hash += bytes[i];
    }
    return hash;
}
```

对于由编译器提供的命名空间中的属性，我们使用熟悉的域解析运算符(`::`)来引用它们：

```c++
// 在“gnu”命名空间中使用“nodebug”属性，不要为此函数生成调试信息
[[gnu::nodebug]]
float Madd(float a, float b, float c)
{
    return a * b + c;
}
```

自C++17起，在命名空间中使用多个属性时，有一个避免重复命名空间名称的快捷方式

```c++
// [[gnu::returns_nonnull]] and [[gnu::nodebug]]
[[using gnu: returns_nonnull, nodebug]]
void* Allocate(std::size_t size)
{
    if (size < 4096)
    {
        return SmallAlloc(size);
    }
    return BigAlloc(size);
}
```

在C++中，属性可以出现在很多地方：变量、函数、名称、代码块、返回值等等。如果添加属性在逻辑上是合理的，那么它很可能被允许。

## Conclusion  结论

C++的结构化绑定是对C#解构的一种不同处理方式。我们不需要编写任何代码就可以为结构体和数组创建结构化绑定。
</br>对于类似元组的类型，我们不得不比C#中的Deconstruct方法编写更多的代码。
</br>幸运的是，由于标准库中存在std::tuple类模板，它显然是类似元组的，因此支持解构，所以这种情况很少需要。

C++的属性是语言中相对较少的、实际上比其C#对应物功能更弱的部分。
</br>它满足编译时的目的，例如通过控制警告和优化，但由于语言中缺乏反射，不支持任何运行时用例。
</br>如果需要，需要第三方库（例如）来添加运行时反射，但这些库并没有集成到核心语言中。
</br>这可能在C++23或未来的某个版本中发生变化，因为已经有很多工作在将编译时反射集成到语言中。






















