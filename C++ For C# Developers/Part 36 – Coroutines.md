# C++ For C# Developers: Part 36 – Coroutines


> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/6240),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

在今天介绍 C++ 语言的最后一篇文章中，我们将探讨 C++20 的一项新功能：协程。这些函数类似于 C# 迭代器函数（即具有 `yield` 的函数）和 C# 异步函数。
</br>协程有很多有趣的方面，所以让我们深入研究探索吧！

## Coroutine Basics  协程基础知识

对于C++来说，我们通常被赋予构建高级功能的底层工具。
</br>在协程的情况下，我们得到了构建C#迭代器函数的等价物（即`yield`）和C#异步函数的等价物（即`await`）所需的工具。
</br>因为这些相对高级的功能并没有直接集成到语言中，我们可以自定义它们的工作方式，并构建出许多类似的功能。

就像C#的迭代器函数通过一个或多个`yield return`或`yield break`语句变得如此简单一样
C++的协程通过一个或多个`co_yield`、`co_return`或`co_await`运算符变得如此。
在C++中，我们不像在C#中那样在函数上放置`async`关键字。协程可以直接进行`co_await`。以下是两种语言之间的等效性：

|                                    | C#                                  | C++            |
|------------------------------------|-------------------------------------|----------------|
| Yield a value 生成一个值                | `yield return x;`                   | `co_yield x;`  |
| End with no value  直接跳出            | `yield break;`                      | `co_return;`   |
| End with a value   以值结尾            | `yield return x;`then`yield break;` | `co_return x;` |
| Wait for an async process   等待异步进程 | `await x;`                          | `co_await x;`  |


让我们开始看一下我们可以在 C++ 中构建的最简单的协程：

```c++
ReturnObj SimpleCoroutine()
{
    DebugLog("Start of coroutine");
    co_return;
    DebugLog("End of coroutine");
}
```

这是一个协程，因为它包含 `co_return`。我们没有像在 C# 中那样使用 `IEnumerable` 作为返回类型，而是使用 `ReturnObj`。
</br>此类型必须是具有特定成员的类。C++ 只是称它为“返回对象”，但它通常以其用途命名：`Generator` 用于迭代器，`Task` 用于异步任务，`Lazy` 用于延迟计算，等等。
</br>不过，我们的 `ReturnObj` 本质上是无用的，因此它有一个非常通用的名称。

我们来看一下 `ReturnObj`：

```c++
struct ReturnObj
{
    ReturnObj()
    {
        DebugLog("ReturnObj ctor");
    }
 
    ~ReturnObj()
    {
        DebugLog("ReturnObj dtor");
    }
 
    struct promise_type
    {
        promise_type()
        {
            DebugLog("promise_type ctor");
        }
 
        ~promise_type()
        {
            DebugLog("promise_type dtor");
        }
 
        ReturnObj get_return_object()
        {
            DebugLog("promise_type::get_return_object");
            return ReturnObj{};
        }
 
        NeverSuspend initial_suspend()
        {
            DebugLog("promise_type::initial_suspend");
            return NeverSuspend{};
        }
 
        void return_void()
        {
            DebugLog("promise_type::return_void");
        }
 
        NeverSuspend final_suspend()
        {
            DebugLog("promise_type::final_suspend");
            return NeverSuspend{};
        }
 
        void unhandled_exception()
        {
            DebugLog("promise_type unhandled_exception");
        }
    };
};
```

`ReturnObj` 主要用于在协程生命周期的各个时间点调用 `DebugLog`。该生命周期大致类似于 C# 迭代器函数或 `async` 函数的生命周期。
</br>在幕后，有一个实现状态机的类。该类将函数的当前状态和所有局部变量保存为字段，并保存函数的编译器重写版本以作该状态机。

在 C++ 中，我们通常会使用实现 C# 迭代器和异步功能的“返回对象”类。
</br>C++20 缺少这样的类，但社区已经填补了空白， 直到它们可以作为下一个版本（C++23）的标准库的一部分提供。
</br>为了在本文中了解这些类的工作原理，我们实际上将实现我们自己的类。

因此，查看 `ReturnObj`，我们看到它有一个 `promise_type` 成员类。
</br>如果这个类不存在或没有这个名字，我们将收到编译器错误。这是协程向其调用方做出的 “promise” 的类型。其目的是控制协程在各个执行点的行为。

