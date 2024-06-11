# Part 5: Statements

是时候为我们语言的语法添加一些“正式”的语句了。 我想能够编写像这样的代码行：

```
print 2 + 3 * 5;
print 18 - 6/3 + 4*2;
```

当然，由于我们忽略了空格，没有必要要求一个语句的所有标记都在同一条线上。每个语句都以关键字 `print` 开始并以分号结束。所以这些将变成我们语言中的新标记。

## BNF Description of the Grammar

我们已经看到了表达式的BNF表示法。现在让我们定义上述类型的语句的BNF语法：

```
statements: statement
     | statement statements
     ;

statement: 'print' expression ';'
     ;
```

输入文件由几个语句组成。它们要么是一个语句，要么是一个语句后面跟着更多的语句。每个语句都以关键字print开始，然后是一个表达式，然后是一个分号。

## Changes to the Lexical Scanner

在我们能够解析上述语法之前，我们需要向现有代码中添加一些更多的内容。让我们从词法扫描器开始。

添加分号的标记将会很容易。现在，是print关键字。

稍后，我们将在语言中有大量的关键字，加上我们的变量标识符，所以我们需要添加一些代码来帮助我们处理它们。

在 `scan.c` 中，我添加了这段代码，我从SubC编译器借来的。它将字母数字字符读入一个缓冲区，直到遇到一个非字母数字字符。

```c
// Scan an identifier from the input file and
// store it in buf[]. Return the identifier's length
static int scanident(int c, char *buf, int lim) {
  int i = 0;

  // Allow digits, alpha and underscores
  while (isalpha(c) || isdigit(c) || '_' == c) {
    // Error if we hit the identifier length limit,
    // else append to buf[] and get next character
    if (lim - 1 == i) {
      printf("identifier too long on line %d\n", Line);
      exit(1);
    } else if (i < lim - 1) {
      buf[i++] = c;
    }
    c = next();
  }
  // We hit a non-valid character, put it back.
  // NUL-terminate the buf[] and return the length
  putback(c);
  buf[i] = '\0';
  return (i);
}
```

我们还需要一个函数来识别语言中的关键字。一种方法是有一个关键字列表，然后遍历列表并使用 `strcmp()` 将每个关键字与 `scanident()` 的缓冲区进行比较。SubC的代码有一个优化：在进行 `strcmp()` 之前先匹配第一个字母。这加快了与几十个关键字的比较。现在我们不需要这个优化，但我为了以后已经把它放进去了：

```c
// Given a word from the input, return the matching
// keyword token number or 0 if it's not a keyword.
// Switch on the first letter so that we don't have
// to waste time strcmp()ing against all the keywords.
static int keyword(char *s) {
  switch (*s) {
    case 'p':
      if (!strcmp(s, "print"))
        return (T_PRINT);
      break;
  }
  return (0);
}
```

现在，在 `scan()` 的switch语句的底部，我们添加了这段代码来识别分号和关键字：

```c
    case ';':
      t->token = T_SEMI;
      break;
    default:

      // If it's a digit, scan the
      // literal integer value in
      if (isdigit(c)) {
        t->intvalue = scanint(c);
        t->token = T_INTLIT;
        break;
      } else if (isalpha(c) || '_' == c) {
        // Read in a keyword or identifier
        scanident(c, Text, TEXTLEN);

        // If it's a recognised keyword, return that token
        if (tokentype = keyword(Text)) {
          t->token = tokentype;
          break;
        }
        // Not a recognised keyword, so an error for now
        printf("Unrecognised symbol %s on line %d\n", Text, Line);
        exit(1);
      }
      // The character isn't part of any recognised token, error
      printf("Unrecognised character %c on line %d\n", c, Line);
      exit(1);
```

我还添加了一个全局文本缓冲区来存储关键字和标识符：

```c
#define TEXTLEN         512             // Length of symbols in input
extern_ char Text[TEXTLEN + 1];         // Last identifier scanned
```

## Changes to the Expression Parser

到目前为止，我们的输入文件只包含一个表达式；因此，在我们的Pratt解析器代码中的 `binexpr()`（在 `expr.c` 中），我们有这段代码来退出解析器：

```c
  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);
```

