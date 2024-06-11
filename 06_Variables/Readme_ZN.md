# Part 6: Variables

我刚完成了向编译器中添加全局变量的工作，正如我所预料的，这是一项相当多的工作。此外，编译器中的几乎每个文件都在这个过程中被修改了。所以这部分的旅程将会很长。

## What Do We Want from Variables?

我们希望能够

* 声明变量
* 使用变量来获取存储的值
* 给变量赋值

这里是input02，它将是我们的测试程序：

```
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

最明显的变化是，语法现在有了变量声明、赋值语句和表达式中的变量名。然而，在我们讨论这些之前，让我们看看如何实现变量。

## The Symbol Table

每个编译器都需要一个[符号表](https://en.wikipedia.org/wiki/Symbol_table)。稍后，我们将持有不仅仅是全局变量。但现在，这是表中条目的结构（来自defs.h）：

```c
// Symbol table structure
struct symtable {
  char *name;                   // Name of a symbol
};
```

我们在data.h中有一个符号数组：

```c
#define NSYMBOLS        1024            // Number of symbol table entries
extern_ struct symtable Gsym[NSYMBOLS]; // Global symbol table
static int Globs = 0;                   // Position of next free global symbol slot
```

Globs实际上在sym.c中，这是管理符号表的文件。在这里我们有这些管理功能：

* `int findglob(char *s)`：确定符号s是否在全局符号表中。如果找到，返回其槽位位置，否则返回-1。
* `static int newglob(void)`：获取一个新的全局符号槽位的位置，或者如果我们用完了位置就报错。
* `int addglob(char *name)`：将一个全局符号添加到符号表中。返回符号表中的槽号。

代码相当直接，所以我不会在这里讨论代码。有了这些功能，我们可以查找符号并添加新符号到符号表。

## Scanning and New Tokens

如果你看下示例输入文件，我们需要一些新的标记：

* 'int'，称为T_INT
* '='，称为T_EQUALS
* 标识符名称，称为T_IDENT

将'='添加到 `scan()` 中很容易：

```c
  case '=':
    t->token = T_EQUALS; break;
```

我们可以将'int'关键字添加到 `keyword()` 中：

```c
  case 'i':
    if (!strcmp(s, "int"))
      return (T_INT);
    break;
```

对于标识符，我们已经在使用 `scanident()` 将单词存储到Text变量中。而不是如果一个单词不是关键字就报错，我们可以返回一个T_IDENT标记：

```c
   if (isalpha(c) || '_' == c) {
      // Read in a keyword or identifier
      scanident(c, Text, TEXTLEN);

      // If it's a recognised keyword, return that token
      if (tokentype = keyword(Text)) {
        t->token = tokentype;
        break;
      }
      // Not a recognised keyword, so it must be an identifier
      t->token = T_IDENT;
      break;
    }
```

## The New Grammar

我们准备看看我们输入语言的语法变化。像以前一样，我将用BNF范式定义它：

```
 statements: statement
      |      statement statements
      ;

 statement: 'print' expression ';'
      |     'int'   identifier ';'
      |     identifier '=' expression ';'
      ;

 identifier: T_IDENT
      ;
