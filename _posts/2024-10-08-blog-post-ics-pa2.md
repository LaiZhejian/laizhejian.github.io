---
title: 'NJU ICS2024 PA 作业心得（二）'
date: 2024-09-15
permalink: /posts/2024/10/08/ics-pa2/
tags:
  - ICS
---

# NJU ICS2024 PA 作业心得（二）

## RTFSC问题处理

这部分一定要很**仔细很仔细**的阅读RISCV的手册，否则后边会吃苦头的。

我们这里从框架的角度对取指之前和执行之后的操作进行一些简要的补充介绍，方便读者结合帮助文档了解NEMU的整体流程。

* 取指之前

  1. risv32中每条指令的长度为4个字节，这些指令以二进制文件读写的形式被堆排到了bin文件，nemu中定义了位置参数获得需要的bin文件路径。

  2. 在交互控制台使用`si`或者`c`执行`$NEMU_HOME/src/cpu/cpu-exec.c`文件下的`cpu_exec`函数，该函数调用同文件下的`execute`函数执行若干次指令后。

  3. `execute`函数中创建了一个结构体`Decode`，`pc`指代要被译码的指令地址，`snpc`指代`pc`之后的下一个地址，`dnpc`只带指令执行后，`pc`该前往的地址，`inst`只带了指令的二进制代码，`logbuf`则是用来存指令的反汇编结果。

     然后通过调用若干次`exec_once`函数执行顺序若干次指令。

  4. `exec_once`函数维护了结构体`Decode`的内容，调用了`isa_exec_once`以此进行取指，译码、执行

* 执行（更新`pc`后）之后
  1. 在`exec_once`函数中记录指令信息和反汇编结果到`Decode.logbuf`中。
  2. 在`execute`函数中调用`trace_and_difftest`函数检查NEMU中我们的译码行为，并打印追踪日志（所有的XTRACE应该都在这里记录结果）。每次调用`exec_once`函数会检查NEMU的状态（出现异常则停止执行指令）
  3. 在`cpu_exec`函数中打印执行结果。

### riscv32由于指令字长限制应该如何载入32位常数

两者都使用分段加载的方式，先加载高位，再使用立即数指令将低位部分合并到寄存器中，以此完成32位常数的加载操作。这种方法有效利用了32位指令的有限空间，避免了直接在一条指令中编码整个32位常数的限制，下面我们给出RISC-V32的解决方案。

1. **`lui`（Load Upper Immediate）指令**：
   - 使用`lui`指令将一个32位常数的高20位加载到寄存器的高位部分。例如，`lui t0, 0x12345`会将`0x12345000`加载到寄存器`t0`中。
2. **`addi`（Add Immediate）指令**：
   - 使用`addi`指令加载低12位的立即数，并将其与`lui`加载的高位相加。例如，`addi t0, t0, 0x678`会将寄存器`t0`中的值更新为`0x12345678`。

### riscv32指令如何实现

本部分是这一小节的重点，有许多容易踩的坑（需要你仔细研读官方文档），

* **一定要注意运算数是有符号还是无符号！**
* **一定要按照文档描述按照制定的运算数类型（64位or32位）进行运算！**
* **一定要按照文档描述对异常情况进行实现！**

在这一部分为了能通过所有指令测试集（如果指令测试集通过，证明某条指令实现应该没有问题了，之后如果遇到BUG，请对自己实现的指令有充足的信心，不用溯源到此处），我们需要总共实现6中类型的指令R、I、S、B、U、J，需要在`decode_operand`添加`TYPE_*`以及他们立即数获取方式（参照下图）。

![immediate number](/images/image-2024-10-08-blog-post-ics-pa2-1.png)

为了获取立即数，NEMU框架给我们提供了一些很棒的宏定义：

* `BITS(x, hi, lo)`获取二进制数$x$第$[lo,hi]$的比特内容
* `SEXT(x, len)`有符号拓展$x$到二进制下$len$位。这里的实现用到了位域，巧妙到通过限制有符号int的长度接受到$x$的符号位信息，从而再转化成无符号类型实现有符号位拓展。

接着的实现就是根据报错信息找到缺失的指令，根据`opcode`,`funct*`锁定具体指令实现即可。

## 程序, 运行时环境与AM问题处理

### 如何通过批处理模式运行NEMU

因为AM文件夹中Makefile肯定通过了某种方式编译了nemu，并传入了参数，我们需要找到这一步，然后想办法将批处理参数`-b`传入。

通过阅读`$AM_HOME/Makefile line 97`的内容`-include $(AM_HOME)/scripts/$(ARCH).mk`可以发现这里引入了`$(AM_HOME)/scripts/$(ARCH).mk`，由于我们的`ARCH=riscv32-nemu`，于是再查看`$(AM_HOME)/scripts/riscv32-nemu.mk`发现他引用了`$AM_HOME/scripts/platform/nemu.mk`。

在这个文件的第31行`$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) run ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin`发现它传给nemu的参数是`NEMUFLAGS`这个变量。细心的你会发现，在我们的测试文件的Makefile中，我们会把命令行中的`ARGS`参数传给Makefile`ARGS`变量，我们需要将这个变量传给Makefile中的`NEMUFLAGS`才对。

但很快你会发现如果改了这一步，会导致NEMU无法输出`nemu.log`了，于是我们尝试去寻找`-l xxx/nemu.log`参数定义的位置，发现他在`$AM_HOME/scripts/platform/nemu.mk`中，但是这个变量在声明的时候**没有添加`override`关键字**。这就会导致+=运算无法对命令行传来的变量生效，因为命令行同名变量会覆盖Makefile文件中定义的同名变量，需要加`override`专门在针对命令行同名变量进行运算。

### 如何实现sprintf 

* `sprintf`：将可变参数列表获取后提供给`vsprintf`，将结果输出到一个字符串即可。

* `printf`：将可变参数列表获取后提供给`vsprintf`，将结果输出到一个字符串，然后再利用`putch`住个字符输出即可

