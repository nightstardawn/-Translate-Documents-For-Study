# C++ For C# Developers: Part 14 – Inheritance

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5711),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

现在我们知道了如何在C++中初始化结构和其它类型，我们可以看看继承，并学习如何使结构相互继承。与C#类的继承相比，这里有很多扩展功能。继续阅读，了解基础知识以及多重继承和虚继承等高级特性！

## Base Structs  基本结构体

> 简单来说就是父类，或是基类

继承的基本语法（也称为派生）在 C++ 中对于结构看起来与在 C# 中对于类的语法相同。我们在结构体名称后添加 `：BaseType`：

```c++
struct GameEntity
{
    static const int32_t MaxHealth = 100;
    int32_t Health = MaxHealth;
 
    float GetHealthPercent()
    {
        return ((float)Health) / MaxHealth;
    }
};
 
struct MovableGameEntity : GameEntity
{
    float Speed = 0;
};
```

在声明而不是定义从另一个结构体继承的结构体时，我们省略基本结构体的名称：

```c++
struct MovableGameEntity; // No base struct name
 
struct MovableGameEntity : GameEntity
{
    float Speed = 0;
};
```

此继承在 C++ 中的含义与 C# 中的含义相同：`MovableGameEntity` 继承自 `GameEntity`。这意味着 `GameEntity` 的所有数据成员和成员函数都作为子对象成为 `MovableGameEntity` 的一部分。我们可以编写代码来使用 parent 和 child 结构体的内容：

```c++
MovableGameEntity mge{};
mge.Health = 50;
DebugLog(mge.Health, mge.Speed, mge.GetHealthPercent()); // 50, 0, 0.5
```

通常，任何对象的大小至少为1字节。这个规则的例外之一是当一个基结构体没有非静态数据成员时。在这种情况下，它可能不会为从它派生的结构体的大小增加任何字节。这个例外的例外是，如果第一个非静态数据成员也是基结构体类型。

```c++
// 没有非静态数据成员
struct Empty
{
    void SayHello()
    {
        DebugLog("hello");
    }
};
 
// 从Empty派生没有增加大小
struct Vector2 : Empty
{
    float X;
    float Y;
};
 
// 大小增加，因为第一个非静态数据成员也是一个Empty
struct ExceptionToException : Empty
{
    Empty E;
    int32_t X;
};
 
void Foo()
{
    Vector2 vec{};
    DebugLog(sizeof(vec)); // 8 (尺寸没有增加)
    vec.SayHello(); // "hello"
 
    DebugLog(sizeof(ExceptionToException)); // 8 (大小从4增加！)
}
```

继承关系在指针和引用之间也存在：

```c++
MovableGameEntity mge{};
GameEntity* pge = &mge; // 指针是兼容的
GameEntity& rge = mge; // 引用是兼容的
DebugLog(mge.Health, pge->Health, rge.Health); // 100, 100, 100
```

派生结构可以有与它们的基结构相同名称的成员。例如，`ArmoredGameEntity`可能从其装甲中获得额外的生命值：

```c++
struct ArmoredGameEntity : GameEntity
{
    static const int32_t MaxHealth = 50;
    int32_t Health = MaxHealth;
};
```


这引入了在引用 `ArmoredGameEntity` 的 `Health` 成员时的歧义：它是 `GameEntity` 中声明的数据成员还是 `ArmoredGameEntity` 中的？编译器会选择被引用的类型中的成员：

```c++
// 在ArmoredGameEntity中引用，因此使用ArmoredGameEntity中的Health
ArmoredGameEntity age{};
DebugLog(age.Health); // 50
 
// 在GameEntity中引用，因此使用GameEntity中的Health
GameEntity& ge = age;
DebugLog(ge->Health); // 100
```


为了解决这种歧义，我们可以使用作用域解析运算符来明确引用继承层次结构中特定子对象的成员：`StructType::Member`。

```c++
struct ArmoredGameEntity : GameEntity
{
    static const int32_t MaxHealth = 50;
    int32_t Health = MaxHealth;
 
    void Die()
    {
        Health = 0; // Health in ArmoredGameEntity
        GameEntity::Health = 0; // Health in GameEntity
    }
};
 
ArmoredGameEntity age{};
GameEntity& ge = age;
DebugLog(age.Health, ge.Health); // 50, 100
age.Die();
DebugLog(age.Health, ge.Health); // 0, 0
```
虽然这样做并不常见，但我们也可以从 struct 层次结构之外引用特定的 structs：

