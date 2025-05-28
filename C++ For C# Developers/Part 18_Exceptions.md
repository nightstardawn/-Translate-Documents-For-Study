# C++ For C# Developers: Part 18 – Exceptions

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5793),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

与C#一样，C++也使用异常作为其主要错误处理机制之一。今天我们将全面了解它们：抛出、捕获、对析构函数的影响、未捕获时会发生什么，以及更多内容。

## Throwing Exceptions  引发异常

引发异常的语法在 C++ 中看起来与在 C# 中几乎相同：
```c++
throw e;
```
这两种语言之间的主要区别在于，C#要求异常对象必须是继承自System.Exception的类实例。
</br>C++标准库中有一个std::exception类，但我们有权忽略它，并抛出任何类型的对象：类实例、枚举、原始数据类型、指针等。

```c++
class IOException {};
enum class ErrorCode { FileNotFound };
 
void Foo()
{
    // Class instance
    throw IOException{};
 
    // Enum
    throw ErrorCode::FileNotFound;
 
    // Primitive
    throw 1;
}
```

请注意，这里的IOException类实例不是一个指向类实例的指针或引用。这是C#的要求，因为所有类实例变量都是管理引用。
</br>在这里，我们抛出对象本身，但如果我们想的话，也可以抛出指针：

```c++
IOException ex;
 
void Foo()
{
    // Pointer to a class instance
    throw &ex;
}
```

通常我们会看到单独的 `throw` 语句，但就像在 C# 7.0 中一样，它实际上是一个表达式，可以是更复杂语句的一部分。
</br>这里就是它与三元/条件运算符常见用法相结合的例子：

```c++
class InvalidId{};
const int32_t MAX_PLAYERS = 4;
int32_t highScores[MAX_PLAYERS]{};
 
int32_t GetHighScore(int32_t playerId)
{
    return playerId < 0 || playerId >= MAX_PLAYERS ?
        throw InvalidId{} :
        highScores[playerId];
}
```

## Catching Exceptions  捕获异常

使用 try 和 catch 块捕获异常，就像在 C# 中一样：

```c++
void Foo()
{
    const int32_t id = 4;
    try
    {
        GetHighScore(id);
    }
    catch (InvalidId)
    {
        DebugLog("Invalid ID", id);
    }
}
```

在这个例子中，我们没有给捕获的异常对象命名，因为仅仅抛出异常就足以让catch块中的代码处理。
</br>在C#中，这样做也是允许的，同样也可以给捕获的异常命名：

```c++
struct InvalidId
{
    int32_t Id;
};
const int32_t MAX_PLAYERS = 4;
int32_t highScores[MAX_PLAYERS]{};
 
int32_t GetHighScore(int32_t playerId)
{
    return playerId < 0 || playerId >= MAX_PLAYERS ?
        throw InvalidId{playerId} :
        highScores[playerId];
}
 
void Foo()
{
    try
    {
        GetHighScore(4);
    }
    catch (InvalidId ex)
    {
        DebugLog("Invalid ID", ex.Id);
    }
}
```

在这个版本中，InvalidId 包含无效的 ID，因此我们给捕获块的异常对象命名以便访问它。

```c++
struct InvalidId
{
    int32_t Id;
};
struct NoHighScore
{
    int32_t PlayerId;
};
const int32_t MAX_PLAYERS = 4;
int32_t highScores[MAX_PLAYERS]{-1, -1, -1, -1};
 
int32_t GetHighScore(int32_t playerId)
{
    if (playerId < 0 || playerId >= MAX_PLAYERS)
    {
        throw InvalidId{playerId};
    }
    const int32_t highScore = highScores[playerId];
    return highScore < 0 ? throw NoHighScore{playerId} : highScore;
}
 
void Foo()
{
    try
    {
        GetHighScore(2);
    }
    catch (InvalidId ex)
    {
        DebugLog("Invalid ID", ex.Id);
    }
    catch (NoHighScore ex)
    {
        DebugLog("No high score for player with ID", ex.PlayerId);
    }
}
```