所以如果要实现`sprintf`最关键的步骤是实现`vsprintf`

`vsprintf` 是一个格式化输出函数，用于将格式化的数据输出到字符串中。该函数的设计思想基于处理格式化字符串中的不同占位符，并根据可变参数的类型将其转换为相应的字符串形式，最后将结果拼接到输出缓冲区 `out` 中。以下是该函数的设计思路：

####  1. 输入参数

- `char *out`：输出缓冲区指针，用于存储格式化后的字符串。
- `const char *fmt`：格式化字符串，包含普通字符和格式化占位符（例如 `%d`，`%s` 等）。
- `va_list ap`：可变参数列表，包含需要格式化输出的数据。

#### 2. 遍历格式化字符串

使用 `while (*fmt != '\0')` 遍历格式化字符串 `fmt`，逐个字符处理。处理分为两种情况：

1. 如果当前字符不是 `%`，将其作为普通字符直接拷贝到输出缓冲区中，并更新缓冲区指针和总输出长度。
2. 如果当前字符是 `%`，则需要进入格式化处理逻辑。

#### 3. 格式化占位符解析

当遇到 `%` 时，函数进入解析模式，依次解析各类格式化修饰符：

1. **标志位解析**：
   - 通过 `switch` 语句处理常见的标志位，如 `-`（左对齐）、`+`（显示正负号）、`0`（填充零）、`#`（特殊格式）等。每个标志位被映射为一个二进制标志位，并存储在 `flags` 变量中。
2. **宽度解析**：
   - 如果格式说明符中指定了宽度（例如 `%10d` 表示输出宽度至少为 10），该宽度要么是直接给出的数字，要么是通过 `*` 从 `va_list` 参数中动态获取。
3. **精度解析**：
   - 如果存在精度修饰符（例如 `%.2f` 表示小数点后保留 2 位），解析精度值。同样，精度值可以是直接指定的数字或通过 `*` 从 `va_list` 参数中获取。
4. **类型说明符解析**：
   - 格式说明符的最后一个字符为类型说明符（如 `d`, `s`, `x` 等），用于确定当前参数的数据类型。根据类型说明符的不同，调用不同的处理函数来进行参数格式化。

#### 4. 不同类型的格式化处理

- **整数类型** (`d`, `i`, `u`, `x`, `o`)：调用 `int_to_str` 函数，将整数值按照不同的进制转换为字符串（支持十进制、八进制、十六进制等），并根据指定的宽度和精度进行格式化输出。

- **字符类型** (`c`)：直接将 `char` 类型的参数输出到缓冲区。

- **字符串类型** (`s`)：将字符串按照指定的宽度和精度进行处理，支持左对齐或右对齐。

- **指针类型** (`p`)：将指针地址转换为十六进制格式输出，并加上 `0x` 前缀。

- **特殊类型**：

  * `%`：直接输出 `%` 符号。

  - `n`：将当前输出字符的长度存储到相应的指针位置。

#### 5. 输出结果拼接

在处理每个占位符后，格式化后的结果通过指针操作被拼接到输出缓冲区 `out` 中，并更新当前缓冲区指针 `buf` 以及总长度 `total_len`。

#### 6. 返回结果

当格式化字符串处理完毕后，在输出缓冲区末尾添加字符串终止符 `\0`，并返回输出缓冲区的长度。

### 如何实现string库

使用`man 3 <function_name>`仔细根据手册实现即可，否则可能存在问题，这里也有一点容易踩的坑，

* `strnpy`如果$n$超出了$det$的长度，就不用添加字符串终止符 `\0`了
* `strcmp`要注意字符串中的字符一般为无符号char，在这里如果要对读入的字符进行无符号转换，否则对于在ASCII码在128及以后的字符的大小比较会出现问题。

### 如何编写klib库的测试程序

尽量用多层for循环嵌套，枚举给定两字符串的起始位置和结束为止，以及操作长度$n$（如果有的话），这样一般是可以覆盖所有情况的（包括前面坑点提到的边界情况）。

### itrace的环形缓冲区如何实现

我的实现是新建了`$NEMU_HOME/src/cpu/iringbuf.c`文件，在里面维护了一个环形队列，对外暴露的接口是一个函数`void record_inst(uint64_t pc, const char *asm_code)`传入了当前记录指令的pc指针和反汇编字符串，以及`void info_inst_records()`函数打印当前队列的信息。

在`$NEMU_HOME/src/cpu/cpu-exec.c`的`trace_and_difftest`函数中，将原本打印指令的位置更改成将指令记录到缓冲区。在同文件下的`assert_fail_msg`函数中，在NEMU框架`ASSERT`中断时输出调用`info_inst_records`。

### mtrace如何实现

在`$NEMU_HOME/src/memory/paddr.c`中`paddr_read`和`paddr_write`函数成功调用后记录信息即可。

### ftrace如何实现

首先我们需要了解elf文件的结构，这里就需要我们首先使用`riscv64-linux-gnu-readelf -a`获取elf文件的信息，再通过`hd`命令验证我们获取的elf文件信息，我们就可以很清晰的了解elf文件的架构了，下面我们进行一个简单的介绍。

#### ELF文件结构的整体组成

ELF文件由三个主要部分组成：

1. **ELF Header**（ELF头）：描述整个文件的组织结构，包括文件类型、目标架构、入口点等基本信息。

   ELF头位于文件的最前面，包含整个ELF文件的元数据信息。其结构固定，大小通常为64字节（32位格式为52字节），主要包含以下字段：

   - **魔数（Magic Number）**：标识这是一个ELF文件，前4个字节为固定值 `0x7F`、`E`、`L`、`F`。

   - Class：文件的架构类型，32位（`ELFCLASS32`）或64位（`ELFCLASS64`）。

   - Data：数据编码，表示字节序的排列方式（小端`LSB`或大端`MSB`）。

   - Version：文件格式的版本，通常为`1`。

   - OS ABI：目标操作系统接口的ABI（应用程序二进制接口）。

   - 文件类型：指定这是可执行文件（`ET_EXEC`）、可重定位文件（`ET_REL`）或共享库（`ET_DYN`）。

   - 目标架构：如x86（`EM_386`）或ARM（`EM_ARM`）等。

   - 入口点地址：程序开始执行的虚拟地址。

   - Program Header表偏移量：指向程序头表的偏移量。

   - **Section Header表偏移量**：指向节头表的偏移量。

   - 标志：与处理器相关的标志。

   - 头部的大小：ELF头的大小。

   - Program Header表的条目大小及数量。

   - **Section Header表的条目大小及数量。**
   - **Section header string表在节头表中的Index**

