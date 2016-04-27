# v9-cpu

## 概述
v9-cpu是一个假想的简单的32-bit RISC CPU，用于操作系统教学实验和练习．

## 寄存器组:
总共有 9 个寄存器,其中 7 个为 32 位,2 个为 64 位(浮点寄存器)。本文档只针对 CPU
进行描述,对于相关的硬件配套外设(例如中断控制器等)在此不做介绍。其中：

 - a, b, c : 三个32-bit通用寄存器
 - f, g 两个64-bit浮点寄存器,是用来进行各种指令操作的
 - sp 为当前栈底指针，按64-bit(8字节)对齐
   - usp: user stack，在用户态,sp是usp
   - ssp: kernel stack，在内核台，sp是ksp,用户态应用不可见ssp
 - pc 为32-bit程序计数器（指向下一条指令），按32-bit(4字节)对齐，其指向的内存内容（即具体的指令值）会放到ir中，给CPU解码并执行
 - tsp 为栈顶指针(本 CPU 的栈是从顶部往底部增长),按64-bit(8字节)对齐
 - flags 为内部状态寄存器(包括当前的运行模式,是否中断使能,是否有自陷,以及是否使用虚拟地址等)，可通过特定指令访问相关bit
 
 

### flags寄存器标志位
 - user   : 1; 用户态或内核态(user mode or not.)
 - iena   : 1; 中断使能/失效 (interrupt flag.)
 - trap   : 1; 异常/错误编码 (fault code.)
 - paging : 1; 页模式使能/失效（ virtual memory enabled or not.）

 
## 指令集

总共有209条指令,一条指令大小为32bit, 具体的命令编码在指令的低8位,高24位为操作数。
但很多指令只是某一类指令的多种形式，很容易触类旁通。

整体来看,指令分为如下几类:

 - 运算指令：如ADD, SUB等
 - 跳转指令：如JMP, JSR，LEV等
 - 访存（Load/Store）指令：如LL, LBL, SL等
 - 系统命令:如HALT, RTI, IDLE，SSP, USP,IVEC, PDIR，目的是为了操作系统设计
 - 扩展函数库命令：如MCPY/MCMP/MCHR/MSET, MATH类，NET类，目的为了简化编译器设计

按照指令对条件的需求，指令可分类如下：

 - HALT, RTI, IDLE等，不需要立即数来参与，也不需要读取或修改寄存器值。
 - ADD, MCPY，PSHA等，不需要立即数来参与，但需要读取或修改寄存器值。
 - ADDI, S/L系列，B系列等，需要立即数和寄存器来参与。
 


v9-cpu的指令集如下：


### 显示格式
```
指令名字　指令值　　指令含义
```

### 指令集明细

#### 指令值格式
```
0xiiiiiicc OR 0x......cc
i --> 操作数　immediate OR operand0 (简称imme)
. --> CPU解码不会用到高24位
c --> 指令编码
```

