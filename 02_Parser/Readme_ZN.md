# Part 2: Introduction to Parsing

在编译器编写之旅的这一部分，我将介绍解析器的基础知识。正如我在第一部分中提到的，解析器的工作是识别输入的语法和结构要素，并确保它们符合*语言的语法*。

我们已经有几个可以扫描的语言要素，即我们的标记:

* 四个基本的数￥￥算符: `*`, `/`, `+` 和 `-`
* 具有 1 个或多个数字 `0` 到 `9` 的十进制整数

现在让我们定义一个语法，让我们的解析器能够识别。

## BNF: Backus-Naur Form

你在处理计算机语言时可能会遇到[BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)的使用。我将在这里介绍足够表达我们想要识别的语法的BNF语法。

我们希望有一个用整数表达数学表达式的语法。以下是该语法的BNF描述：

```
expression: number
          | expression '*' expression
          | expression '/' expression
          | expression '+' expression
          | expression '-' expression
          ;

number:  T_INTLIT
         ;
```

垂直线 `|` 用于分隔语法中的选项，因此上面的内容表示：

* 表达式可以是一个数字，或者
* 表达式是由一个 '*` 标记分隔的两个表达式，或者
* 表达式是由一个 '/` 标记分隔的两个表达式，或者
* 表达式是由一个 '+` 标记分隔的两个表达式，或者
* 表达式是由一个 '-` 标记分隔的两个表达式
* 数字始终是一个 T_INTLIT 标记

显然，BNF中语法的定义是*递归的*：一个表达式通过参照其他表达式来定义。但是，有一种*递归的结束*方法：当一个表达式实际上是一个数字时，这总是一个 T_INTLIT 标记，因此不再是递归的。

在BNF中，我们称"expression"和"number"为*非终结符*，因为它们是由语法规则生成的。然而，T_INTLIT 是*终结符*，因为它不由任何规则定义。相反，它是语言中已经识别的标记。类似地，四个数￥￥算符标记也是终结符。

## Recursive Descent Parsing

鉴于我们的语言的语法是递归的，尝试递归解析是有意义的。我们需要读入一个标记，然后*预读*下一个标记。根据下一个标记，我们可以决定解析输入时应采取哪种路径。这可能需要递归调用已经被调用的函数。

在我们的情况下，任何表达式中的第一个标记将是一个数字，后面可能跟着数￥￥算符。之后可能只有一个单独的数字，或者可能是一个全新表达式的开始。我们该如何递归解析这个过程呢？

我们可以编写伪代码如下：

```
function expression() {
  Scan and check the first token is a number. Error if it's not
  Get the next token
  If we have reached the end of the input, return, i.e. base case

  Otherwise, call expression()
}
```

让我们在输入 `2 + 3 - 5 T_EOF` 上运行这个函数，其中 `T_EOF` 是一个代表输入结束的标记。我将对每次调用 `expression()` 进行编号。

```
expression0:
  Scan in the 2, it's a number
  Get next token, +, which isn't T_EOF
  Call expression()

    expression1:
      Scan in the 3, it's a number
      Get next token, -, which isn't T_EOF
      Call expression()

        expression2:
          Scan in the 5, it's a number
          Get next token, T_EOF, so return from expression2

      return from expression1
  return from expression0
```

是的，该函数能够递归解析输入 `2 + 3 - 5 T_EOF`。

当然，我们并没有对输入做任何操作，但这不是解析器的工作。解析器的工作是*识别*输入，并警告任何语法错误。其他人将负责对输入进行*语义分析*，即理解和执行这个输入的含义。

> > 后来你会发现，这实际上并不是真的。通常将语法分析和语义分析交织在一起是有意义的。

## Abstract Syntax Trees

要进行语义分析，我们需要编写代码来解释识别的输入，或者将其转换为另一种格式，例如汇编代码。在这段旅程中，我们将为输入构建一个解释器。但是为了达到这个目标，我们首先要将输入转换成抽象语法树（AST）。

我强烈建议你阅读这篇关于AST的简短解释：