2. **Program Header Table**（程序头表）：描述如何将文件映射到内存中的不同段，用于执行时加载可执行文件。

   程序头表描述了如何将文件中的数据段映射到内存，以便程序执行。每个程序头条目描述了一个内存段的属性。程序头表在执行时用于决定哪些部分加载到内存中。

3. **Section Header Table**（节头表）：描述文件中的各个节（Section）的属性，用于链接和重定位。

   节头表的每个条目（节头）用于描述文件中的各个节（section），这些节在链接和重定位阶段起作用。每个节都有自己特定的作用，如存储代码、数据、符号表、重定位信息等。每个节头条目通常包含以下信息：

   - **Section Name（节名称）：通过节名称表（Section header string）来索引。**

   - Section Type（节类型）：描述该节的内容，如可重定位数据（`SHT_RELA`）、符号表（`SHT_SYMTAB`）、字符串表（`SHT_STRTAB`）等。

   - Section Flags（节标志）：描述节的属性，如可写、可执行、分配到内存等。

   - Section Address（节地址）：如果该节被加载到内存中，它的虚拟地址。

   - **Section Offset（文件偏移）：节在文件中的偏移量。**

   - **Section Size（节大小）：节在文件中的大小。**

   - Link：与其他节的关联，如符号表与字符串表的关联。

   - Info：附加信息，不同类型的节该字段含义不同。

   - Address Alignment（对齐）：节在文件中或内存中的对齐要求。

   - **Entry Size（条目大小）：如果节中包含固定大小的条目（如符号表），该字段指定条目大小。**

   典型的ELF文件会包含以下节：

   - .text：代码段。

   - .data：已初始化的数据。

   - .bss：未初始化的数据。

   - **.symtab：符号表，包含所有符号及其地址信息。**

   - **.strtab：字符串表，存储符号名称和节名称。**

   - .rel.text 或 .rela.text：重定位表，针对代码段的重定位信息。

我们首先要明确以下信息：

* 我们通过符号表获取函数的地址以及函数的名称在字符串表中的偏移量，通过字符串表译码偏移量获取函数名称。
* 节名称表，符号表和字符串表，虽然中文叫表，但实际上他们是一个节（section）。
* 节名称表，符号表和字符串表的节头信息存储在节头表中。节头表是一个数组，这个数组在内存中连续，具体位置（偏移量信息）和大小由ELF头给出。
* ELF头信息位于文件最开头，偏移量为0。
* 我们需要的信息已经在上述的结构组成中**黑体加粗**了。

#### 获取函数名与函数起始地址的映射的流程

1. 打开并读取ELF文件

   通过 `fopen` 打开传入的ELF文件。如果文件无法打开，输出错误信息并终止程序。

   使用 `fread` 读取ELF头部（`Elf32_Ehdr`），以获取文件的基本信息，如节头表的偏移和节的数量等。

2. 验证文件是否是ELF格式

   检查ELF头部的魔数字段（`e_ident`），确保它是一个合法的ELF文件。如果魔数不匹配，输出错误信息并终止程序。

3. 读取节头表（Section Headers）

   根据ELF头部中的节头表偏移量，节的数量和每一个节头表的大小，分配足够的内存来存储节头表（`Elf32_Shdr`），并使用 `fseek` 和 `fread` 从文件中读取节头表。

4. 读取节名称表

   节名称表是一个特殊的section，用于存储所有节的名称。在节头表中，通过索引找到节名称表（`shstrtab`），并读取其内容。

5. 查找符号表和字符串表

   遍历所有节头条目，寻找 `.symtab`（符号表）和 `.strtab`（字符串表）的节头，分别用于存储符号和符号名称。（如果愿意的话，可以根据节头条目提供的节类型信息二次确认我们找到了正确的表）。

​	如果没有找到这两个节，输出错误信息并终止程序。

6. 读取符号表和字符串表

   根据 `.symtab` 节头中的偏移和大小，读取符号表。

   根据 `.strtab` 节头中的偏移和大小，读取符号名称字符串表。

7. 将函数符号添加到符号表

   遍历符号表中的每个符号（`Elf32_Sym`），对于每个符号，检查它的类型是否是 `STT_FUNC`（即函数类型）。如果是函数符号，调用 `add_symbol` 将符号的名称、地址和大小添加到内存中的符号表中。

#### 实现FTRACE

1. 定义`const char *find_symbol(uint64_t addr)`函数从保存的映射表中找到内存地址对应的函数
2. 涉及函数跳转的命令，`jal`是直接跳转到目标地址进行函数调用，他的跳转地址是一个立即数，所以是一个编译时已经知道的具体偏移量，因此是直接调用。`jalr`的目标地址是由寄存器提供的，这是一个运行时才能确定的偏移量，因此是间接调用。同时由于函数返回时，我们并不能在编译阶段就知道每个函数返回到具体哪个地址，因此我们需要在每次调用前将当前pc存入返回地址寄存器`ra`中，利用`jalr ra, 0(ra)`实现返回。
3. 所以我们知道`jal`是直接调用，`jalr`在传入的立即数为0，且调用了返回地址寄存器`ra`时是在返回函数，其他情况下在间接调用，

### 如何实现DiffTest

检测`ref_r`中的寄存器信息是否和全局变量`cpu`中的一样即可，被忘了pc寄存器

## 输入输出问题处理