捕获块按照它们列出的顺序进行检查，并且首先匹配的类型对应的捕获块将被执行。

在 C++ 中捕获所有类型的异常看起来有点不同。我们使用 `catch （...） {} `而不是 `catch {}`：

```c++
void Foo()
{
    try
    {
        GetHighScore(2);
    }
    catch (...)
    {
        DebugLog("Couldn't get high score");
    }
}
```

就像在C#中，无论异常是否有名称，我们都可以使用throw;来重新抛出捕获的异常：

```c++
void Foo()
{
    try
    {
        GetHighScore(2);
    }
    catch (...)
    {
        throw;
    }
}
```

C# 具有异常筛选器功能：
```csharp
catch (Exception e) when (e.Message == "kaboom")
{
    Console.WriteLine("kaboom!");
}
catch (Exception e) when (e.Message == "boom")
{
    Console.WriteLine("boom!");
}
```

这在 C++ 中不可用，但我们可以使用常规代码（如 switch）进行近似处理。请记住，C# 异常筛选器在堆栈展开之前进行评估，此近似值在之后进行评估：

```c++
enum class IOError { FileNotFound, PermissionDenied };
 
void Foo()
{
    try
    {
        DeleteFile("/path/to/file");
    }
    catch (IOError err)
    {
        switch (err)
        {
            case IOError::FileNotFound:
                DebugLog("file not found");
                break;
            case IOError::PermissionDenied:
                DebugLog("permission denied");
                break;
            default:
                throw;
        }
    }
}
```

最后，C++ 具有另一种形式的 try-catch 块，它们放置在函数级别：

```c++
void Foo() try
{
    GetHighScore(2);
}
catch (...)
{
    DebugLog("Couldn't get high score");
}
```

这些与包含整个函数的try语句类似。
</br>使用它的主要原因是能够在构造函数的初始化列表中捕获异常。因为这些代码不会出现在函数体中，所以没有其他方法可以编写包含它们的try块。

在函数级别的捕获块被调用时，所有已构造的数据成员已经被销毁。在捕获块的末尾，异常会自动通过隐式抛出重新抛出；类似于在void函数末尾的隐式返回；

```c++
struct HighScore
{
    int32_t Value;
 
    HighScore(int32_t playerId) try
        : Value(GetHighScore(playerId))
    {
    }
    catch (...)
    {
        DebugLog("Couldn't get high score");
    }
};
 
void Foo()
{
    try
    {
        HighScore hs{2};
    }
    catch (NoHighScore ex)
    {
        DebugLog("No high score for player", ex.PlayerId);
    }
}
 
// 打印结果:
// * Couldn't get high score
// * No high score for player 2
```

参数，而不是局部变量，可以在函数级别的捕获块中使用，并且它们甚至可以返回：

```c++
int32_t GetHighScoreOrDefault(int32_t playerId, int32_t defaultVal) try
{
    return GetHighScore(playerId);
}
catch (...)
{
    DebugLog(
        "Couldn't get high score for", playerId,
        ". Returning default value", defaultVal);
    return defaultVal;
}
 
void Foo()
{
    DebugLog(GetHighScoreOrDefault(2, -1));
}
 
// 打印结果:
// * Couldn't get high score for 2. Returning default value -1
// * -1
```

## Exception Specifications 异常规格

C++函数分为非抛出异常和可能抛出异常两种。默认情况下，所有函数都是可能抛出异常的，除非是析构函数和没有调用可能抛出异常的函数的编译器生成的函数。

```c++
// 常规函数可能抛出异常
void Foo() {}
 
struct MyStruct
{
    // 编译器生成的构造函数不会抛出异常
    //MyStruct()
    //{
    //}
 
    // 析构函数是非抛出异常的
    ~MyStruct()
    {
    }
};
```

这些信息被编译器用来生成更优化的代码，并在我们意外地在非抛出函数中抛出异常时启用编译时检查。