```c++
ArmoredGameEntity age{};
ArmoredGameEntity* page = &age;
 
// 使用 page->GameEntity::Health 引用 GameEntity 中的 Health
DebugLog(age.Health, age.GameEntity::Health); // 50, 100
 
age.Die();
 
// 通过指针使用 page->GameEntity::Health 引用 GameEntity 中的 Health
DebugLog(age.Health, page->GameEntity::Health); // 0, 0
```

这有点像在C#中使用基类，只不过我们可以引用层次结构中的任何结构体，而不仅仅是直接基类类型。

```c++
struct MagicArmoredGameEntity : ArmoredGameEntity
{
    static const int32_t MaxHealth = 20;
    int32_t Health = MaxHealth;
};
 
MagicArmoredGameEntity mage{};
DebugLog(mage.Health); // 20
DebugLog(mage.ArmoredGameEntity::Health); // 50
DebugLog(mage.GameEntity::Health); // 100
```

## Constructors and Destructors 构造函数和析构函数

> 之前原作者提供了一个有关 


与在 C# 中一样，构造函数是从层次结构的顶部到底部调用的。与 C# 的非确定性析构函数/终结器不同，C++ 析构函数的调用顺序与构造函数相同。
</br>回想一下，数据成员是按声明顺序构造的，而析构则按相反的顺序构造。这两个属性组合在一起，给出以下确定性顺序：

```c++
struct LogLifecycle
{
    const char* str;
 
    LogLifecycle(const char* str)
        : str(str)
    {
        DebugLog(str, "constructor");
    }
 
    ~LogLifecycle()
    {
        DebugLog(str, "destructor");
    }
};
 
struct GameEntity
{
    LogLifecycle a{"GameEntity::a"};
    LogLifecycle b{"GameEntity::b"};
 
    GameEntity()
    {
        DebugLog("GameEntity constructor");
    }
 
    ~GameEntity()
    {
        DebugLog("GameEntity destructor");
    }
};
 
struct ArmoredGameEntity : GameEntity
{
    LogLifecycle a{"ArmoredGameEntity::a"};
    LogLifecycle b{"ArmoredGameEntity::b"};
 
    ArmoredGameEntity()
    {
        DebugLog("ArmoredGameEntity constructor");
    }
 
    ~ArmoredGameEntity()
    {
        DebugLog("ArmoredGameEntity destructor");
    }
};
 
void Foo()
{
    ArmoredGameEntity age{};
    DebugLog("--after variable declaration--");
} // Note: destructor of 'age' called here
 
// Logs printed:
//   GameEntity::a, constructor
//   GameEntity::b, constructor
//   GameEntity constructor
//   ArmoredGameEntity::a, constructor
//   ArmoredGameEntity::b, constructor
//   ArmoredGameEntity constructor
//   --after variable declaration--
//   ArmoredGameEntity destructor
//   ArmoredGameEntity::b, destructor
//   ArmoredGameEntity::a, destructor
//   GameEntity destructor
//   GameEntity::b, destructor
//   GameEntity::a, destructor
```

正如我们上面看到的，基结构体的默认构造函数是隐式调用的。然而，如果没有默认构造函数，则必须显式调用它。语法看起来与C#中的语法相似，只是使用基结构体类型而不是关键字`base`：

```c++
struct GameEntity
{
    static const int32_t MaxHealth = 100;
    int32_t Health;
 
    GameEntity(int32_t health)
    {
        Health = health;
    }
};
 
struct ArmoredGameEntity : GameEntity
{
    static const int32_t MaxArmor = 100;
    int32_t Armor = 0;
 
    ArmoredGameEntity()
        : GameEntity(MaxHealth)
    {
    }
};
```

这实际上只是我们用于初始化数据成员的初始值设定项列表的一部分：

```c++
struct ArmoredGameEntity : GameEntity
{
    static const int32_t MaxArmor = 100;
    int32_t Armor = 0;
 
    ArmoredGameEntity()
        : GameEntity(MaxHealth), Armor(MaxArmor)
    {
    }
};
```
由于初始化的顺序始终如上，因此我们将初始化器按什么顺序放置并不重要。对于数据成员和基本结构都是如此：

```c++
struct ArmoredGameEntity : GameEntity
{
    static const int32_t MaxArmor = 100;
    int32_t Armor = 0;
 
    ArmoredGameEntity()
        // 顺序不重要
        // GameEntity 依然在 Armor 之前初始化
        : Armor(MaxArmor), GameEntity(MaxHealth)
    {
    }
};
```