AM和NEMU的关系是，NEMU通过调用C函数库，模拟了一系列外设的接口，所以我们需要去阅读`$NEMU_HOME/src/device`下的文件，看NEMU是如何进行MMIO的，从而我们可以在AM中去读写正确的信息给NEMU处理。

### volatile有什么用

比如，假设 `p` 指向一个硬件设备的寄存器。如果没有使用 `volatile` 修饰，编译器可能会认为这个寄存器的值在程序运行期间不会被外部改变，因此它会把读取 `p` 的操作优化为只读取一次，将值缓存在寄存器中，而不再频繁地从该地址重新读取。这对于普通变量来说是合理的优化，但对于设备寄存器，这种优化可能会导致问题。原因是设备寄存器的值可能随时发生变化（比如通过外部硬件设备更新状态），编译器缓存的值不再是最新的，导致程序读取到的设备状态并不正确，从而产生错误行为。

### 如何实现时钟

阅读NEMU中`timer.c`源码，可以看到NEMU的模拟实现是通过调用`<time.h>`获得当前的时间，存储到了`rtc_port_base`的16个字节中。同时通过其中的`rtc_io_handler`可以看到如果我们读取了高位字节，NEMU就会获得最新的时间。所以我们在实现的时候**一定要先读高位字节**，才能刷新设备缓存，获取最新时间。

### 如何实现malloc

注意好申请的地址要对齐，这里由于我的电脑是64位我就设置了8字节的对齐，即每次申请的空间大小必须向8的整数倍对齐。

### 如何实现DTRACE

通过阅读文档我们知道，NEMU通过`map_read`和`map_write`进行MMIO的操作，所以只要在这两个函数里记录log即可。

### 如何实现键盘

阅读NEMU中`keyboard.c`文件中的`send_key`函数可以看到键盘每一次按下的信息是将`keymap`中的键位信息与掩码通过按位或的方式记录。我们在AM中解析出这两段信息，写入`kbd`即可。

### 如何实现VGA

阅读NEMU中`vga.c`可以看到显示的长宽在`vgactl_port_base`中的存储顺序。在AM中我们需要提取出这些信息并记录在config中，因为通过阅读`am-test`中的测试程序，可以发现程序需要获取这些信息。

`__am_gpu_fbdraw`中需要将提供的`ctl`中的打印信息正确写入VGA的寄存器，这里就是简单的二维数组的映射，我们需要在`x, y`作为矩形的左上角端点，写入一个`w*h`的矩形图像信息。如果此时要求控制信息要求同步，我们也要将这个信息及时写入VGA的寄存器中。

### 如何实现声卡

NEMU中由于我们使用`SDL`库来模拟声卡，在每次我们调用声卡信息的时候，我们也需要使用刚才的信息初始化一个新的SDL（因为配置信息可能改变），并且设置好他的回调函数。SDL回调函数定义了我们如何处理寄存器中提供的音频信息，由于我们的缓存区有限，我们使用循环队列的方式向缓冲区中写入数据。因此我们需要正确的从缓存区中读出数据并写入stream中。

在AM的`__am_audio_play`中，我们需要正确维护好循环队列缓冲区，向其中写入数据。

## 源码

