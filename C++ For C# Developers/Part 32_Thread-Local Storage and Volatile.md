# C++ For C# Developers: Part 32 – Thread-Local Storage and Volatile

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6166),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

在C#中，语言级别支持变量的线程存储。对于`volatile`关键字也是如此。
</br>C++也支持线程变量，但带有线程初始化和线程释放。它也有一个`volatile`关键字，但其含义与C#大不相同。
</br>继续阅读，了解如何在每种语言中正确使用这些功能。

> 原作者有简单的讲解：我这边贴一下我自己博客[TLS的文档](https://github.com/nightstardawn/myBlog/blob/main/source/_posts/CSLanguage/CS%E7%9F%A5%E8%AF%86%E6%8B%93%E5%B1%95/TLS_ThreadLocalStorage.md)

线程局部存储是一种为每个线程存储一个变量的方式。C# 和 C++ 都支持这种方式。
</br>在 C# 中，我们通过将 `[ThreadStatic]` 属性添加到静态字段上来实现。一个常见的错误是由于字段的初始化只运行一次，就像其他静态字段一样，而不是每个线程运行一次。

```csharp
// C#
public class Counter
{
    // 每个线程存储一个整型变量
    // 只初始化一次，而不是每个线程都初始化
    [ThreadStatic] public static int Value = 1;
}
 
Action a = () => DebugLog(Counter.Value);
Thread t1 = new Thread(new ThreadStart(a));
Thread t2 = new Thread(new ThreadStart(a));
t1.Start();
t2.Start();
t1.Join();
t2.Join();
 
// 第一个线程运行并且第一次使用Counter将Value初始化为1
// 第二个线程运行但没有初始化Value。使用默认值0。
// 输出：1然后0
```

C++使用关键字`thread_local`而不是属性。
</br>这个关键字可以应用于类似于C#中的静态数据成员。它还可以应用于全局作用域、命名空间作用域或任何级别的块作用域：

```c++
// 全局变量
thread_local int global = 1;
 
namespace Counters
{
    // 命名空间变量
    thread_local int ns = 1;
}
 
struct Counter
{
    // 静态数据成员
    // 非const静态数据成员不允许内联初始化
    static thread_local int member;
};
// 类外初始化是允许的
thread_local int Counter::member = 1;
 
void Foo()
{
    // 局部变量
    thread_local int local = 1;
 
    {
        // 任何嵌套块中的变量
        thread_local int block = 1;
    }
}
```

此外， `thread_local`可以被标记为`static`或`extern` ，以控制链接(Part5)：

```c++
// 全局变量可以是静态的或外部的
static thread_local int global1 = 1;
extern thread_local int global2 = 2;
 
namespace Counters
{
    // 命名空间变量可以是静态的或外部的
    static thread_local int ns1 = 1;
    extern thread_local int ns2 = 2;
}
 
void Foo()
{
    // 局部变量可以是静态的，但不能是extern
    static thread_local int local = 1;
 
    {
        // 嵌套块变量可以是静态的，但不能是外部的
        static thread_local int block = 1;
    }
}
```

请注意，静态关键字不会影响它们的存储持续时间。所有 `thread_local` 变量都是在线程开始时分配和初始化的。
</br>初始化的确切顺序没有指定，因此不应依赖于它。这与 C# 中的初始化不同，在 C# 中初始化根本不会按线程进行。

```c++
// 为每个线程初始化，而不仅仅是像C#中那样只初始化一次
static thread_local int counter = 1;
 
auto a = []{ DebugLog(counter); };
std::thread t1{a};
std::thread t2{a};
t1.join();
t2.join();
 
// 第一个线程运行并初始化计数器为1
// 第二个线程运行并初始化计数器为1
// 输出：1然后1
```

如果初始化抛出异常，程序将调用`std::terminate`来关闭。

```c++
struct Throws
{
    Throws()
    {
        throw 123;
    }
};
 
// Initializing throws an exception which calls std::terminate
static thread_local Throws t{};
```

线程局部变量在线程结束时会被释放和去初始化：

```c++
struct LogLifecycle
{
    int Value = 1;
 
    LogLifecycle()
    {
        DebugLog("ctor");
    }
 
    ~LogLifecycle()
    {
        DebugLog("dtor");
    }
};
 
thread_local LogLifecycle x{};
 
auto a = []{ DebugLog(x.Value); };
std::thread t1{a};
std::thread t2{a};
t1.join();
t2.join();
 
// 可能的注解输出，取决于线程的执行顺序：
//   ctor     // 第一个线程初始化 x
//   ctor     // 第二个线程初始化 x
//   1        // 第一个线程打印 x.Value
//   dtor     // 第一个线程销毁 x
//   1        // 第二个线程打印 x.Value
//   dtor     // 第二个线程销毁 x
```

在C#中，此类每个线程的初始化和销毁都需要手动实现。

## Volatile  挥发性的

C#和C++都使用了`volatile`关键字，但这两个语言中的意义不同。在C#中，`volatile`关键字旨在用于线程同步。
它保证了`volatile`变量的原子性读取和写入，这意味着它们不会被其他线程中断。

为了确保原子性，只有某些类型在C#中可以是`volatile`：

- 所有引用类型（如类、接口、数组等）
- 指针（仅限不安全上下文）
- 简单类型，如 sbyte、byte、short、ushort、int、uint、char、float 和 bool。
- 具有以下基本类型之一的 enum 类型：byte、sbyte、short、ushort、int 或 uint。
- 已知为引用类型的泛型类型参数。
- IntPtr 和 UIntPtr。

所有其他类型，包括 `double`、`long` 以及所有结构体，都不能是 `volatile`：

> 这里可以先去看这篇[文章](https://blog.csdn.net/qq_39847278/article/details/145320928)
> 之后读者会再写一篇文章来简单的聊一聊这个的。(画饼～(∠・ω< )⌒★)

```csharp
// C#
public class Name
{
    public string First;
    public string Last;
}
 
public struct IntWrapper
{
    public int Value;
}
 
public enum IntEnum : int
{
}
 
public enum LongEnum : long
{
}
 
unsafe public class Volatiles<T>
    where T : class
{
    // OK: reference type
    volatile Name RefType;
 
    // OK: type parameter known to be a reference type due to where constraint
    volatile T TypeParam;
 
    // OK: pointer
    volatile int* Pointer;
 
    // OK: permitted primitive type
    volatile int GoodPrimitive;
 
    // Compiler error: denied primitive type
    volatile long BadPrimitive;
 
    // OK: enum based on permitted primitive type
    volatile IntEnum GoodEnum;
 
    // Compiler error: enum based on denied primitive type
    volatile LongEnum BadEnum;
 
    // Compiler error: structs can't be volatile
    // No exception for structs that only have one field that can be volatile
    volatile IntWrapper Struct;
 
    // OK: Special-case for IntPtr and UIntPtr structs
    volatile IntPtr SpecialPtr1;
    volatile UIntPtr SpecialPtr2;
}
```

在C#中，唯一可以被声明为`volatile`的变量是类和结构体的字段。局部变量和参数不能被声明为`volatile`。

C# 还会隐式地添加内存栅栏来禁用可能由执行“乱序”指令的CPU执行的操作重排和数据缓存。
对于每个对`volatile`变量的读取，都会插入一个“获取栅栏”，而对于每个写入，都会插入一个“释放栅栏”

```csharp
// C#
public struct Counter
{
    public volatile int Value;
 
    public void Increment()
    {
        // 读取操作会获得一个隐式的获取栅栏
        int cur = this.Value; // 围栏获取
 
        int next = cur + 1;
 
        // 写入获得隐式释放栅栏
        this.Value = next; // 释放围栏
    }
}
```

另一方面，C++对`volatile`的实现方式不同。关键字是相同的，但它并不是用来进行线程同步的。相反，它是用来实现内存映射硬件访问的：

```c++
// 硬件设备使用32位整数报告其状态
enum class DeviceStatus : int32_t
{
    OK = 0,
    Stuck = 1,
    Fault = 2,
};
 
// 指向一个volatile的DeviceStatus指针
// 它存储在固定的位置：内存映射地址
volatile DeviceStatus* pDeviceStatus = (volatile DeviceStatus*)100;
 
while (true)
{
    // 读取并打印设备状态
    DebugLog("Device status:", *pDeviceStatus);
 
    // 等待一秒钟
    std::this_thread::sleep_for(std::chrono::seconds{1});
}
```

在这里，`volatile`关键字被应用于`pDeviceStatus`指向的`DeviceStatus`。
这告诉编译器它不能假设它对那个32位整数的读取者和写入者有完全的可见性。它必须假设它可能被外部访问，例如当设备驱动程序将设备的状态写入内存地址`100`时。

因此，编译器不允许像这样“优化”我们的循环：

```c++
// 仅读取指针一次
// 将其存储为局部变量，可能由寄存器支持
DeviceStatus status = *pDeviceStatus;
 
while (true)
{
    // 从局部变量中打印设备状态
    // 没有缓存缺失的可能性！
    DebugLog("Device status:", status);
 
    // 等待一秒钟
    std::this_thread::sleep_for(std::chrono::seconds{1});
}
```

上述“优化”使代码运行更快，因为在通过`pDeviceStatus`指针读取时没有缓存未命中。状态只读取一次并存储在CPU寄存器中，这基本上是免费读取的。
</br>编译器看不到内核驱动程序对内存地址100的写入，因此它可以假设这是一个安全的优化。

唯一的问题是，我们记录的设备状态不能再改变。通过将`pDeviceStatus`指向的值标记为`volatile`，编译器被禁止进行这种优化。它必须假设存在一个可能更改设备状态的外部写入器。

另一个`volatile`的效果是，编译器不允许对其他`volatile`变量重新排序读取和写入操作：

```c++
// 设备状态
enum class DeviceStatus : int32_t
{
    OK = 0,
    Stuck = 1,
    Fault = 2,
    CommandAccepted = 3,
    CommandRejected = 4,
};
 
// 向设备发送的指令
enum class DeviceCommand : int32_t
{
    Retry = 1,
};
 
// 内存映射设备I/O
volatile DeviceStatus* pDeviceStatus = (volatile DeviceStatus*)100;
volatile DeviceCommand* pDeviceCommand = (volatile DeviceCommand*)200;
 
while (true)
{
    if (*pDeviceStatus == DeviceStatus::Stuck) // read
    {
        *pDeviceCommand = DeviceCommand::Retry; // write
        while (*pDeviceStatus != DeviceStatus::CommandAccepted) // read
        {
        }
        if (*pDeviceStatus == DeviceStatus::CommandRejected || // read
            *pDeviceStatus == DeviceStatus::Stuck) // read
        {
            throw std::runtime_error{"Failed to get device un-stuck"};
        }
    }
 
    // 等待一秒钟
    std::this_thread::sleep_for(std::chrono::seconds{1});
}
```

如果没有使用`volatile`，编译器可以自由地对这些读写操作进行重新排序，只要它遵守“as-if”规则，即代码的工作效果“如同”编译器没有进行重新排序。
</br>下面是它可能的样子：

```c++
// 无volatile的内存映射设备I/O
DeviceStatus* pDeviceStatus = (DeviceStatus*)100;
DeviceCommand* pDeviceCommand = (DeviceCommand*)200;
 
while (true)
{
    if (*pDeviceStatus == DeviceStatus::Stuck)
    {
        // Read status first
        DeviceStatus status = *pDeviceStatus;
 
        // Write command second
        *pDeviceCommand = DeviceCommand::Retry;
 
        // Check status
        while (status != DeviceStatus::CommandAccepted)
        {
            status = *pDeviceStatus;
        }
        if (*pDeviceStatus == DeviceStatus::CommandRejected || // read
            *pDeviceStatus == DeviceStatus::Stuck) // read
        {
            throw std::runtime_error{"Failed to get device un-stuck"};
        }
    }
 
    // Wait for one second
    std::this_thread::sleep_for(std::chrono::seconds{1});
}
```

在这个没有`volatile`中，编译器决定我们应该先读取状态，然后再写入命令。
</br>这可能会导致我们读取到之前命令的旧`CommandRejected`状态，然后即使在我们的重试命令被接受的情况下，也会抛出异常。
</br>通过应用`volatile`关键字，我们禁用这种重排序，并保证我们的`volatile`读写操作按照它们被写入的顺序发生。

到目前为止，我们没有看到C++对`volatile`读取和写入是原子或隔离的保证，就像在C#中那样。 这是因为C++中根本不是这种情况。
</br>这是一个关键的区别，它对它们在多线程等场景中的使用以及性能有影响。

由于缺乏原子性保证，C++中的任何类型都可以是易变的。
</br>没有必要仅仅因为访问它们可能不是原子的，就禁止struct、`double`和`long`。
</br>正如我们已经看到的，也可以采用指向 `volatile` 变量的指针（和引用）：

```c++
struct Vector3d
{
    double X;
    double Y;
    double Z;
};
 
volatile Vector3d V{2, 4, 6}; // Struct
volatile uint64_t L; // Long
volatile double D; // Double
volatile int A[1000]; // Array
```

此外，在C++中，任何变量都可以是`volatile`的。我们不仅限于数据成员。我们可以使局部变量、嵌套块变量、参数、全局变量和命名空间成员成为`volatile`：

```c++
volatile int global;
 
namespace Volatiles
{
    volatile int ns;
}
 
void Foo(volatile int param)
{
    volatile int local;
 
    {
        volatile int block;
    }
}
```

`volatile` 关键字是类似于 `const` 的 “type 限定符”。简写 “cv” 和 “cv-qualified” 通常用于谈论这两个限定词。
</br>与 `const` 一样，非 `volatile` 类型可以隐式被视为 `volatile` 类型，但反之则不然。
</br>这同样适用于被视为 `const` 和 `volatile` 类型的非 `volatileconst` 类型：

```c++
int nc_nv = 100;
const int c_nv = 200;
volatile int nc_v = 300;
const volatile int c_v = 400;
 
{
    int& i1 = nc_nv; // OK
    int& i2 = c_nv; // Compiler error: removes const
    int& i3 = nc_v; // Compiler error: removes volatile
    int& i4 = c_v; // Compiler error: removes const and volatile
}
 
{
    const int& i1 = nc_nv; // OK
    const int& i2 = c_nv; // OK
    const int& i3 = nc_v; // Compiler error: removes volatile
    const int& i4 = c_v; // Compiler error: removes volatile
}
 
{
    volatile int& i1 = nc_nv; // OK
    volatile int& i2 = c_nv; // Compiler error: removes const
    volatile int& i3 = nc_v; // OK
    volatile int& i4 = c_v; // Compiler error: removes const
}
 
{
    const volatile int& i1 = nc_nv; // OK
    const volatile int& i2 = c_nv; // OK
    const volatile int& i3 = nc_v; // OK
    const volatile int& i4 = c_v; // OK
}
```

这里的通用规则是，我们可以将变量视为“更const”或“更volatile”，但不能视为“更不const”或“更不volatile”，因为这会移除重要的限制。

请注意，我们应用于数据成员的`mutable`关键字不是像`const`这样的类型限定符。
</br>它实际上是一个类似于`static`、`extern`或`registe`r以及`thread_local`的“存储类指定符”，它仅适用于数据成员。
</br>这就是为什么我们不能像`const int`一样声明具有类型`mutable int`的局部或全局变量。

## Conclusion  结论

这两种语言都有线程局部存储(TLS)和`volatile`关键字，但它们之间存在显著差异。
- C++中的线程局部存储可以应用于更多种类的变量，例如局部变量和全局变量。它还保证了每个线程的初始化，而C#只初始化一次。它还具备线程终止时的解初始化功能。
- C#代码需要手动实现每个线程的初始化和解初始化。

至于`volatile`关键字，它在C#和C++中的预期用途和实现方式存在显著差异。
- 在C#中，我们得到保证的原子访问和内存栅栏，这对于同步多个线程非常有用。
- 在C++中，我们只是禁用了一些可能会妨碍内存映射I/O的编译器优化。
线程同步通常使用其他工具来解决，例如互斥锁和标准库中的`std::atomic`类模板。
由于这两个语言中关键字的命名相同，许多程序员认为它们的功能也相同。重要的是要知道情况并非如此，并且需要在每种语言中适当地使用该关键字。