## Multiple Inheritance  多重继承

与 C# 不同，C++ 支持从多个结构派生的结构：

> 也就是：c++一个结构体(类),可以继承多个结构体(类)。c#中只能继承一个结构体(类),可以继承多个结构体(类),但是可以继承多个接口

```c++
struct HasHealth
{
    static const int32_t MaxHealth = 100;
    int32_t Health = MaxHealth;
};
 
struct HasArmor
{
    static const int32_t MaxArmor = 20;
    int32_t Armor = MaxArmor;
};
 
// 玩家同时继承自HasHealth和HasArmor
struct Player : HasHealth, HasArmor
{
    static const int32_t MaxLives = 3;
    int32_t NumLives = MaxLives;
};
```

这意味着`HasHealth`和`HasArmor`是`Player`的子对象，`Player`“是”`HasHealth`，“是”`HasArmor`。我们使用它的方式与使用单继承相同：

```c++
// 两个基结构体的成员都是可访问的
Player p{};
DebugLog(p.Health, p.Armor, p.NumLives); // 100, 20, 3
 
// 可以获取到基类的引用
HasHealth& hh = p;
DebugLog(hh.Health); // 100
 
// 可以获取到基类的指针
HasArmor* ha = &p;
DebugLog(ha->Armor); // 20
```

这就解释了为什么 C++ 中没有 `base` 关键字：可能有多个 base。当显式引用子对象的成员时，我们总是使用它的名称：

```c++
Player p{};
 
// 访问HasHealth子对象的成员
DebugLog(p.HasHealth::Health); // 100
 
// 访问HasArmor子对象的成员
DebugLog(p.HasArmor::Armor); // 20
 
// 访问Player子对象的成员
DebugLog(p.Player::NumLives); // 3
```

现在让我们重新介绍 `LogLifecycle` 实用程序结构，看看多重继承如何影响构造函数和析构函数的顺序：

```c++
struct LogLifecycle
{
    const char* str;
 
    LogLifecycle(const char* str)
        : str(str)
    {
        DebugLog(str, "constructor");
    }
 
    ~LogLifecycle()
    {
        DebugLog(str, "destructor");
    }
};
 
struct HasHealth
{
    LogLifecycle a{"HasHealth::a"};
    LogLifecycle b{"HasHealth::b"};
 
    static const int32_t MaxHealth = 100;
    int32_t Health = MaxHealth;
 
    HasHealth()
    {
        DebugLog("HasHealth constructor");
    }
 
    ~HasHealth()
    {
        DebugLog("HasHealth destructor");
    }
};
 
struct HasArmor
{
    LogLifecycle a{"HasArmor::a"};
    LogLifecycle b{"HasArmor::b"};
 
    static const int32_t MaxArmor = 20;
    int32_t Armor = MaxArmor;
 
    HasArmor()
    {
        DebugLog("HasArmor constructor");
    }
 
    ~HasArmor()
    {
        DebugLog("HasArmor destructor");
    }
};
 
struct Player : HasHealth, HasArmor
{
    LogLifecycle a{"Player::a"};
    LogLifecycle b{"Player::b"};
 
    static const int32_t MaxLives = 3;
    int32_t NumLives = MaxLives;
 
    Player()
    {
        DebugLog("Player constructor");
    }
 
    ~Player()
    {
        DebugLog("Player destructor");
    }
};
 
void Foo()
{
    Player p{};
    DebugLog("--after variable declaration--");
} // Note: destructor of 'p' called here
 
// Logs printed:
//   HasHealth::a, constructor
//   HasHealth::b, constructor
//   HasHealth constructor
//   HasArmor::a, constructor
//   HasArmor::b, constructor
//   HasArmor constructor
//   Player::a, constructor
//   Player::b, constructor
//   Player constructor
//   --after variable declaration--
//   Player destructor
//   Player::b, destructor
//   Player::a, destructor
//   HasArmor destructor
//   HasArmor::b, destructor
//   HasArmor::a, destructor
//   HasHealth destructor
//   HasHealth::b, destructor
//   HasHealth::a, destructor
```

我们在这里看到，基结构体的构造函数是按照它们的派生顺序调用的：在这个例子中是 `HasHealth`，然后是 `HasArmor`。
</br>它们的析构函数按相反的顺序调用。这类似于数据成员，数据成员按声明顺序构造，并以相反的顺序析构。

