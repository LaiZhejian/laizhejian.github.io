---
title: 'NJU ICS2024 PA 作业心得（三）'
date: 2024-12-24
permalink: /posts/2024/12/24/ics-pa3/
tags:
  - ICS
---

> 记录了在RISC-V架构下实现异常响应机制、系统调用和文件系统的过程，包括NEMU模拟器的扩展、AM抽象层的异常处理以及Nanos-lite操作系统的系统调用实现，并分析了ELF加载和程序执行流程。


##  需要参考的内容

1. [RISC-V ABIs Specification](https://d3s.mff.cuni.cz/files/teaching/nswi200/202324/doc/riscv-abi.pdf)：是一组规则和规范，定义了在 RISC-V 架构上编写和链接程序的方式。它确保了不同语言编写的代码、不同编译器生成的代码以及操作系统之间的兼容性。**在这部分PA中我们需要参考其中定义的每个寄存器的功能，以及对应存储传递内容的含义。**
2. [The RISC-V Instruction Set Manual--Volume II: Privileged Architecture](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)：定义了 RISC-V 架构的特权模式和相关功能。这本手册的核心内容是描述操作系统、虚拟化和其他需要硬件支持的高特权操作所需的架构支持。**在这部分PA中我们需要参考其中的控制状态寄存器相关指令，包括实现方式及具体意义。**

## 穿越时空的旅程问题处理

我们首先需要明确的是`cte_init`函数的作用是什么，在文档中已经给了详细的说明，我们也需要在`$AM_HOME/am/src/riscv/nemu/cte.c`中`cte_init`函数的代码分别找到对应实现：

1. 设置异常入口地址

   ```c
   asm volatile("csrw mtvec, %0" : : "r"(__am_asm_trap));
   ```

   这里csrw是一个针对控制状态寄存器读写操作的指令，上述的汇编指令将`am_asm_trap`函数的地址（指针）写入到了`mtvec`寄存器中。

2. 注册一个事件处理回调函数

   ```c
   user_handler = handler;
   ```

   这个事件处理回调函数是由更上一层的操作系统提供的，在[之后](##实现SYS_yield系统调用问题处理)我们再详细介绍她的功能，现在只要知道他是一个函数指针`Context*(*handler)(Event, Context*)`，通过给定异常事件类型以及和上下文便可以处理异常事件，并将异常处理的结果写入上下文的寄存器中，方便用户程序得知结果。

再进一步的我们阅读`$AM_HOME/am/src/riscv/nemu/trap.S`中关于`am_asm_trap`的代码我们可以看到这个函数

1. 创建一段上下文大小的内存空间，并把CPU寄存器的值存入其中。

2. 分别将控制状态寄存器读取到一些特定的寄存器，从而将控制状态寄存器的值存入第一步创建的内存空间中，方便后续恢复上下文。

3. 将内存空间中保存的上下文作为参数（根据ABI手册规定`a0`寄存器是函数传入的第一个参数）交给`$AM_HOME/am/src/riscv/nemu/cte.c`中的`__am_irq_handle`处理异常。

4. `__am_irq_handle`函数会根据上下文信息确定事件类型交由操作系统的事件处理函数。

   *需要说明的是，事件的类型判断本应当与操作系统有关而与AM无关，所以一个更科学的写法是把这事件类型的判断放在`user_handler`中完成。*

5. 通过1，2步保存的上下文信息，恢复上下文并返回。

### 如何实现异常响应机制？

1. 完善`isa_raise_intr`的功能，从而成功实现`yield`函数的调用。

   通过阅读`$AM_HOME/am/src/riscv/nemu/cte.c`中`yield`函数的源码我们可以看出来，它首先将`-1`这个特殊的异常号存入`a7`寄存器中，然后调用`ecall`指令。

   根据RISCV手册中规定，`ecall`指令会触发异常并进入更高的特权级（如 S 模式或 M 模式）处理系统调用。也就是说`ecall`指令应当调用`isa_raise_intr`函数，正确的对控制状态寄存器进行赋值后，将`pc`移动到`mtvec`寄存器中。并且通过观察已经给出的`ebreak`指令可以看出，在PA中默认我们在M-Mode，因此`PRIV`字段为`0b11`，从而我们可以获得对应的`ecall`指令码。

   再看`isa_raise_intr`，它需要负责对控制状态寄存器进行正确的赋值。那么为了实现这个功能，我们首先得在CPU中创建这样的控制状态寄存器，也就是在`$NEMU_HOME/src/isa/riscv32/include/isa-def.h`中，更改CPU结构体。并且为了通过DIFFTEST，我们需要在CPU初始化时把`mstatus`寄存器赋值位`0x1800`。

   通过阅读源码可以看到`$NEMU_HOME/src/monitor/monitor.c`中调用了`init_isa`函数，这个函数在`$NEMU_HOME/src/isa/riscv32/init.c`中。同时这个函数调用了同文件下的`restart`函数完成CPU初始化，在其中对`mstatus`寄存器进行更改即可。

   然后通过阅读RISCV手册，我们了解到在`ecall`指令调用下，`mcause`寄存器需要存入`11`，对应`Environment call from M-mode`。

2. 实现相关指令。通过阅读源码，出现了一些在PA2中未曾实现的指令`ecall`、`csrr`、`csrw`、`mret`。这些指令有的涉及到了控制状态寄存器的读写，每个状态控制寄存器的编号，可以通过RISCV手册查询获得。

### 如何重新组织Context结构体？

通过查看`$AM_HOME/am/src/riscv/nemu/trap.S`中关于`OFFSET`的代码，我们知道在开辟的内存空间中要在低位存通用寄存器，高位存控制状态寄存器。

## 用户程序与系统调用问题解决

### 如何为Nanos-lite实现正确的事件分发问题处理？

Nanos-lite是一个操作系统，它在`/home/dream/ics2024/nanos-lite/src/irq.c`处提供了异常事件的处理函数`do_event`，在该函数中对`EVENT_YIELD`进行打印处理即可。

### 堆和栈在哪里？

* **堆**：堆用于动态内存分配（如 `malloc` 或 `new`），其大小和内容在程序运行时才会确定。因为堆的使用是动态的，无法在编译阶段确定其内容或大小，所以无法将堆的内容预先包含在可执行文件中。

- **栈**：栈用于函数调用、局部变量和中断上下文保存，其内容也在程序运行时动态变化。尽管栈的初始位置和大小通常是由编译器或操作系统定义的，但栈内容本身取决于程序的执行流，无法预先包含。

### FileSiz与MemSiz的区别是什么？

ELF 文件中的段在加载时可能需要额外的内存分配，导致两者的值不同。主要原因包括：

1. BSS 段

   BSS 段（未初始化数据）在文件中通常不存储实际内容，而是仅在运行时分配所需的内存。由于从`[VirtAddr + FileSiz, VirtAddr + MemSiz)`**可能对应BSS段存储的那些在内存空间中需要初始化为0的数据**，因此需要清零。

   例子:

   - 一个未初始化的全局变量 `int arr[1000];`，占用 4000 字节内存。

   - 在 ELF 文件中，BSS 段的 `FileSiz` 为 0，因为没有实际数据，但 `MemSiz` 为 4000，用于分配运行时内存。

2. 对齐和填充

   ELF 段通常需要按照内存对齐要求加载，可能会分配额外的空间以满足对齐需求。这些对齐的空白区域不会占用文件存储，但在内存中需要分配。

### loader函数怎么实现？

其实和之前的FTRACE实现很像，我们首先得了解ELF文件的结构。

1. 在ELF文件最开头存储了ELF文件头，通过这个文件头我们可以检查ELF魔数，编译ELF文件的指令集，找到程序段。
2. 我们访问所有程序段，并且通过`PT_LOAD`标志判断是否要在运行时载入，并按照ELF文件的要求将这些内容初始化在指定的内存位置，供程序运行时读写。

### SYS_yield系统调用怎么实现？

现在我们需要捋一捋系统调用的整个过程：当程序（Navy_app）通过操作系统调用`_syscall_`进行系统调用后，该函数通过调用`ecall`指令，进而转入NEMU中`isa_raise_intr`进行硬件负责的中断处理程序跳转，当NEMU确定好该异常对应的控制状态寄存器后，然后转入AM中将会进行上下文的保存，并进入`__am_irq_handle`获得这个异常的事件类型，并将事件交由Nanos中`do_event`，这个事件处理回调函数。如果这个事件的类型是系统调用，那么会转入Nanos的`do_syscall`中执行一些操作系统级别的指令（例如文件读写等）处理异常。

再回到这道题目本身，我们需要实现`SYS_yield`，根据`man syscall`我们知道了`a7`寄存器存储了系统调用的类型，`a0,a1`寄存器分别保存第一返回值和第二返回值。这个系统调用的规范在`$NAVY_APPS_HOME/libs/libos/src/syscall.c`的`_syscall_`宏定义中被实现了。那根据这个要求在`do_syscall`中进行对应的功能实现即可。

*很令笔者费解的一点在于，为什么既有`EVENT_YIELD`又有`SYSCALL_YIELD`，感觉好像存在功能的冗余？若有见解，还望广大读者不吝赐教。*

### hello程序是什么, 它从而何来, 要到哪里去？

`hello.c` 是一个 C 源文件，存储在磁盘上。当我们使用编译器（如 `gcc` 或 `clang`）编译它时，生成一个可执行的 ELF 文件（如 `hello`）。这个文件仍然在磁盘上，直到被加载到内存。

当我们在操作系统中用loader根据ELF文件信息数据与命令内容文件加载到了内存中的正确区域中。

它的第一条指令由ELF文件指定在`e_entry`位置上。这个指针指向`$AM_HOME/am/src/riscv/nemu/start.S`的`_start`函数。然后被NEMU从该位置从内存中的该位置读取第一条指令并译码顺序执行。

Hello程序打印字符串的过程

1. 用户态的函数调用
   `printf("Hello, world!\n") `会调用标准库中的 `_write` 函数，该函数会调用`write`系统调用。
   `write` 系统调用通过系统调用号和参数向操作系统发起请求。
2. 系统调用切换到内核态
   `write` 系统调用切换到内核态（通过 `syscall` 指令）。
   参数（如文件描述符、缓冲区地址、长度）通过寄存器传递到内核。
3. 内核处理数据
   内核根据文件描述符（`FD_STDOUT`，值为 1）识别终端设备。
   将用户缓冲区中的字符串数据拷贝到NEMU模拟的串口驱动程序`putch`。
4. 驱动程序输出字符
   字符可能通过`putch`发送到屏幕。
5. 用户看到结果
   最终，字符串被显示在终端窗口或硬件屏幕上。

## 文件系统问题处理

### 如何实现如sbrk, open, close, gettimeofday等系统调用？

这些系统调用直接查看手册，根据规范实现即可，如果需要调动外设，通过AM定义好的函数接口调用。

关于文件系统的实现，一定要注意好在每次读入文件时，**文件指针的初始化**，忘记将其归位为0可能导致一些BUG！并且可以关于其中虚拟文件系统的设计思想非常有参考意义，和程序设计中的多态非常像，对读写行为进行了很好抽象。

## 精彩纷呈的应用程序问题处理

### 之后的关于一些VGA和键盘事件的实现，还有一些APP的怎么实现？

笔者完成了VGA和键盘事件的实现，在VGA的实现过程中需要去阅读SDL的源码，完整的复刻其行为才能使得程序正确运行。在最后一章节的APP实现中，感觉与ICS的内容核心有所分离，没那么偏向于系统的模拟与实现，且需要耗费大量时间，因此从Flappy Bird开始的所有APP均未实现。

## 源码

```c
/* cte.c 粘一些代表性的，不全粘贴 */ 
Context* __am_irq_handle(Context *c) {
  if (user_handler) {
    Event ev = {0};
    switch (c->mcause) {
      case 11: 
        if (c->gpr[17] == -1)  // check a7 value
          ev.event = EVENT_YIELD;
        else 
          ev.event = EVENT_SYSCALL;
        c->mepc += 4;
        break;
      default: ev.event = EVENT_ERROR; break;
    }
    c = user_handler(ev, c);
    assert(c != NULL);
  }

  return c;
}
```

```c
/* syscall.c 粘一些代表性的，不全粘贴 */ 
void do_syscall(Context *c) {
  uintptr_t a[4];
  a[0] = c->GPR1;
  a[1] = c->GPR2;
  a[2] = c->GPR3;
  a[3] = c->GPR4;
  
  // int i;
  // printf("\n%-10s\t%-10s\t%s\n", "REG_NAME", "HEX_VALUE", "DEC_VALUE");
  // for (i = 0; i < 32; ++i) {
  //   printf("%-10s\t%#010x\t%-12u\n", "", c->gpr[i], c->gpr[i]);
  // }
  // printf("%-10s\t%#010x\t%-12u\n", "mstatus", c->mstatus, c->mstatus);
  // printf("%-10s\t%#010x\t%-12u\n", "mcause", c->mcause, c->mcause);
  // printf("%-10s\t%#010x\t%-12u\n", "mepc", c->mepc, c->mepc);

  switch (a[0]) {
    case SYS_exit: if (SYS_DEBUG) Log("Syscall: exit"); halt(a[1]); break;
    case SYS_yield: if (SYS_DEBUG) Log("Syscall: yield"); yield(); c->GPRx = 0; break;
    case SYS_open: if (SYS_DEBUG) Log("Syscall: open"); c->GPRx = fs_open((char *)a[1], a[2], a[3]); break;
    case SYS_read: if (SYS_DEBUG) Log("Syscall: read"); c->GPRx = fs_read(a[1], (void *)a[2], a[3]); break;
    case SYS_write: if (SYS_DEBUG) Log("Syscall: write"); c->GPRx = fs_write(a[1], (void *)a[2], a[3]); break;
    case SYS_lseek: if (SYS_DEBUG) Log("Syscall: lseek"); c->GPRx = fs_lseek(a[1], a[2], a[3]); break;
    case SYS_close: if (SYS_DEBUG) Log("Syscall: close"); c->GPRx = fs_close(a[1]); break;
    case SYS_brk: if (SYS_DEBUG) Log("Syscall: brk"); c->GPRx = 0; break; // fail return -1
    case SYS_gettimeofday: if (SYS_DEBUG) Log("Syscall: gettimeofday"); c->GPRx = sys_gettimeofday((struct timeval *)a[1], (struct timezone *)a[2]); break;
    default: panic("Unhandled syscall ID = %d", a[0]);
  }

}
```

```c
/* inst.c 粘一些代表性的，不全粘贴 */ 

  INSTPAT("??????? ????? ????? 001 ????? 11100 11", csrrw  , I, rd? (R(rd) = CSR(imm)) : 0; CSR(imm) = src1);
  INSTPAT("??????? ????? ????? 010 ????? 11100 11", csrrs  , I, R(rd) = CSR(imm); src1? (CSR(imm) |= src1) : 0);
  INSTPAT("0000000 00000 00000 000 00000 11100 11", ecall  , N, s->dnpc = isa_raise_intr(11, s->pc)); // Environment call from M-mode
  INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak , N, NEMUTRAP(s->pc, R(10))); // R(10) is $a0
  INSTPAT("0011000 00010 00000 000 00000 11100 11", mret   , N, s->dnpc = CSR(0x341)); // Get return address from 0x341
```

```c
/* fs.c 粘一些代表性的，不全粘贴 */ 

static Finfo file_table[] __attribute__((used)) = {
  [FD_STDIN]  = {"stdin", 0, 0, 0, invalid_read, serial_write},
  [FD_STDOUT] = {"stdout", 0, 0, 0, invalid_read, serial_write},
  [FD_STDERR] = {"stderr", 0, 0, 0, invalid_read, invalid_write},
  [FD_EVENTS] = {"/dev/events", 0, 0, 0, events_read, invalid_write},
  [FD_FB]     = {"/dev/fb", 0, 0, 0, invalid_read, fb_write},
  {"/proc/dispinfo", 0, 0, 0, dispinfo_read, invalid_write },
#include "files.h"
};

void init_fs() {
  // initialize the size of /dev/fb
  AM_GPU_CONFIG_T gpu_config = io_read(AM_GPU_CONFIG);
  int width = gpu_config.width;
  int height = gpu_config.height;
  file_table[FD_FB].size = width * height * sizeof(uint32_t);
}

int fs_open(const char *pathname, int flags, int mode) {
  for (int i = 0; i < sizeof(file_table) / sizeof(file_table[0]); i++) {
    if (strcmp(pathname, file_table[i].name) == 0) {
      file_table[i].open_offset = 0;
      return i;
    }
  }
  assert(0);
  return -1;
}

size_t fs_read(int fd, void *buf, size_t len) {
  size_t nread = 0;
  if (file_table[fd].read == NULL)
    nread = ramdisk_read(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  else
    nread = file_table[fd].read(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  if (fd != FD_STDIN) {
    file_table[fd].open_offset = file_table[fd].open_offset + nread > file_table[fd].size ? file_table[fd].size : file_table[fd].open_offset + nread;
  }
  return fd >= 0 ? nread : -1;
}

size_t fs_write(int fd, const void *buf, size_t len) {
  size_t nwrite = 0;
  if (file_table[fd].write == NULL)
    nwrite = ramdisk_write(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  else
    nwrite = file_table[fd].write(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  if (fd != FD_STDOUT && fd != FD_STDERR) {
    file_table[fd].open_offset = file_table[fd].open_offset + nwrite > file_table[fd].size ? file_table[fd].size : file_table[fd].open_offset + nwrite;
  }
  return fd >= 0 ? nwrite : -1;
}

size_t fs_lseek(int fd, size_t offset, int whence) {
  switch (whence) {
    case SEEK_SET: file_table[fd].open_offset = offset; break;
    case SEEK_CUR: file_table[fd].open_offset += offset; break;
    case SEEK_END: file_table[fd].open_offset = file_table[fd].size + offset; break;
    default: assert(0);
  }
  if (file_table[fd].open_offset < 0 || file_table[fd].open_offset > file_table[fd].size) {
    Log("fs_lseek: offset out of range");
    return -1;
  }
  return file_table[fd].open_offset >= 0 ? file_table[fd].open_offset : -1;
}

int fs_close(int fd) {
  return 0;
}
```

```c
/* loader.c 粘一些代表性的，不全粘贴 */ 

static uintptr_t loader(PCB *pcb, const char *filename) {
  int fd = fs_open(filename, 0, 0);
  if (fd == -1) {
      printf("Failed to open file %s\n", filename);
      assert(0);
  }
  Elf_Ehdr ehdr;
  if (fs_read(fd, &ehdr, sizeof(Elf_Ehdr)) == 0) {
      printf("Failed to read ELF header\n");
      assert(0);
  }
  if (memcmp(ehdr.e_ident, ELFMAG, SELFMAG) != 0) {
      printf("Not an ELF file\n");
      assert(0);
  }
  if (ehdr.e_machine != EXPECT_TYPE) {
      printf("Not an ELF file of target ISA\n");
      assert(0);
  }
  Elf_Phdr phdr[ehdr.e_phnum];
  fs_lseek(fd, ehdr.e_phoff, SEEK_SET);
  if (fs_read(fd, phdr, ehdr.e_phnum * sizeof(Elf_Phdr)) == 0) {
      printf("Failed to read program headers\n");
      assert(0);
  }
  for (int i = 0; i < ehdr.e_phnum; i++) {
      if (phdr[i].p_type == PT_LOAD) {
          fs_lseek(fd, phdr[i].p_offset, SEEK_SET);
          if (fs_read(fd, (void *)phdr[i].p_vaddr, phdr[i].p_filesz) == 0) {
              printf("Failed to read segment %d\n", i);
              assert(0);
          }
          if (phdr[i].p_filesz < phdr[i].p_memsz) {
              memset((void *)(phdr[i].p_vaddr + phdr[i].p_filesz), 0, phdr[i].p_memsz - phdr[i].p_filesz);
          }
      }
  }
  fs_close(fd);
  return ehdr.e_entry;
}
```

```c
/* device.c 粘一些代表性的，不全粘贴 */ 

int sys_gettimeofday(struct timeval *tv, struct timezone *tz) {
  uint64_t us = io_read(AM_TIMER_UPTIME).us;
  tv->tv_sec = us / 1000000;
  tv->tv_usec = us % 1000000;
  return 0;
}

size_t serial_write(const void *buf, size_t offset, size_t len) {
  for (int i = 0; i < len; i++) {
    putch(((char *)buf + offset)[i]);
  }
  return len;
}

size_t events_read(void *buf, size_t offset, size_t len) {
  AM_INPUT_KEYBRD_T kb = io_read(AM_INPUT_KEYBRD);
  int nread = snprintf((char*)buf, len, "%s %s\n", kb.keydown ? "kd" : "ku", keyname[kb.keycode]);
  if (kb.keycode == AM_KEY_NONE)
    nread = 0;
  return nread;
}

size_t dispinfo_read(void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T gpu_config = io_read(AM_GPU_CONFIG);
  return snprintf((char *)buf, len, "WIDTH: %d\nHEIGHT: %d\n", gpu_config.width, gpu_config.height);
}

size_t fb_write(const void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T gpu_config = io_read(AM_GPU_CONFIG);
  int width = gpu_config.width;
 
  offset /= 4;
  len /= 4;
 
  int y = offset / width;
  int x = offset % width;
 
  io_write(AM_GPU_FBDRAW, x, y, (void *)buf, len, 1, true);
 
  return len;
}
```

```c
/* fixdptc.h 粘一些代表性的，不全粘贴 */ 

static inline fixedpt fixedpt_muli(fixedpt A, int B) {
	return A * B;
}

/* Divides a fixedpt number with an integer, returns the result. */
static inline fixedpt fixedpt_divi(fixedpt A, int B) {
	return A / B;
}

/* Multiplies two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_mul(fixedpt A, fixedpt B) {
	return A * (B >> FIXEDPT_FBITS);
}


/* Divides two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_div(fixedpt A, fixedpt B) {
	return A / (B >> FIXEDPT_FBITS);
}

static inline fixedpt fixedpt_abs(fixedpt A) {
	return A < 0 ? -A : A;
}

static inline fixedpt fixedpt_floor(fixedpt A) {
	if (fixedpt_fracpart(A) == 0) return A;
	if (A > 0) return A & ~FIXEDPT_FMASK;
	else return (A & ~FIXEDPT_FMASK) - FIXEDPT_ONE;
}

static inline fixedpt fixedpt_ceil(fixedpt A) {
	if (fixedpt_fracpart(A) == 0) return A;
	if (A > 0) return (A & ~FIXEDPT_FMASK) + FIXEDPT_ONE;
	else return A & ~FIXEDPT_FMASK;
}
```

```c
/* event.c 粘一些代表性的，不全粘贴 */ 

int SDL_PollEvent(SDL_Event *ev) {
  char buf[64];
  if (NDL_PollEvent(buf, sizeof(buf))) {
    char type[8];
    char key[16];
    if (sscanf(buf, "%s %s", type, key) == 2) {
      ev->key.type = (strcmp(type, "kd") == 0) ? SDL_KEYDOWN : SDL_KEYUP;
      for (int i = 0; i < sizeof(keyname) / sizeof(keyname[0]); i++) {
        if (strcmp(key, keyname[i]) == 0) {
          ev->key.keysym.sym = i;
          return 1;
        }
      }
    }
  }
  return 0;
}

int SDL_WaitEvent(SDL_Event *event) {
  int ret;
  while (1) {
    ret = SDL_PollEvent(event);
    if (ret) {
      return ret;
    }
  }
}
```

```c
/* NDL.c 粘一些代表性的，不全粘贴 */ 

uint32_t NDL_GetTicks() {
  // return the time in milliseconds
  assert(gettimeofday(&tv, NULL) == 0);
  return tv.tv_sec * 1000 + tv.tv_usec / 1000 - start_time;

}

int NDL_PollEvent(char *buf, int len) {
  int fd = open("/dev/events", 0, 0);
  if (fd < 0) return 0;
  int nread = read(fd, buf, len);
  assert(close(fd) == 0);
  return nread > 0 ? 1 : 0;
}

void NDL_OpenCanvas(int *w, int *h) {

  if (getenv("NWM_APP")) {
    int fbctl = 4;
    fbdev = 5;
    screen_w = *w; screen_h = *h;
    char buf[64];
    int len = sprintf(buf, "%d %d", screen_w, screen_h);
    // let NWM resize the window and create the frame buffer
    write(fbctl, buf, len);
    while (1) {
      // 3 = evtdev
      int nread = read(3, buf, sizeof(buf) - 1);
      if (nread <= 0) continue;
      buf[nread] = '\0';
      if (strcmp(buf, "mmap ok") == 0) break;
    }
    close(fbctl);
  }
  else {
    canvas_h = *h == 0 || canvas_h > screen_h ? screen_h : *h;
    canvas_w = *w == 0 || canvas_w > screen_w ? screen_w : *w;
    *h = canvas_h;
    *w = canvas_w;
    canvas_x = (screen_w - canvas_w) / 2;
    canvas_y = (screen_h - canvas_h) / 2;
  }
}

void NDL_DrawRect(uint32_t *pixels, int x, int y, int w, int h) {
  w = w ? w : canvas_w;
  h = h ? h : canvas_h;
  int fd = open("/dev/fb", 0, 0);
  for (int i = 0; i < h && y + i < canvas_h; ++i) {
    lseek(fd, ((y + canvas_y + i) * screen_w + (x + canvas_x)) * sizeof(uint32_t), SEEK_SET);
    write(fd, pixels + i * w,  (w < canvas_w - x ? w : canvas_w - x) * sizeof(uint32_t));
  }
  assert(close(fd) == 0);

}

int NDL_Init(uint32_t flags) {
  if (getenv("NWM_APP")) {
    evtdev = 3;
  }
  int fd = open("/proc/dispinfo", 0, 0);
  char buf[64];
  int nread = read(fd, buf, sizeof(buf) - 1);
  if (nread > 0) {
    buf[nread] = '\0';
    int i = 0;
    while (buf[i] != ':') ++i;
    screen_w = atoi(buf + i + 1);
    ++i;
    while (buf[i] != ':') ++i;
    screen_h = atoi(buf + i + 1);
  }
  else {
    assert(0);
  }

  gettimeofday(&tv, NULL);
  start_time = tv.tv_sec * 1000 + tv.tv_usec / 1000;

  return 0;
}
```
