# C++ For C# Developers: Part 1 – Introduction

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5535),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

今天我们将从绝对的基础开始——原始类型和字面量——我们将通过整个系列的剩余部分来构建这些内容。
</br>尽管这个主题听起来很基础，但当我们从像C#这样的语言转换过来时，其中一些内容可能会相当令人震惊。

## Types  类型

让我们从整数开始，它们在两个方面令人惊讶：它们的定义有多宽松，以及它们的类型有多少。类型名本身由一个或多个部分组成：


| Part                            | 	Meaning                                                                                                 |
|---------------------------------|----------------------------------------------------------------------------------------------------------|
| signed, unsigned, or none       | 	If the type is signed or not. None means signed.</br>类型是否有符号。None 表示已签名。                                |
| short, long, long long, or none | 	Size classification of the integer. Not an exact size! None means int.</br>整数的大小分类。不是确切的尺寸！None 表示 int。 |
| int or none                     | 	Explicitly state that this is an integer. None states this implicitly.</br>明确声明这是一个整数。None 隐含地说明了这一点。   |

以下是所有24种排列组合，包括在常见平台上的位大小：


| C# Type | 	C++ Type              | 	Windows Size | 	Unix Size |
|---------|------------------------|---------------|------------|
| short	  | short                  | 	16           | 	16        |
| short	  | short int              | 	16           | 	16        |
| short	  | signed short           | 	16           | 	16        |
| short	  | signed short int       | 	16           | 	16        |
| ushort	 | unsigned short         | 	16           | 	16        |
| ushort	 | unsigned short int     | 	16           | 	16        |
| int	    | int	                   | 32	           | 32         |
| int	    | signed	                | 32	           | 32         |
| int	    | signed int             | 	32	          | 32         |
| uint	   | unsigned	              | 32	           | 32         |
| uint	   | unsigned int           | 	32           | 	32        |
| N/A     | 不适用	long               | 	32           | 	64        |
| N/A     | 不适用	long int           | 	32           | 	64        |
| N/A     | 不适用	long int           | 	32           | 	64        |
| N/A     | 不适用	signed long        | 	32           | 	64        |
| N/A     | 不适用	signed long int    | 	32           | 	64        |
| N/A     | 不适用	unsigned long	     | 32	           | 64         |
| N/A     | 不适用	unsigned long int  | 	32           | 	64        |
| long	   | long long              | 	64           | 	64        |
| long	   | long long int          | 	64           | 	64        |
| long	   | signed long long       | 	64           | 	64        |
| long	   | signed long long int   | 	64           | 	64        |
| ulong	  | unsigned long long	    | 64            | 	64        |
| ulong	  | unsigned long long int | 	64           | 	64        |

还有一种类型叫做`size_t`，它是一个32位或64位的无符号整数，这取决于编译时使用的CPU。

有四种 8 位类型：

| C# Type | 	C++ Type     | 	x86 and x64	ARM |
|---------|---------------|------------------|
| bool	   | bool	         | N/A  	           |N/A  |
| sbyte	  | char	         | Signed  	        |Unsigned|  
| sbyte	  | signed char   | 	Signed          |	Signed| 
| byte	   | unsigned char | 	Signed          |	Signed |

使用`char`命名的类型是因为它们最初用于ASCII字符串中的字符。还有更大的字符类型：


| C# Type | C++ Type   | 	Windows Size | 	Unix Size |
|---------|------------|---------------|------------|
| N/A     | 	char8_t	  | 8	            | 8          |
| N/A     | 	char16_t	 | 16            | 	16        |
| N/A     | 	char32_t	 | 32            | 	32        |
| N/A     | 	wchar_t	  | 16	           | 32         |

接下来，我们将介绍浮点类型，包括`long double`类型：

| C# Type  | C++ Type      | x86 Size | ARM Size |
|----------|---------------|----------|----------|
| `float`  | `float`       | 32       | 32       |
| `double` | `double`      | 64       | 64       |
| N/A      | `long double` | 80       | 64       |

