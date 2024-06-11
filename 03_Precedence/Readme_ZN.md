# Part 3: Operator Precedence

我们在编译器编写之旅的前一部分看到，解析器并不一定强制执行我们语言的语义。它只强制执行语法和文法的结构规则。

我们最终得到的代码计算了像 `2 * 3 + 4 * 5` 这样的表达式的错误值，因为代码创建了一个看起来像这样的抽象语法树（AST）：

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

而不是：


```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

为了解决这个问题，我们必须在我们的解析器中添加代码来执行运算符优先级。至少有两种方法可以做到这一点：

* 在语言的文法中明确运算符优先级
* 用运算符优先级表影响现有的解析器

## Making the Operator Precedence Explicit

这是我们旅程上一部分的文法：


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

注意，四个数学运算符之间没有区别。让我们调整文法，使它们有所区别：


```
expression: additive_expression
    ;

additive_expression:
      multiplicative_expression
    | additive_expression '+' multiplicative_expression
    | additive_expression '-' multiplicative_expression
    ;

multiplicative_expression:
      number
    | number '*' multiplicative_expression
    | number '/' multiplicative_expression
    ;

number:  T_INTLIT
         ;
```

我们现在有两种类型的表达式：加法表达式和乘法表达式。注意，文法现在强制数字只作为乘法表达式的一部分。这强制了 '*' 和 '/' 运算符更紧密地绑定到它们两边的数字，从而具有更高的优先级。

任何加法表达式实际上要么是一个乘法表达式本身，要么是一个加法（即乘法）表达式，后面跟着一个 '+' 或 '-' 运算符，然后是另一个乘法表达式。加法表达式现在的优先级比乘法表达式低得多。

## Doing The Above in the Recursive Descent Parser

我们如何将上述版本的文法实现到我们的递归下降解析器中？我在文件 `expr2.c` 中完成了这项工作，我将在下面介绍代码。

答案是有一个 `multiplicative_expr()` 函数来处理 '*' 和 '/' 运算符，以及一个 `additive_expr()` 函数来处理较低优先级的 '+' 和 '-' 运算符。

两个函数都将读取某物和一个运算符。然后，当有相同优先级的后续运算符时，每个函数将解析更多的输入，并将左右两半部分与第一个运算符结合起来。

然而，`additive_expr()` 将不得不推迟到更高优先级的 `multiplicative_expr()` 函数。以下是如何做到这一点的。

## `additive_expr()`

```c
// Return an AST tree whose root is a '+' or '-' binary operator
struct ASTnode *additive_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the left sub-tree at a higher precedence than us
  left = multiplicative_expr();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // Loop working on token at our level of precedence
  while (1) {
    // Fetch in the next integer literal
    scan(&Token);

    // Get the right sub-tree at a higher precedence than us
    right = multiplicative_expr();

    // Join the two sub-trees with our low-precedence operator
    left = mkastnode(arithop(tokentype), left, right, 0);

    // And get the next token at our precedence
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // Return whatever tree we have created
  return (left);
}
```

一开始，我们立即调用 `multiplicative_expr()`，以防第一个运算符是高优先级的 '*' 或 '/'。那个函数只有在遇到低优先级的 '+' 或 '-' 运算符时才会返回。

因此，当我们进入 `while` 循环时，我们知道我们有一个 '+' 或 '-' 运算符。我们循环直到输入中没有剩余的标记，即当我们遇到 T_EOF 标记时。

在循环内部，我们再次调用 `multiplicative_expr()`，以防任何未来的运算符比我们的优先级更高。同样，当它们不是时，这将返回。

一旦我们有了左子树和右子树，我们可以用我们上次循环中得到的运算符将它们结合起来。这会重复，所以如果我们有表达式 `2 + 4 + 6`，我们最终会得到 AST 树：

``` 
       +
      / \
     +   6
    / \
   2   4
```

但如果 `multiplicative_expr()` 有自己的更高优先级运算符，我们将结合包含多个节点的子树。

## multiplicative_expr()

```c
// Return an AST tree whose root is a '*' or '/' binary operator
struct ASTnode *multiplicative_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // While the token is a '*' or '/'
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    // Fetch in the next integer literal
    scan(&Token);
    right = primary();

    // Join that with the left integer literal
    left = mkastnode(arithop(tokentype), left, right, 0);

    // Update the details of the current token.
    // If no tokens left, return just the left node
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // Return whatever tree we have created
  return (left);
}
```

代码与 `additive_expr()` 类似，除了我们调用 `primary()` 来获取真正的整数字面量！我们也只在我们有高优先级级别的运算符时循环，即 '*' 和 '/' 运算符。一旦我们遇到一个低优先级运算符，我们简单地返回我们到目前为止构建的子树。这将回到 `additive_expr()` 来处理低优先级运算符。

## Drawbacks of the Above

以上使用显式运算符优先级构建递归下降解析器的方法可能会因为需要达到正确优先级级别的所有函数调用而变得效率低下。还需要有处理每个运算符优先级级别的函数，所以我们最终会有很多代码行。

## The Alternative: Pratt Parsing

减少代码量的一种方法是使用 [Pratt 解析器](https://en.wikipedia.org/wiki/Pratt_parser))，它与每个标记相关联的优先级值表，而不是具有在文法中复制显式优先级的函数。

在这一点上，我强烈推荐你阅读 Bob Nystrom 写的 [Pratt Parsers: Expression Parsing Made Easy](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)。Pratt 解析器仍然让我头疼，所以尽可能多地阅读并熟悉基本概念。

## `expr.c`: Pratt Parsing

我在 expr.c 中实现了 Pratt 解析，它是 expr2.c 的直接替代品。让我们开始游览。

首先，我们需要一些代码来确定每个标记的优先级：

```c
// Operator precedence for each token
static int OpPrec[] = { 0, 10, 10, 20, 20,    0 };
//                     EOF  +   -   *   /  INTLIT