多重继承可能会引入称为“菱形问题”的歧义，指的是继承层次结构的形状。例如，请考虑以下结构：

```c++
struct Top
{
    const char* Id;
 
    Top(const char* id)
        : Id(id)
    {
    }
};
 
struct Left : Top
{
    const char* Id = "Left";
 
    Left()
        : Top("Top of Left")
    {
    }
};
 
struct Right : Top
{
    const char* Id = "Right";
 
    Right()
        : Top("Top of Right")
    {
    }
};
 
struct Bottom : Left, Right
{
    const char* Id = "Bottom";
};
```

给定一个 `Bottom`，很容易引用它的 `Id` 以及 `Left` 和 `Right` 子对象的 `Id`：

```c++
Bottom b{};
DebugLog(b.Id); // Bottom
DebugLog(b.Left::Id); // Left
DebugLog(b.Right::Id); // Right
```

但是，`Left` 和 `Right` 子对象都有一个 `Top` 子对象。因此，这是模棱两可的，并会导致编译器错误：

```c++
Bottom b{};
 
// 编译器错误：模糊。左或右的top子对象？
DebugLog(b.Top::Id);
```

为了消除歧义，我们需要显式引用 `Left` 或 `Right` 子对象。一种方法是引用这些子对象：

```c++
Bottom b{};
Left& left = b;
DebugLog(left.Top::Id); // Top of Left
Right& right = b;
DebugLog(right.Top::Id); // Top of Right
```

## Virtual Inheritance  虚拟继承

如果我们不希望 Bottom 对象包含两个 Top 子对象，那么我们可以改用“虚拟继承”。
</br>这与普通继承类似，不同之处在于编译器将只为公共基本结构生成一个子对象。我们通过在派生自的结构体的名称前添加关键字 virtual 来启用它：

```c++
struct Top
{
    const char* Id = "Top Default";
};
 
struct Left : virtual Top
{
    const char* Id = "Left";
};
 
struct Right : virtual Top
{
    const char* Id = "Right";
};
 
struct Bottom : virtual Left, virtual Right
{
    const char* Id = "Bottom";
};
 
// 在 Left Right Bottom 中 引用的是同一个 Top 子对象
Bottom b{};
Left& left = b;
Right& right = b;
DebugLog(b.Top::Id); // Top Default
DebugLog(left.Top::Id); // Top Default
DebugLog(right.Top::Id); // Top Default
 
// 改变 Left 的 Top 改变的也是 Right 和 bottom 的Top  
left.Top::Id = "New Top of Left";
DebugLog(b.Top::Id); // New Top of Left
DebugLog(left.Top::Id); // New Top of Left
DebugLog(right.Top::Id); // New Top of Left
 
// 对于 Right 也是相同的
right.Top::Id = "New Top of Right";
DebugLog(b.Top::Id); // New Top of Right
DebugLog(left.Top::Id); // New Top of Right
DebugLog(right.Top::Id); // New Top of Right
```

注意，虚继承和普通继承可以混合使用。在这种情况下，我们为所有共同的虚基类结构得到一个子对象，并为所有非虚基类结构各自得到一个子对象。
例如，这个Bottom有两个Top子对象：一个是通过Left和Right的虚继承得到的，另一个是通过Middle的非虚继承得到的：

```c++
struct Top
{
    const char* Id = "Top Default";
};
 
struct Left : virtual Top
{
    const char* Id = "Left";
};
 
struct Middle : Top
{
    const char* Id = "Middle";
};
 
struct Right : virtual Top
{
    const char* Id = "Right";
};
 
struct Bottom : virtual Left, Middle, virtual Right
{
    const char* Id = "Bottom";
};
 
// 在 Left Right Bottom 中 引用的是同一个 Top 子对象
// 在Middle 的 Top 是另一个子对象
Bottom b{};
Left& left = b;
Middle& middle = b;
Right& right = b;
DebugLog(left.Top::Id); // Top Default
DebugLog(middle.Top::Id); // Top Default
DebugLog(right.Top::Id); // Top Default
 
// 改变 Left 的 Top 改变的是虚拟继承的 Top
left.Top::Id = "New Top of Left";
DebugLog(left.Top::Id); // New Top of Left
DebugLog(middle.Top::Id); // Top Default (note: 没有改变)
DebugLog(right.Top::Id); // New Top of Left
 
// 在Right同理
right.Top::Id = "New Top of Right";
DebugLog(left.Top::Id); // New Top of Right
DebugLog(middle.Top::Id); // Top Default (note: 没有改变)
DebugLog(right.Top::Id); // New Top of Right
 
// 改变 Middle 的 Top 改变的是非虚拟继承的 Top
middle.Top::Id = "New Top of Middle";
DebugLog(left.Top::Id); // New Top of Right (note: 没有改变)
DebugLog(middle.Top::Id); // New Top of Middle
DebugLog(right.Top::Id); // New Top of Right (note: 没有改变)
```

