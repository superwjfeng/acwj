# Part 0: Introduction

我决定踏上编写编译器的旅程。过去我写过一些[汇编程序](https://github.com/DoctorWkt/pdp7-unix/blob/master/tools/as7)，也为一种无类型语言编写了一个[简单的编译器](https://github.com/DoctorWkt/h-compiler)。但我从未写过能够自我编译的编译器。所以，这将是我这次旅程的目标。

在这个过程中，我打算记录我的工作，以便其他人可以跟随。这同时也有助于我澄清思路和想法。希望你和我都会觉得这很有用！

## Goals of the Journey

以下是我在这次旅程中设定的目标与非目标 non-goals：

- 编写一个能够自我编译的编译器。我认为，如果编译器能够编译自身，它才能被称为一个*真正的*编译器。（译者注：即要实现一个自举 bootstrapping 的编译器）
- 至少针对一个真实的硬件平台。我见过一些为假想机器生成代码的编译器。我希望我的编译器能在真实的硬件上工作。此外，如果可能的话，我希望编写这样一个编译器：它可以支持不同硬件平台的多个后端。
- 实践优先于研究。在编译器领域有大量的研究。我想从零开始这段旅程，因此我会采取实用的方法而非理论密集的方法。话虽如此，在某些时候我还是需要引入（并实现）一些基于理论的东西。
- 遵循KISS原则（keep it simple）：保持简单，蠢货！在这里我肯定会使用Ken Thompson的原则："当有疑问时，使用蛮力。"
- 采取许多小步骤来达到最终目标。我会将旅程分解成许多简单的步骤，而不是采取大的飞跃。这将使得每一个对编译器的新添加都是一件一口大小、易于消化的事情。

## Target Language

选择目标语言是一件困难的事。如果我选择像Python、Go等高级语言，那么我将不得不实现一大堆库和类，因为它们是语言内置的。

我可以为像Lisp这样的语言编写一个编译器，但这些[可以轻易完成](ftp://publications.ai.mit.edu/ai-publications/pdf/AIM-039.pdf)。

相反，我回到了老路，我打算为C语言的一个子集编写一个编译器，足以让编译器能够自我编译。

C语言只比汇编语言高出一步（对于C语言的某些子集来说，而非[C18标准](https://en.wikipedia.org/wiki/C18_(C_standard_revision))），这将有助于简化将C代码编译成汇编代码的任务。哦，我还喜欢C语言。

## The Basics of a Compiler's Job

编译器的工作就是输入一门语言（一般是一门高级语言），输出一门其他的语言（一般是一门相比输入语言低级的语言）。主要的步骤如下：

![](Figs/parsing_steps.png)

 + 做[词法分析](https://en.wikipedia.org/wiki/Lexical_analysis)来得到词素。在某些语言中，`=` 是和 `==` 不同的，所以不能只是读取一个 `=`。我们把这些词素也称为 tokens。
 + [解析输入](https://en.wikipedia.org/wiki/Parsing)，即识别输入的语法和结构元素，并确保它们符合语言的*语法*。例如，你的语言可能有这样的决策结构：

```C
if (x < 23) {
	print("x is smaller than 23\n");
}
```

> 但是在其他语言里可以写成：

```C
if (x < 23):
	print("x is smaller than 23\n")
```

> This is also the place where the compiler can detect syntax errors, like if
the semicolon was missing on the end of the first *print* statement.

 + 对输入进行[语义分析](https://en.wikipedia.org/wiki/Semantic_analysis_(compilers))，即理解输入的含义。这实际上与识别语法和结构不同。例如，在英语中，一个句子可能具有 `<主语> <动词> <形容词> <宾语>` 的形式。以下两个句子具有相同的结构，但完全不同的含义：

```
David ate lovely bananas.
Jennifer hates green tomatoes.
```

 + [翻译](https://en.wikipedia.org/wiki/Code_generation_(compiler))：将输入的意思转换成另一种语言。在这里，我们逐部分将输入转换为一种更低级别的语言。

## Resources

互联网上有许多关于编译器的资源。以下是我将要查看的一些资源。

### Learning Resources

如果你想从一些关于编译器的书籍、论文和工具开始，我强烈推荐以下列表：

- [精选的优秀编译器、解释器和运行时资源列表](https://github.com/aalhour/awesome-compilers) 作者 Ahmad Alhour

### Existing Compilers

虽然我打算构建自己的编译器，但我计划参考其他编译器以获取灵感，很可能也会借用它们的一些代码。以下是我将要参考的编译器：

- [SubC](http://www.t3x.org/subc/) 作者 Nils M Holm
- [Swieros C编译器](https://github.com/rswier/swieros/blob/master/root/bin/c.c) 作者 Robert Swierczek
- [fbcc](https://github.com/DoctorWkt/fbcc) 作者 Fabrice Bellard
- [tcc](https://bellard.org/tcc/) 同样由 Fabrice Bellard 和其他人开发
- [catc](https://github.com/yui0/catc) 作者 Yuichiro Nakada
- [amacc](https://github.com/jserv/amacc) 作者 Jim Huang
- [Small C](https://en.wikipedia.org/wiki/Small-C) 由 Ron Cain、 James E. Hendrix 创建，其他人衍生

特别是，我将使用许多来自 SubC 编译器的想法，和一些代码。

## Setting Up the Development Environment

假设你想加入这次旅程，以下是你需要准备的。我将使用 Linux 开发环境，所以下载并设置你最喜欢的 Linux 系统：我正在使用 Lubuntu 18.04。

我将针对两个硬件平台：Intel x86-64 和 32位 ARM。我将使用运行 Lubuntu 18.04 的 PC 作为 Intel 目标平台，以及运行 Raspbian 的 Raspberry Pi 作为 ARM 目标平台。

在 Intel 平台上，我们需要一个现有的 C 编译器。因此安装这个包（我提供 Ubuntu/Debian 的命令）：

```cmd
$ sudo apt-get install build-essential
```

如果对于普通 Linux 系统还需要任何其他工具，请告知我。

最后，克隆一份这个 Github 仓库的副本。

## The Next Step

在我们的编写编译器之旅的下一部分，我们将从扫描输入文件并找到构成我们语言的词汇元素——*标记*（tokens）的代码开始。[下一步](https://gpt.momenta.works/01_Scanner/Readme.md)