接下来我们来逐个解释:

- 第一个执行点由 `get_return_object` 表示。顾名思义，这用于获取 “return object”，在本例中我们简单地返回一个默认构造的 object。
- 第二个点是 `initial_suspend`，它在 `get_return_object` 之后调用，以确定在开始时要做什么。
- 第三，我们有 `return_void`，当协程使用没有返回值的 `co_return` 时调用它：即仅 `co_return`;。
- 第四，调用 `final_suspend` 以确定当协程到达执行结束时要做什么。
- 第五，当异常逃离协程时，会调用未处理的异常处理程序。

> 这里读者补充一下C#迭代器函数和async函数的生命周期
> </br>首先是**迭代器**:
> ```csharp
> IEnumerable<int> GetNumbers()
> {
>     yield return 1;
>     yield return 2;
>     yield return 3;
> }
> ```
> 生命周期阶段：
> 1. **调用函数**
>     - 返回一个枚举器对象（`IEnumerable<T>`或`IEnumerator<T>`）。
>     - **不会立即执行函数体**，仅创建状态机实例。
> 2. **第一次调用**`MoveNext()`
>     - 执行到第一个`yield return`。
>     - 状态机保存当前执行位置和局部变量状态。
>     - 返回`true`并提供当前值（`Current`属性）。
> 3. **后续调用**`MoveNext()`
>     - 从上一次暂停的位置恢复执行。
>     - 遇到下一个`yield return`时再次暂停。
>     - 若执行到`yield break`或函数结尾，则返回`false`。
> 4. **资源清理**
>     - 如果实现了`IDisposable`，调用`Dispose()`释放资源（如文件句柄）。
>     - `finally`块在迭代结束、`Dispose()`调用或提前终止时执行。
> 
> #### 状态机原理：
> 
> 编译器生成一个类，管理：
> 
> - 当前状态（整数标签，指向下一个`yield`位置）。
> - 局部变量值。
> - 实现`IEnumerator`接口的逻辑。
>
> 然后是 async 函数的生命周期
> 
> ```csharp
> async Task<int> GetDataAsync()
> {
>     await Task.Delay(100);
>     return 42;
> }
> ```
>
> 生命周期阶段：
> 
> 1. **调用函数**
>     - 同步执行到第一个`await`。
>     - 遇到未完成的`await`时，返回一个`Task`（或派生类）。
> 2. **暂停（****`await`****）**
>     - 保存上下文（局部变量、执行位置）。
>     - 释放当前线程（通常返回到线程池）。
>     - 将未完成的`Task`返回给调用方。
> 3. **恢复**
>     - 当等待的`Task`完成时，由同步上下文（如 UI 线程或线程池）恢复执行。
>     - 从`await`之后的位置继续执行。
> 4. **完成**
>     - 若函数正常结束，标记`Task`为`RanToCompletion`并设置结果。
>     - 若抛出异常，标记`Task`为`Faulted`并保存异常。
>     - 若通过`CancellationToken`取消，标记为`Canceled`。
> 5. **资源清理**
>     - `using`和`finally`块在退出时自动执行（即使中途`await`）。
> 
> 状态机原理：
> 
> 编译器生成一个状态机类（`IAsyncStateMachine`），管理：
> 
> - 状态（当前执行位置）。
> - 上下文（局部变量、执行环境）。
> - `Task`的构建和完成通知。

`initial_suspend` 和 `final_suspend` 都返回默认构造的 `NeverSuspend` 类，因此接下来让我们看看它：

```c++
#include <coroutine>
 
struct NeverSuspend
{
    NeverSuspend()
    {
        DebugLog("NeverSuspend ctor");
    }
 
    ~NeverSuspend()
    {
        DebugLog("NeverSuspend dtor");
    }
 
    bool await_ready()
    {
        DebugLog("NeverSuspend::await_ready");
        return true;
    }
 
    void await_suspend(std::coroutine_handle<>)
    {
        DebugLog("NeverSuspend::await_suspend");
    }
 
    void await_resume()
    {
        DebugLog("NeverSuspend::await_resume");
    }
};
```