```c
/* inst.c 粘一些代表性的，不全粘贴 */ 
INSTPAT_START();
  INSTPAT("0000000 ????? ????? 000 ????? 01100 11", add    , R, R(rd) = (int32_t)((int32_t)src1 + (int32_t)src2));
  INSTPAT("0000001 ????? ????? 000 ????? 01100 11", mul    , R, R(rd) = (int32_t)((int64_t)(int32_t)src1 * (int64_t)(int32_t)src2));
  INSTPAT("0000001 ????? ????? 001 ????? 01100 11", mulh   , R, R(rd) = (int32_t)(((int64_t)(int32_t)src1 * (int64_t)(int32_t)src2) >> 32));
  INSTPAT("0000001 ????? ????? 010 ????? 01100 11", mulhsu , R, R(rd) = ((int64_t)(int32_t)src1 * (uint64_t)(uint32_t)src2) >> 32);
  INSTPAT("0000001 ????? ????? 011 ????? 01100 11", mulhu  , R, R(rd) = (uint32_t)(((uint64_t)(uint32_t)src1 * (uint64_t)(uint32_t)src2) >> 32));
  INSTPAT("0100000 ????? ????? 000 ????? 01100 11", sub    , R, R(rd) = src1 - src2);
  INSTPAT("0000000 ????? ????? 010 ????? 01100 11", slt    , R, R(rd) = (int32_t)src1 < (int32_t)src2);
  INSTPAT("0000000 ????? ????? 011 ????? 01100 11", sltu   , R, R(rd) = (uint32_t)src1 < (uint32_t)src2);
  INSTPAT("0000000 ????? ????? 100 ????? 01100 11", xor    , R, R(rd) = src1 ^ src2);
  INSTPAT("0000001 ????? ????? 100 ????? 01100 11", div    , R, R(rd) = src2 ? (src1 == (1 << 31) && src2 == -1) ? (1 << 31) : (int32_t)((int32_t)src1 / (int32_t)src2) : -1);
  INSTPAT("0000001 ????? ????? 101 ????? 01100 11", divu   , R, R(rd) = src2 ? (uint32_t)((uint32_t)src1 / (uint32_t)src2) : (uint32_t)((1ll << 32) - 1));
  INSTPAT("0000000 ????? ????? 001 ????? 01100 11", sll    , R, R(rd) = (uint32_t)src1 << BITS(src2, 4, 0));
  INSTPAT("0000000 ????? ????? 101 ????? 01100 11", srl    , R, R(rd) = (uint32_t)src1 >> BITS(src2, 4, 0));
  INSTPAT("0100000 ????? ????? 101 ????? 01100 11", sra    , R, R(rd) = (int32_t)src1 >> BITS(src2, 4, 0));
  INSTPAT("0000000 ????? ????? 110 ????? 01100 11", or     , R, R(rd) = src1 | src2);
  INSTPAT("0000001 ????? ????? 110 ????? 01100 11", rem    , R, R(rd) = src2 ? (src1 == (1 << 31) && src2 == -1) ? 0 : (int32_t)src1 % (int32_t)src2 : src1);
  INSTPAT("0000001 ????? ????? 111 ????? 01100 11", remu   , R, R(rd) = src2 ? (uint32_t)src1 % (uint32_t)src2 : src1);
  INSTPAT("0000000 ????? ????? 111 ????? 01100 11", and    , R, R(rd) = src1 & src2);

  INSTPAT("??????? ????? ????? 000 ????? 00000 11", lb     , I, R(rd) = SEXT(Mr((uint32_t)((int32_t)src1 + (int32_t)imm), 1), 8));
  INSTPAT("??????? ????? ????? 001 ????? 00000 11", lh     , I, R(rd) = SEXT(Mr((uint32_t)((int32_t)src1 + (int32_t)imm), 2), 16));
  INSTPAT("??????? ????? ????? 010 ????? 00000 11", lw     , I, R(rd) = SEXT(Mr((uint32_t)((int32_t)src1 + (int32_t)imm), 4), 32));
  INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu    , I, R(rd) = Mr((uint32_t)((int32_t)src1 + (int32_t)imm), 1));
  INSTPAT("??????? ????? ????? 101 ????? 00000 11", lhu    , I, R(rd) = Mr((uint32_t)((int32_t)src1 + (int32_t)imm), 2));
  INSTPAT("??????? ????? ????? 000 ????? 00100 11", addi   , I, R(rd) = (int32_t)((int32_t)src1 + (int32_t)imm));  // mv
  INSTPAT("??????? ????? ????? 010 ????? 00100 11", slti   , I, R(rd) = (int32_t)src1 < (int32_t)imm);
  INSTPAT("??????? ????? ????? 011 ????? 00100 11", sltiu  , I, R(rd) = (uint32_t)src1 < (uint32_t)imm);
  INSTPAT("??????? ????? ????? 100 ????? 00100 11", xori   , I, R(rd) = src1 ^ imm);
  INSTPAT("000000? ????? ????? 001 ????? 00100 11", slli   , I, R(rd) = (uint32_t)src1 << BITS(imm, 4, 0));
  INSTPAT("000000? ????? ????? 101 ????? 00100 11", srli   , I, R(rd) = (uint32_t)src1 >> BITS(imm, 4, 0));
  INSTPAT("010000? ????? ????? 101 ????? 00100 11", srai   , I, R(rd) = (int32_t)src1 >> BITS(imm, 4, 0));
  INSTPAT("??????? ????? ????? 110 ????? 00100 11", ori    , I, R(rd) = src1 | imm);
  INSTPAT("??????? ????? ????? 111 ????? 00100 11", andi   , I, R(rd) = src1 & imm);
  // INSTPAT("??????? ????? ????? 000 ????? 00011 11", fence  , I, R(rd) = src1 & imm);
  INSTPAT("??????? ????? ????? 000 ????? 11001 11", jalr   , I, R(rd) = s->snpc; s->dnpc = (uint32_t)((int32_t)src1 + (int32_t)imm) & -2ull; check_jalr(rd, imm, s)); // ret

  INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb     , S, Mw((uint32_t)((int32_t)src1 + (int32_t)imm), 1, src2));
  INSTPAT("??????? ????? ????? 001 ????? 01000 11", sh     , S, Mw((uint32_t)((int32_t)src1 + (int32_t)imm), 2, src2));
  INSTPAT("??????? ????? ????? 010 ????? 01000 11", sw     , S, Mw((uint32_t)((int32_t)src1 + (int32_t)imm), 4, src2));

  INSTPAT("??????? ????? ????? 000 ????? 11000 11", beq    , B, s->dnpc = src1 == src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);
  INSTPAT("??????? ????? ????? 001 ????? 11000 11", bne    , B, s->dnpc = src1 != src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);
  INSTPAT("??????? ????? ????? 100 ????? 11000 11", blt    , B, s->dnpc = (int32_t)src1 < (int32_t)src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);
  INSTPAT("??????? ????? ????? 101 ????? 11000 11", bge    , B, s->dnpc = (int32_t)src1 >= (int32_t)src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);
  INSTPAT("??????? ????? ????? 110 ????? 11000 11", bltu   , B, s->dnpc = (uint32_t)src1 < (uint32_t)src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);
  INSTPAT("??????? ????? ????? 111 ????? 11000 11", bgeu   , B, s->dnpc = (uint32_t)src1 >= (uint32_t)src2 ? (uint32_t)((int32_t)s->pc + (int32_t)imm) : s->dnpc);

  INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc  , U, R(rd) = s->pc + imm);
  INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui    , U, R(rd) = imm);

  INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal    , J, R(rd) = s->snpc; s->dnpc = s->pc + imm; check_jal(s));


  INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak , N, NEMUTRAP(s->pc, R(10))); // R(10) is $a0
  INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv    , N, INV(s->pc));

void check_jal(Decode *s) {
  if (elf_loaded) {
    const char *func_name = find_symbol(s->dnpc);
    if (func_name != NULL) {
        call_depth++;
        log_write("F\t%#010x\t%*sCall to %s @ %#010x\n", s->pc, call_depth * 2, "", func_name, s->dnpc);
    }
  }
}

void check_jalr(word_t rd, word_t imm, Decode *s) {
  uint32_t i = s->isa.inst;
  int rs1 = BITS(i, 19, 15);
  if (elf_loaded) {
    if (rd == 0 && rs1 == 1 && imm == 0) {
        // Function return
        const char *func_name = find_symbol(s->pc);
        if (func_name != NULL) {
            log_write("F\t%#010x\t%*sReturn from %s\n", s->pc, call_depth * 2, "", func_name);
        }
        if (call_depth > 0) call_depth--;
    }
    else {
        // Indirect function call
        const char *func_name = find_symbol(s->dnpc);
        if (func_name != NULL) {
            call_depth++;
            log_write("F\t%#010x\t%*sIndirect call to %s @ %#010x\n", s->pc, call_depth * 2, "", func_name, s->dnpc);
        }
    }
  }
}
```

