---
title: 'NJU ICS2024 PA 作业心得（一）'
date: 2024-09-15
permalink: /posts/2024/09/15/ics-pa1/
tags:
  - ICS
---

# NJU ICS2024 PA 作业心得（一）

> 由于自己并不是NJU 2024的学生，因此“堂而皇之”的把这份心得发在了网上，并且只是仅供**非以此课程作为自己当前学年保研课**的同学参考。如若这是你的目前正在进修的保研课程，请速速关闭此网页！！！

## 序幕（工具的准备）

【该章节只在PA1中出现】由于PA0涉及简单的系统安装与环境配置，因此并不打算单独写一个心得博客。但自己在过程中也使用了一些好用的小工具，在此分享一下。

### **Vscode** but not Vim. 

对不起Vim，我知道你是神，但我真的不会用。Vscode真的太好用了，有GUI，有那么多好用的插件，甚至背景都能自定义，我太爱他（bushi

### clangd

`clangd`是一个开源的语言服务器，可以配合[CompileDB项目](https://github.com/nickdiego/compiledb)生成compile_commands.json（所有符号表索引数据库），方便只参与编译的文件代码进行快速跳转。
具体食用方式：

1. `pip3 install compiledb` 安装compiledb。
2. 在`NEMU_PATH`下运行`compiledb make`，在`NEMU_PATH`下生成`compile_commands.json`文件。
3. 在自己的环境中使用`apt-get install clangd`安装clangd
4. vscode安装clangd插件后，设置`--compile-commands-dir=nemu`。这样会找到你vscode工作目录下`nemu`目录里的`compile_commands.json`

然后就可以快乐的通过这种方式找到一堆之前使用C++插件找不到的宏定义啦

### CodeLLDB

这个项目太需要Debug了，我们使用lldb作为调试工具（当然也可以使用gdb）。

具体食用方式：

1. 在自己的环境中使用`apt-get install lldb`安装lldb

2. vscode安装CodeLLDB插件

3. 在`Run and Debug`功能下修改配置文件`launch.json`，添加如下配置

   ```json
   {
       "configurations": [
           
           {
               "name": "C/C++: gcc build and debug active file",
               "type": "lldb",
               "request": "launch",
               "program": "${workspaceFolder}/nemu/build/riscv32-nemu-interpreter",
               "args": [],
               "cwd": "${workspaceFolder}/nemu",
           },
           {
               "name": "C/C++: gcc build and debug current file",
               "type": "lldb",
               "request": "launch",
               "program": "${fileDirname}/${fileBasenameNoExtension}",
               "args": [],
               "cwd": "${workspaceFolder}/nemu",
               "preLaunchTask": "C/C++: gcc build active file",
           }
       ],
       "version": "2.0.0"
   }
   ```

### GitLens & Git Graph

GitLens可以方便的让你在编辑器中看到上一次commit的提交情况。

Git Graph可以方便的让你看到git log信息，查看方式是在点击`Sys Control`功能后单击最右边的小图标`View git graph`。

### WakaTime

记录你的项目时长，统计通关时间。在注册账户后找到个人token，在vscode通过`ctrl+shift+P`找到`WakaTime: Api Key`，在提示内填入即可。

## RTFSC问题处理

### 宏的工作原理

C语言中的宏（Macro）是预处理指令，由预处理器在实际编译之前处理。宏的工作原理主要包括文本替换、条件编译和文件包含等功能。

对象宏基本上是一个简单的替换。预处理器会在编译代码之前将所有宏名替换为宏定义中的内容。例如：

```c
#define PI 3.14159
```

函数宏允许宏定义带有参数，这使得宏的替换可以根据参数动态变化。函数宏在形式上类似于函数，但在实际处理上仍然是文本替换。例如：

```c
#define SQUARE(x) ((x) * (x))
```

宏还常用于条件编译，根据不同的编译条件包含或排除代码块。例如：

```c
#ifdef DEBUG
    printf("Debug information\n");
#endif
```

### parse_args参数哪里来？

从main的参数直接传递而来，argc是命令行参数的数量，argv是一个常量字符串数组的地址。

### 处理assert报错

```text
[src/monitor/monitor.c:20 welcome] Exercise: Please remove me in the source code and compile NEMU again.
riscv32-nemu-interpreter: src/monitor/monitor.c:21: welcome: Assertion `0' failed.
```

通过报错信息可以定位错误在`src/monitor/monitor.c`文件21行，删去`assert(0)`即可

### cmd_c为什么传入-1

因为此时用户的目的是继续执行程序，执行多少步是未知的，所以期望下程序会执行无限步直至CPU进入停止状态。为了无限步执行，所以这里需要传入一个参数的最大值。由于在NEMU中，参数是uint32_t类型，所以其最大值是全1二进制串，对应为int32的-1。

## 基础设施问题处理

没有什么难度，按照需求实现即可。注意我们在RTFSC部分已经了解到

> 内存通过在`nemu/src/memory/paddr.c`中定义的大数组`pmem`来模拟. 在客户程序运行的过程中, 总是使用`vaddr_read()`和`vaddr_write()` (在`nemu/src/memory/vaddr.c`中定义)来访问模拟的内存. vaddr, paddr分别代表虚拟地址和物理地址。

所以在实现扫描内存时，要用到`vaddr_read`参数，当然为了打印的好看，我们可以设置一下格式化字符串给出列名，并打印制定的长度，详情参考[wiki百科](https://zh.wikipedia.org/wiki/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2)。

## 表达式求值问题处理

### 如何进行词法分析

详情请参考[正则表达式](https://www.runoob.com/regexp/regexp-syntax.html)，构建正确的正则表达式并分别给予对应的type。

*第一个坑点在于，`regex.h`这个C库提供的正则表达式不包含`\d`，`\s`这样的元字符，所以找数字时需要用`[0-9]`等方式代替。*

*第二个坑点在于，由于单token长度最大可以超过32，所以记得处理数字过长导致超过uint32_t表示范围的情况（至少打印一个warning），方便之后找到BUG原因。u_int32最大值约为4e9，即十进制表示10位，十六进制表示8位。*

### 如何实现简单的计算功能

一般我们实现计算功能的方式是通过栈内外优先级的方式，但是讲义通过BNF表达式，给出了一个递归式的求值框架，我们对他进行分析

```c
eval(p, q) {
  if (p > q) {
    /* Bad expression */
  }
  else if (p == q) {
    /* Single token.
     * For now this token should be a number.
     * Return the value of the number.
     */
  }
  else if (check_parentheses(p, q) == true) {
    /* The expression is surrounded by a matched pair of parentheses.
     * If that is the case, just throw away the parentheses.
     */
    return eval(p + 1, q - 1);
  }
  else {
    /* We should do more things here. */
  }
}
```

1. 我们从后向前探索这个框架，先看最后一个else条件框。它代表了一个普通表达式，我们如果想要分治求解，就需要将普通表达式，抽象成`<expr> <operator> <expr>`的形式，将两个`<expr>`递归求值，由于我们知道分治的合并过程需要在`<expr>`计算之后才能进行，因此为了满足原表达式的计算规则，`<operator>`必须是其中**优先级最低**的运算符，这样才能保证在合并过程中该运算符被最后运算。

   在[利用栈的算法](https://blog.csdn.net/qq_43432875/article/details/102515831)中，我们将括号也视为一种运算符，利用栈内外优先级的方法轻松解决了括号的问题，但此时栈内外优先级是相同的，我们如何处理括号呢？在遇到左括号时，要把与之匹配的右括号构成的**合法括号内的表达式看作一个整体**，由于括号内计算优先级最高，所以寻找匹配有括号过程中不要记录该部分的运算符优先级。如若我们无法找到合法括号，那意味着表达式不合法，需要及时中断。

2. 在上一步我们只是将合法括号表达式看作一个整体交由函数递归处理。函数现在仍然无法处理括号，所以我们需要一个检测算法，将最外层可以被去除且不影响表达式内部合法性的括号去除，这是因为`<expr> ::= "(" <expr> ")"`。

3. 如果$p=q$那么此时一定只剩余一个运算数，即`<expr> ::= <number>`，我们根据这个运算数的数据类型将其转化成`word_t`的数字即可。

4. 如果$p<q$，意味着要不第一次传入的表达式index存在问题，要不然在递归过程中第三个if对应的括号内部表达式为空。这种情况我们直接`assert(0)`即可。

第1，3，4步都很好实现，问题转移到了第2步`check_parentheses(p, q)`该如何实现呢？根据讲义，`check_parentheses(p, q)`只有在传入`"(" <expr> ")"`时才返回`true`，那么为了实现这个功能，我们定义

* 返回值`0`代表传入的表达式内含有**不合法的括号序列**
* 返回值`-1`代表传入的表达式内**括号序列合法**，但是**不能删除表达式左右端的运算符**
* 返回值`1`代表传入的表达式内**括号序列合法**，且**表达式左右端的运算符可以删除**（即括号）

它得做到以下几种情况的处理：

1. 检测传入表达式内的括号是否合法，不合法直接返回`0` 。【针对样例`(4 + 3)) * ((2 - 1)`】

2. 检测表达式两断是否有左右括号，没有直接返回`-1`。 【针对样例`4 + 3 * (2 - 1)`】

3. 递归检测去除括号的内部表达式包含的的括号序列是否合法（查看返回值是否为`true`或者`-1`），合法返回`true`，不合法返回`-1`。【针对样例`(4 + 3) * (2 - 1)`】

   *需要说明的是这里虽然内部不合法，但是函数顺序执行，如果了第1步的检测来到这里，就意味着当前表达式符合返回值`-1`的定义，而非返回值`0`的定义。*

4. 由于要有递归终点，当$p<q$时，我们需要返回`-1`。

*注意在这里，我们不检测`<expr>`本身的合法性，他的合法性由`eval`函数检测，例如在$p=q$时对于最后一个剩余字符必须为运算数的类型之一。*

### 如何拓展表达式求值功能

Congratulations！因为到了这里，一般就不会写出啥BUG了。拓展功能设计两个方面：

1. 兼容单目运算符

   为了做到第一点，我们需要在`eval`函数中将单目运算符与双目运算符区别开进行计算即可。但形如`-（单目-取反/双目-减法）`和`*（单目-解引用/双目-乘法）`这样，一个字符对应两种运算符的情况，我们只需要在`eval`函数之前找到连续运算符，将连续运算符段中靠后的那个运算符设置为对应的单目运算符即可。

2. 添加更多运算符，运算数类型

   这个就太简单了，加正则表达式，加优先级，加`eval`中运算符对应的运算方式... 

   *解引用运算直接用`vaddr_read()`函数读取1个uint32_t长度（4个字节）的数据即可。*

除此之外，还是建议在代码中多添加`assert`检查格式是否有问题方便之后遇到问题时DEBUG，例如单目运算符计算时必须保证第一个字符的类型一定是运算符类型。

### 如何编写测试样例生成和测试程序

#### 测试样例生成程序

根据BNF表达式，递归构造字符串，但一个值得我们思考的问题，我们如何给这个构造函数设置一个递归终点？一种解决方法是如果长度超出限制，那就舍弃并重新构造，但这样的方式可能会引发栈帧溢出的错误。另一种解决方式是给构造函数多增加一个输入参数限制最大的生成长度，如果函数当前的最大生成长度低于某一阈值则停止递归调用。

根据合法的表达式生成正确答案的过程实际上是把生成的表达式作为C代码的一部分，让其编译执行后打印出表达式的结果。

值得一提的是根据框架已经给好的C代码可以看出，`eval`函数返回值是int，`expr`函数的返回值才是`uint32_t`。

#### 测试程序

打开文件后，读入每一行的字符串，使用`strtok`分隔`gold_answer`和`expression`，检测`eval(expresssion)=gold_answer`即可。

## 其他问题处理

### 如何添加监视点功能

维护一个[支持增删改查的链表](https://www.cnblogs.com/ranjiewen/p/5428839.html)即可，不过多赘述了。

由于`watchpoint.c`需要提供一些接口（如查看表达式值是否发生变化），且该文件没有对应的头文件，所以记得要在`sdb.c`顶部给出函数声明。

### watchpoint文件为什么要用一些static变量

使用`static`修饰的变量具有**内部链接性**，意味着它只能在定义它的源文件中可见，其他文件无法引用。这样使得监视点池有良好的封装性，不会被其他文件窥探并修改其中的内容。

### GDB会提供哪些信息

GDB（GNU调试器）可以提供多种信息，包括：

1. **程序状态**：当前执行的行、调用栈、寄存器值。
2. **变量信息**：局部变量、全局变量及其值。
3. **内存状态**：内存中的数据及其地址。
4. **源代码**：与当前执行点相关联的源代码行。
5. **程序输出**：输出流的信息，方便调试。
6. **断点管理**：设置、删除断点及其命中情况。

### GDB中把断点设置在指令的非首字节会发生什么

在GDB中，如果把断点设置在指令的非首字节（例如，中间或末尾），会发生以下情况：

1. **无法正常设置断点**：GDB通常会拒绝在非首字节设置断点，因为断点是通过修改指令的机器码来实现的。如果你尝试在非首字节设置断点，GDB会报告错误。
2. **可能导致程序崩溃或异常**：如果指令的非首字节被修改为中断指令，程序在执行时可能会导致不可预测的行为，包括崩溃或错误。
3. **指令分割**：某些指令可能在字节边界上有特定要求（例如，ARM架构的Thumb模式），在不正确的位置设置断点可能导致指令无法正确解析。

断点的工作原理是用一种特殊的指令（通常是中断指令或陷入指令）替换目标地址处的机器码，从而暂停程序执行。由于指令通常由多个字节组成，修改非首字节可能导致指令的不完整性，从而造成程序崩溃或无法执行。

### NEMU和GDB分别是怎样调试程序的

模拟器（如NEMU）用于模拟整个硬件环境，使软件在不同架构上运行，而调试器（如GDB）专注于调试已编译的程序，帮助开发者分析和修复错误。NEMU模拟CPU和系统行为，支持基本调试功能；GDB则提供丰富的调试工具，允许设置断点、单步执行和检查程序状态。

### riscv32有哪几种指令格式

* **R型（R-Type）**：典型指令有算术运算（如`ADD`，`SUB`）、逻辑运算等。
* **I型（I-Type）**：用于立即数操作、加载类指令，典型指令有`ADDI`（立即数加法）、`LOAD`（加载内存到寄存器）。
* **S型（S-Type）**：用于存储类指令，典型指令有`STORE`（存储寄存器到内存）。
* **B型（B-Type）**：用于条件分支跳转类指令，典型指令有`BEQ`（相等条件分支）、`BNE`（不等条件分支）。
* **U型（U-Type）**：用于指令中带有大立即数（高20位）操作，典型指令有`LUI`（加载高立即数）、`AUIPC`（将立即数加到PC）。
* **J型（J-Type）**：用于无条件跳转指令，典型指令有`JAL`（跳转并链接）。

### LUI指令的行为是什么

LUI（Load Upper Immediate）指令将一个20位的立即数加载到目标寄存器的高20位，低12位填充为0。这通常用于构建较大的立即数或地址。

### mstatus寄存器的结构是怎么样的

`mstatus`寄存器的结构包含多个字段，包括：

- **MIE**：机器中断使能位。
- **MPIE**：机器中断使能的先前值。
- **MPRV**：用于指示是否在特权级下访问。
- **XS**：扩展状态位，指示浮点状态。
- **FS**：浮点状态位，指示浮点单元的状态。

### 如何统计代码行数

基本思路：`find`找到所有.c和.h文件，`grep`过滤空行，`wc`统计行数

讲义中有提及到[包含空行的代码统计方式]((https://nju-projectn.github.io/ics-pa-gitbook/ics2024/linux.html#%E7%BB%9F%E8%AE%A1%E4%BB%A3%E7%A0%81%E8%A1%8C%E6%95%B0))，我们需要进行一些更改。

```Makefile
count:
	@echo "line counts = $(shell find $(NEMU_HOME)/ -name "*.c" -o -name "*.h" | xargs grep -v '^\s*$$' | wc -l)"

.PHONY: run gdb run-env clean-tools clean-all $(clean-tools) count
```

### gcc中的Wall和Werror什么作用？

-Wall

- **作用**：启用大多数的警告信息，这些警告通常表示潜在的代码问题。使用`-Wall`可以帮助开发者发现代码中的错误和不规范的地方。
- **使用原因**：通过开启更多的警告，开发者可以及时识别和修复可能导致未定义行为或其他问题的代码，从而提高代码质量和可靠性。

-Werror

- **作用**：将所有警告视为错误。如果编译过程中出现任何警告，编译器将停止编译，并返回错误状态。
- **使用原因**：使用`-Werror`可以强制开发者关注代码中的警告，因为它们被视为阻碍编译的错误。这有助于确保代码在发布前是干净的，没有潜在问题。

## 源码

```c
/* sdb.c 粘一些代表性的，不全粘贴 */ 
static int cmd_x(char* args) {
  /* extract the first argument */
  char* arg = strtok(NULL, " ");

  word_t N;
  vaddr_t EXPR;

  if (arg == NULL) {
    /* no argument is given */
    printf("ERROR: x need two argument: N implies length, EXPR implies expression");
  }
  else {
    /* read the first argument */
    sscanf(arg, "%u", &N);
          
    arg = strtok(NULL, " ");

    if (arg == NULL) {
      /* only one argument is given */
      printf("ERROR: x need two argument: N implies length, EXPR implies expression");
    }
    else {
      bool success;

      EXPR = (vaddr_t)expr(arg, &success);

      if (!success) {
        printf("EXPR is invalid");
        assert(0);
      }
      else {
        int i;
        printf("%-10s\t%-10s\t%s\n", "VADDR", "HEX_VALUE", "DEC_VALUE");
        for (i = 0; i < N; ++i) {
          word_t value = vaddr_read(EXPR, sizeof(word_t));
          printf("%#010x\t%#010x\t%u\n", EXPR, value, value);
          EXPR += sizeof(word_t);
        }
      }
    }
  }

  return 0;
}
```

---



```c
/* expr.c 粘一些代表性的，不全粘贴 */ 
static int check_parentheses(int p, int q) {
  // check if the expression is surrounded by a matched pair of parentheses
  if (p > q)
    return -1;
  int cnt = 0, i;
  for (i = p; i <= q; ++i) {
    if (tokens[i].type == '(')
      ++cnt;
    else if (tokens[i].type == ')')
      --cnt;
    if (cnt < 0)
      return 0;
  }
  if (cnt)
    return 0;
  if (tokens[p].type != '(' || tokens[q].type != ')')
    return -1;
  int result = check_parentheses(p + 1, q - 1);
  if (result) /* value is -1 or 1 */
    return 1;
  else
    return -1;
}

static int get_op_level(int op_type) {
  switch (op_type) {
  case TK_OR:
    return 1;
  case TK_AND:
    return 2;
  case TK_EQ: case TK_NEQ:
    return 3;
  case '+': case '-':
    return 4;
  case '*': case '/':
    return 5;
  case TK_NEG: case TK_DEREF: case TK_NOT:
    return 6;
  default:
    printf("ERROR: invalid operator %d (enum index)\n", op_type);
    assert(0);
  }
}

static void find_unary_op() {
  int i;
  for (i = 0; i < nr_token; ++i) {
    if (tokens[i].type == '-' && (i == 0 || is_bin_op(tokens[i - 1].type))) {
      tokens[i].type = TK_NEG;
    }
    else if (tokens[i].type == '*' && (i == 0 || is_bin_op(tokens[i - 1].type))) {
      tokens[i].type = TK_DEREF;
    }
    else if (tokens[i].type == '!' && (i == 0 || is_bin_op(tokens[i - 1].type))) {
      tokens[i].type = TK_NOT;
    }
  }
}

static int eval(int p, int q) {
  if (p > q) {
    /* Bad expression */
    printf("ERROR: bad expression\n");
    assert(0);
  }
  else if (p == q) {
    /* Single token.
     * For now this token should be a number.
     * Return the value of the number.
     */
    long long num;
    if (tokens[p].type == TK_HEX)
      sscanf(tokens[p].str, "%llx", &num);
    else if (tokens[p].type == TK_DEC)
      sscanf(tokens[p].str, "%lld", &num);
    else if (tokens[p].type == TK_REG) {
      bool success = true;
      int result = isa_reg_str2val(tokens[p].str, &success);
      if (!success) {
        printf("ERROR: invalid register %s\n", tokens[p].str);
        assert(0);
      }
      return result;
    }
    else {
      printf("ERROR: invalid token %d (enum index)\n", tokens[p].type);
      assert(0);
    }
    if (num > 0x100000000) {
      printf("WARNING: the input number [%s] is too large", tokens[p].str);
      assert(0);
    }
    return (word_t)num;
  }
  else if (check_parentheses(p, q) == true) {
    /* The expression is surrounded by a matched pair of parentheses.
     * If that is the case, just throw away the parentheses.
     */
    return eval(p + 1, q - 1);
  } else {
    int i, current_priority, op_pos = -1, priority = 0x3f3f3f3f;
    int val1, val2;

    /* Find the position of dominant operator */
    for (i = p; i <= q; ++i) {
      if (tokens[i].type == '(') {
        int cnt = 1;
        while (cnt && i < q) {
          ++i;
          if (tokens[i].type == '(')
            ++cnt;
          else if (tokens[i].type == ')')
            --cnt;
        }
        if (cnt) {
          printf("ERROR: bad expression %d-%d\n", p, q);
          assert(0);
        }
      }else if(is_bin_op(tokens[i].type) || is_unary_op(tokens[i].type)) {
        current_priority = get_op_level(tokens[i].type);
        if (current_priority <= priority) {
          priority = current_priority;
          op_pos = i;
        }
      }
    }

    if (is_unary_op(tokens[op_pos].type)) {
      /* unary operator */
      assert(op_pos == p);
     switch (tokens[op_pos].type) {
     case TK_DEREF:
        return vaddr_read(eval(op_pos + 1, q), 4);
      case TK_NEG:
        return -eval(op_pos + 1, q);
      case TK_NOT:
        return !eval(op_pos + 1, q);
      default:
        printf("ERROR: invalid operator %d (enum index)\n", tokens[op_pos].type);
        assert(0); // impossible
     } 
    }

    val1 = eval(p, op_pos - 1);
    val2 = eval(op_pos + 1, q);

    switch (tokens[op_pos].type) {
      /* TODO: Possible arithmetic overflows are not handled */
    case '+':
      return val1 + val2;
    case '-':
      return val1 - val2;
    case '*':
      return val1 * val2;
    case '/':
      if (!val2)
        printf("ERROR: division by zero\n"), assert(0);
      return val1 / val2;
    case TK_EQ:
      return val1 == val2;
    case TK_NEQ:
      return val1 != val2;
    case TK_AND:
      return val1 && val2;
    case TK_OR:
      return val1 || val2;
    default: assert(0);
    }
  }
}
```

---

```c
/* expr.c 粘一些代表性的，不全粘贴 */ 
WP *new_wp() {
  if (free_ == NULL) {
    printf("No enough watchpoints.\n");
    assert(0);
    return NULL;
  }
  WP *wp = free_;
  free_ = free_->next;
  wp->next = head;
  head = wp;
  return wp;
}

void free_wp(WP *wp) {
  if (wp == NULL) {
    return;
  }
  WP *p = head;
  if (p == wp) {
    head = head->next;
    wp->next = free_;
    free_ = wp;
    return;
  }
  while (p->next != NULL) {
    if (p->next == wp) {
      p->next = wp->next;
      wp->next = free_;
      free_ = wp;
      return;
    }
    p = p->next;
  }
}
```