// Check that we have a binary operator and
// return its precedence.
static int op_precedence(int tokentype) {
  int prec = OpPrec[tokentype];
  if (prec == 0) {
    fprintf(stderr, "syntax error on line %d, token %d\n", Line, tokentype);
    exit(1);
  }
  return (prec);
}
```

更高的数字（例如 20）意味着比更低的数字（例如 10）具有更高的优先级。

现在，你可能会问：既然有一个叫做 OpPrec[] 的查找表，为什么还要有一个函数呢？答案是：发现语法错误。

考虑一个看起来像 `234 101 + 12` 的输入。我们可以扫描前两个标记。但是，如果我们简单地使用 OpPrec[] 来获取第二个 `101` 标记的优先级，我们就不会注意到它不是一个运算符。因此，`op_precedence()` 函数强制执行正确的语法。

现在，我们没有每个优先级级别的函数，我们有一个单一的表达式函数，它使用运算符优先级表：

```c
// Return an AST tree whose root is a binary operator.
// Parameter ptp is the previous token's precedence.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // While the precedence of this token is
  // more than that of the previous token precedence
  while (op_precedence(tokentype) > ptp) {
    // Fetch in the next integer literal
    scan(&Token);

    // Recursively call binexpr() with the
    // precedence of our token to build a sub-tree
    right = binexpr(OpPrec[tokentype]);

    // Join that sub-tree with ours. Convert the token
    // into an AST operation at the same time.
    left = mkastnode(arithop(tokentype), left, right, 0);

    // Update the details of the current token.
    // If no tokens left, return just the left node
    tokentype = Token.token;
    if (tokentype == T_EOF)
      return (left);
  }

  // Return the tree we have when the precedence
  // is the same or lower
  return (left);
}
```

首先，请注意，这仍然像之前的解析器函数一样是递归的。这一次，在我们被调用之前，我们会接收到找到的标记的优先级级别。`main()`将使用最低优先级 0 调用我们，但我们会以更高的值调用自己。

你也应该注意到，代码与 `multiplicative_expr()` 函数非常相似：读取整数字面值，获取操作符的标记类型，然后循环构建树。

不同之处在于循环的条件和主体：

```c
multiplicative_expr():
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    scan(&Token); right = primary();

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }

binexpr():
  while (op_precedence(tokentype) > ptp) {
    scan(&Token); right = binexpr(OpPrec[tokentype]);

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }
```

在 Pratt 解析器中，当下一个操作符的优先级高于当前标记时，我们不仅仅是用 `primary()` 获取下一个整数字面值，而是使用 `binexpr(OpPrec[tokentype])` 调用自己来提高操作符的优先级。

一旦我们遇到一个与我们的优先级级别相同或更低的标记，我们就会简单地：

```c
  return (left);
```

这将是具有许多节点和在调用我们的运算符之上优先级的子树，或者它可能是一个与我们相同优先级的单个整数文字值，用于表示操作符。

现在我们有一个用于表达式解析的单个函数。它使用一个小的辅助函数来强制执行运算符的优先级，因此实现了我们语言的语义。

## Putting Both Parsers Into Action

你可以编写两个程序，一个使用每个解析器：

```
$ make parser                                        # Pratt Parser
cc -o parser -g expr.c interp.c main.c scan.c tree.c

$ make parser2                                       # Precedence Climbing
cc -o parser2 -g expr2.c interp.c main.c scan.c tree.c
```

你也可以使用我们旅程前面部分的相同输入文件来测试这两个解析器：

```
$ make test
(./parser input01; \
 ./parser input02; \
 ./parser input03; \
 ./parser input04; \
 ./parser input05)
15                                       # input01 result
29                                       # input02 result
syntax error on line 1, token 5          # input03 result
Unrecognised character . on line 3       # input04 result
Unrecognised character a on line 1       # input05 result

$ make test2
(./parser2 input01; \
 ./parser2 input02; \
 ./parser2 input03; \
 ./parser2 input04; \
 ./parser2 input05)
15                                       # input01 result
29                                       # input02 result
syntax error on line 1, token 5          # input03 result
Unrecognised character . on line 3       # input04 result
Unrecognised character a on line 1       # input05 result

```

## Conclusion and What's Next

现在我们有：

* 一个能识别并返回语言中标记的扫描器
* 一个能识别我们语法、报告语法错误并构建抽象语法树的解析器
* 解析器使用的优先级表，实现语言的语义
* 一个解释器，深度优先遍历抽象语法树，并计算输入表达式的结果

我们还没有的是编译器。但我们非常接近完成我们的第一个编译器！

在我们撰写编译器的下一部分旅程中，我们将替换解释器。取而代之的是，我们将编写一个生成针对每个具有数￥￥算符的AST节点的x86-64汇编代码的转换器。我们还将生成一些汇编序言和尾声，以支持生成器输出的汇编代码。[Next step](../04_Assembly/Readme.md)