#### system
```
HALT	0xiiiiii00	halt system

//可用于分配函数中的局部变量
ENT		0xiiiiii01  sp += imme
//用于函数返回
LEV		0xiiiiii02  pc =  *sp,	sp += imme+8

//跳转指令
JMP		0xiiiiii03	pc += imme
JMPI	0xiiiiii04	pc += imme+ra>>2


//用于函数调用
JSR		0xiiiiii05	*sp = pc,	sp -= 8,	pc += imme
JSRA	0x......06	*sp = pc,	sp -= 8,	pc += ra

#### load to register a
```
LL		0xiiiiii0e	ra = *(*unit)  (sp+imme)
LLS		0xiiiiii0f	ra = *(*short) (sp+imme)
LLH		0xiiiiii10	ra = *(*ushort)(sp+imme)
LLC		0xiiiiii11	ra = *(*char)  (sp+imme)
LLB		0xiiiiii12	ra = *(*uchar) (sp+imme)
LX		0xiiiiii1c	ra = *(*unit)  global_addr(ra+imme)
LXS		0xiiiiii1d	ra = *(*short) global_addr(ra+imme)
LXH		0xiiiiii1e	ra = *(*ushort)global_addr(ra+imme)
LXC		0xiiiiii1f	ra = *(*char)  global_addr(ra+imme)
LXB		0xiiiiii20	ra = *(*uchar) global_addr(ra+imme)
LI		0xiiiiii23	ra = imme
LHI		0xiiiiii24	ra = (ra<<24)|(imme>>8)
```

#### load to register b
```
LBL		0xiiiiii26	rb = *(*uint)  (sp+imme)
LBLS	0xiiiiii27	rb = *(*short) (sp+imme)
LBLH	0xiiiiii28	rb = *(*ushort)(sp+imme)
LBLC	0xiiiiii29	rb = *(*char)  (sp+imme)
LBLB	0xiiiiii2a	rb = *(*uchar) (sp+imme)
LBX		0xiiiiii34	rb = *(*uint)  global_addr(rb+imme)
LBXS	0xiiiiii35	rb = *(*short) global_addr(rb+imme)
LBXH	0xiiiiii36	rb = *(*ushort)global_addr(rb+imme)
LBXC	0xiiiiii37	rb = *(*char)  global_addr(rb+imme)
LBXB	0xiiiiii38	rb = *(*uchar) global_addr(rb+imme)
LBI		0xiiiiii3b  rb = imme
LBHI	0xiiiiii3c  rb = (rb<<24)|(imme>>8)
```

#### store register a to memory 
```
SL		0xiiiiii40	*(*uint)  (sp+imme) = (uint)  (ra)
SLH		0xiiiiii41	*(*ushort)(sp+imme) = (ushort)(ra)
SLB		0xiiiiii42	*(*uchar) (sp+imme) = (ushort)(ra)
SX		0xiiiiii4a  *(*uint)  global_addr(rb+imme) = (uint)  (ra)
SXH		0xiiiiii4b	*(*ushort)global_addr(rb+imme) = (ushort)(ra)
SXB		0xiiiiii4c	*(*uchar) global_addr(rb+imme) = (ushort)(ra)
```
#### arithmetic
```
ADD		0x......53  ra = ra+rb
ADDI	0xiiiiii54	ra = ra+imme
MUL		0x......59	ra = (int)(ra)*(int)(rb)
MULI	0xiiiiii5a	ra = (int)(ra)*(int)(imme)
DIV		0x......5c	ra = (int)(ra)/(int)(rb)
DIVI	0xiiiiii5d	ra = (int)(ra)/(int)(imme)
DVU		0x......5f	ra = (uint)(ra)/(uint)(rb)
DVUI	0xiiiiii60	ra = (uint)(ra)/(uint)(imme)
MOD		0x......62	ra = (int)(ra)%(int)(rb)
MODI	0xiiiiii63	ra = (int)(ra)%(int)(imme)
MDU		0x......65	ra = (uint)(ra)%(uint)(rb)
MDUI	0xiiiiii66	ra = (uint)(ra)%(uint)(imme)
AND		0x......68	ra = ra&rb
ANDI	0xiiiiii69	ra = ra&imme
OR		0x......6b	ra = ra|rb
ORI		0xiiiiii6c	ra = ra|imme
XOR		0x......6e	ra = ra^rb
XORI	0xiiiiii6f	ra = ra^imme
SHL		0x......71	ra = ra<<(uint)(rb)
SHLI	0xiiiiii72	ra = ra<<(uint)(imme)
SHR		0x......74	ra = (int)(ra)>>(uint)(rb)
SHRI	0xiiiiii75	ra = (int)(ra)>>(uint)(imme)
SRU		0x......77	ra = (uint)(ra)>>(uint)(rb)
SRUI	0xiiiiii78	ra = (uint)(ra)>>(uint)(imme)
EQ		0x......7a  ra = (ra == rb)
NE		0x......7c	ra = (ra != rb)
LT		0x......7e	ra = ((int)a < (int)b)
LTU		0x......7f	ra = ((uint)a < (uint)b)
GE		0x......81	ra = ((int)a > (int)b)
GEU		0x......82	ra = ((uint)a > (uint)b)
```
#### conditional branch
```
BZ		0xiiiiii84	if (ra == 0)  pc = pc+imme
BNZ		0xiiiiii86  if (ra != 0)  pc = pc+imme
```

#### misc
```
NOP		0x......9b	no operation.