我们可以通过两种方式覆盖默认分类。首先，在函数的参数列表后添加 `noexcept`，就像我们放置 `override` 或 `const` 的地方一样：

```c++
void Foo() noexcept // 强制非抛出异常
{
    throw 1; // 编译器警告：在非抛出函数中抛出异常
}
```

我们可以通过在其后添加一个编译时表达式来使`noexcept`条件化：

```c++
void Foo() noexcept(FOO_THROWS == 1)
{
    throw 1;
}
```

可以使用诸如`-DFOO_THROWS=1`之类的编译器选项来设置`FOO_THROWS`，从而在不更改代码的情况下更改函数的抛出分类。

我们也可以用同样的方式将 noexcept 添加到函数指针中：

```c++
void Foo() noexcept
{
    throw 1;
}
 
void Goo() noexcept(FOO_THROWS == 1)
{
    throw 1;
}
 
void (*pFoo)() noexcept = Foo;
void (*pGoo)() noexcept(FOO_THROWS == 1) = Goo;
```

第二种更改默认值的方式在C++11中被弃用，并在C++17和C++20中完全删除。它曾经用来指定函数可以抛出的异常类型，或者函数根本不会抛出任何异常：

```c++
// 强制非抛出异常
// 在C++11中已弃用，在C++20中已移除
void Foo() throw()
{
    throw 1; // 编译器警告：此函数不会抛出异常
}
 
// 可以抛出一个整数或浮点数
// 在C++11中已弃用，在C++17中被移除
void Goo(int a) throw(int, float)
{
    if (a == 1)
    {
        throw 123; // Throw an int
    }
    else if (a == 2)
    {
        throw 3.14f; // Throw a float
    }
}
```

## Stack Unwinding  堆栈展开

就像我们在C#中抛出异常一样，C++中抛出的异常会回溯调用栈，寻找可以处理该异常的捕获块。
</br>这会在C#中触发finally块，但C++没有finally块。相反，局部变量的析构函数会被调用，而不需要任何像finally这样的显式语法：

```c++
struct File
{
    FILE* handle;
 
    File(const char* path)
    {
        handle = fopen(path, "r");
    }
 
    ~File()
    {
        fclose(handle);
    }
 
    void Write(int32_t val)
    {
        fwrite(&val, sizeof(val), 1, handle);
    }
};
 
void Foo()
{
    File file{"/path/to/file"};
    int32_t highScore = GetHighScore(123);
    file.Write(highScore);
}
```

在这个例子中，如果`GetHighScore`抛出异常，`file`的析构函数将被调用，文件句柄将被关闭并归还给操作系统。
</br>如果`GetHighScore`没有抛出异常，`file`的生命周期将在函数结束时结束，其析构函数将被调用。
</br>在两种情况下，都防止了资源泄漏，不需要编写`try`或`finally`块。

当调用栈回溯以寻找合适的捕获块时，我们可能到达调用栈的根函数，但仍未捕获到异常。
</br>在这种情况下，调用C++标准库函数`std::terminate`。这会调用`std::terminate_handler`函数指针。
</br>它默认指向一个调用`std::abort`的函数，这实际上会导致程序崩溃。
</br>我们可以设置自己的`std::terminate_handle`r函数指针，通常在调用`std::abort`之前执行某种类型的崩溃报告：

```c++
void SaveCrashReport()
{
    // ...
}
 
void OnTerminate()
{
    SaveCrashReport();
    std::abort();
}
 
std::set_terminate(OnTerminate);
 
// ... anywhere else in the program ...
 
throw 123; // calls OnTerminate if not caught
```

`std：：terminate` 也会在许多其他情况下被调用。其中一种情况是在堆栈展开期间调用的析构函数本身引发异常：