此类也有一些必要成员。
- 首先，调用 `await_ready` 以检查协程是否应暂停。当协程是同步的（如本例）时，我们可以返回 `true` 来指示不应有挂起。
- 其次，`await_suspend` 从标准库的协程头文件传递 `std：：coroutine_handle<>` 对象。
  </br>在此示例中，我们没有使用它，甚至没有给它命名，但这是一种在暂停协程时访问协程状态的方法。
- 第三个也是最后一个，当协程恢复时调用 `await_resume`。

```c++
void Foo()
{
    DebugLog("Calling coroutine");
    ReturnObj ret = SimpleCoroutine();
    DebugLog("Done");
}
```

```c++
Calling coroutine
promise_type ctor
promise_type::get_return_object
ReturnObj ctor
promise_type::initial_suspend
NeverSuspend ctor
NeverSuspend::await_ready
NeverSuspend::await_resume
NeverSuspend dtor
Start of coroutine
promise_type::return_void
promise_type::final_suspend
NeverSuspend ctor
NeverSuspend::await_ready
NeverSuspend::await_resume
NeverSuspend dtor
promise_type dtor
Done
ReturnObj dtor
```

当然，我们首先从“调用协程”开始，但紧接着事情就变得有趣起来。
我们看到`promise_type`类被实例化了。这是使用`operator new`分配的，所以默认情况下会将其放在堆上，但我们可以重载该运算符(Part11)来控制这种行为。

接下来调用 `get_return_object` 来构建返回对象，这将产生对 `ReturnObj` 构造函数的调用。

此时，我们第一次有机会通过调用 `initial_suspend` 来暂停。
我们通过调用 `NeverSuspend` 构造函数作为提供对此初始挂起的控制的一种丰富方式来做出响应。
我们的 `await_ready` 被调用，我们返回 `true` 以表示我们不想暂停，因此我们收到了对 `await_resume` 的调用，然后我们的 `NeverSuspend` 被销毁，因为初始暂停阶段已经结束，因此它的工作完成了。

由于我们没有挂起，因此协程实际上可以开始了！
我们看到 “Start of coroutine” 并立即调用 `co_return;`，后者调用 `return_void`。
由于我们已经终止了协程，因此我们获得了对 `final_suspend` 的调用，然后我们再次创建并返回一个经历相同生命周期的 `NeverSuspend` 对象。

协程现在已经结束，所以我们的 `promise_type` 被销毁，控制权在调用 `SimpleCoroutine` 后立即返回给 `Foo`。
我们打印 “Done”，然后 `ReturnObj` 局部变量超出范围并被销毁。


请注意，“协程结束”从未被记录。
这是因为在我们能够记录之前，我们调用了`co_return；`。这就像在普通函数中调用`return;`一样结束了协程，所以那个语句从未被执行。

## Lazy Evaluation  惰性评估

既然我们已经很好地掌握了协程的生命周期，让我们用它来做些有用的事情。
</br>比如说，我们有一些计算成本高昂的任务，但我们想推迟计算它，甚至可能根本不计算它。我们可以使用协程来实现这一点。
</br>不过，我们需要进行一些更改，从AlwaysSuspend挂起策略开始：

```c++
struct AlwaysSuspend
{
    AlwaysSuspend()
    {
        DebugLog("AlwaysSuspend ctor");
    }
 
    ~AlwaysSuspend()
    {
        DebugLog("AlwaysSuspend dtor");
    }
 
    bool await_ready()
    {
        DebugLog("AlwaysSuspend::await_ready");
        return false;
    }
 
    void await_suspend(std::coroutine_handle<>)
    {
        DebugLog("AlwaysSuspend::await_suspend");
    }
 
    void await_resume()
    {
        DebugLog("AlwaysSuspend::await_resume");
    }
};
```

`AlwaysSuspend` 和 `NeverSuspend` 之间的唯一区别是，我们在 `await_ready` 中返回 `false` 以指示我们应该暂停。

现在让我们看看我们的新 “return object” Lazy：

