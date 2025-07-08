# C++ For C# Developers: Part 46 – Other Containers Library


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6817),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

类似数组的容器(Part45)并非我们所需的唯一容器。
</br>今天我们将探讨 C++标准库提供的非数组容器，包括其对应于 `Dictionary` 、 `HashSet` 和 `LinkedList` 的等价物。

## Unordered Map  无序映射

`<unordered_map>` 头文件提供了 C++ 的 `Dictionary` 对应物： `std::unordered_map` 。
与其他容器（如 `std::vector`）类似，它“拥有”存储键和值的内存。以下是 API 的一个示例：

```c++
#include <unordered_map>
 
void Foo()
{
    // 整数键到浮点值的哈希表
    std::unordered_map<int, float> ifum{};
 
    // 添加键值对
    ifum.insert({ 123, 3.14f });
 
    // 读取123映射的值
    DebugLog(ifum[123]); // 3.14
 
    // 尝试读取456映射的值
    // 没有这样的键，因此插入默认初始化的值
    DebugLog(ifum[456]); // 0
 
    // 查询大小
    DebugLog(ifum.empty()); // false
    DebugLog(ifum.size()); // 2
    DebugLog(ifum.max_size()); // Maybe 768614336404564650
 
    // 尝试读取并在找不到键时抛出异常
    DebugLog(ifum.at(123)); // 3.14
    DebugLog(ifum.at(1000)); // 抛出 std::out_of_range 异常
 
    // insert() 不覆盖
    ifum.insert({ 123, 2.2f }); // 不覆盖3.14
    DebugLog(ifum[123]); // 3.14
 
    // insert_or_assign()会覆盖
    ifum.insert_or_assign(123, 2.2f); // 覆盖 3.14
    DebugLog(ifum[123]); // 2.2
 
    // emplace() 在原地构造
    ifum.emplace(456, 1.123f);
 
    // 删除一个元素
    ifum.erase(456);
    DebugLog(ifum.size()); // 1
} // ifum的析构函数释放存储键和值的内存
```

当可能存在多个相同键时，也有一个 `std::unordered_multimap` 可用。
</br>C# 没有这个类模板的等效项，但可以用 `Dictionary<TKey, List<TValue>>` 来近似。以下是使用方法：

```c++
#include <unordered_map>
 
void Foo()
{
    // 创建一个空的multimap，它将int映射到float
    std::unordered_multimap<int, float> ifumm{};
 
    // 插入两个相同的键，但值不同
    ifumm.insert({ 123, 3.14f });
    ifumm.insert({ 123, 2.2f });
 
    // 检查映射到123键的值有多少个
    DebugLog(ifumm.count(123)); // 2
 
    // C++20：检查是否有任何值映射到123键
    DebugLog(ifumm.contains(123)); // true
 
    // 找到123键的一个键值对
    const auto& found = ifumm.find(123);
    DebugLog(found->first, found->second); // Maybe 123, 3.14
 
    // 遍历123键的所有键值对
    auto range = ifumm.equal_range(123);
    for (auto i = range.first; i != range.second; ++i)
    {
        DebugLog(i->first, i->second); // 123, 3.14 and 123, 2.2
    }
 
    // 删除所有具有给定键的键值对
    ifumm.erase(123);
    DebugLog(ifumm.size()); // 0
} // ifumm的析构函数释放键和值的内存
```

C++20 增加了一个与数组容器类型一起使用的 `std::erase_if` 非成员函数，用于处理 `std::unordered_map` 和 `std::unordered_multimap`。

```c++
#include <unordered_map>
 
// 为每个映射创建2个键值对
std::unordered_multimap<int, float> umm{ {123, 3.14}, {456, 2.2f} };
std::unordered_map<int, float> um{ {123, 3.14}, {456, 2.2f} };
 
// 删除所有键值对，其中键小于200
auto lessThan200 = [](const auto& pair) {
    const auto& [key, value] = pair;
    return key < 200;
};
std::erase_if(um, lessThan200);
std::erase_if(umm, lessThan200);
 
// 与“123”键相关联的值为0
DebugLog(um.count(123)); // 0
DebugLog(umm.count(123)); // 0
```

## Map

`<map>` 头文件提供了 `std::unordered_map` 和 `std::unordered_multimap` 的有序版本。
</br>它们自然地被称为 `std::map` 和 `std::multimap` 。它们的 API 基础与无序版本非常相似，这有助于泛型编程，但也存在一些差异。