## Virtual Functions  虚拟函数

C++ 中的成员函数可以是虚拟的。这与 C# 中的虚拟方法直接类似。
</br>下面介绍了如何由派生的 `Bow` 结构的 `Attack` 成员函数覆盖基本 `Weapon` 结构的 `Attack` 成员函数：

> 这里的读者原本是想写一下 C# 中virtual相关的内容的，但是作者后面有提到，就不过多赘述了
> 回忆不起来的参考[微软官方文档](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/virtual)

```c++
struct Enemy
{
    int32_t Health = 100;
};
 
struct Weapon
{
    int32_t Damage = 0;
 
    explicit Weapon(int32_t damage)
    {
        Damage = damage;
    }
 
    virtual void Attack(Enemy& enemy)
    {
        enemy.Health -= Damage;
    }
};
 
struct Bow : Weapon
{
    Bow(int32_t damage)
        : Weapon(damage)
    {
    }
 
    virtual void Attack(Enemy& enemy)
    {
        enemy.Health -= Damage;
    }
};
 
Enemy enemy{};
DebugLog(enemy.Health); // 100
 
Weapon weapon{10};
weapon.Attack(enemy);
DebugLog(enemy.Health); // 90
 
Bow bow{20};
bow.Attack(enemy);
DebugLog(enemy.Health); // 70
 
Weapon& weaponRef = bow;
weaponRef.Attack(enemy);
DebugLog(enemy.Health); // 50
 
Weapon* weaponPointer = &bow;
weaponPointer->Attack(enemy);
DebugLog(enemy.Health); // 30
```

请注意，C++中的默认行为是成员函数会覆盖基结构体的虚函数。
</br>这与C#中的默认行为不同，C#中默认会创建一个具有相同名称的新函数，类似于使用了new关键字来声明该方法。

</br>要在 C# 中重写，我们必须使用 `override` 关键字。
</br>在 C++ 中，这是可选的，并且是主要使用的，因此如果我们的覆盖成员函数不再与基本结构中的虚函数匹配，编译器将生成错误作为提醒。
</br>需要注意一点的是 `override` 关键字位于函数签名之后 ，而不是像 C# 中那样位于函数签名之前：

```c++
struct Bow : Weapon
{
    Bow(int32_t damage)
        : Weapon(damage)
    {
    }
 
    virtual void Attack(Enemy& enemy) override
    {
        enemy.Health -= Damage;
    }
 
    // 编译器错误：没有基结构有这个要重写的虚函数
    virtual float SimulateAttack(Enemy& enemy) override
    {
        return enemy.Health - Damage;
    }
};
```

C++ 还可以替代 C# 的 `abstract` 关键字，用于我们根本不想提供成员函数的实现。
这些成员函数称为 “纯虚拟” ，后面有一个 `= 0` 而不是一个 `body`：

```c++
struct Weapon
{
    int32_t Damage = 0;
 
    explicit Weapon(int32_t damage)
    {
        Damage = damage;
    }
 
    virtual void Attack(Enemy& enemy) = 0;
};
```

就像在 C# 中，存在任何抽象方法都意味着我们必须将类本身标记为抽象 ，具有任何纯虚成员函数的 C++ 类都是隐式抽象的，因此无法实例化：

```c++
// 编译错误：武器是一个抽象结构体，不能被实例化
Weapon weapon{10};
```