```c++
class Lazy
{
    struct promise_type;
 
    std::coroutine_handle<promise_type> handle;
    bool haveVal{ false };
 
public:
 
    Lazy(std::coroutine_handle<promise_type> handle)
        : handle{ handle }
    {
        DebugLog("Lazy ctor");
    }
 
    Lazy(const Lazy&) = delete;
 
    Lazy(Lazy&& s)
        : handle(s.handle)
    {
        DebugLog("Lazy move ctor");
        s.handle = nullptr;
    }
 
    ~Lazy()
    {
        DebugLog("Lazy dtor");
        if (handle)
        {
            handle.destroy();
        }
    }
 
    Lazy& operator=(const Lazy&) = delete;
 
    Lazy& operator=(Lazy&& s)
    {
        DebugLog("Lazy move assignment operator");
        handle = s.handle;
        s.handle = nullptr;
        return *this;
    }
 
    int GetValue()
    {
        DebugLog("Lazy::GetValue");
        if (!haveVal)
        {
            handle.resume();
            haveVal = true;
        }
        return handle.promise().value;
    }
```

首先，我们声明`promise_type`，以便在稍后定义它之前，编译器知道它是一个结构体。
</br>这使得我们可以声明一个`std::coroutine_handle<promise_type>`数据成员。这是标准库中的一个类模板，它为我们返回类型（Lazy）提供了对协程`promise_type`的访问。
</br>我们在构造函数中获取handle并将其保存到该数据成员中。我们还存储了是否已经计算了值。

接下来，我们通过删除复制构造函数和复制赋值运算符来禁止复制Lazy。
</br>这是因为`Lazy`通过在其析构函数中调用`handle.destroy()`来“拥有handle。
</br>我们不希望两个`Lazy`的副本被销毁并两次调用`handle.destroy()`。
</br>尽管如此，移动构造和赋值是可行的，所以我们定义了这些函数。

最后，我们提供了一个`GetValue`函数而不是将值公开。这让我们能够控制调用handle.resume()来计算值。
</br>我们使用`handle.promise()`来获取`promise_type`的引用并访问计算出的值数据成员。
</br>我们使用`haveVal`来记住是否已经调用过`handle.resume()`，这样我们就不会两次恢复协程。


现在让我们看一下 `promise` 类型，它仍然是 `Lazy` 的成员：

```c++
struct promise_type
{
    int value{0};
 
    promise_type()
    {
        DebugLog("promise_type ctor");
    }
 
    ~promise_type()
    {
        DebugLog("promise_type dtor");
    }
 
    Lazy get_return_object()
    {
        DebugLog("promise_type::get_return_object");
        return Lazy{
            std::coroutine_handle<promise_type>::from_promise(*this) };
    }
 
    AlwaysSuspend initial_suspend()
    {
        DebugLog("promise_type::initial_suspend");
        return AlwaysSuspend{};
    }
 
    void return_value(int value)
    {
        DebugLog("promise_type::return_value");
        this->value = value;
    }
 
    AlwaysSuspend final_suspend()
    {
        DebugLog("promise_type::final_suspend");
        return AlwaysSuspend{};
    }
 
    void unhandled_exception()
    {
        DebugLog("promise_type unhandled_exception");
    }
};
```

第一个更改是 `value` 现在只是一个数据成员。我们不再使用 `new int` 来分配我们自己的值存储。
</br>`coroutine_handle` 正在为我们处理这件事。
</br>因此，我们必须在 `get_return_object` 中传递给 `std::coroutine_handle<promise_type>::from_promise(*this) Lazy` 构造函数。
</br>这将为 `Promise` 对象创建 `handle`。

第二个变化是我们的 `initial_suspend` 和 `final_suspend` 函数现在返回 `AlwaysSuspend` 对象。
</br>挂起可防止协程函数被执行，直到 `GetValue` 显式恢复该函数

第三个也是最后一个变化是我们现在有一个 `return_value` 而不是 `return_void`。
</br>这允许将 `co_return` 值传递给我们，并且至少在此示例中，我们可以将其存储在 `GetValue` 中以供以后检索。

现在我们可以在协程中使用 `Lazy`：

```c++
Lazy VeryExpensiveCalculation()
{
    DebugLog("Start of coroutine");
    co_return 123;
    DebugLog("End of coroutine");
}
```

在此示例中，这显然不是一个真正昂贵的计算，因为我们只需 `co_return 123;`。