```

一个标识符作为T_IDENT标记返回，我们已经拥有解析print语句的代码。但是，由于我们现在有三种类型的语句，编写一个函数来处理每一种语句是有意义的。我们在stmt.c中的顶级 `statements()` 函数现在看起来像这样：

```c
// Parse one or more statements
void statements(void) {

  while (1) {
    switch (Token.token) {
    case T_PRINT:
      print_statement();
      break;
    case T_INT:
      var_declaration();
      break;
    case T_IDENT:
      assignment_statement();
      break;
    case T_EOF:
      return;
    default:
      fatald("Syntax error, token", Token.token);
    }
  }
}
```

我已经将旧的print语句代码移动到了 `print_statement()` 中，你可以自己浏览。

## Variable Declarations

让我们看看变量声明。这在一个新的文件decl.c中，因为我们将来会有很多其他类型的声明。

```c
// Parse the declaration of a variable
void var_declaration(void) {

  // Ensure we have an 'int' token followed by an identifier
  // and a semicolon. Text now has the identifier's name.
  // Add it as a known identifier
  match(T_INT, "int");
  ident();
  addglob(Text);
  genglobsym(Text);
  semi();
}
```

`ident()` 和 `semi()` 函数是 `match()` 的包装器：

```c
void semi(void)  { match(T_SEMI, ";"); }
void ident(void) { match(T_IDENT, "identifier"); }
```

回到 `var_declaration()`，一旦我们将标识符扫描到Text缓冲区，我们可以使用 `addglob(Text)` 将其添加到全局符号表中。代码允许变量被多次声明（目前）。

## Assignment Statements

这里是stmt.c中 `assignment_statement()` 的代码：

```c
void assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int id;

  // Ensure we have an identifier
  ident();

  // Check it's been defined then make a leaf node for it
  if ((id = findglob(Text)) == -1) {
    fatals("Undeclared variable", Text);
  }
  right = mkastleaf(A_LVIDENT, id);

  // Ensure we have an equals sign
  match(T_EQUALS, "=");

  // Parse the following expression
  left = binexpr(0);

  // Make an assignment AST tree
  tree = mkastnode(A_ASSIGN, left, right, 0);

  // Generate the assembly code for the assignment
  genAST(tree, -1);
  genfreeregs();

  // Match the following semicolon
  semi();
}
```

我们有两个新的AST节点类型。A_ASSIGN将左子树中的表达式赋值给右子树。右子树将是一个A_LVIDENT节点。

我为什么称这个节点为A_LVIDENT？因为它代表一个左值标识符。那么什么是[lvalue](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue)？

左值是一个与特定位置绑定的值。在这里，它是内存地址，该地址保存变量的值。当我们执行：

```
area = width * height;
```

我们把右侧（即右值）的结果赋值给左侧的变量（即左值）。右值没有绑定到特定位置。在这里，表达式的结果可能在某个任意寄存器中。

还要注意，尽管赋值语句的语法是

```
  identifier '=' expression ';'
```

我们将使表达式成为A_ASSIGN节点的左子树，并将A_LVIDENT的详细信息保存在右子树中。为什么？因为我们需要在将值保存到变量之前先计算表达式。

## Changes to the AST Structure

现在我们需要在A_INTLIT AST节点中存储一个整数值，或者在A_IDENT AST节点中存储符号的详细信息。我在defs.h中的AST结构中添加了一个联合来实现这一点：

```c
// Abstract Syntax Tree structure
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  struct ASTnode *left;         // Left and right child trees
  struct ASTnode *right;
  union {
    int intvalue;               // For A_INTLIT, the integer value
    int id;                     // For A_IDENT, the symbol slot number
  } v;
};
```

## Generating the Assignment Code

让我们现在看看 `genAST()` 在gen.c中的变更

```c
int genAST(struct ASTnode *n, int reg) {
  int leftreg, rightreg;

  // Get the left and right sub-tree values
  if (n->left)
    leftreg = genAST(n->left, -1);
  if (n->right)
    rightreg = genAST(n->right, leftreg);

  switch (n->op) {
  ...
    case A_INTLIT:
    return (cgloadint(n->v.intvalue));
  case A_IDENT:
    return (cgloadglob(Gsym[n->v.id].name));
  case A_LVIDENT:
    return (cgstorglob(reg, Gsym[n->v.id].name));
  case A_ASSIGN:
    // The work has already been done, return the result
    return (rightreg);
  default:
    fatald("Unknown AST operator", n->op);
  }

```

注意我们首先计算左子树的AST，然后得到一个寄存器号，该寄存器包含左子树的值。我们现在将这个寄存器号传递给右子树。我们需要这样做对于A_LVIDENT节点，以便 `cgstorglob()` 函数在cg.c中知道哪个寄存器保存了赋值表达式的右值结果。

所以，考虑这个AST树：

```
           A_ASSIGN
          /        \
     A_INTLIT   A_LVIDENT
        (3)        (5)
```

我们调用 `leftreg = genAST(n->left, -1);` 来评估 A_INTLIT 操作。这将会 `return (cgloadint(n->v.intvalue));`，即用值 3 加载一个寄存器并返回寄存器ID。

然后，我们调用 `rightreg = genAST(n->right, leftreg);` 来评估 A_LVIDENT 操作。这将会 `return (cgstorglob(reg, Gsym[n->v.id].name));`，即将寄存器的值存储到名称在 `Gsym[5]` 中的变量中。

接下来我们转到 A_ASSIGN 情况。好吧，我们所有的工作已经完成了。右值仍在寄存器中，所以让我们把它留在那里并返回它。稍后，我们将能够执行如下表达式：

```
a= b= c = 0;
```

其中赋值不仅是一个语句，也是一个表达式。

## Generating x86-64 Code

你可能已经注意到我将原来的 `cgload()` 函数的名称更改为 `cgloadint()`。这更具体了。现在我们有一个用于从全局变量中加载值的函数（在 `cg.c` 中）：

```c
int cgloadglob(char *identifier) {
  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it
  fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", identifier, reglist[r]);
  return (r);
}
```

同样，我们需要一个函数来将寄存器的值保存到一个变量中：

```c
// Store a register's value into a variable
int cgstorglob(int r, char *identifier) {
  fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], identifier);
  return (r);
}
```

我们还需要一个函数来创建一个新的全局整数变量：

```c
// Generate a global symbol
void cgglobsym(char *sym) {
  fprintf(Outfile, "\t.comm\t%s,8,8\n", sym);
}
```

当然，我们不能让解析器直接访问这段代码。相反，在 `gen.c` 中的通用代码生成器中有一个充当接口的函数：

```c
void genglobsym(char *s) { cgglobsym(s); }
```

## Variables in Expressions

所以现在我们可以给变量赋值了。但是如何将变量的值放入表达式中呢？好吧，我们已经有一个 `primary()` 函数用于获取整数字面值。让我们修改它以加载变量的值：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
  case T_INTLIT:
    // For an INTLIT token, make a leaf AST node for it.
    n = mkastleaf(A_INTLIT, Token.intvalue);
    break;

  case T_IDENT:
    // Check that this identifier exists
    id = findglob(Text);
    if (id == -1)
      fatals("Unknown variable", Text);

    // Make a leaf AST node for it
    n = mkastleaf(A_IDENT, id);
    break;

  default:
    fatald("Syntax error, token", Token.token);
  }

  // Scan in the next token and return the leaf node
  scan(&Token);
  return (n);
}
```