有了我们的新语法，每个表达式都以分号结束。因此，我们需要更改表达式解析器中的代码，以便发现 `T_SEMI` 标记并退出表达式解析：

```c
// Return an AST tree whose root is a binary operator.
// Parameter ptp is the previous token's precedence.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If we hit a semicolon, return just the left node
  tokentype = Token.token;
  if (tokentype == T_SEMI)
    return (left);

    while (op_precedence(tokentype) > ptp) {
      ...

          // Update the details of the current token.
    // If we hit a semicolon, return just the left node
    tokentype = Token.token;
    if (tokentype == T_SEMI)
      return (left);
    }
}
```

## Changes to the Code Generator

我想保持通用代码生成器在gen.c中与CPU特定代码在cg.c中分开。这也意味着编译器的其他部分应该只调用gen.c中的函数，只有gen.c应该调用cg.c中的代码。

为此，我在gen.c中定义了一些新的“前端”函数：

```c
void genpreamble()        { cgpreamble(); }
void genpostamble()       { cgpostamble(); }
void genfreeregs()        { freeall_registers(); }
void genprintint(int reg) { cgprintint(reg); }
```

## Adding the Parser for Statements

我们有一个新文件stmt.c。这将包含我们语言中所有主要语句的解析代码。现在，我们需要解析我上面给出的BNF语法的语句。这是通过这个单一函数完成的。我已经将递归定义转换为循环：

```c
// Parse one or more statements
void statements(void) {
  struct ASTnode *tree;
  int reg;

  while (1) {
    // Match a 'print' as the first token
    match(T_PRINT, "print");

    // Parse the following expression and
    // generate the assembly code
    tree = binexpr(0);
    reg = genAST(tree);
    genprintint(reg);
    genfreeregs();

    // Match the following semicolon
    // and stop if we are at EOF
    semi();
    if (Token.token == T_EOF)
      return;
  }
}
```

在每次循环中，代码找到一个T_PRINT标记。然后它调用binexpr()来解析表达式。最后，它找到T_SEMI标记。如果跟随T_EOF标记，我们跳出循环。

在每个表达式树之后，调用gen.c中的代码将树转换为汇编代码，并调用汇编printint()函数来打印最终值。

## Some Helper Functions

在上面的代码中，有一些新的辅助函数，我已经将它们放入一个新的文件misc.c中：

```c
// Ensure that the current token is t,
// and fetch the next token. Otherwise
// throw an error 
void match(int t, char *what) {
  if (Token.token == t) {
    scan(&Token);
  } else {
    printf("%s expected on line %d\n", what, Line);
    exit(1);
  }
}

// Match a semicon and fetch the next token
void semi(void) {
  match(T_SEMI, ";");
}
```

这些是解析器中语法检查的一部分。稍后，我将添加更多的短函数来调用 `match()`，使我们的语法检查更容易。

## Changes to `main()`

`main()` 过去直接调用 `binexpr()` 来解析旧输入文件中的单个表达式。现在它这样做：

```c
scan(&Token);                 // Get the first token from the input
genpreamble();                // Output the preamble
statements();                 // Parse the statements in the input
genpostamble();               // Output the postamble
fclose(Outfile);              // Close the output file and exit
exit(0);
```

## Trying It Out

这就是新代码和更改的代码。让我们给新代码一个尝试。这是新的输入文件，input01：

```
print 12 * 3;
print 
   18 - 2
      * 4; print
1 + 2 +
  9 - 5/2 + 3*5;
```

是的，我决定检查我们是否在多行上分散了标记。要编译并运行输入文件，请执行 `make test`：

```make
$ make test
cc -o comp1 -g cg.c expr.c gen.c main.c misc.c scan.c stmt.c tree.c
./comp1 input01
cc -o out out.s
./out
36
10
25
```

And it works!

## Conclusion and What's Next

我们已经为我们的语言添加了第一个“真正的”语句语法。我已经用BNF符号定义了它，但使用循环而不是递归来实现它更容易。别担心，我们很快就会回到递归解析。

在此过程中，我们不得不修改扫描器，添加对关键字和标识符的支持，并更清晰地分离通用代码生成器和CPU特定生成器。

在我们编译器编写之旅的下一部分，我们将向语言添加变量。这将需要大量的工作。[Next step](../06_Variables/Readme.md)