我们采用与以前相同的方式调用它，但现在需要使用 `GetValue`：

```c++
void Foo()
{
    DebugLog("Calling coroutine");
    Lazy ret = VeryExpensiveCalculation();
    DebugLog("Get value first time");
    DebugLog(ret.GetValue());
    DebugLog("Get value second time");
    DebugLog(ret.GetValue());
}
```
运行此命令将输出以下日志消息：
```c++
Calling coroutine
promise_type ctor
promise_type::get_return_object
Lazy ctor
promise_type::initial_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
Get value first time
Lazy::GetValue
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Start of coroutine
promise_type::return_value
promise_type::final_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
123
Get value second time
Lazy::GetValue
123
Lazy dtor
AlwaysSuspend dtor
promise_type dtor
```

开头部分都是相同的，只是我们看到 `AlwaysSuspend` 用于 `initial_suspend` 阶段。
</br>在此之后， 我们不会立即看到 “Start of coroutine”。
</br>在它运行之前，我们会看到 “Get value first time” ，这表示我们可以在调用协程的点和决定获取计算值的值之间注入任意代码。

调用 `GetValue` 后，我们会看到 `handle.resume（）` 调用提示 `await_resume`。
</br>初始挂起现已完成，因此 `AlwaysSuspend` 将被销毁，协程开始执行。

然后，该值被“计算”并提供给 co_return 调用该值 return_value。
</br>之后，我们进入最后的暂停阶段，简单地暂停。控制权将交还给 `GetValue`，`GetValue` 将值返回给 `Foo` 并打印出来。

对 `GetValue` 的第二次调用跳过 `handle.resume（）` 调用，只返回值。此时，我们没有看到任何协程函数被调用。

最后，`Lazy` 对象超出范围并调用 `handle.destroy（）` 来销毁 `promise_type` 和最终挂起的 `AlwaysSuspend` 对象。

## Yielding 生成

协程的下一个常见用法涉及 `co_yield`。这种 “generator” 模式允许我们生成一系列值。
如果我们愿意，这个系列甚至可以是无限的。我们的示例生成器协程如下所示：

```c++
Generator Squares(int count)
{
    DebugLog("Start of coroutine");
    for (int i = 1; i < count+1; ++i)
    {
        int square = i * i;
        DebugLog("Yielding", square, "for", i);
        co_yield square;
        DebugLog("Done yielding", square, "for", i);
    }
    DebugLog("End of coroutine");
}
```

在这种情况下，我们只生成整数的第一个计数平方。我们不使用 `co_return`，而是使用 `co_yield` 来输出单个值，并能够在该运算符之后立即选取函数。

为了支持这一点，我们需要一个名为 `Generator` 的更新的 “return object” 类：

```c++
class Generator
{
    struct promise_type;
 
    std::coroutine_handle<promise_type> handle;
 
public:
 
    Generator(std::coroutine_handle<promise_type> handle)
        : handle{ handle }
    {
        DebugLog("Generator ctor");
    }
 
    Generator(const Generator&) = delete;
 
    Generator(Generator&& s)
        : handle(s.handle)
    {
        DebugLog("Generator move ctor");
        s.handle = nullptr;
    }
 
    ~Generator()
    {
        DebugLog("Generator dtor");
        if (handle)
        {
            handle.destroy();
        }
    }
 
    Generator& operator=(const Generator&) = delete;
 
    Generator& operator=(Generator&& s)
    {
        DebugLog("Generator move assignment operator");
        handle = s.handle;
        s.handle = nullptr;
        return *this;
    }
 
    bool MoveNext()
    {
        DebugLog("Generator::MoveNext");
        handle.resume();
        bool done = handle.done();
        DebugLog("done?", done);
        return !done;
    }
 
    int GetValue()
    {
        DebugLog("Generator::GetValue");
        return handle.promise().value;
    }
 
    struct promise_type
    {
        int value{ 0 };
 
        promise_type()
        {
            DebugLog("promise_type ctor");
        }
 
        ~promise_type()
        {
            DebugLog("promise_type dtor");
        }
 
        Generator get_return_object()
        {
            DebugLog("promise_type::get_return_object");
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this) };
        }
 
        AlwaysSuspend initial_suspend()
        {
            DebugLog("promise_type::initial_suspend");
            return AlwaysSuspend{};
        }
 
        AlwaysSuspend yield_value(int value)
        {
            DebugLog("promise_type::yield_value", value);
            this->value = value;
            return AlwaysSuspend{};
        }
 
        void return_void()
        {
            DebugLog("promise_type::return_void");
        }
 
        AlwaysSuspend final_suspend()
        {
            DebugLog("promise_type::final_suspend");
            return AlwaysSuspend{};
        }
 
        void unhandled_exception()
        {
            DebugLog("promise_type unhandled_exception");
        }
    };
};
```
这里几乎没有什么变化！我们不再有 `haveVal` 数据成员，因为这不适用于生成器。
相反，我们已将一些 `GetValue` 拆分为新的 `MoveNext`。
现在，我们在 `MoveNext` 中调用 `handle.resume（）` 并使用 `handle.done（）` 检查协程是否已完成。`GetValue` 现在只从 `promise` 对象获取值。