因为与 C# 不同，`重载运算符`是非静态的，因此它们也可能是虚拟的。在此示例中，`+=` 运算符会升级 `Weapon`：
> c# 中的重载运算符是静态的 一个简单的例子：
>```c++
> public class Point
> {
>      public int X { get; set; }
>      public int Y { get; set; }
>  
>      public Point(int x, int y)
>      {
>          X = x;
>          Y = y;
>      }
>  
>      // 重载 + 运算符
>      public static Point operator +(Point p1, Point p2)
>      {
>          return new Point(p1.X + p2.X, p1.Y + p2.Y);
>      }
> }
>```
```c++
struct Weapon
{
    int32_t Damage = 0;
 
    explicit Weapon(int32_t damage)
    {
        Damage = damage;
    }
 
    virtual void operator+=(int32_t numLevels)
    {
        Damage += numLevels;
    }
};
 
struct Bow : Weapon
{
    int32_t Range;
 
    Bow(int32_t damage, int32_t range)
        : Weapon(damage), Range(range)
    {
    }
 
    virtual void operator+=(int32_t numLevels) override
    {
        // 显式调用基结构重载的运算符
        Weapon::operator+=(numLevels);
 
        Range += numLevels;
    }
};
 
Bow bow{20, 10};
DebugLog(bow.Damage, bow.Range); // 20, 10
 
bow += 5;
DebugLog(bow.Damage, bow.Range); // 25, 15
 
Weapon& weaponRef = bow;
weaponRef += 5;
DebugLog(bow.Damage, bow.Range); // 30, 20
```

此外，与 C# 不同的是，用户定义的`转换运算符`是非静态的，因此它们也可能是虚拟的：
> C# 中转换运算符也是静态的，看一个简单的例子：
> ```c++
> public class Weapon
> {
>     public int Damage { get; protected set; }
> 
>     public Weapon(int damage)
>     {
>         Damage = damage;
>     }
> 
>     // 隐式转换运算符
>     public static implicit operator int(Weapon weapon)
>     {
>         return Damage;
>     }
> }
> 
> Weapon weapon = new Weapon(20);
> int weaponVal = weapon;
> Debug.Log($"{bowVal}"); // 输出: 20
> ```
```c++
struct Weapon
{
    int32_t Damage = 0;
 
    explicit Weapon(int32_t damage)
    {
        Damage = damage;
    }
 
    virtual operator int32_t()
    {
        return Damage;
    }
};
 
struct Bow : Weapon
{
    int32_t Range;
 
    Bow(int32_t damage, int32_t range)
        : Weapon(damage), Range(range)
    {
    }
 
    virtual operator int32_t() override
    {
        // 显式调用基结构体的用户定义转换操作符
        return Weapon::operator int32_t() + Range;
    }
};
 
Bow bow{20, 10};
Weapon& weaponRef = bow;
int32_t bowVal = bow;
int32_t weaponRefVal = weaponRef;
DebugLog(bowVal, weaponRefVal); // 30, 30
```

最后，析构函数可以是虚拟的。这很常见，因为它确保派生的结构析构函数始终被调用：

```c++
struct ReadOnlyFile
{
    FILE* ReadFileHandle;
 
    ReadOnlyFile(const char* path)
    {
        ReadFileHandle = fopen(path, "r");
    }
 
    virtual ~ReadOnlyFile()
    {
        fclose(ReadFileHandle);
        ReadFileHandle = nullptr
    }
};
 
struct FileCopier : ReadOnlyFile
{
    FILE* WriteFileHandle;
 
    FileCopier(const char* path, const char* writeFilePath)
        : ReadOnlyFile(path)
    {
        WriteFileHandle = fopen(writeFilePath, "w");
    }
 
    virtual ~FileCopier()
    {
        fclose(WriteFileHandle);
        WriteFileHandle = nullptr;
    }
};
 
FileCopier copier("/path/to/input/file", "/path/to/output/file");
 
//1.调用派生结构的析构函数 即 这里再调用FileCopier的析构函数
//2.在基结构上调用虚析构函数 即 这里调用ReadOnlyFile的虚析构函数
//如果这不是虚的，则只会调用ReadOnlyFile的析构函数
ReadOnlyFile& rof = copier;
rof.~ReadOnlyFile();
```

当没有其他成员是成为纯虚拟的合适候选者时，纯虚拟析构函数也是将结构标记为抽象的常用方法。这就像在 C# 中标记类抽象而不标记其任何内容抽象 。

## Stopping Inheritance and Overrides 停止继承和覆盖

C# 具有 [sealed](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/sealed) 关键字，该关键字可应用于类以阻止其他类派生该类。
C++ 具有用于此目的的 final 关键字。它位于 struct name 之后和任何 base struct names 之前：