* [通过AST提升语法分析水平](https://www.aidutu.cn/info/link?url=https://medium.com/basecs/leveling-up-ones-parsing-game-with-asts-d7a6fc2400ff) by Vaidehi Joshi

这篇文章写得很好，真的有助于解释AST的目的和结构。 不用担心，等你回来时我还会在这里。

我们将要构建的AST中每个节点的结构在 `defs.h` 文件中有描述。

```c
// AST node types
enum {
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE, A_INTLIT
};

// Abstract Syntax Tree structure
struct ASTnode {
  int op;                               // "Operation" to be performed on this tree
  struct ASTnode *left;                 // Left and right child trees
  struct ASTnode *right;
  int intvalue;                         // For A_INTLIT, the integer value
};
```

一些AST节点，例如具有 `op` 值为 `A_ADD` 和 `A_SUBTRACT` 的节点，有两个子AST，分别由 `left` 和 `right` 指向。之后，我们将对子树的值进行加法或减法运算。

另外，具有 `op` 值为 `A_INTLIT` 的AST节点表示一个整数值。它没有子树，只有 `intvalue` 字段中的值。

## Building AST Nodes and Trees

`tree.c` 中的代码包含了构建AST的函数。最通用的函数是 `mkastnode()`，它接受AST节点的所有四个字段的值。该函数会分配节点，填充字段的值，并返回指向该节点的指针：

```c
// Build and return a generic AST node
struct ASTnode *mkastnode(int op, struct ASTnode *left,
                          struct ASTnode *right, int intvalue) {
  struct ASTnode *n;

  // Malloc a new ASTnode
  n = (struct ASTnode *) malloc(sizeof(struct ASTnode));
  if (n == NULL) {
    fprintf(stderr, "Unable to malloc in mkastnode()\n");
    exit(1);
  }
  // Copy in the field values and return it
  n->op = op;
  n->left = left;
  n->right = right;
  n->intvalue = intvalue;
  return (n);
}
```

根据这一点，我们可以编写更具体的函数，用于创建叶子节点（即没有子节点的节点），以及创建具有单个子节点的节点：

```c
// Make an AST leaf node
struct ASTnode *mkastleaf(int op, int intvalue) {
  return (mkastnode(op, NULL, NULL, intvalue));
}

// Make a unary AST node: only one child
struct ASTnode *mkastunary(int op, struct ASTnode *left, int intvalue) {
  return (mkastnode(op, left, NULL, intvalue));
```

## Purpose of the AST

我们将使用AST来存储我们识别的每个表达式，这样我们之后就可以递归地遍历它，计算表达式的最终值。我们确实希望处理数￥￥算符的优先级。以下是一个例子。

考虑表达式 `2 * 3 + 4 * 5`。现在，乘法比加法具有更高的优先级。因此，我们希望*绑定*乘法操作数，并在进行加法之前执行这些操作。

如果我们生成的AST树看起来像这样：

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

那么，在遍历该树时，我们会先执行 `2*3`，然后是 `4*5`。一旦我们得到这些结果，就可以将它们传递到树的根部进行加法运算。

## A Naive Expression Parser

现在，我们可以重复使用扫描器中的标记值作为AST节点的操作值，但我喜欢将标记和AST节点的概念分开。因此，首先，我要编写一个函数来将标记值映射到AST节点的操作值。这部分代码，以及解析器的其余部分，都在 `expr.c` 中。

```c
// Convert a token into an AST operation.
int arithop(int tok) {
  switch (tok) {
    case T_PLUS:
      return (A_ADD);
    case T_MINUS:
      return (A_SUBTRACT);
    case T_STAR:
      return (A_MULTIPLY);
    case T_SLASH:
      return (A_DIVIDE);
    default:
      fprintf(stderr, "unknown token in arithop() on line %d\n", Line);
      exit(1);
  }
}
```

switch语句中的默认语句在无法将给定标记转换为AST节点类型时触发。它将成为解析器中语法检查的一部分。

我们需要一个函数来检查下一个标记是否是整数字面量，并创建一个AST节点来保存这个字面量值。以下是该函数：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;

  // For an INTLIT token, make a leaf AST node for it
  // and scan in the next token. Otherwise, a syntax error
  // for any other token type.
  switch (Token.token) {
    case T_INTLIT:
      n = mkastleaf(A_INTLIT, Token.intvalue);
      scan(&Token);
      return (n);
    default:
      fprintf(stderr, "syntax error on line %d\n", Line);
      exit(1);
  }
}
```

这个假设需要一个全局变量 `Token`，并且它已经从输入中扫描到了最近的标记。在 `data.h` 文件中：

```c
extern_ struct token    Token;
```

还有在 `main()` 中：

```c
  scan(&Token);                 // Get the first token from the input
  n = binexpr();                // Parse the expression in the file
```

现在我们可以写解析器的代码：

```c
// Return an AST tree whose root is a binary operator
struct ASTnode *binexpr(void) {
  struct ASTnode *n, *left, *right;
  int nodetype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  if (Token.token == T_EOF)
    return (left);

  // Convert the token into a node type
  nodetype = arithop(Token.token);