```c
/* stdio.c 粘一些代表性的，不全粘贴 */ 

int vsprintf(char *out, const char *fmt, va_list ap) {
    char* buf = out;
    int total_len = 0;

    while (*fmt != '\0') {
        if (*fmt != '%') {
            *buf++ = *fmt++;
            total_len++;
        } else {
            fmt++;
            // Parse flags
            int flags = 0;
            int width = 0;
            int precision = -1;
            // int length = 0;
            int specifier = 0;

            while (*fmt == '-' || *fmt == '+' || *fmt == ' ' || *fmt == '#' || *fmt == '0') {
                switch (*fmt) {
                    case '#': flags |= 0x02; break; // Hash
                    case '+': flags |= 0x04; break; // Plus sign
                    case ' ': flags |= 0x08; break; // Space
                    case '-': flags |= 0x10; break; // Left align
                    case '0': flags |= 0x20; break; // Zero padding
                }
                fmt++;
            }

            // Parse width
            if (*fmt == '*') {
                width = va_arg(ap, int);
                fmt++;
            } else {
                while (*fmt >= '0' && *fmt <= '9') {
                    width = width * 10 + (*fmt++ - '0');
                }
            }

            // Parse precision
            if (*fmt == '.') {
                fmt++;
                precision = 0;
                if (*fmt == '*') {
                    precision = va_arg(ap, int);
                    fmt++;
                } else {
                    while (*fmt >= '0' && *fmt <= '9') {
                        precision = precision * 10 + (*fmt++ - '0');
                    }
                }
            }

            // // Parse length modifier
            // if (*fmt == 'h' || *fmt == 'l') {
            //     length = *fmt++;
            // }

            // Parse specifier
            specifier = *fmt++;

            // Handle specifier
            if (specifier == 'd' || specifier == 'i') {
                int value = va_arg(ap, int);
                char temp[65];
                int len = int_to_str(value, temp, 10, 0, width, precision > 0 ? precision : 0, flags);
                for (int i = 0; i < len; i++) {
                    *buf++ = temp[i];
                }
                total_len += len;
            } else if (specifier == 'o') {
                unsigned int value = va_arg(ap, unsigned int);
                char temp[65];
                int len = int_to_str(value, temp, 8, 1, width, precision > 0 ? precision : 0, flags);
                for (int i = 0; i < len; i++) {
                    *buf++ = temp[i];
                }
                total_len += len;
            } else if (specifier == 'u') {
                unsigned int value = va_arg(ap, unsigned int);
                char temp[65];
                int len = int_to_str(value, temp, 10, 1, width, precision > 0 ? precision : 0, flags);
                for (int i = 0; i < len; i++) {
                    *buf++ = temp[i];
                }
                total_len += len;
            } else if (specifier == 'x' || specifier == 'X') {
                unsigned int value = va_arg(ap, unsigned int);
                char temp[65];
                if (specifier == 'X') flags |= 0x01; // Uppercase
                int len = int_to_str(value, temp, 16, 1, width, precision > 0 ? precision : 0, flags);
                for (int i = 0; i < len; i++) {
                    *buf++ = temp[i];
                }
                total_len += len;
            } else if (specifier == 'c') {
                char c = (char)va_arg(ap, int);
                *buf++ = c;
                total_len++;
            } else if (specifier == 's') {
                char *str = va_arg(ap, char*);
                int len = precision >= 0 ? precision : strlen(str);
                if (width > len && !(flags & 0x10)) { // Right align
                    for (int i = 0; i < width - len; i++) {
                        *buf++ = ' ';
                        total_len++;
                    }
                }
                for (int i = 0; i < len && str[i] != '\0'; i++) {
                    *buf++ = str[i];
                    total_len++;
                }
                if (width > len && (flags & 0x10)) { // Left align
                    for (int i = 0; i < width - len; i++) {
                        *buf++ = ' ';
                        total_len++;
                    }
                }
            } else if (specifier == 'p') {
                void *ptr = va_arg(ap, void*);
                unsigned long value = (unsigned long)ptr;
                char temp[65];
                flags |= 0x02; // Force '0x'
                int len = int_to_str(value, temp, 16, 1, width, precision > 0 ? precision : 0, flags);
                for (int i = 0; i < len; i++) {
                    *buf++ = temp[i];
                }
                total_len += len;
            } else if (specifier == 'f' || specifier == 'F' || specifier == 'e' || specifier == 'E' || specifier == 'g' || specifier == 'G') {
                assert(0);
                // double value = va_arg(args, double);
                // char temp[320];
                // int len = double_to_str(value, temp, precision >= 0 ? precision : 6, specifier, flags, width);
                // for (int i = 0; i < len; i++) {
                //     /*buf++ = temp[i];
                // }
                // total_len += len;
            } else if (specifier == '%') {
                *buf++ = '%';
                total_len++;
            } else if (specifier == 'n') {
                int *ptr = va_arg(ap, int*);
                *ptr = total_len;
            } else {
                // Invalid specifier
                assert(0);
            }
        }
    }

    *buf = '\0';
    return buf - out;
}

int sprintf(char *out, const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    int total_len = vsprintf(out, fmt, args);
    va_end(args);
    return total_len;
}

int snprintf(char *out, size_t n, const char *fmt, ...) {
  panic("Not implemented");
}

int vsnprintf(char *out, size_t n, const char *fmt, va_list ap) {
  panic("Not implemented");
}

static int int_to_str(int value, char *buffer, int base, int is_unsigned, int width, int precision, int flags) {
    char temp[65];
    int i = 0;
    unsigned int uvalue = value;
    int is_negative = 0;

    if (!is_unsigned && value < 0) {
        is_negative = 1;
        uvalue = -value;
    }

    if (uvalue == 0) {
        temp[i++] = '0';
    } else {
        while (uvalue != 0) {
            int digit = uvalue % base;
            if (digit < 10) {
                temp[i++] = digit + '0';
            } else {
                if (flags & 0x01) { // Uppercase for X
                    temp[i++] = digit - 10 + 'A';
                } else {
                    temp[i++] = digit - 10 + 'a';
                }
            }
            uvalue /= base;
        }
    }

    // Handle precision
    while (i < precision) {
        temp[i++] = '0';
    }

    // Add prefix for '#' flag
    if (flags & 0x02) {
        if (base == 8 && temp[i-1] != '0') {
            temp[i++] = '0';
        } else if (base == 16) {
            temp[i++] = flags & 0x01 ? 'X' : 'x';
            temp[i++] = '0';
        }
        else {
          assert(0);
        }
    }

    int len = i;
    // Handle zero-padding and width
    int padding = width - len - 1; // 1-width for the sign

    if (padding > 0 && (flags & 0x20) && !(flags & 0x10)) { // Right align
      while (padding-- > 0) {
          temp[i++] = '0';
      }
    }

    if (is_negative) {
        temp[i++] = '-';
    } else if (flags & 0x04) { // '+' flag
        temp[i++] = '+';
    } else if (flags & 0x08) { // ' ' flag
        temp[i++] = ' ';
    }

    if (padding > 0 && !(flags & 0x20) && !(flags & 0x10)) { // Right align
      while (padding-- > 0) {
          temp[i++] = ' ';
      }
    }

    reverse_str(temp, i);

    // Copy to buffer
    for (int j = 0; j < i; j++) {
        *buffer++ = temp[j];
    }

    // Right padding if left align
    if (padding > 0 && (flags & 0x10)) {
        while (padding-- > 0) {
            *buffer++ = ' ';
        }
    }

    return i;
}
```