唯一的其他变化是我们现在有一个 `yield_value` 而不是 `return_value`。
这是使用给定给 `co_yield` 的值调用的，并返回 `AlwaysSuspend` 以控制协程执行该点的暂停。
我们还有一个 `return_void`，因为没有 `co_return`，因此我们在协程的末尾有一个隐式 `co_return`;。

现在让我们调用这个协程：

```c++
void Foo()
{
    DebugLog("Calling coroutine");
    Generator ret = Squares(3);
    while (ret.MoveNext())
    {
        DebugLog("Get value");
        DebugLog(ret.GetValue());
    }
}
```

运行此命令会得到以下日志：

```c++
Calling coroutine
promise_type ctor
promise_type::get_return_object
Generator ctor
promise_type::initial_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
Generator::MoveNext
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Start of coroutine
Yielding 1 for 1
promise_type::yield_value 1
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
done? false
Get value
Generator::GetValue
1
Generator::MoveNext
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Done yielding 1 for 1
Yielding 4 for 2
promise_type::yield_value 4
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
done? false
Get value
Generator::GetValue
4
Generator::MoveNext
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Done yielding 4 for 2
Yielding 9 for 3
promise_type::yield_value 9
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
done? false
Get value
Generator::GetValue
9
Generator::MoveNext
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Done yielding 9 for 3
End of coroutine
promise_type::return_void
promise_type::final_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
done? true
Generator dtor
AlwaysSuspend dtor
promise_type dtor
```

我们可以看到，当通过 `MoveNext` 调用 `handle.resume（）` 时，协程会反复暂停和恢复。
值将输出到 `yield_value` 函数，并通过 `GetValue` 返回给 `Foo`。

由于像这样的“生成器”表示一系列值，因此通常会将其调整为 C++ 的“迭代器”范例，以便它可以在基于范围的 for 循环中使用。
为此，我们将所需的 `begin` 和 `end` 函数添加到 `Generator` 中，并使它们返回一个满足基于范围的 for 循环的所有要求的对象：

```c++
class Iterator
{
    Generator& owner;
    bool done;
 
public:
 
    Iterator(Generator& o, bool d)
        : owner(o)
        , done(d)
    {
        if (!done)
        {
            MoveNext();
        }
    }
 
    void MoveNext()
    {
        owner.handle.resume();
        done = owner.handle.done();
    }
 
    bool operator!=(const Iterator& other) const
    {
        return done != other.done;
    }
 
    Iterator& operator++()
    {
        MoveNext();
        return *this;
    }
 
    int operator*() const
    {
        return owner.handle.promise().value;
    }
};
 
Iterator begin()
{
    return Iterator{ *this, false };
}
 
Iterator end()
{
    return Iterator{ *this, true };
}
```

现在我们可以像这样使用协程：

```c++
for (int val : Squares(3))
{
    DebugLog(val);
}
```

我们将得到预期的值：

```c++
1
4
9
```

## Asynchronous Coroutines  异步协程

最后一个协程关键字是 `co_await`。它通常用于创建类似于 C# 异步函数的异步协程。
</br>例如，我们可能编写 `file = co_await DownloadUrl("https://test.com/big-file");` 以暂停我们的协程，直到文件下载完成。