请注意在 T_IDENT 情况下进行的语法检查，以确保在尝试使用它之前已经声明了变量。

还要注意，检索变量值的AST叶节点是A_IDENT节点。保存到变量中的叶节点是A_LVIDENT节点。这就是*rvalues*和*lvalues*之间的区别。

## Trying It Out

我认为关于变量声明的部分就到这里了，让我们试着用 `input02` 文件来测试一下：

```
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

`make test`

```
$ make test
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
               stmt.c sym.c tree.c
...
./comp1 input02
cc -o out out.s
./out
17
```

正如你所看到的，我们计算了 `fred + jim`，即 5 + 12，结果是 17。以下是 `out.s` 中的新汇编行：

```
        .comm   fred,8,8                # Declare fred
        .comm   jim,8,8                 # Declare jim
        ...
        movq    $5, %r8
        movq    %r8, fred(%rip)         # fred = 5
        movq    $12, %r8
        movq    %r8, jim(%rip)          # jim = 12
        movq    fred(%rip), %r8
        movq    jim(%rip), %r9
        addq    %r8, %r9                # fred + jim
```

## Other Changes

我可能还进行了一些其他更改。我唯一记得的主要更改是在 `misc.c` 中创建了一些辅助函数，以便更容易地报告致命错误：

```c
// Print out fatal messages
void fatal(char *s) {
  fprintf(stderr, "%s on line %d\n", s, Line); exit(1);
}

void fatals(char *s1, char *s2) {
  fprintf(stderr, "%s:%s on line %d\n", s1, s2, Line); exit(1);
}

void fatald(char *s, int d) {
  fprintf(stderr, "%s:%d on line %d\n", s, d, Line); exit(1);
}

void fatalc(char *s, int c) {
  fprintf(stderr, "%s:%c on line %d\n", s, c, Line); exit(1);
}
```

## Conclusion and What's Next

这是一项艰巨的工作。我们不得不编写符号表管理的开始部分。我们必须处理两种新的语句类型。我们还必须添加一些新的标记和一些新的AST节点类型。最后，我们还必须添加一些代码来生成正确的x86-64汇编输出。

尝试编写一些示例输入文件，并查看编译器是否能按预期工作，特别是检测语法错误和语义错误（未经声明的变量使用）。

在我们撰写编译器的下一部分旅程中，我们将向我们的语言添加六个比较操作符。这将使我们能够在随后的部分中开始处理控制结构。[Next step](../07_Comparisons/Readme.md)