让我们从 `std::map` 开始，它通常是一个红黑树。
</br>这个容器的 C# 等价物最接近 `OrderedDictionary` ，但这个类不像 `std::map` 和 `Dictionary` 那样支持泛型键和值类型。
</br>以下是 std::map 功能的一些示例：

```c++
#include <map>
 
void Foo()
{
    // 创建一个包含三个键的映射
    std::map<int, float> m{ {456, 2.2f}, {123, 3.14}, {789, 42.42f} };
 
    // 许多其他容器中的功能都可用
    DebugLog(m.size()); // 3
    DebugLog(m.empty()); // false
    m.insert({ 1000, 2000.0f });
    m.erase(123);
    m.emplace(100, 9.99f);
 
    // 按键排序是保证的
    for (const auto& item : m)
    {
        DebugLog(item.first, item.second);
        // Prints:
        //   100, 9.99
        //   456, 2.2
        //   789, 42.42
        //   1000, 2000
    }
} // 对象m的析构函数释放键和值的内存
```

和 `std::unordered_multimap` 一样， `std::multimap` 也没有 C# 的对应物。
</br>我们可以用 `OrderedDictionary` 来近似表示，其值是 `List<TValue>` 对象，但同样存在 `OrderedDictionary` 不支持泛型键和值类型的缺点。
</br>无论如何， `std::multimap` 支持映射多个相同的键，并且通常也是红黑树。值按插入顺序存储：

```c++
#include <map>
 
void Foo()
{
    // 创建一个具有三个键的多映射：其中两个键是重复的
    std::multimap<int, float> mm{ {456, 42.42f}, {123, 3.14f}, {123, 2.2f} };
 
    // 许多其他容器中的功能都可用
    DebugLog(mm.size()); // 3
    DebugLog(mm.empty()); // false
    mm.insert({ 1000, 2000.0f });
    mm.erase(456);
    mm.emplace(100, 9.99f);
 
    // 按键排序是保证的
    for (const auto& item : mm)
    {
        DebugLog(item.first, item.second);
        // Prints:
        //   100, 9.99
        //   123, 3.14
        //   123, 2.2
        //   1000, 2000
    }
} // mm的析构函数释放键和值的内存
```

## Unordered Set  无序集合

头文件 `<unordered_set>` 提供了 C# 中 `HashSet` 的等效功能： `std::unordered_set` 。
</br>它类似于 `std::unordered_map` ，但只有键，因此 API 更简单：

```c++
#include <unordered_set>
 
void Foo()
{
    // 创建一个包含四个值的集合
    std::unordered_set<int> us{ 123, 456, 789, 1000 };
 
    // 许多其他容器中的功能都可用
    DebugLog(us.size()); // 4
    DebugLog(us.empty()); // false
    us.insert(2000);
    us.erase(456);
    us.emplace(100);
    DebugLog(us.count(123)); // 1
} // us的析构函数释放值内存
```

一个 `std::unordered_multiset` 也可以支持多个相同的值。
</br>C# 没有这个版本的实现，但可以使用 `Dictionary<TKey, int>` 通过用值来计数键来近似实现它。
</br>这需要额外的内存，并且与 `std::unordered_multiset` 的直接 API 相比，使用起来有些笨拙。

```c++
#include <unordered_set>
 
void Foo()
{
    // 创建一个包含六个值的集合：其中两个是重复的
    std::unordered_multiset<int> ums{ 123, 456, 123, 789, 1000, 1000 };
 
    // 许多其他容器中的功能都可用
    DebugLog(ums.size()); // 6
    DebugLog(ums.empty()); // false
    ums.insert(2000);
    ums.erase(123); // erases both
    ums.emplace(100);
    DebugLog(ums.count(1000)); // 2
} // ums的析构函数释放值内存
```

## Set

和 `map` 一样，`set` 类的有序版本也是可用的。
</br>`<set>` 头文件提供了 `std::set` 和 `std::multiset` 。两者都是用更多的红黑树实现的。
</br>C# 有一个与 `std::set` 相当的 `SortedSet` 类，但与 `std::multiset` 不相当。

`std::set` 和 `std::multiset` 的 API 也与它们的无序版本非常相似。这是 `std::set` ：