> 简单回顾一下 C# 中 sealed ：
> ```csharp
> // 基类
> public class Animal
> {
>     public virtual void MakeSound()
>     {
>         Console.WriteLine("Animal makes a sound");
>     }
>     // 密封方法：禁止子类重写 Draw
>     public sealed override void Draw()
>     {
>         Console.WriteLine("Drawing a circle");
>     }
> }
> 
> // 密封类：禁止其他类继承 Dog
> public sealed class Dog : Animal
> {
>     public override void MakeSound()
>     {
>         Console.WriteLine("Dog barks: Woof!");
>     }
> }
> ```

```c++
struct Vector1
{
    float X;
};
 
// 如果继承 Vector2 将会显示编译错误
struct Vector2 final : Vector1
{
    float Y;
};
 
// 编译错误 ： Vector2 是 final
struct Vector3 : Vector2
{
    float Z;
};
```

C# 中的 sealed 关键字还可以应用于重写以停止进一步重写的方法和属性。C++ 中的 final 关键字也用于此目的。
</br>与 override 关键字一样，它位于成员函数签名之后。这两个关键字可以一起使用，因为它们具有不同但相关的含义。

```c++
struct Vector1
{
    float X;
 
    // 允许 override
    virtual void DrawPixel(float r, float g, float b)
    {
        GraphicsLibrary::DrawPixel(X, 0, r, g, b);
    }
};
 
struct Vector2 : Vector1
{
    float Y;
 
    // 1.重写 DrawPiexl
    // 2.不允许子类重写 DrawPiexl
    virtual void DrawPixel(float r, float g, float b) override final
    {
        GraphicsLibrary::DrawPixel(X, Y, r, g, b);
    }
};
 
struct Vector3 : Vector2
{
    float Z;
 
    // 编译错误：基类结构（Vector2）中的DrawPixel是final的
    virtual void DrawPixel(float r, float g, float b) override
    {
        GraphicsLibrary::DrawPixel(X/Z, Y/Z, r, g, b);
    }
};
```

## C# Equivalency  C# 等效性

我们已经看到了 C# 概念的 C++ 等效项，例如`抽象类`和`密封类`，以及`抽象` 、 `虚拟` 、 `重写`和`密封`方法。
</br>C# 还有其他几个功能，这些功能没有明确的 C++ 等效项。相反，这些是用通用结构功能惯用地实现的。

首先，C# 具有专用的[接口](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/interface)概念。它就像一个基类，它不能有字段，不能有非抽象方法，是抽象的，并且可以被类多次继承。
</br>在 C++ 中，我们总是有多重继承，因此我们需要做的就是不添加任何数据成员或非抽象成员函数：

```c++
//类似于接口：
//* 没有数据成员
//* 没有非抽象成员函数
//* 是抽象的（由于Log是纯虚函数的缘故）
//* 允许多重继承（在C++中始终启用）
struct ILoggable
{
    virtual void Log() = 0;
};
 
// 要“实现”一个“接口”，只需派生并重写所有成员函数
struct Vector2 : ILoggable
{
    float X;
    float Y;
 
    Vector2(float x, float y)
        : X(x), Y(y)
    {
    }
 
    virtual void Log() override
    {
        DebugLog(X, Y);
    }
};
 
// 使用“接口”，而不是“具体类”
void LogTwice(ILoggable& loggable)
{
    loggable.Log();
    loggable.Log();
}
 
Vector2 vec{2, 4};
LogTwice(vec); // 2, 4 then 2, 4
```

接下来，我们有 C# 中的[分部类](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/partial-type)。这允许我们将一个类的内容拆分为多个文件。我们至少可以通过两种方式在 C++ 中模拟这一点。
</br>首先，也是更常见的是，我们将结构体的定义放在头文件（例如 .h）中，并将其成员定义拆分到多个翻译单元（例如 .cpp）文件中。

```c++
// In Player.h
struct Player
{
    const static int32_t MaxHealth = 100;
    const static int32_t MaxLives = 3;
    int32_t Health = MaxHealth;
    int32_t NumLives = MaxLives;
    float PosX = 0;
    float PosY = 0;
    float DirX = 0;
    float DirY = 0;
    float Speed = 0;
 
    void TakeDamage(int32_t amount);
    void Move(float time);
};
 
// In PlayerCombat.cpp
#include "Player.h"
void Player::TakeDamage(int32_t amount)
{
    Health -= amount;
}
 
// In PlayerMovement.cpp
#include "Player.h"
void Player::Move(float time)
{
    float distance = Speed * time;
    PosX += DirX * distance;
    PosY += DirY * distance;
}
 
// In Game.cpp
#include "Player.h"
Player player;
player.DirX = 1;
player.Speed = 1;
player.TakeDamage(10);
DebugLog(player.Health); // 90
player.Move(5);
DebugLog(player.PosX, player.PosY); // 5, 0
```