```c++
struct Boom
{
    ~Boom() noexcept(false) // 强制可能抛出异常的
    {
        DebugLog("boom!");
 
        // 如果在栈回溯期间被调用，这将调用std::terminate，否则它就像正常一样抛出异常
        throw 123;
    }
};
 
void Foo()
{
    try
    {
        Boom boom{};
        throw 456; // 调用 boom 的析构函数
    }
    catch (...)
    {
        DebugLog("never printed");
    }
}
```

另一种调用 `std::terminate` 的方式是如果非抛出函数抛出异常：

```c++
struct Boom
{
    ~Boom() // Non-throwing
    {
        throw 123; // 编译器警告：在非抛出函数中抛出异常
    }
};
 
void Foo()
{
    try
    {
        Boom boom{};
    }
    catch (...)
    {
        DebugLog("never printed");
    }
}
```

或者如果静态变量的构造函数抛出异常：

```c++
struct Boom
{
    Boom()
    {
        throw 123;
    }
};
 
static Boom boom{};
```

请注意，在静态局部变量的构造函数中抛出异常并不会调用`std::terminate`。相反，构造函数将在下次函数调用时再次被调用：

```c++
struct Boom
{
    Boom()
    {
        throw 123;
    }
};
 
void Goo()
{
    static Boom boom{}; // 静态局部变量其构造函数抛出异常
}
 
void Foo()
{
    for (int i = 0; i < 3; ++i)
    {
        try
        {
            Goo();
        }
        catch (...)
        {
            DebugLog("caught"); // Prints three times
        }
    }
}
```

## Slicing  切片

一个常见的错误是捕获继承层次结构中的类实例。我们通常希望捕获基类（IOError），以隐式捕获所有派生类（FileNotFound、PermissionDenied）。
</br>这将导致“切片”掉派生类的基类子对象。由于子对象实际上是为作为派生类对象的一部分而设计的，这可能会导致错误。

为了看到这一点是如何发生的，请考虑以下情况，其中没有尊重虚函数：

```c++
struct Exception
{
    const char* Message;
 
    virtual void Print()
    {
        DebugLog(Message);
    }
};
 
struct IOException : Exception
{
    const char* Path;
 
    IOException(const char* path, const char* message)
    {
        Path = path;
        Message = message;
    }
 
    virtual void Print() override
    {
        DebugLog(Message, Path);
    }
};
 
FILE* OpenLogFile(const char* path)
{
    FILE* handle = fopen(path, "r");
    return handle == nullptr ? throw IOException{path, "Can't open"} : handle;
}
 
void Foo()
{
    try
    {
        FILE* handle = OpenLogFile("/path/to/log/file");
        // ... use handle
    }
    // 捕获基类将其从整个IOException中分离出来
    catch (Exception ex)
    {
        // 调用Exception的Print，而不是IOException的Print
        ex.Print();
    }
}
```

要解决此问题，只需按引用捕获：

```c++
catch (Exception& ex)
```

通常使用const引用会更好，因为很少需要修改异常对象：

```c++
catch (const Exception& ex)
```

无论哪种方式，现在都会调用相应的 virtual function，我们将得到正确的错误消息：

```c++
Can't open /path/to/log/file
```

## Conclusion  结论

C++中的异常与C#中的异常在广义上相似。这两种语言都可以抛出类实例并捕获一个、多个或任意类型的异常。
</br>C++缺少捕获过滤器，但可以用像switch语句这样的普通代码来模拟。
</br>它没有finally，因为不需要记住添加try或finally块，而是使用析构函数来代替。


C++还获得了抛出对象的能力，而不仅仅是引用。
</br>这些对象不必是类实例，原始数据类型、枚举和指针也是允许的。
</br>我们还可以通过使用noexcept规范来获得编译器安全网和更好的优化。
</br>当异常未被捕获时，我们可以挂钩到std::terminate_handler以添加崩溃报告或在进行程序退出之前采取任何其他行动。

这就是异常的全部内容！随着我们继续系列，我们将看到它们如何被整合到其他语言特性中，例如动态分配和运行时类型转换。请保持关注！
