C++中没有十进制类型，但像[GMP](https://gmplib.org/)这样的库提供了类似的功能。

鉴于CPU和操作系统之间大小的不确定性，避免使用许多这些类型，而改用具有特定大小的类型是一种最佳实践。
</br>这些类型可以在标准库或游戏引擎API中找到。这样使一切变得多么简单：

| Meaning                       | C# Type   | C++ Type   | Unreal Type |
|-------------------------------|-----------|------------|-------------|
| Boolean                       | `bool`    | `bool`     | `bool`      |
| 8-bit signed integer          | `sbyte`   | `int8_t`   | `int8`      |
| 8-bit unsigned integer        | `byte`    | `uint8_t`  | `uint8`     |
| 16-bit signed integer         | `short`   | `int16_t`  | `int16`     |
| 16-bit unsigned integer       | `ushort`  | `uint16_t` | `uint16`    |
| 8-bit character               | N/A       | `char8_t`  | `CHAR8`     |
| 16-bit character              | `char`    | `char16_t` | `CHAR16`    |
| 32-bit character              | N/A       | `char32_t` | `CHAR32`    |
| 32-bit signed integer         | `int`     | `int32_t`  | `int32`     |
| 32-bit unsigned integer       | `uint`    | `uint32_t` | `uint32`    |
| 64-bit signed integer         | `long`    | `int64_t`  | `int64`     |
| 64-bit unsigned integer       | `ulong`   | `uint64_t` | `uint64`    |
| 32-bit floating point number  | `float`   | `float`    | `float`     |
| 128-bit floating point number | `decimal` | N/A        | N/A         |

## Literals  字面量

现在我们知道了所有这些类型，让我们通过编写一些字面量来表示它们。首先，也是最明显的，是booleans：

| Literal | Type   | Value |
|---------|--------|-------|
| `true`  | `bool` | `1`   |
| `false` | `bool` | `0`   |

接下来是整数。它们由四个部分组成：

| Part                                           | Meaning                                                                                                                                                                                                                                                              |
|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `0x`,`0X`,`0`,`0b`,`0B`or none                 | The chosen base: hexadecimal, octal, or binary. None means decimal.</br>基础：十六进制、八进制或二进制。无表示十进制。                                                                                                                                                                      |
| `0123456789abcdefABCDEF'`,`01234567'`, or`01'` | Digits of the chosen base.`'`characters are optional separators like`_`in C#.</br>所选基数的数字。`'`字符是可选的分隔符，如C#中的`_`。                                                                                                                                                     |
| `u`,`U`, or none                               | If the integer is unsigned. None means signed for decimal and octal, unsigned for hexadecimal and binary.</br>整数是无符号的。None表示十进制和八进制是有符号的，十六进制和二进制是无符号的。                                                                                                              |
| `l`,`L`,`ll`,`LL`, or none                     | The size classification. None means “the smallest size that can fit the value” from the`int`size classification to`long`then to`long long`. Note: can be swapped with`u`or`U`, if specified</br>大小分类。None表示“能够容纳该值的最小大小”，从int大小分类到long，然后到long long。注意：如果指定，可以与u或U互换 |

这里有一些例子：

| Literal                                 | Type            | Base                   | Signed              | Size                  |
|-----------------------------------------|-----------------|------------------------|---------------------|-----------------------|
| `123`                                   | `int`           | Decimal (default)      | Signed (default)    | `int`(default)        |
| `5000000000`                            | `long`          | Decimal (default)      | Signed (default)    | `long`(default)       |
| `123u`                                  | `unsigned int`  | Decimal (default)      | Unsigned (explicit) | `int`(default)        |
| `123ul`                                 | `unsigned long` | Decimal (default)      | Unsigned (explicit) | `long`(explicit)      |
| `123lu`                                 | `unsigned long` | Decimal (default)      | Unsigned (explicit) | `long`(explicit)      |
| `0x123456`                              | `int`           | Hexadecimal (explicit) | Signed (default)    | `int`(default)        |
| `0xffffffff`                            | `unsigned int`  | Hexadecimal (explicit) | Unsigned (default)  | `int`(default)        |
| `0xffffffffff`                          | `long`          | Hexadecimal (explicit) | Signed (default)    | `long`(default)       |
| `0xFFFFFFFFll`                          | `long long`     | Hexadecimal (explicit) | Signed (default)    | `long long`(explicit) |
| `0b10101010'01010101'10101010'01010101` | `unsigned int`  | Binary (explicit)      | Unsigned (default)  | `int`(default)        |
| `0123`                                  | `int`           | Octal (explicit)       | Signed (default)    | `int`(default)        |

接下来是浮点字面量，它们也分为四个部分：

| Part                                                                     | Meaning                                                                                                                                                                                                                                       |
|--------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `0x`,`0X`, or none                                                       | Choose hexadecimal, or none for decimal </br>选择十六进制，或无表示十进制                                                                                                                                                                                   |
| `0123456789abcdefABCDEF.'`                                               | Digits of the chosen base.`'`characters are optional separators like`_`in C#. May end in`.`for whole numbers.</br>所选基数的数字。`'`字符是可选的分隔符，类似于C#中的`_`。对于整数，可能以`.`结尾。                                                                              |
| `e`,`e`then`+-`then`0123456789`,`p`,`p`then`+-`then`0123456789`, or none | Exponent`x`to multiply digits by`10^x`. Always required for hexadecimal and required for decimal if there’s no`.`in the digits.`e`for decimal and`p`for hexadecimal.</br>指数`x`，用于将数字乘以`10^x`。对于十六进制总是需要，对于没有点的数字的十进制也是必需的。十进制使用`e`，十六进制使用`p`。 |
| `f`,`F`,`l`,`L`, or none                                                 | Size classification of`float`(`f`) or`long double`(`l`). None means`double`.</br>`float`（f）或`long double`（l）的大小分类。None表示`double`。                                                                                                             |

以下是浮点字面量的示例：

| Literal            | Type     | Base        |
|--------------------|----------|-------------|
| `12.34`            | `double` | Decimal     |
| `12.34f`           | `float`  | Decimal     |
| `12.34F`           | `float`  | Decimal     |
| `12.34e2`          | `double` | Decimal     |
| `12.34e-2`         | `double` | Decimal     |
| `12.34e-2f`        | `float`  | Decimal     |
| `12.e1`            | `double` | Decimal     |
| `12'34.56'78f`     | `float`  | Decimal     |
| `0x12p2`           | `double` | Hexadecimal |
| `0x12.p2`          | `double` | Hexadecimal |
| `0x12'34'56.78p2f` | `float`  | Hexadecimal |


最后，我们有几种形式的字符字面量：

| Form    | Meaning                                                                                                     |
|---------|-------------------------------------------------------------------------------------------------------------|
| `'c'`   | `char`type if`c`fits, otherwise`int`type, with character`c`</br>如果 `c` 适合，则为 `char` 类型，否则为 `int` 类型，字符为 `c` |
| `u8'c'` | `char8_t`type with UTF-8 character`c`          </br>使用 UTF-8 字符 `c` 的 `char8_t` 类型                          |
| `u'c'`  | `char16_t`type with UTF-16 character`c`        </br>`char16_t` UTF-16 字符 `c` 的类型                            |
| `U'c'`  | `char32_t`type with UTF-32 character`c`        </br>使用 UTF-32 字符 `c` 的 `char32_t` 类型                        |
| `L'c'`  | `wchar_t`type with character`c`                </br>字符 `c` 的 `wchar_t` 类型                                   |
| `'abc'` | `int`type representing multiple characters`abc`</br>表示多个字符 `abc` 的 `int` 类型                                 |

字符可以是它们集合中的任何东西（例如UTF-8），但不能是`'`、`\`和换行符。要获取这些以及其他特殊字符，请使用转义序列：

| Meaning                   | Escape Sequence | Note                          | Example            |
|---------------------------|-----------------|-------------------------------|--------------------|
| Single quote              | `\'`            |                               |                    |
| Double quote              | `\"`            |                               |                    |
| Question mark             | `\?`            |                               |                    |
| Backslash                 | `\`             |                               |                    |
| Bell                      | `\a`            |                               |                    |
| Backspace                 | `\b`            |                               |                    |
| Form feed                 | `\f`            |                               |                    |
| Line feed                 | `\n`            |                               |                    |
| Carriage return           | `\r`            |                               |                    |
| Tab                       | `\t`            |                               |                    |
| Vertical tab              | `\v`            |                               |                    |
| Octal value               | `\ABC`          | `\ABC`is the octal value      | `\0`is`NUL`        |
| Hexadecimal value         | `\xAB`          | `\AB`is the hexadecimal value | `\x41`is`A`        |
| 16-bit Unicode code point | `\uABCD`        | `\ABCD`is the code point      | `\u03b1`is`Î±`     |
| 32-bit Unicode code point | `\UABCDEFGH`    | `\ABCDEFGH`is the code point  | `\U0001F389`is`🎉` |

这里有一些示例字符字面量：

| Literal      | Type       | Decimal Value |
|--------------|------------|---------------|
| `'A'`        | `char`     | `65`          |
| `'?'`        | `char`     | `63`          |
| `u8'A'`      | `char8_t`  | `65`          |
| `u'Î±'`      | `char16_t` | `945`         |
| `U'\x1f389'` | `char32_t` | `127881`      |
| `'ab'`       | `int`      | `127881`      |

## Conclusion  结论

C++ 文本类似于 C# 文本，但在几个方面有所不同。通常，您可以使用两种语言编写完全相同的代码并获得相同的效果。不过，有几种极端情况，因此了解有关该语言工作原理的一些细节非常重要。

> 这里可能比较陌生的是Literals(字面量)
> </br>简单理解就是直接在代码中标识的值，比如：42，3.14，114514等等
> </br>Literals主要就是各种字面量所标识的类型











