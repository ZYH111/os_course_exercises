# lab0 SPOC思考题

##**提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在linux, ucore, v9-cpu中是如何具体体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？  
答：若需要支持进程，则在硬件设计上需要支持时钟中断、用户态和内核态之间的切换，应该提供的特权指令是从内核态切换到用户态的指令。若需要支持虚存，则在硬件设计上需要支持段页式内存管理，应该提供的特权指令是设置段基地址的指令，以及设置页目录起始地址的指令。若需要支持文件系统，则在硬件设计上应当支持对磁盘的读取，应该提供的特权指令是读取磁盘扇区内容的指令。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？  
答：x86的实模式下只能使用20位的地址线（即寻址空间为1MB），且程序中使用的地址为物理地址，用户程序很容易破坏操作系统；而保护模式下可以使用全部32位地址线（即寻址空间为4GB），支持段页式内存管理，支持多个特权级，能够防止用户程序破坏操作系统。物理地址的含义是数据在物理内存中存放的实际地址，线性地址是逻辑地址加上段基地址，逻辑地址是程序员在程序中使用的地址。

- 理解list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```
答：表示字段占用的位数。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？  
答：这个题的代码用gcc 5.4.0编译无法通过，因为将`SETGATE`宏展开后`SETGATE`宏的内容的第一条语句会变为`intr.gd_off_15_0 = (uint32_t)3 & 0xffff`，而`intr`不是一个含有字段`gd_off_15_0`的结构体，所以会出现编译错误。

### 课堂实践练习

#### 练习一

请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

  - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)

  - ##### [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)

  - ##### [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)

  - ##### [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)

  - ##### [[IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)]

答：选取的汇编代码如下（这段汇编代码来自LAB 1的文件`boot/bootasm.S`）：

```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
```

这段代码的含义是首先从8042的0x64端口读取数据到寄存器%al，计算%al与0x2进行逻辑与运算后的结果，若结果等于0则重新读取，直到某次读取到%al的值与0x2进行逻辑与运算后的结果不等于0（即）为止。这时将%al的内容设为0xd1，将寄存器%al的内容输出到8042的0x64端口。

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore中宏定义的用途，并举例描述其含义。  
答：利用宏进行复杂数据结构中的数据访问（例：LAB 4的文件`kern/process/proc.h`中的宏`le2proc`对进程控制块链表进行访问，根据链表节点找到进程控制块）；利用宏进行数据类型转换（例：文件`libs/def.h`中的宏`to_struct`对根据结构体中的成员变量的地址找到结构体的起始地址）；常用功能的代码片段优化（例：LAB 2的文件`kern/mm/pmm.h`中的宏`KADDR`根据物理地址求出内核虚拟地址）。