现在，让我们把 `Lazy` 返回对象支持设 `co_await`。我们真正需要做的就是向其添加一些成员函数：

```c++
bool await_ready()
{
    DebugLog("Lazy::await_ready");
    const auto done = handle.done();
    DebugLog("Done?", done);
    return handle.done();
}
 
void await_suspend(std::coroutine_handle<> awaitHandle)
{
    DebugLog("Lazy::await_suspend");
    DebugLog("Resuming handle");
    handle.resume();
    DebugLog("Resuming awaitHandle");
    awaitHandle.resume();
}
 
auto await_resume()
{
    DebugLog("Lazy::await_resume");
    return handle.promise().value;
}
```

这些成员函数对应于使用 `co_await` 的协程中的不同执行点。
</br>这些正是我们在 `NeverSuspend` 和 `AlwaysSuspend` 中看到的 `suspended` 函数。事实上，这些类可以与 `co_await` 一起使用！

```c++
Lazy AwaitSuspender()
{
    co_await NeverSuspend{};
}
```

现在我们在 `Lazy` 中拥有了它们，我们可以从其他协程中使用它们。让我们在 `Squares` 协程中将 `VeryExpensiveCalculation` 重新用作计数源：

```c++
Lazy VeryExpensiveCalculation()
{
    DebugLog("Start of coroutine");
    co_return 3;
    DebugLog("End of coroutine");
}
 
Generator Squares()
{
    int count = co_await VeryExpensiveCalculation();
 
    for (int i = 1; i < count + 1; ++i)
    {
        int square = i * i;
        co_yield square;
    }
}
```
我们可以像以前一样运行它：
```c++
void Foo()
{
    DebugLog("Calling coroutine");
    for (int val : Squares())
    {
        DebugLog(val);
    }
}
```

这是我们得到的日志：

```c++
Calling coroutine
promise_type ctor
promise_type::get_return_object
Generator ctor
promise_type::initial_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
AlwaysSuspend::await_resume
AlwaysSuspend dtor
promise_type ctor
promise_type::get_return_object
Lazy ctor
promise_type::initial_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
Lazy::await_ready
Done? false
Lazy::await_suspend
Resuming handle
AlwaysSuspend::await_resume
AlwaysSuspend dtor
Start of coroutine
promise_type::return_value
promise_type::final_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
Resuming awaitHandle
Lazy::await_resume
Lazy dtor
AlwaysSuspend dtor
promise_type dtor
promise_type::yield_value 1
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
1
AlwaysSuspend::await_resume
AlwaysSuspend dtor
promise_type::yield_value 4
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
4
AlwaysSuspend::await_resume
AlwaysSuspend dtor
promise_type::yield_value 9
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
9
AlwaysSuspend::await_resume
AlwaysSuspend dtor
promise_type::return_void
promise_type::final_suspend
AlwaysSuspend ctor
AlwaysSuspend::await_ready
AlwaysSuspend::await_suspend
Generator dtor
AlwaysSuspend dtor
promise_type dtor
```

我们可以在开始时看到对 `Lazy：：await_ready`、`Lazy：：await_suspend` 和 `Lazy：：await_resume` 的调用，因为 `Squares` 使用 `co_await` 来获取计数 。
</br>之后，我们只是看到 Squares 像以前一样继续生成序列。

请注意，`co_await` 并不像在 C# 中那样意味着任何特定的多线程系统。
</br>我们可以像这里一样自由地将所有内容都放在一个线程上，使用线程池，通过 job 系统漏斗协程返回对象，或者我们认为合适的任何其他事情。

## Conclusion  结论

C++ 中的协程与 C# 中的迭代器函数和异步函数的用途非常相似。像往常一样，C++ 对它们的行为提供较低级别的控制。
</br>这种级别的控制至少足以实现这两种 C# 功能，并且随着越来越多的代码库采用 C++20，它可能会用于各种创意目的。

虽然无法保证，但 C++ 编译器通常会积极优化协程。
</br>正如我们今天看到的，这种即时的同步使用可能会避免任何堆分配，并且大多数（如果不是全部）函数调用和存储需求将被简单地优化掉，只留下原始循环或常量。
</br>当无法做到这一点时，可以灵活地使用它们，例如通过长期存储异步协程 return 和 promise 对象。


















