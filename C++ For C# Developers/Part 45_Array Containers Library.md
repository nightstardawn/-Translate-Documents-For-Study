# C++ For C# Developers: Part 45 – Array Containers Library


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6767),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

我们经常使用某些容器类型，如 `map` 和动态数组，而其他类型，如链表和队列，则使用较少。
</br>然而，它们几乎是每个程序中的基本结构，也是泛型编程的典范。
</br>与 C# 类似，C++ 的标准库提供了多种容器类型。今天我们将开始逐一介绍它们，首先从各种数组的容器开始！

## Vector  向量

让我们从一个最常用的容器类型开始： `std::vector` 。这个类在 `<vector>` 中，相当于 C#中的 `List` ，因为它实现了一个动态数组。以下是它的 API 示例：

```c++
#include <vector>

void Foo()
{
    // 创建空的 int 类型 vector
    std::vector<int> v{};
 
    // 在末尾添加元素
    v.push_back(123);
    
    //删除末尾元素
    v.pop_back();
    //删除索引元素
    v.erase(v.begin() + 2); 
 
    // 在末尾原地构造元素
    v.emplace_back(456);
 
    // 获取大小信息
    DebugLog(v.empty()); // false
    DebugLog(v.size()); // 有效数据大小 2
    DebugLog(v.capacity()); // 容量 至少为 2
    DebugLog(v.max_size()); // 可能为 4611686018427387903
 
    // 调整容量请求
    v.reserve(100); // 注意：不能缩小容量
    DebugLog(v.capacity()); // 100
    v.shrink_to_fit();
    DebugLog(v.capacity()); // 可能为 2
 
    // 缩容到仅保留第一个元素
    v.resize(1);
 
    // 在末尾添加两个默认初始化的元素
    v.resize(3);
 
    // 使用下标操作符访问元素
    v[2] = 789;
    DebugLog(v[0], v[1], v[2]); // 123, 0, 789
 
    // 访问首尾元素
    DebugLog(v.front()); // 123
    v.back() = 1000;
    DebugLog(v[2]); // 1000
 
    // 获取指向首元素的指针
    int* p = v.data();
    DebugLog(p[0], p[1], p[2]); // 123, 0, 1000
 
    // 创建包含四个元素的 vector
    std::vector<int> v2{ 2, 4, 6, 8 };
 
    // 比较两个 vector 的元素
    DebugLog(v == v2); // false
 
    // 用 v2 的元素替换 v 的元素
    v = v2;
    DebugLog(v.size()); // 4
    DebugLog(v[0], v[1], v[2], v[3]); // 2, 4, 6, 8
} // 析构函数自动释放 v 和 v2 的内存
```

在 C++20 中， `<vector>` 头文件还提供了一些非成员函数来从 `std::vector` 中删除元素：

```c++
#include <vector>
 
std::vector<int> v1{ 100, 200, 200, 200, 300 };
 
// 删除所有等于200的元素
std::vector<int>::size_type numErased = std::erase(v1, 200);
DebugLog(numErased); // 3
DebugLog(v1.size()); // 2
DebugLog(v1[0], v1[1]); // 100, 300
 
std::vector<int> v2{ 1, 2, 3, 4, 5 };
 
// 删除所有偶数
numErased = std::erase_if(v2, [](int x) { return (x % 2) == 0; });
DebugLog(numErased); // 2
DebugLog(v2.size()); // 3
DebugLog(v2[0], v2[1], v2[2]); // 1, 3, 5
```

和 `std::string` 一样， `std::vector` “拥有”存储元素的内存。
</br>它在构造函数和`push_back`等函数需要时分配内存，并在 `shrink_to_fit` 等函数需要扩展时以及析构函数中释放内存。
</br>与 C#中的 List 管理类不同，它不会被垃圾回收，因为 C++没有垃圾回收器。如果需要，我们可以使用引用计数的 `std::shared_ptr` 来模拟这一功能。

## Array  数组

