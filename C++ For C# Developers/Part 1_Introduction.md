# C++ For C# Developers: Part 1 – Introduction

> 注：本文译自该[博客](https://www.jacksondunstan.com/articles/5530),仅供学习参考
> </br>在这个类似这种提示框内，皆为本人看法，如有错误欢迎指正

在Unity之外，C#很少被用作游戏编程语言。C++在Unreal、Cryengine、Lumberyard以及几乎所有专有游戏工作室引擎中都被广泛使用。
</br>本系列是为那些了解C#的Unity游戏程序员准备的，他们希望拓宽自己的技能，以便能够有效地为其他引擎编写代码，甚至为Unity编写C++脚本。
</br>今天，我们将从C++的历史、标准库、工具、社区和文档的介绍开始。继续阅读以开始学习！

## History  历史

C++ 的前身是 C，于 1972 年首次亮相。它仍然是最常用的语言 ，C++ 排在第四位，C# 排在第五位

C++于1979年以“C with Classes”的名字开始。C++这个名字后来在1982年出现。
最初的C++编译器Cfront输出C源文件，然后编译成机器码。那个编译器早已被取代，现代编译器都直接将C++编译成机器码。

1989年，语言中增加了“C++ 2.0”的主要功能，随后该语言由ISO标准化，并于1998年正式发布。
口语中，这被称为C++98，并开始了将年份添加到语言版本名称中的惯例。
它还正式化了通过委员会和各个工作组设计和标准化语言的过程。

2003年对语言进行的微小改动导致了C++03的诞生，但“现代C++”时代随着C++11对语言的巨大变革而开始。
这也加快了标准化进程，从之前的八年间隔缩短到了三年。这意味着我们得到了C++14的微小改动，C++17的相对较大改动，而C++20将带来巨大的变化。
像Unreal、Cryengine和Lumberyard这样的游戏引擎都支持最新的版本——C++17，并且标准化后可能会支持C++20。

到目前为止，这种语言与C语言几乎没有相似之处。许多C代码仍然可以编译为C++，但惯用的C++仅在外观上与C相似。

## Standard Library  标准库

C++的每个版本都包含所谓的“标准库”。这个库通常被称为“STL”，即“标准模板库”，因为它大量使用了C++语言的一个特性——模板。这个库与语言本身一样，由ISO标准化。

标准库类似于.NET的框架类库或CoreFX。其架构方法是让C++语言拥有强大的底层语言特性，以便更多功能可以在库中实现，而不是直接包含在语言中。
例如，语言本身不包含字符串类。 相反，标准库提供了一个使用底层语言特性高效实现的字符串类。

以下表格显示了标准库的主要部分及其在.NET中的大致等效部分：

| Standard Library Section </br>标准库部分 | c++                    | C#                                 |
|-------------------------------------|------------------------|------------------------------------|
| Language support                    | 	numeric_limits::max   | 	int.MaxValue                      |
| Concepts                            | 	default_initializable | 	where T : new()                   |
| Diagnostics	                        | exception              | 	System.Exception                  |
| Utilities                           | 	tuple<int, float>	    | (int, float)                       |
| Strings	                            | string                 | 	System.String                     |
| Containers                          | 	vector	               | List                               |
| Iterators                           | 	begin()               | 	GetEnumerator()                   |
| Ranges	                             | views::filter          | 	Enumerable.Where                  |
| Algorithms	                         | transform              | 	Enumerable.Select                 |
 | Numerics                            | 	accumulate	           | Enumerable.Aggregate               |
 | Localization	                       | toupper                | 	Char.ToUpper                      |
 | I/O	                                | fstream	               | FileStream                         |
 | File system                         | 	copy                  | 	File.Copy                         |
 | Regular expressions                 | 	regex                 | 	Regex                             |
| Atomic operations	                  | atomic++               | 	            Interlocked.Increment |
 | Threading	                          | thread	                | Thread                             |

一些游戏编程环境不使用标准库，或者至少将其使用量降到最低。艺电（EA）实现了一个名为EASTL的自己的版本。虚幻引擎（Unreal）有许多内置的类似类型（如`FString`与`string`）和函数（如`MakeUnique`与`make_unique`）。
这些库从与标准库相同的底层语言特性中获益，但它们使用这些特性来高效地重新实现在其他许多语言中可能作为语言特性的功能。

## Tools  工具

主要的工具当然是编译器。如今有很多不错的选择，但以下是一些最受欢迎的：

| Compiler                      | 	Cost          | 	Open Source | 	Platforms             |
|-------------------------------|----------------|--------------|------------------------|
| Microsoft Visual Studio       | 	Free and Paid | 	No          | 	Windows               |
| GCC (GNU Compiler Collection) | 	Free          | 	Yes	        | Windows, macOS, Linux  |
| Clang                         | 	Free	         | Yes	         | Windows, macOS, Linux  |
| Intel C++                     | 	Free          | 	No          | 	Windows, macOS, Linux |

也有许多IDE可供选择，它们通常具备文本编辑、编译器执行、交互式调试等功能。以下是一些流行的选项：


| IDE                          | 	Cost          | 	Open Source | 	Platforms            |
|------------------------------|----------------|--------------|-----------------------|
| Microsoft Visual Studio      | 	Free and Paid | 	No          | 	Windows              |
| Apple Xcode                  | 	Free	         | No	          | macOS                 |
| JetBrains CLion	             | Paid           | 	No	         | Windows, macOS, Linux |
| Microsoft Visual Studio Code | 	Free          | 	Yes	        | Windows, macOS, Linux |

许多静态分析器，也称为“linters”，以及动态分析器都可用。
</br>Clang sanitizers套件是免费、开源的，并且支持Unreal。
</br>还有像Coverity SAST这样的商业工具也可用。
</br>Clang格式和许多IDE可以强制执行风格指南并自动重新格式化代码。

## Documentation  文档

C++标准可以购买，但几乎没有任何C++开发者真的购买它。有一个免费提供的草案版本，它与正式版本几乎相同，但由于它非常长且技术性极强，所以它也只是一种最后的参考。
大多数开发者不会直接阅读标准本身，而是像阅读C#的Microsoft Docs（即MSDN）一样阅读诸如cppreference.com之类的参考网站。

C++有许多指南文档。 C++核心指南、Google C++风格指南以及特定引擎的标准都普遍使用。
特别是，C++核心指南有一个配套的指南支持库（GSL），用于执行和促进这些指南。