# Part 1: Introduction to Lexical Scanning

我们的编译器编写之旅始于一个简单的词法扫描器。 如我在上一部分中提到的，扫描器的工作是识别输入语言中的词法单元或 tokens。

我们将从一种仅具有五个词法单元的语言开始：

* 四个基本数学运算符：`*`、`/`、`+`和`-`
* 由 1 个或多个数字 `0` 到 `9` 组成的十进制整数

我们扫描到的每个标记都将被存储在这个结构体中（来自 `defs.h`）：

```c
// Token structure
struct token {
  int token;
  int intvalue;
};
```

`token` 字段可以是下面的几个值之一（来自 `defs.h`）：

```c
// Tokens
enum {
  T_PLUS, T_MINUS, T_STAR, T_SLASH, T_INTLIT
};
```

当标记是 `T_INTLIT`（即整数文字）时，`intvalue` 字段将保存我们扫描到的整数的值。

## Functions in `scan.c`

`scan.c` 文件包含了我们词法扫描器的函数。我们将逐个字符从输入文件中读取。然而，有时候我们需要“撤销”一个字符，如果我们在输入流中读取得太远。我们还想要追踪当前所在的行号，以便在调试消息中打印行号信息。所有这些功能都由 `next()` 函数完成：

```c
// Get the next character from the input file.
static int next(void) {
  int c;

  if (Putback) {                // Use the character put
    c = Putback;                // back if there is one
    Putback = 0;
    return c;
  }

  c = fgetc(Infile);            // Read from input file
  if ('\n' == c)
    Line++;                     // Increment line count
  return c;
}
```

`Putback` 和 `Line` 变量以及我们的输入文件指针都是在 `data.h` 中定义的：

```c
extern_ int     Line;
extern_ int     Putback;
extern_ FILE    *Infile;
```

所有的 C 文件都将包含这个代码，其中 `extern_` 将被替换为 `extern`。但是 `main.c` 将会移除 `extern_`，因此这些变量将“归属”于 `main.c`。

最后，我们如何将一个字符放回输入流呢？就是这样：

```c
// Put back an unwanted character
static void putback(int c) {
  Putback = c;
}
```

## Ignoring Whitespace

我们需要一个函数，它读取并默默地跳过空白字符，直到遇到非空白字符，然后将其返回。因此有：

```c
// Skip past input that we don't need to deal with, 
// i.e. whitespace, newlines. Return the first
// character we do need to deal with.
static int skip(void) {
  int c;

  c = next();
  while (' ' == c || '\t' == c || '\n' == c || '\r' == c || '\f' == c) {
    c = next();
  }
  return (c);
}
```

## Scanning Tokens: `scan()`

现在我们可以在跳过空白字符的同时读取字符；如果我们读取了多余的字符，我们也可以将其放回。现在我们可以编写我们的第一个词法扫描器了：

```c
// Scan and return the next token found in the input.
// Return 1 if token valid, 0 if no tokens left.
int scan(struct token *t) {
  int c;

  // Skip whitespace
  c = skip();

  // Determine the token based on
  // the input character
  switch (c) {
  case EOF:
    return (0);
  case '+':
    t->token = T_PLUS;
    break;
  case '-':
    t->token = T_MINUS;
    break;
  case '*':
    t->token = T_STAR;
    break;
  case '/':
    t->token = T_SLASH;
    break;
  default:
    // More here soon
  }

  // We found a token
  return (1);
}
```

对于简单的单字符标记，我们只需将每个识别的字符转换成一个标记。你可能会问：为什么不直接将识别的字符放入 `struct token` 中呢？答案是，以后我们需要识别像 `==` 这样的多字符标记和关键字如 `if` 和 `while`。因此，拥有一个标记值的枚举列表将会让工作更加轻松。

## Integer Literal Values

事实上，我们已经不得不面对这种情况，因为我们还需要识别像 `3827` 和 `87731` 这样的整数字面值。下面是在 `switch` 语句中缺失的 `default` 代码：