有时我们想使用数组，但不想承担动态数组的开销，或者想将其分配在栈上而不是堆上。
</br>在 C#中，我们会使用 `stackalloc` 或 `fixed` 缓冲区。如果堆分配可以接受，我们也可以使用管理数组。
</br>在 C++中，我们可以使用数组，但这有一些显著的缺点。它会退化成指针，其大小不容易检查，并且缺少 `std::vector` 提供的所有便捷成员函数。


为了解决这些问题， `<array>` 头文件提供了 `std::array` 类模板，用于存储静态大小的数组。 
`std::vector` 持有指向数组第一个元素的指针，而 `std::array` 则持有实际的数组。
这是 `struct Vector { int* P; };` 和 `struct Array { int E[100]; };` 之间的区别。
这种区别使得 `std::array` 可以在栈上分配。它还要求将大小指定为模板参数：

```c++
#include <array>
 
// 创建一个包含3个整型元素的数组
std::array<int, 3> a{ 1, 2, 3 };
 
// 查询其大小
DebugLog(a.size()); // 3
DebugLog(a.max_size()); // 3
DebugLog(a.empty()); // false
 
// 读取和写入其元素
DebugLog(a[0], a[1], a[2]); // 1, 2, 3
a[0] = 10;
DebugLog(a.front()); // 10
DebugLog(a.back()); // 3
 
// 获取第一个元素的指针
int* p = a.data();
DebugLog(p[0], p[1], p[2]); // 10, 2, 3
 
// 创建另一个包含3个整数的数组
std::array<int, 3> a2{ 10, 2, 3 };
 
// 比较数组元素
DebugLog(a == a2); // true
```

请注意， `std::array` 不要求它在栈上分配。我们可以像任何其他类一样轻松地在堆上分配一个。

```c++
#include <array>
 
std::array<int, 3>* a = new std::array<int, 3>{ 1, 2, 3 };
DebugLog((*a)[0], (*a)[1], (*a)[2]); // 1, 2, 3
delete a;
```

C++20 增加了非成员 `std::to_array` 函数，用于将数组复制到 `std::array`：

```c++
#include <array>
 
// 普通数组
int a[3] = { 1, 2 };
 
// 将普通数组复制到 std::array 中
std::array<int, 3> c{ std::to_array(a) };
 
// 改变一个不会改变另一个
a[0] = 10;
c[1] = 20;
DebugLog(a[0], a[1]); // 10, 2
DebugLog(c[0], c[1]); // 1, 20
```

## Valarray

`<valarray>` 头文件是数字库(Part)的一部分，也是一个容器。 
`std::basic_string` 也是如此。与 `std::vector` 类似，它们都基于数组。
主要区别在于操作这些数组的成员函数集。
由于 `std::basic_string` 提供了查找子字符串的函数， `std::valarray` 更侧重于对 `std::valarray` 对象的每个元素或两个 `std::valarray` 对象中的元素对进行操作。

```c++
#include <valarray>
 
void Foo()
{
    // 创建一个包含两个 int 类型的数组
    std::valarray<int> va1{ 10, 20 };
 
    // 创建一个包含两个 int 类型的数组
    std::valarray<int> va2{ 10, 30 };
 
    // 在每个中比较元素0，每个中的元素1，依此类推
    // 返回一个比较结果的valarray
    std::valarray<bool> eq{ va1 == va2 };
    DebugLog(eq[0], eq[1]); // true, false
 
    // 元素对应相加
    std::valarray<int> sums{ va1 + va2 };
    DebugLog(sums[0], sums[1]); // 20, 50
 
    // 访问元素
    DebugLog(va1[0]); // 10
    va1[1] = 200;
    DebugLog(va1[0], va1[1]); // 10, 200
 
    // 将元素1向前移动，用零填充
    std::valarray<int> shifted{ va1.shift(1) };
    DebugLog(shifted[0], shifted[1]); // 200, 0
 
    // 将元素1向前移动，围绕到后面
    std::valarray<int> cshifted{ va1.cshift(1) };
    DebugLog(cshifted[0], cshifted[1]); // 200, 10
 
    // 将所有元素复制到另一个valarray中
    va1 = va2;
    DebugLog(va1[0], va1[1]); // 10, 30
 
    // 使用每个元素调用一个函数并将返回值分配给它
    std::valarray<int> plusOne{ va1.apply([](int x) { return x + 1; }) };
    DebugLog(plusOne[0], plusOne[1]); // 11, 33
 
    // 取2的4次方和3的2次方
    std::valarray<float> bases{ 2, 3 };
    std::valarray<float> powers{ 4, 2 };
    std::valarray<float> squares{ std::pow(bases, powers) };
    DebugLog(squares[0], squares[1]); // 16, 9
} // Destructors free memory of all valarrays
```

