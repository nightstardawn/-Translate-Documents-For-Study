# C++ For C# Developers: Part 10 – Struct Basics

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5644),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

现在我们已经了解了结构体的基础知识 ，让我们向它们添加函数吧！今天，我们将探讨成员函数和重载运算符。

## Member Functions  成员函数

与 C# 一样，C++ 中的结构可以包含函数。这些函数在 C# 中称为“方法”，在 C++ 中称为“成员函数”。
</br>它们的外观和工作方式与 C# 中的基本相同：

```c++
struct Vector2
{
    float X;
    float Y;
 
    float SqrMagnitude()
    {
        return this->X*this->X + this->Y*this->Y;
    }
};
```


成员函数隐式传递一个指针，该指针指向它们所在的结构的实例。
在本例中，它的 type 为 `Vector2*`。除了使用 `this->X` 或 `（*this）` 之外。`X` 而不是 this 的 X 的用法与 C# 中的用法相同。它也是可选的，同样像 C#：

与 C# 不同，但为了与其他 C++ 函数和数据成员初始化保持一致，我们可以拆分函数的声明和定义。如果我们这样做，我们需要将定义放在类之外：

```c++
struct Vector2
{
    float X;
    float Y;
 
    // Declaration
    float SqrMagnitude();
};
 
// Definition
float Vector2::SqrMagnitude()
{
    return X*X + Y*Y;
}
```

请注意，当我们这样做时，我们需要指定我们定义的函数的声明位置。为此，我们在函数名称的开头加上 `Vector2：：` 前缀。

仅在结构定义中声明成员函数声明是很常见的。
</br>该结构定义通常放在头文件 （例如 `Vector2.h`）中，成员函数定义放在转换单元（例如 `Vector2.cpp`）中。
</br>这通过仅编译一次成员函数定义来减少编译时间，同时允许成员函数由任何 `#include` 成员函数声明的头文件调用。

现在我们有一个成员函数，让我们调用它吧！

```c++
Vector2 v;
v.X = 2;
v.Y = 3;
float sqrMag = v.SqrMagnitude();
DebugLog(sqrMag); // 13
```

调用成员函数的工作方式与在 C# 中一样，并且与我们访问数据成员的方式一致。如果我们有一个指针，我们会使用 `->` 而不是 `.`：

```c++
Vector2* p = &v;
float sqrMag = p->SqrMagnitude();
```

我们之前看到的适用于全局函数的所有规则都适用于成员函数。这包括对重载的支持：

```c++
struct Weapon
{
    int32_t Damage;
};
 
struct Potion
{
    int32_t HealAmount;
};
 
struct Player
{
    int32_t Health;
 
    void Use(Weapon weapon, Player& target)
    {
        target.Health -= weapon.Damage;
    }
 
    void Use(Potion potion)
    {
        Health += potion.HealAmount;
    }
};
 
Player player;
player.Health = 50;
 
Player target;
target.Health = 50;
 
Weapon weapon;
weapon.Damage = 10;
 
player.Use(weapon, target);
DebugLog(target.Health); // 40
 
Potion potion;
potion.HealAmount = 20;
 
player.Use(potion);
DebugLog(player.Health); // 70
```

请记住，成员函数采用隐式 `this` 参数。除了所有显式参数之外，我们还需要能够基于该参数进行重载。
</br>重载 `this` 的类型没有意义，但 C++ 确实为我们提供了一种重载方法，具体取决于成员函数是在左值引用还是右值引用上调用的：

```c++
struct Test
{
    // 仅允许在左值对象上调用此操作
    void Log() &
    {
        DebugLog("lvalue-only");
    }
 
    // 仅允许在右值对象上调用此操作
    void Log() &&
    {
        DebugLog("rvalue-only");
    }
 
    // 允许在左值或右值对象上调用此操作
    // 注意：如果上述任一存在则不允许
    void Log()
    {
        DebugLog("lvalue or rvalue");
    }
};
 
// 假装“左值或右值”版本没有被定义...
 
// 因为 'test' 有一个名字，所以它是一个左值
Test test;
test.Log(); // lvalue-only
 
// 因为'Test()'没有名字，所以它是一个右值
Test().Log(); // rvalue-only
```

我们很快就会更深入地介绍结构体的初始化，但目前 `Test（）` 是一种创建 `Test` 结构体实例的方法。

最后，成员函数可以是静态的，具有与 C# 相似的语法和含义：

```c++
struct Player
{
    int32_t Health;
 
    static int32_t ComputeNewHealth(int32_t oldHealth, int32_t damage)
    {
        return damage >= oldHealth ? 0 : oldHealth - damage;
    }
};
```

