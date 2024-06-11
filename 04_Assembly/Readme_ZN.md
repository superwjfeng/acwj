# Part 4: An Actual Compiler

是时候兑现我写一个编译器的承诺了。所以，在我们旅程的这一部分，我们将用生成x86-64汇编代码的代码替换我们程序中的解释器。

## Revising the Interpreter

在我们这么做之前，重新审视一下 `interp.c` 中的解释器代码是值得的：

```c
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  if (n->left) leftval = interpretAST(n->left);
  if (n->right) rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:      return (leftval + rightval);
    case A_SUBTRACT: return (leftval - rightval);
    case A_MULTIPLY: return (leftval * rightval);
    case A_DIVIDE:   return (leftval / rightval);
    case A_INTLIT:   return (n->intvalue);

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

`interpretAST()` 函数 以深度优先的方式遍历给定的AST树。它首先评估任何左侧子树，然后是右侧子树。最后，它使用当前树底部的op值来操作这些子节点。

如果op值是四个数学运算符之一，那么就执行这个数学运算。如果op值表示该节点仅仅是一个整数字面量，就返回字面量值。

该函数返回这棵树的最终值。并且，因为它是递归的，它将一次一个子子树地计算整个树的最终值。

## Changing to Assembly Code Generation

我们将编写一个通用的汇编代码生成器。这反过来将调用一组特定于CPU的代码生成函数。

这是 `gen.c` 中的通用汇编代码生成器：

```c
// Given an AST, generate
// assembly code recursively
static int genAST(struct ASTnode *n) {
  int leftreg, rightreg;

  // Get the left and right sub-tree values
  if (n->left) leftreg = genAST(n->left);
  if (n->right) rightreg = genAST(n->right);

  switch (n->op) {
    case A_ADD:      return (cgadd(leftreg,rightreg));
    case A_SUBTRACT: return (cgsub(leftreg,rightreg));
    case A_MULTIPLY: return (cgmul(leftreg,rightreg));
    case A_DIVIDE:   return (cgdiv(leftreg,rightreg));
    case A_INTLIT:   return (cgload(n->intvalue));

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

看起来熟悉，对吧？我们正在做同样的深度优先树遍历。这次：

* A_INTLIT: 将一个寄存器加载为字面量值
* 其他运算符：对两个寄存器执行数学函数 这两个寄存器分别保存左子节点和右子节点的值

与传递值不同，`genAST()` 中的代码传递寄存器标识符。例如 `cgload()` 将值加载到寄存器并返回加载值的寄存器的标识。

`genAST()` 本身返回当前树点最终值的寄存器标识。这就是为什么上面的代码获取寄存器标识：

```c
if (n->left) leftreg = genAST(n->left);
if (n->right) rightreg = genAST(n->right);
```

## Calling `genAST()`

`genAST()` 将只计算给它的表达式的值。我们需要打印出这个最终计算结果。我们还需要用一些引导代码（序言）和一些尾随代码（尾声）来包装我们生成的汇编代码。这是 `gen.c` 中的另一个函数：

> 译者注：
>
> preamble

```c
void generatecode(struct ASTnode *n) {
  int reg;

  cgpreamble();
  reg= genAST(n);
  cgprintint(reg);      // Print the register with the result as an int
  cgpostamble();
}
```

## The x86-64 Code Generator

这就是通用代码生成器。现在我们需要看一些真正的汇编代码的生成。目前，我以x86-64 CPU为目标，因为这仍然是最常见的Linux平台之一。所以，打开 `cg.c` 让我们浏览一下。

### Allocating Registers

任何CPU的寄存器数量都是有限的。我们将不得不分配一个寄存器来保存整数字面量值，以及我们对它们执行的任何计算。然而，一旦我们使用了值，我们通常可以丢弃该值，从而释放保存它的寄存器。

然后我们可以重用该寄存器来保存另一个值。

有三个函数处理寄存器分配：

* `freeall_registers()`：将所有寄存器设置为可用
* `alloc_register()`：分配一个空闲寄存器
* `free_register()`：释放一个已分配的寄存器

我不会逐一介绍代码，因为它很直接，但有一些错误检查。目前，如果我用完了寄存器，程序将会崩溃。稍后，我们将处理用完空闲寄存器的情况。

代码使用通用寄存器：r0, r1, r2 和 r3。有一个字符串表，包含实际的寄存器名称：

```c
static char *reglist[4]= { "%r8", "%r9", "%r10", "%r11" };
```

这使得这些函数相对独立于CPU架构。

### Loading a Register

这是在 `cgload()` 中完成的：分配一个寄存器，然后movq指令将字面量值加载到分配的寄存器中。

```c
// Load an integer literal value into a register.
// Return the number of the register
int cgload(int value) {

  // Get a new register
  int r= alloc_register();

  // Print out the code to initialise it
  fprintf(Outfile, "\tmovq\t$%d, %s\n", value, reglist[r]);
  return(r);
}
```

### Adding Two Registers

`cgadd()` 接收两个寄存器编号，并生成将它们相加的代码。结果保存在一个寄存器中，然后释放另一个寄存器以备将来使用

```c
// Add two registers together and return
// the number of the register with the result
int cgadd(int r1, int r2) {
  fprintf(Outfile, "\taddq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

注意，加法是可交换的，所以我本可以将r2加到r1而不是r1加到r2。返回具有最终值的寄存器的标识。

### Multiplying Two Registers

这与加法非常相似，并且操作也是可交换的，所以任何寄存器都可以返回：

```c
// Multiply two registers together and return
// the number of the register with the result
int cgmul(int r1, int r2) {
  fprintf(Outfile, "\timulq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

### Subtracting Two Registers

减法不是可交换的：我们必须确保顺序正确。第二个寄存器从第一个寄存器中减去，所以我们返回第一个并释放第二个：

```c
// Subtract the second register from the first and
// return the number of the register with the result
int cgsub(int r1, int r2) {
  fprintf(Outfile, "\tsubq\t%s, %s\n", reglist[r2], reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### Dividing Two Registers

除法也不满足交换律，因此之前的注释仍然适用。在x86-64上，这甚至更加复杂。我们需要从`r1`中加载 `%rax` 寄存器作为*被除数*。这需要使用 `cqo` 来扩展为八个字节。然后，`idivq` 会将 `%rax` 与`r2`中的除数相除，将*商*留在 `%rax` 中，所以我们需要将其复制到 `r1` 或 `r2` 中。然后我们可以释放另一个寄存器。

```c
// Divide the first register by the second and
// return the number of the register with the result
int cgdiv(int r1, int r2) {
  fprintf(Outfile, "\tmovq\t%s,%%rax\n", reglist[r1]);
  fprintf(Outfile, "\tcqo\n");
  fprintf(Outfile, "\tidivq\t%s\n", reglist[r2]);
  fprintf(Outfile, "\tmovq\t%%rax,%s\n", reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### Printing A Register

在x86-64指令集中没有打印寄存器的十进制数的指令。为了解决这个问题，汇编序言包含一个名为 `printint()` 的函数，它接受一个寄存器参数并调用 `printf()` 以十进制形式进行打印。

我不会提供 `cgpreamble()` 中的代码，但它还包含了 `main()` 的开始代码，这样我们就可以组装我们的输出文件来得到一个完整的程序。`cgpostamble()` 的代码也不在这里给出，它只是简单地调用 `exit(0)` 来结束程序。

然而，这里是 `cgprintint()` 的代码：

```c
void cgprintint(int r) {
  fprintf(Outfile, "\tmovq\t%s, %%rdi\n", reglist[r]);
  fprintf(Outfile, "\tcall\tprintint\n");
  free_register(r);
}
```

在Linux x86-64 中，函数的第一个参数应该位于 `%rdi` 寄存器中，因此在调用 `printint` 之前，我们将我们的寄存器移动到 `%rdi` 寄存器中。

## Doing Our First Compile

这就是x86-64代码生成器的全部内容。在 `main()` 中还有一些额外的代码来打开 `out.s` 作为我们的输出文件。我也保留了解释器在程序中，这样我们可以确认我们的汇编计算与解释器对输入表达式的答案相同。

让我们编译编译器并运行它在input01上：

```make
$ make
cc -o comp1 -g cg.c expr.c gen.c interp.c main.c scan.c tree.c

$ make test
./comp1 input01
15
cc -o out out.s
./out
15
```

是的！第一个15是解释器的输出。第二个15是汇编的输出。

## Examining the Assembly Output

那么，汇编输出到底是什么呢？嗯，这是输入文件：

```
2 + 3 * 5 - 8 / 3
```

这里是这个输入的 `out.s`，带有注释：

```
        .text                           # Preamble code
.LC0:
        .string "%d\n"                  # "%d\n" for printf()
printint:
        pushq   %rbp
        movq    %rsp, %rbp              # Set the frame pointer
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movl    -4(%rbp), %eax          # Get the printint() argument
        movl    %eax, %esi
        leaq    .LC0(%rip), %rdi        # Get the pointer to "%d\n"
        movl    $0, %eax
        call    printf@PLT              # Call printf()
        nop
        leave                           # and return
        ret

        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp              # Set the frame pointer
                                        # End of preamble code

        movq    $2, %r8                 # %r8 = 2
        movq    $3, %r9                 # %r9 = 3
        movq    $5, %r10                # %r10 = 5
        imulq   %r9, %r10               # %r10 = 3 * 5 = 15
        addq    %r8, %r10               # %r10 = 2 + 15 = 17
                                        # %r8 and %r9 are now free again
        movq    $8, %r8                 # %r8 = 8
        movq    $3, %r9                 # %r9 = 3
        movq    %r8,%rax
        cqo                             # Load dividend %rax with 8
        idivq   %r9                     # Divide by 3
        movq    %rax,%r8                # Store quotient in %r8, i.e. 2
        subq    %r8, %r10               # %r10 = 17 - 2 = 15
        movq    %r10, %rdi              # Copy 15 into %rdi in preparation
        call    printint                # to call printint()

        movl    $0, %eax                # Postamble: call exit(0)
        popq    %rbp
        ret
```

我们现在有了一个合法的编译器：一个程序，它以一种语言接收输入，并生成该输入的另一种语言的翻译。

我们仍然需要将输出汇编成机器代码并与支持库链接，但这是我们现在可以手动执行的。稍后，我们将编写一些代码来自动执行此操作。

## Conclusion and What's Next

从解释器转换为通用代码生成器很简单，但然后我们必须编写一些代码来生成真正的汇编输出。为此，我们必须考虑如何分配寄存器：目前，我们有一个简单的解决方案。我们还必须处理一些x86-64的奇特之处，比如idivq指令。

我还没有触及的一件事是：为什么要费心为表达式生成AST呢？当然，当我们在Pratt解析器中遇到 `+` 标记时，我们本可以调用 `cgadd()`，其他运算符也是如此。我会留给你思考这个问题，但我会在一两步后回来讨论它。

在我们编写编译器的旅程的下一部分，我们将向我们的语言添加一些语句，这样它就开始像一个合适的计算机语言了。 [Next step](../05_Statements/Readme.md)