与 `std::vector` 类似， `std::valarray` “拥有”存储其元素的内存。C# 没有这个类模板的等效物。

存在一些辅助类，可以通过将类的实例传递给重载的 `[]` 运算符，从 `std::valarray` 中“切片”出多个元素：

```c++
#include <valarray>
 
std::valarray<int> va1{ 10, 20, 30, 40, 50, 60, 70 };
 
// 从索引1开始的切片，包含2个元素，步长为0
std::slice s{ 1, 2, 0 };
 
// 将valarray切片以获取指向切片的slice_array
std::slice_array<int> sa{ va1[s] };
 
// 将切片复制到新的valarray中
std::valarray<int> sliced{ sa };
DebugLog(sliced.size()); // 2
DebugLog(sliced[0], sliced[1]); // 20, 30
 
// 从索引1开始，大小为2和3，步长为1和2的切片
std::gslice g{ 1, {2, 3}, {1, 2} };
 
// 将valarray切片以获取一个指向切片的gslice_array
std::gslice_array ga{ va1[g] };
 
// 将切片复制到新的valarray中
std::valarray<int> gsliced{ ga };
DebugLog(gsliced.size()); // 6
DebugLog(gsliced[0], gsliced[1], gsliced[2]); // 20, 40, 60
DebugLog(gsliced[3], gsliced[4], gsliced[5]); // 30, 50, 70
```

## Deque  双向队列

`std::deque` 发音类似于“deck”，位于 `<deque>` 中，是一种拥有其元素的队列。
</br>在内部，它持有数组的列表，但这一点被其 API 隐藏了，看起来就像是一个与 `std::vector` 类似的连续数组。
</br>这意味着元素访问涉及第二次间接引用，但从 `std::deque` 的开头和结尾添加和删除元素非常快。
</br>C# 没有这种容器类型的等效物。以下是使用方法：


```c++
#include <deque>
 
void Foo()
{
    // 创建一个包含三个浮点数的双向队列
    std::deque<float> d{ 10, 20, 30 };
 
    // 大小
    DebugLog(d.size()); // 3
    DebugLog(d.max_size()); // Maybe 4611686018427387903
    DebugLog(d.empty()); // false
 
    // 访问
    DebugLog(d.front()); // 10
    d[1] = 200;
    DebugLog(d[1]); // 200
    DebugLog(d.back()); // 30
 
    // 增
    d.push_front(5);// 在前面添加元素
    d.push_back(35);// 在后面添加元素
    DebugLog(d[0], d[1], d[2], d[3], d[4]); // 5, 10, 200, 30, 35
    d.pop_front(); // 删除前面元素
    d.pop_back();// 删除后面元素
    DebugLog(d[0], d[1], d[2]); // 10, 200, 30
 
    // 将容器缩小指定大小
    // 删除除了前两个元素之外的所有元素
    d.resize(2);
    DebugLog(d.size()); // 2
    DebugLog(d[0], d[1]); // 10, 200
 
    // 比较两个双向队列的元素
    std::deque<float> d2{ 10, 200 };
    DebugLog(d == d2); // true
 
    // 删除特定元素的所有值
    std::deque<float>::size_type numErased = std::erase(d, 10);
    DebugLog(numErased); // 1
    DebugLog(d[0]); // 200
 
    // 移除所有函数返回true的元素
    numErased = std::erase_if(d2, [](float x) { return x < 100; });
    DebugLog(numErased); // 1
    DebugLog(d2[0]); // 200
}  // 析构函数释放所有双端队列的内存
```

## Queue  队列