```c
/* iringbuf.c 粘一些代表性的，不全粘贴 */ 

void record_inst(uint64_t pc, const char *asm_code) {
    inst_buf[inst_buf_pos].pc = pc;
    // Safely copy the assembly code to prevent buffer overflow
    strncpy(inst_buf[inst_buf_pos].asm_code, asm_code, sizeof(inst_buf[inst_buf_pos].asm_code) - 1);
    inst_buf[inst_buf_pos].asm_code[sizeof(inst_buf[inst_buf_pos].asm_code) - 1] = '\0'; // Ensure null-termination
    inst_buf_pos = (inst_buf_pos + 1) % INST_BUF_SIZE;  // Update write position to create a ring buffer
    if (inst_buf_count < INST_BUF_SIZE) {
        inst_buf_count++;  // Increment the count of recorded instructions
    }
}
```

```c
/* elf.c */ 

#include <elf.h>

typedef struct {
    char *name;
    uint64_t addr;
    uint64_t size;
} FunctionSymbol;

static FunctionSymbol *symbol_table = NULL;
static int symbol_count = 0;
static int symbol_capacity = 0;
bool elf_loaded = false;

void add_symbol(const char *name, uint64_t addr, uint64_t size) {
    if (symbol_count >= symbol_capacity) {
        symbol_capacity = symbol_capacity == 0 ? 1024 : symbol_capacity * 2;
        symbol_table = realloc(symbol_table, symbol_capacity * sizeof(FunctionSymbol));
    }
    symbol_table[symbol_count].name = strdup(name);
    symbol_table[symbol_count].addr = addr;
    symbol_table[symbol_count].size = size;
    symbol_count++;
}

const char *find_symbol(uint64_t addr) {
    if (!elf_loaded) return NULL;
    for (int i = 0; i < symbol_count; i++) {
        if (addr >= symbol_table[i].addr && addr < symbol_table[i].addr + symbol_table[i].size) {
            return symbol_table[i].name;
        }
    }
    return NULL;
}

void init_elf(const char *elf_file) {
    if (elf_file == NULL) {
        elf_loaded = false;
        return;
    }

    FILE *fp = fopen(elf_file, "rb");
    Assert(fp, "Cannot open '%s'", elf_file);

    elf_loaded = true;

    // Read ELF header
    Elf32_Ehdr ehdr;
    if (fread(&ehdr, sizeof(Elf32_Ehdr), 1, fp) != 1) {
        printf("Failed to read ELF header\n");
        exit(1);
    }

    // Verify ELF magic number
    if (memcmp(ehdr.e_ident, ELFMAG, SELFMAG) != 0) {
        printf("Not an ELF file\n");
        exit(1);
    }

    // Read section headers
    Elf32_Shdr *shdrs = malloc(ehdr.e_shentsize * ehdr.e_shnum);
    fseek(fp, ehdr.e_shoff, SEEK_SET);
    if (fread(shdrs, ehdr.e_shentsize, ehdr.e_shnum, fp) != ehdr.e_shnum) {
        printf("Failed to read section headers\n");
        exit(1);
    }

    // Read section header string table
    Elf32_Shdr shstr_shdr = shdrs[ehdr.e_shstrndx];
    char *shstrtab = malloc(shstr_shdr.sh_size);
    fseek(fp, shstr_shdr.sh_offset, SEEK_SET);
    if (fread(shstrtab, shstr_shdr.sh_size, 1, fp) != 1) {
        printf("Failed to read section header string table\n");
        exit(1);
    }

    // Find .symtab and .strtab sections
    Elf32_Shdr *symtab_shdr = NULL;
    Elf32_Shdr *strtab_shdr = NULL;
    for (int i = 0; i < ehdr.e_shnum; i++) {
        char *section_name = &shstrtab[shdrs[i].sh_name];
        if (shdrs[i].sh_type == SHT_SYMTAB && strcmp(section_name, ".symtab") == 0) {
            symtab_shdr = &shdrs[i];
        } else if (shdrs[i].sh_type == SHT_STRTAB && strcmp(section_name, ".strtab") == 0) {
            strtab_shdr = &shdrs[i];
        }
    }

    if (symtab_shdr == NULL || strtab_shdr == NULL) {
        printf("Failed to find .symtab or .strtab in ELF file\n");
        exit(1);
    }

    // Read symbol table
    int sym_count = symtab_shdr->sh_size / symtab_shdr->sh_entsize;
    Elf32_Sym *symtab = malloc(symtab_shdr->sh_size);
    fseek(fp, symtab_shdr->sh_offset, SEEK_SET);
    if (fread(symtab, symtab_shdr->sh_entsize, sym_count, fp) != sym_count) {
        printf("Failed to read symbol table\n");
        exit(1);
    }

    // Read string table
    char *strtab = malloc(strtab_shdr->sh_size);
    fseek(fp, strtab_shdr->sh_offset, SEEK_SET);
    if (fread(strtab, strtab_shdr->sh_size, 1, fp) != 1) {
        printf("Failed to read string table\n");
        exit(1);
    }

    // Store function symbols
    for (int i = 0; i < sym_count; i++) {
        Elf32_Sym sym = symtab[i];
        char *name = &strtab[sym.st_name];
        if (ELF32_ST_TYPE(sym.st_info) == STT_FUNC) {
            add_symbol(name, sym.st_value, sym.st_size);
        }
    }

    // Free allocated memory
    free(shdrs);
    free(shstrtab);
    free(symtab);
    free(strtab);
    fclose(fp);
}
```