另一种不太常见的方法是使用虚拟继承将多个结构组合成一个：

```c++
// In PlayerShared.h
struct PlayerShared
{
    const static int32_t MaxHealth = 100;
    const static int32_t MaxLives = 3;
    int32_t Health = MaxHealth;
    int32_t NumLives = MaxLives;
    float PosX = 0;
    float PosY = 0;
    float DirX = 0;
    float DirY = 0;
    float Speed = 0;
};
 
// In PlayerCombat.h
#include "PlayerShared.h"
struct PlayerCombat : virtual PlayerShared
{
    void TakeDamage(int32_t amount)
    {
        Health -= amount;
    }
};
 
// In PlayerMovement.h
#include "PlayerShared.h"
struct PlayerMovement : virtual PlayerShared
{
    void Move(float time)
    {
        float distance = Speed * time;
        PosX += DirX * distance;
        PosY += DirY * distance;
    }
};
 
// In Player.h
#include "PlayerCombat.h"
#include "PlayerMovement.h"
struct Player : virtual PlayerCombat, virtual PlayerMovement
{
};
 
// In Game.cpp
#include "Player.h"
Player player;
player.DirX = 1;
player.Speed = 1;
player.TakeDamage(10);
DebugLog(player.Health); // 90
player.Move(5);
DebugLog(player.PosX, player.PosY); // 5, 0
```

这种方法允许重新组合各部分（`PlayerCombat` 和 `PlayerMovement`）以形成其他类型，例如可以战斗但不能移动的 `StationaryPlayer`：

```c++
// In StationaryPlayer.h
#include "PlayerCombat.h"
struct StationaryPlayer : virtual PlayerCombat
{
};
 
// In Game.cpp
#include "StationaryPlayer.h"
StationaryPlayer stationary;
stationary.TakeDamage(10);
stationary.Move(5); // 编译器错误：Move 不是一个成员函数
```

最后，所有 C# 类都将 System.Object 作为其最终基类。C# 结构和其他值类型在以特定方式（如调用 GetHashCode 方法）使用时，将隐式“装箱”到 System.Object。
</br>C++ 没有像这样的终极基本结构，但可以创建一个，所有其他结构都可以从它派生。

```c++
// In Object.h
// base struct
struct Object
{
    virtual int32_t GetHashCode()
    {
        return HashBytes((char*)this, sizeof(*this));
    }
};
 
// In Player.h
#include "Object.h"
struct Player : Object
{
    const static int32_t MaxHealth = 100;
    const static int32_t MaxLives = 3;
    int32_t Health = MaxHealth;
    int32_t NumLives = MaxLives;
    float PosX = 0;
    float PosY = 0;
    float DirX = 0;
    float DirY = 0;
    float Speed = 0;
 
    void TakeDamage(int32_t amount);
    void Move(float time);
 
    // 根据需要可以进行重写，和 C# 中一样
    virtual int32_t GetHashCode() override
    {
        return 123;
    }
};
 
// In Vector2.h
#include "Object.h"
struct Vector2 : Object
{
    float X;
    float Y;
 
    // 不允许按需重写，就像在 C# 中一样
    // virtual int32_t GetHashCode() override
};
 
// 可以传递任何结构体
// 因为我们将每个结构体都从Object继承而来
void LogHashCode(Object& obj)
{
    DebugLog(obj.GetHashCode());
}
 
// 可以传递一个Player，因为它继承自Object
Player player;
LogHashCode(player);
 
// 可以传递一个Vector2，因为它继承自Object
Vector2 vector;
LogHashCode(vector);
```

`boxing`的通用解决方案同样是可能的，我们将在本系列的后面介绍这些技术。

## Conclusion  结论

C++ 结构继承在许多方面都是 C# 类继承的超集。
</br>它支持多重继承、虚拟继承、虚拟重载运算符、虚拟用户定义的转换函数以及 Level3 中的跳过级子对象规范（如 Level1：：X）。

在其他方面，C++ 继承比 C# 继承更精简。
</br>它没有对接口或分部类的专门支持，也不要求像 C# 中的 Object 那样使用最终的基本结构。
</br>为了恢复这些功能，我们依靠扩展功能集增加的灵活性来构建我们自己的接口、 分部类和 Object 类型。