为了调用它，我们使用 `struct` 类型而不是该类型的实例来引用成员函数。这与在 C# 中一样，只是我们使用 `：：` 而不是 `.`，这在 C++ 中引用类型的内容是正常的：

```c++
DebugLog(Player::ComputeNewHealth(100, 15)); // 85
DebugLog(Player::ComputeNewHealth(10, 15)); // 0
```

由于 `static` 成员函数不对特定 `struct` 对象进行作，因此它们没有隐式 `this` 参数。这使它们与常规函数指针兼容：

```c++
// 获取静态成员函数的函数指针
int32_t (*cnh)(int32_t, int32_t) = Player::ComputeNewHealth;
 
// Call it
DebugLog(cnh(100, 15)); // 85
DebugLog(cnh(10, 15)); // 0
```

## Overloaded Operators  重载运算符

C# 和 C++ 都允许大量运算符重载，但也存在相当多的差异。让我们从一些基本的东西开始：

```c++
struct Vector2
{
    float X;
    float Y;
 
    Vector2 operator+(Vector2 other)
    {
        Vector2 result;
        result.X = X + other.X;
        result.Y = Y + other.Y;
        return result;
    }
};
 
Vector2 a;
a.X = 2;
a.Y = 3;
 
Vector2 b;
b.X = 10;
b.Y = 20;
 
Vector2 c = a + b;
DebugLog(a.X, a.Y); // 2, 3
DebugLog(b.X, b.Y); // 10, 20
DebugLog(c.X, c.Y); // 12, 23
```

在这里，我们看到一个重载的二进制 `+` 运算符。它看起来就像一个成员函数，只是它的名称是 `operator+` 而不是像 `Use` 这样的标识符。
这与 C# 不同，在 C# 中，重载的运算符是静态的，因此需要采用两个参数。如果需要 C# 样式的静态方法，则可以在结构外部声明重载运算符：

```c++
struct Vector2
{
    float X;
    float Y;
};
 
Vector2 operator+(Vector2 a, Vector2 b)
{
    Vector2 result;
    result.X = a.X + b.X;
    result.Y = a.Y + b.Y;
    return result;
}
 
// 用法相同
```

另一个区别是，当重载运算符在结构中定义时，可以使用 `operator+` 代替成员函数名称来直接调用重载运算符：

```c++
Vector2 d = a.operator+(b);
DebugLog(d.X, d.Y);
```

下表比较了哪些运算符在两种语言中可能重载：

| Operator     | C++                                   | C#                                  |
|--------------|---------------------------------------|-------------------------------------|
| `+x`         | Yes                                   | Yes                                 |
| `-x`         | Yes                                   | Yes                                 |
| `!x`         | Yes                                   | Yes                                 |
| `~x`         | Yes                                   | Yes                                 |
| `x++`        | Yes                                   | Yes, but same for`x++`and`++x`      |
| `x--`        | Yes                                   | Yes, but same for`x--`and`--x`      |
| `++x`        | Yes                                   | Yes, but same for`x++`and`++x`      |
| `--x`        | Yes                                   | Yes, but same for`x--`and`--x`      |
| `true`       | N/A                                   | Yes                                 |
| `false`      | N/A                                   | Yes                                 |
| `x + y`      | Yes                                   | Yes                                 |
| `x - y`      | Yes                                   | Yes                                 |
| `x * y`      | Yes                                   | Yes                                 |
| `x / y`      | Yes                                   | Yes                                 |
| `x % y`      | Yes                                   | Yes                                 |
| `x ^ y`      | Yes                                   | Yes                                 |
| `x && y`     | Yes                                   | Yes                                 |
| `x \| y`     | Yes                                   | Yes                                 |
| `x = y`      | Yes                                   | No                                  |
| `x < y`      | Yes                                   | Yes, requires`>`too                 |
| `x > y`      | Yes                                   | Yes, requires`<`too                 |
| `x += y`     | Yes                                   | No, implicitly uses`+`              |
| `x -= y`     | Yes                                   | No, implicitly uses`-`              |
| `x *= y`     | Yes                                   | No, implicitly uses`*`              |
| `x /= y`     | Yes                                   | No, implicitly uses`/`              |
| `x %= y`     | Yes                                   | No, implicitly uses`%`              |
| `x ^= y`     | Yes                                   | No, implicitly uses`^`              |
| `x &= y`     | Yes                                   | No, implicitly uses`&`              |
| `x\|= y`     | Yes                                   | No, implicitly uses`                |`|
| `x << y`     | Yes                                   | Yes                                 |
| `x >> y`     | Yes                                   | Yes                                 |
| `x >>= y`    | Yes                                   | No, implicitly uses`>>`             |
| `x <<= y`    | Yes                                   | No, implicitly uses`<<`             |
| `x == y`     | Yes                                   | Yes, requires`!=`too                |
| `x != y`     | Yes                                   | Yes, requires`==`too                |
| `x <= y`     | Yes                                   | Yes, requires`>=`too                |
| `x >= y`     | Yes                                   | Yes, requires`<=`too                |
| `x <=> y`    | Yes                                   | N/A                                 |
| `x && y`     | Yes, without short-circuiting         | No, implicitly uses`true`and`false` |
| `x \|\| y`   | Yes, without short-circuiting         | No, implicitly uses`true`and`false` |
| `x, y`       | Yes, without left-to-right sequencing | No                                  |
| `x->y`       | Yes                                   | No                                  |
| `x(x)`       | Yes                                   | No                                  |
| `x[i]`       | Yes                                   | No, indexers instead                |
| `x?.[i]`     | N/A                                   | No, indexers instead                |
| `x.y`        | No                                    | No                                  |
| `x?.y`       | N/A                                   | No                                  |
| `x::y`       | No                                    | No                                  |
| `x ? y : z`  | No                                    | No                                  |
| `x ?? y`     | N/A                                   | No                                  |
| `x ??= y`    | N/A                                   | No                                  |
| `x..y`       | N/A                                   | No                                  |
| `=>`         | N/A                                   | No                                  |
| `as`         | N/A                                   | No                                  |
| `await`      | N/A                                   | No                                  |
| `checked`    | N/A                                   | No                                  |
| `unchecked`  | N/A                                   | No                                  |
| `default`    | N/A                                   | No                                  |
| `delegate`   | N/A                                   | No                                  |
| `is`         | N/A                                   | No                                  |
| `nameof`     | N/A                                   | No                                  |
| `new`        | Yes                                   | No                                  |
| `sizeof`     | No                                    | No                                  |
| `stackalloc` | N/A                                   | No                                  |
| `typeof`     | N/A                                   | No                                  |