PSHA	0x......9d	sp -= 8, *sp = a
PSHI	0x......9e	sp -= 8, *sp = imme
PSHB	0x......a0	sp -= 8, *sp = b
POPB	0x......a1	b = *sp, sp += 8
POPA	0x......a3	a = *sp, sp += 8
PSHC	0x......ae	sp -= 8, *sp = c
POPC	0x......af	c = *sp, sp += 8
```

#### cpu idle
```
IDLE 	0x......d1	response hardware interrupt (include timer).
```

## 内存
缺省内存大小为128MB，可以通过启动参数"-m XXX"，设置为XXX MB大小．
在TLB中，设置了4个1MB大小页转换表（page translation buffer array）

 - kernel read page translation table
 - kernel write page translation table
 - user read page translation table
 - user write page translation table

有两个指针tr/tw, tw指向内核态或用户态的read/write　page translation table．
```
tr/tw[page number]=phy page number //页帧号
```
还有一个tpage buffer array, 保存了所有tr/tw中的虚页号，这些虚页号是tr/tw数组中的index 
```
tpage[tpages++] = v //v是page number
```

### 分页机制
#### 相关操作
- PDIR --> page_directory base addr PTBR <--reg a
- SPAG --> enable_paging = a

#### 页表格式

```
                                                              PAGE FRAME
              +-----------+-----------+----------+         +---------------+
              | DIR 10bit |PAGE 10bit |OFF 12bit |         |               |
              +-----+-----+-----+-----+-----+----+         |               |
                    |           |           |              |               |
      +-------------+           |           +------------->|    PHYSICAL   |
      |                         |                          |    ADDRESS    |
      |   PAGE DIRECTORY        |      PAGE TABLE          |               |
      |  +---------------+      |   +---------------+      |               |
      |  |               |      |   |               |      +---------------+
      |  |               |      |   |---------------|              ^
      |  |               |      +-->| PG TBL ENTRY  |--------------+
      |  |---------------|          |---------------|
      +->|   DIR ENTRY   |--+       |               |
         |---------------|  |       |               |
         |               |  |       |               |
         +---------------+  |       +---------------+
                 ^          |               ^
+-------+        |          +---------------+
| PTBR  |--------+
+-------+

```
#### 页目录表项，页表项的属性
```
  PTE_P = 0x001, // present
  PTE_W = 0x002, // writeable
  PTE_U = 0x004, // user
  PTE_A = 0x020, // accessed
  PTE_D = 0x040, // dirty
```

#### 页目录表项，页表项的组成
```
DIR_ENTRY=　[高20位：二级页表地址的高20位（4KB对齐）][低12位属性]　

PT_ENTRY　=　[高20位：物理页帧地址的高20位（4KB对齐）][低12位属性]　
```

### 页访问异常
```
  FMEM,   // bad physical address
  FIPAGE, // page fault on opcode fetch
  FWPAGE, // page fault on write
  FRPAGE, // page fault on read
  USER=16 // user mode exception