```c
  default:

    // If it's a digit, scan the
    // literal integer value in
    if (isdigit(c)) {
      t->intvalue = scanint(c);
      t->token = T_INTLIT;
      break;
    }

    printf("Unrecognised character %c on line %d\n", c, Line);
    exit(1);
```

一旦我们遇到一个十进制数字字符，我们将使用这个第一个字符来调用辅助函数 `scanint()`。它将返回扫描到的整数值。为了实现这一点，它必须逐个读取每个字符，检查它是否是合法的数字，并构建最终的数字。以下是代码：

```c
// Scan and return an integer literal
// value from the input file.
static int scanint(int c) {
  int k, val = 0;

  // Convert each character into an int value
  while ((k = chrpos("0123456789", c)) >= 0) {
    val = val * 10 + k;
    c = next();
  }

  // We hit a non-integer character, put it back.
  putback(c);
  return val;
}
```

我们从零开始的 `val` 值。每当我们得到一个在 `0` 到 `9` 范围内的字符时，我们就用 `chrpos()` 将其转换为一个 `int` 值。然后我们将 `val` 扩大 10 倍，并将新数字加上去。

例如，如果我们有字符 `3`、`2`、`8`，我们可以这样做：

* `val = 0 * 10 + 3`，即 3
* `val = 3 * 10 + 2`，即 32
* `val = 32 * 10 + 8`，即 328

最后，你有没有注意到调用了 `putback(c)`？此时我们发现了一个不是十进制数字的字符。我们不能简单地丢弃它，但幸运的是，我们可以把它放回输入流，以便稍后处理。

此时你可能会问：为什么不直接减去 `'0'` 的 ASCII 值就能使 `c` 成为一个整数？答案是，稍后我们将能够通过 `chrpos("0123456789abcdef")` 来转换十六进制数字。

下面是 `chrpos()` 的代码：

```c
// Return the position of character c
// in string s, or -1 if c not found
static int chrpos(char *s, int c) {
  char *p;

  p = strchr(s, c);
  return (p ? p - s : -1);
}
```

以上就暂时是 `scan.c` 中的词法分析器中的代码

## Putting the Scanner to Work

`main.c` 中的代码让上述的词法扫描器开始工作。 `main()` 函数打开一个文件然后扫描其中的标记：

```c
void main(int argc, char *argv[]) {
  ...
  init();
  ...
  Infile = fopen(argv[1], "r");
  ...
  scanfile();
  exit(0);
}
```

`scanfile()` 函数循环，直到获取新的标记并打印出标记的细节：

```c
// List of printable tokens
char *tokstr[] = { "+", "-", "*", "/", "intlit" };

// Loop scanning in all the tokens in the input file.
// Print out details of each token found.
static void scanfile() {
  struct token T;

  while (scan(&T)) {
    printf("Token %s", tokstr[T.token]);
    if (T.token == T_INTLIT)
      printf(", value %d", T.intvalue);
    printf("\n");
  }
}
```

## Some Example Input Files

我提供了一些示例输入文件，这样你可以看到扫描器在每个文件中找到了哪些标记，以及扫描器拒绝的输入文件。

```
$ make
cc -o scanner -g main.c scan.c

$ cat input01
2 + 3 * 5 - 8 / 3

$ ./scanner input01
Token intlit, value 2
Token +
Token intlit, value 3
Token *
Token intlit, value 5
Token -
Token intlit, value 8
Token /
Token intlit, value 3

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./scanner input04
Token intlit, value 23
Token +
Token intlit, value 18
Token -
Token intlit, value 45
Unrecognised character . on line 3
```

## Conclusion and What's Next

我们从简单的开始，已经有了一个可以识别四个主要数学运算符和整数字面值的简单词法扫描器。我们发现需要跳过空白并在读取过多输入时将字符放回。

单字符标记容易扫描，但多字符标记则稍微复杂一些。最终，`scan()` 函数会在一个 `struct token` 变量中返回输入文件的下一个标记：

```c
struct token {
  int token;
  int intvalue;
};
```

在编译器编写之旅的下一部分中，我们将构建一个递归下降解析器来解释输入文件的语法，并计算并打印出每个文件的最终值。[下一步](../02_Parser/Readme.md)