与 `std::deque` 不同， `<queue>` 中的 `std::queue` 类模板是一个适配器，用于为其他集合提供队列 API。
</br>默认的容器类型是 `std::deque` ，但也可以使用具有 `back` 、 `front` 、 `push_back` 和 `push_front` 成员函数的其他容器。 
</br>`std::queue` 包含此集合类型，并提供由包含的集合实现的成员函数。

```c++
#include <queue>
#include <deque>
 
void Foo()
{
    // 显式地使用 std::deque<int> 来存储 std::queue<int> 的元素
    std::queue<int, std::deque<int>> qd{};
 
    // 使用默认的集合类型，即 std::deque
    // 它最初为空
    std::queue<int> q{};
 
    // 将元素添加到末尾
    q.push(10);
    q.push(20);
    q.emplace(30); // 在原地构造元素 
 
    // 查询大小
    DebugLog(q.size()); // 3
    DebugLog(q.empty()); // false
 
    // 仅访问第一个和最后一个元素
    DebugLog(q.front()); // 10
    DebugLog(q.back()); // 30
 
    // 从前面移除元素
    q.pop();
    DebugLog(q.size()); // 2
    DebugLog(q.front()); // 20
 
    // 将元素复制到另一个队列
    std::queue<int> q2{};
    q2 = q;
    DebugLog(q2.size()); // 2
    DebugLog(q2.front()); // 20
    DebugLog(q2.back()); // 30
} // 析构函数释放所有队列的内存
```

C# 有一个等效的 `Queue` 类。与 `std::queue` 不同，它不是其他集合类型的适配器。相反，它内部实现了一个特定的集合。

## Stack  栈

另一种适配器类型，类似于 `std::queue` ，在 `<stack>` 头文件中的 `std::stack` 。
</br>它也默认使用 `std::deque` 作为其集合类型。由于栈只操作栈顶，可以使用更多类型的集合。
</br>它们只需要具有 `back` 、 `push_back` 和 `pop_back` 成员函数：

```c++
#include <stack>
#include <vector>

void Foo()
{
    // 创建使用 std::vector 作为底层容器的栈
    std::stack<int, std::vector<int>> sv{};

    // 创建使用默认 std::deque 作为底层容器的栈
    std::stack<int> s{};

    // 向栈顶添加元素（后进）
    s.push(10);
    s.push(20);
    s.emplace(30); // 原地构造元素

    // 查询栈大小
    DebugLog(s.size()); // 3
    DebugLog(s.empty()); // false

    // 仅访问栈顶元素（最后添加的元素）
    DebugLog(s.top()); // 30

    // 从栈顶移除元素（先进）
    s.pop();
    DebugLog(s.size()); // 2
    DebugLog(s.top()); // 20

    // 将元素复制到另一个栈
    std::stack<int> s2{};
    s2 = s;
    DebugLog(s2.size()); // 2
    DebugLog(s2.top()); // 20
} // 析构函数自动释放所有栈的内存
```

和 Queue 类似，C# 在 `Stack` 中也有一个 `std::stack` 的等效项。它也不是一个适配器，而是一种独特实现的容器类型

## Conclusion  结论

数组无处不在，因此 C++标准库提供了多种容器类型来包装和访问它们。 
`vector` 、 `array` 和 `valarray` 与 `basic_string` 一样，将元素存储在连续的内存块中。 
- `vector` 类型提供动态调整大小功能
- `array` 提供栈分配和低开销
- `valarray` 提供逐元素访问和高级切片功能。

C#在 `List` 中有与 `vector` 非常相似的对应物，但 `stackalloc` 仅支持未管理类型，这使得它与 `array` 相比功能受限，而且根本没有 `valarray` 的对应物。

当我们需要廉价地向数组的开头添加和删除元素时， `deque` 类型非常实用。
</br>虽然它在内存中不是完全连续的，但仍然大部分是连续的，考虑到现在在`deque`的开头进行插入和删除操作是在 O(1) 而不是 O(N) 中完成的，这可能代表了一个可以接受的权衡。
</br>C# 缺乏这种集合类型。

最后，还有 `queue` 和 `stack` 作为适配器类型，适用于提供必要成员函数的任何集合。而 C# 的 Queue 和 Stack 类则强制要求特定的集合实现。


