```

通过LVAD指令可获得访问异常的虚地址并赋值给寄存器a　

## IO操作
### 写外设（类似串口写）的步骤
 - 1 --> a
 - 一个字符'char' --> b
 - BOUT　　//如果在内核态，在终端上输出一个字符'char', 1-->a，如果在用户态，产生FPRIV异常

### 读外设（类似串口读）的步骤
　- BIN  //如果在内核态，kchar -->a  kchar是模拟器定期轮询获得的一个终端输入字符
 　　
如果iena(中断使能)，则在获得一个终端输入字符后，会产生FKEYBD中断 
 
### 设置timer的timeout
 - val --> a
 - TIME // 如果在内核态，设置timer的timeout为a; 如果在用户态，产生FPRIV异常
 


## 中断/异常
### 一些变量的含义：
 - ivec: 中断向量的地址
 
### 中断/异常类型
```
- FMEM,          // bad physical address 
- FTIMER,        // timer interrupt
- FKEYBD,        // keyboard interrupt
- FPRIV,         // privileged instruction
- FINST,         // illegal instruction
- FSYS,          // software trap
- FARITH,        // arithmetic trap
- FIPAGE,        // page fault on opcode fetch
- FWPAGE,        // page fault on write
- FRPAGE,        // page fault on read
- USER 　　　　      // user mode exception 
```

### 设置中断向量
 - val --> a
 - IVEC // 如果在内核态，设置中断向量的地址ivec为a; 如果在用户态，产生FPRIV异常

### 中断/异常产生的处理
对于中断，CPU在执行指令前进行判断，看是否有中断：

 - 如果终端产生了键盘输入，且iena=1，则ipend |= FKEYBD，0-->iena
 - 如果timer产生了timeout，且iena=1，则ipend |= FTIMER，0-->iena
 
对于异常，即在执行某指令时出现了异常（非法访问内存，非法权限，算术异常等）,或者是陷入（trap）指令，则会有相应的处理

 - 如果当前处于中断屏蔽状态(即iena=0)，则产生fatal错误；
 - 对于异常，设置错误码为异常编码；对于陷入指令，设置错误码为FSYS

然后是统一的善后处理：

 - 如果当前处于用户态(user=1)，则sp=kernel_sp, tr=kernel_tr, tw=kernel_tw,  trap|=USER
 - PUSH当前的pc到当前内核栈
 - PUSH 错误码(fault code)到当前内核栈
 - pc跳转中断向量的地址ivec


## 应用二进制接口（application binary interface，ABI）
ABI描述的内容包括：

 - 数据类型的大小、布局和对齐;
 - 调用约定（控制着函数的参数如何传送以及如何接受返回值），例如，是所有的参数都通过栈传递，还是部分参数通过寄存器传递；哪个寄存器用于哪个函数参数；通过栈传递的第一个函数参数是最先push到栈上还是最后；
 - 系统调用的编码和一个应用如何向操作系统进行系统调用；
 - 以及在一个完整的操作系统ABI中，执行文件的二进制格式、程序库等等。


### 基本数据表示
 - char：8-bit有符号字符类型
 - uchar：8-bit无符号字符类型
 - short：16-bit有符号短整数类型
 - ushort:16-bit无符号短整数类型
 - int:32-bit有符号整数类型
 - uint：32-bit无符号整数类型
 - float:32-bit浮点类型
 - double:64-bit浮点类型
 - pointer:32-bit指针类型

### 对齐要求和字节序
上述的的数据类型，只有当自然对齐的情况，才可以用标准的v9指令直接处理。v9-cpu采用小端（Little-Endian）格式，就是低位字节排放在内存的低地址端，高位字节排放在内存的高地址端。v9-cpu 要求栈对齐到8字节。


### 函数调用约定
####  常规参数传递
v9-cpu ABI (应用程序二进制接口)约定了用栈来传递函数调用参数，所有的参数的都是占用8字节的空间，在调用前，把参数按从右至左的顺序进行压栈。

#### 返回值传递
在返回基本数据类型的情况下，寄存器 a 约定为传递返回值。

### 执行文件的格式
执行文件由文件头和文件体组成。其中，文件头的结构为：

```
struct { 
 uint magic //0xC0DEF00D 文件的magic number 
 uint bss　 //在v9-cpu执行时没有用到
 uint entry //程序的执行入口地址
 uint flags;//程序的数据段起始地址 
} hdr;
```
> BSS是Block Started by Symbol的简称，指用来存放程序中未初始化的全局变量的一块内存区域

执行文件体有两部分组成：

 - text段：紧接在文件头后，代码段（code segment/text segment）通常是指用来存放程序执行代码的一块内存区域。这部分区域的大小在程序运行前就已经确定，
 - data段:紧接在代码段后，数据段（data segment）通常是指用来存放程序中已初始化的全局变量的一块内存区域。数据段属于静态内存分配。

## CPU执行过程
### 一些变量的含义：
主要集中在em.c的cpu()函数中

 - a: a寄存器
 - b: b寄存器
 - c: c寄存器
 - f: f浮点寄存器
 - g: g浮点寄存器
 - ir:　指令寄存器
 - xpc: pc在host内存中的值
 - fpc: pc在host内存中所在页的下一页的起始地址值
 - tpc: pc在host内存中所在页的起始地址值
 - xsp: sp在host内存中的值
 - tsp: sp在host内存中所在页的起始地址值
 - fsp: 辅助判断是否要经过tr/tw的分析
 - ssp: 内核态的栈指针
 - usp: 用户态的栈指针
 - cycle: 指令执行周期计数
 - xcycle:　用于判断外设中断的执行频度，和调整最终的指令执行周期计数（需进一步明确?）
 - timer: 当前时钟计时（和时间时间中断有关）
 - timeout: 时钟周期，当timer>timeout时，产生时钟中断 
 -　detla:　一次指令执行的时间，timer+=delta
 
 ###执行过程概述
 
 1. 以funcall程序为例，首先，读入执行文件的代码段＋数据段到内存的底部，并把pc放置到执行文件头中entry设置的内存位置， 设置可用sp为MEM_SZ-FS_SZ - 数据段和程序段（应该还包括BSS段）
 1. 然后从pc指向的起始地址开始执行
 1. 如果碰到异常或中断，则保存中断的地址，并跳到中断向量的地址ivec处执行