与C#一样，C++语言对重载运算符的参数、返回值和功能几乎没有限制。 相反，这两种语言都依赖于约定。因此，像这样实现一个重载运算符是合法的，但非常奇怪：

```c++
struct Vector2
{
    float X;
    float Y;
 
    int32_t operator++()
    {
        return 123;
    }
};
 
Vector2 a;
a.X = 2;
a.Y = 3;
int32_t res = ++a;
DebugLog(res); // 123
```

上表中一个特别有趣的运算符是 C++20 中引入的` x <=> y`。这称为“三方比较”或“宇宙飞船”运算符。这通常可以使用，而无需 `operator` 重载，如下所示：

```c++
auto res = 1 <=> 2;
 
if (res < 0)
{
    DebugLog("1 < 2"); // 这个被调用
}
else if (res == 0)
{
    DebugLog("1 == 2");
}
else if (res > 0)
{
    DebugLog("1 > 2");
}
```

这与大多数排序比较器类似，其中返回负值表示第一个参数小于第二个参数，返回正值表示更大，返回零表示相等。返回的确切类型没有指定，只是它需要支持这三种比较。

虽然它可以像这样直接使用，但它对于运算符重载特别有价值，因为它意味着所有其他比较运算符的规范实现：`==`、`!=`、`<`、`<=`、`>` 和 `>=`。这允许我们编写直接或间接使用三向比较运算符的代码：

```c++
struct Vector2
{
    float X;
    float Y;
 
    float SqrMagnitude()
    {
        return this->X*this->X + this->Y*this->Y;
    }
 
    float operator<=>(Vector2 other)
    {
        return SqrMagnitude() - other.SqrMagnitude();
    }
};
 
int main()
{
    Vector2 a;
    a.X = 2;
    a.Y = 3;
 
    Vector2 b;
    b.X = 10;
    b.Y = 20;
 
    // 直接使用 <==>
    float res = a <=> b;
    if (res < 0)
    {
        DebugLog("a < b");
    }
 
    // 间接使用 <=>
    if (a < b)
    {
        DebugLog("a < b");
    }
}
```

## Conclusion  结论

今天，我们了解了 C++ 的方法版本，称为成员函数和重载运算符。
</br>成员函数与其 C# 对应函数非常相似，但确实存在差异，例如可选的声明定义拆分、基于左值和右值对象的重载以及转换为函数指针。

重载运算符与 C# 也有其相似之处和不同之处。在 C++ 中，它们可以放置在结构内部并像非静态成员函数一样使用，也可以放置在结构外部并像静态成员函数一样使用。
当在 `struct` 中时，可以像使用 `x.operator+（10）` 一样显式调用它们。相当多的运算符可能会过载，并且通常具有更精细的控制。最后，三向比较 （“spaceship”） 运算符允许在重载比较时删除大量样板。




