```c
/* gpu.c 粘一些代表性的，不全粘贴 */ 

void __am_gpu_config(AM_GPU_CONFIG_T *cfg) {
  uint32_t data = inl(VGACTL_ADDR);
  int width = (data >> 16) & 0xffff;
  int height = data & 0xffff;
  int vmemsz = width * height * sizeof(uint32_t);
  *cfg = (AM_GPU_CONFIG_T) {
    .present = true, .has_accel = false,
    .width = width, .height = height,
    .vmemsz = vmemsz
  };
}

void __am_gpu_fbdraw(AM_GPU_FBDRAW_T *ctl) {
  int x = ctl->x, y = ctl->y, w = ctl->w, h = ctl->h;
  int width = inl(VGACTL_ADDR) >> 16, height = inl(VGACTL_ADDR) & 0xffff;

  uint32_t *pixels = ctl->pixels;
  uint32_t *fb = (uint32_t *)(uintptr_t)FB_ADDR;

  for (int i = y; i < y + h && i < height; i++) {
    for (int j = x; j < x + w && j < width; j++) {
      fb[i * width + j] = pixels[(i - y) * w + (j - x)];
    }
  }

  if (ctl->sync) {
    outl(SYNC_ADDR, 1);
  }
}
```

```c
/* audio.c 粘一些代表性的，不全粘贴 */ 

void __am_audio_play(AM_AUDIO_PLAY_T *ctl) {
  uint8_t *audio_data = (ctl->buf).start;
  uint32_t sbuf_size = inl(AUDIO_SBUF_SIZE_ADDR);
  uint32_t len = (ctl->buf).end - (ctl->buf).start;

  uint8_t *ab = (uint8_t *)(uintptr_t)AUDIO_SBUF_ADDR; 
  for(int i = 0; i < len; i++){
    ab[sbuf_pos] = audio_data[i];
    sbuf_pos = (sbuf_pos + 1) % sbuf_size;  
  }
  outl(AUDIO_COUNT_ADDR, inl(AUDIO_COUNT_ADDR) + len);
}
```

```c
/* am/audio.c 粘一些代表性的，不全粘贴 */ 

void __am_audio_play(AM_AUDIO_PLAY_T *ctl) {
  uint8_t *audio_data = (ctl->buf).start;
  uint32_t sbuf_size = inl(AUDIO_SBUF_SIZE_ADDR);
  uint32_t len = (ctl->buf).end - (ctl->buf).start;

  uint8_t *ab = (uint8_t *)(uintptr_t)AUDIO_SBUF_ADDR; 
  for(int i = 0; i < len; i++){
    ab[sbuf_pos] = audio_data[i];
    sbuf_pos = (sbuf_pos + 1) % sbuf_size;  
  }
  outl(AUDIO_COUNT_ADDR, inl(AUDIO_COUNT_ADDR) + len);
}
```

```c
/* nemu/audio.c 粘一些代表性的，不全粘贴 */ 

void init_sound();

static void audio_io_handler(uint32_t offset, int len, bool is_write) {
  if(audio_base[reg_init] == 1){
    init_sound();
    audio_base[reg_init] = 0;
  }
}

void sdl_audio_callback(void *userdata, uint8_t *stream, int len){
  SDL_memset(stream, 0, len);
  uint32_t used_cnt = audio_base[reg_count];
  len = len > used_cnt ? used_cnt : len;
  
  uint32_t sbuf_size = audio_base[reg_sbuf_size];
  if( (sbuf_pos + len) > sbuf_size ){
    SDL_MixAudio(stream, sbuf + sbuf_pos, sbuf_size - sbuf_pos , SDL_MIX_MAXVOLUME);
    SDL_MixAudio(stream +  (sbuf_size - sbuf_pos), sbuf, len - (sbuf_size - sbuf_pos), SDL_MIX_MAXVOLUME);
  }
  else 
    SDL_MixAudio(stream, sbuf + sbuf_pos, len , SDL_MIX_MAXVOLUME);
  sbuf_pos = (sbuf_pos + len) % sbuf_size;
  audio_base[reg_count] -= len;
}

void init_sound() {
  SDL_AudioSpec s = {};
  s.format = AUDIO_S16SYS;
  s.userdata = NULL;
  s.freq = audio_base[reg_freq];
  s.channels = audio_base[reg_channels];
  s.samples = audio_base[reg_samples];
  s.callback = sdl_audio_callback;
  SDL_InitSubSystem(SDL_INIT_AUDIO);
  SDL_OpenAudio(&s, NULL);
  SDL_PauseAudio(0);
}
```