```c++
#include <set>
 
void Foo()
{
    // 创建一个包含四个值的集合
    std::set<int> s{ 123, 456, 789, 1000 };
 
    // 许多其他容器中的功能都可用
    DebugLog(s.size()); // 4
    DebugLog(s.empty()); // false
    s.insert(2000);
    s.erase(456);
    s.emplace(100);
    DebugLog(s.count(123)); // 1
 
    // 按键排序是保证的
    for (int x : s)
    {
        DebugLog(x);
        // Prints:
        //   100
        //   123
        //   789
        //   1000
        //   2000
    }
} // s的析构函数释放值内存
```

`std::multiset` 也支持多个相同的值，并且通常也是红黑树。以下是它的 API 示例：

```c++
#include <set>
 
void Foo()
{
    // 创建一个包含六个值的集合：其中两个是重复的
    std::multiset<int> ms{ 123, 456, 123, 789, 1000, 1000 };
 
    // 许多其他容器中的功能都可用
    DebugLog(ms.size()); // 6
    DebugLog(ms.empty()); // false
    ms.insert(2000);
    ms.erase(123); // erases both
    ms.emplace(100);
    DebugLog(ms.count(1000)); // 2
 
    // Ordering is guaranteed
    for (int x : ms)
    {
        DebugLog(x);
        // Prints:
        //   100
        //   456
        //   789
        //   1000
        //   1000
        //   2000
    }
} // ms的析构函数释放值内存
```

## List  列表

接下来是 `<list>` 头文件，以及它的 `std::list` 类模板，用于实现双向链表。
这相当于 C# 中的 `LinkedList` 类。它的 API 与 `std::vector` 类似，但支持对链表前端的操作，并且不允许索引，因为那将需要昂贵的链表遍历：

```c++
#include <list>
 
void Foo()
{
    // 创建一个空列表
    std::list<int> li{};
 
    // 添加一些值
    li.push_back(456);
    li.push_front(123);
 
    // 通过插入默认初始化值来增长
    li.resize(5);
 
    // 查询大小
    DebugLog(li.empty()); // false
    DebugLog(li.size()); // 5
 
    // 索引不支持。请使用循环。
    for (int x : li)
    {
        DebugLog(x);
        // Prints:
        //   123
        //   456
        //   0
        //   0
        //   0
    }
 
    // 移除值
    li.pop_back();
    li.pop_front();
 
    // 特殊操作
    li.sort();
    li.remove(0); // 移除所有零
    li.remove_if([](int x) { return x < 200; }); // 删除所有低于200的
} // li的析构函数释放值内存
```

## Forward List  前向列表

单链表 `std::forward_list` 也可以通过 `<forward_list>` 提供。
</br>C# 没有提供等效的容器。API 类似于 `std::vector` 的反向，因为只支持对列表前端的操作。
</br>与大多数其他容器不同，没有提供 `size` 函数，因为这将需要遍历整个列表来计数节点：

```c++
#include <forward_list>
 
void Foo()
{
    // 创建一个空列表
    std::forward_list<int> li{};
 
    // 添加一些值
    li.push_front(123);
    li.push_front(456);
 
    // 通过插入默认初始化值来增长
    li.resize(5);
 
    // 查询大小
    DebugLog(li.empty()); // false
 
    // 索引不支持。请使用循环。
    for (int x : li)
    {
        DebugLog(x);
        // Prints:
        //   123
        //   456
        //   0
        //   0
        //   0
    }
 
    // 移除值
    li.pop_front();
 
    // 特殊操作
    li.sort();
    li.remove(0); // 移除所有零
    li.remove_if([](int x) { return x < 200; }); // 删除所有低于200的
} // li的析构函数释放值内存
```

## Conclusion  结论

C++标准库提供了与数组集合类型（如 std::vector ）相配套的、稳健且一致的、非数组集合类型。
</br>这些 API 非常相似，这对于泛型编程来说很棒，因为集合类型可以很容易地被转化为类型参数。
</br>无论我们需要集合、映射还是列表，需要排序还是哈希，甚至支持重复的键或值，都提供了一个类模板。

相比之下，C# 的功能更为有限，有时没有泛型版本，不支持重复的键，或者没有处理排序的类。
</br>这些可能是不太常见的用例，但当需要时，有标准化的工具可用会让人感到很方便。






