  // Get the next token in
  scan(&Token);

  // Recursively get the right-hand tree
  right = binexpr();

  // Now build a tree with both sub-trees
  n = mkastnode(nodetype, left, right, 0);
  return (n);
}
```

请注意，在这个简单的解析器代码中，没有任何处理不同操作符优先级的内容。目前，该代码将所有操作符视为具有相等的优先级。如果您按照代码解析表达式 `2 * 3 + 4 * 5`，您会发现它构建了这棵抽象语法树（AST）:

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

这绝对是不正确的。它将会先计算 `4*5` 得到 20，然后再进行 `3+20` 得到 23，而不是先进行 `2*3` 得到 6。

那我为什么要这样做呢？我想向您展示编写一个简单的解析器很容易，但要让它也进行语义分析则更加困难。

## Interpreting the Tree

现在我们有了（不正确的）抽象语法树，让我们编写一些代码来解释它。同样，我们将编写递归代码来遍历这棵树。以下是伪代码：

```
interpretTree:
  First, interpret the left-hand sub-tree and get its value
  Then, interpret the right-hand sub-tree and get its value
  Perform the operation in the node at the root of our tree
  on the two sub-tree values, and return this value
```

下面是正确的AST：

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

调用的结构会看起来像下面这样：

```
interpretTree0(tree with +):
  Call interpretTree1(left tree with *):
     Call interpretTree2(tree with 2):
       No maths operation, just return 2
     Call interpretTree3(tree with 3):
       No maths operation, just return 3
     Perform 2 * 3, return 6

  Call interpretTree1(right tree with *):
     Call interpretTree2(tree with 4):
       No maths operation, just return 4
     Call interpretTree3(tree with 5):
       No maths operation, just return 5
     Perform 4 * 5, return 20

  Perform 6 + 20, return 26
```

## Code to Interpret the Tree

这段代码应该放在 `interp.c` 文件中，并遵循上面的伪代码：

```c
// Given an AST, interpret the
// operators in it and return
// a final value.
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  // Get the left and right sub-tree values
  if (n->left)
    leftval = interpretAST(n->left);
  if (n->right)
    rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:
      return (leftval + rightval);
    case A_SUBTRACT:
      return (leftval - rightval);
    case A_MULTIPLY:
      return (leftval * rightval);
    case A_DIVIDE:
      return (leftval / rightval);
    case A_INTLIT:
      return (n->intvalue);
    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

再一次，switch 语句中的默认语句在我们无法解释 AST 节点类型时触发。它将成为我们解析器中语义检查的一部分。

## Building the Parser

这里还有一些其他的代码，比如在`main()`函数中调用解释器。

```c
  scan(&Token);                 // Get the first token from the input
  n = binexpr();                // Parse the expression in the file
  printf("%d\n", interpretAST(n));      // Calculate the final result
  exit(0);
```

可以借助下面的命令来build解析器：

```
$ make
cc -o parser -g expr.c interp.c main.c scan.c tree.c
```

我已经为你提供了一些输入文件供你测试解析器，当然你也可以自己创建。请记住，计算结果是不正确的，但解析器应该能检测到例如连续数字、连续运算符以及输入结束时缺少数字等输入错误。我还向解释器中添加了一些调试代码，这样你就可以看到AST树节点的评估顺序：

```
$ cat input01
2 + 3 * 5 - 8 / 3

$ ./parser input01
int 2
int 3
int 5
int 8
int 3
8 / 3
5 - 2
3 * 3
2 + 9
11

$ cat input02
13 -6+  4*
5
       +
08 / 3

$ ./parser input02
int 13
int 6
int 4
int 5
int 8
int 3
8 / 3
5 + 2
4 * 7
6 + 28
13 - 34
-21

$ cat input03
12 34 + -56 * / - - 8 + * 2

$ ./parser input03
unknown token in arithop() on line 1

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./parser input04
Unrecognised character . on line 3

$ cat input05
23 * 456abcdefg

$ ./parser input05
Unrecognised character a on line 1
```

## Conclusion and What's Next

一个解析器能够识别语言的语法，并检查编译器输入是否符合这个语法。如果不符合，解析器应该输出一个错误消息。由于我们的表达式语法是递归的，我们选择编写一个递归下降解析器来识别我们的表达式。

目前，如上面的输出所示，解析器已经工作了，但它未能正确理解输入的语义。换句话说，它没有计算出表达式的正确值。

在我们编写编译器的下一部分中，我们将修改解析器，使其能够对表达式进行语义分析，从而得到正确的数学结果。[下一步](../03_Precedence/Readme.md)
