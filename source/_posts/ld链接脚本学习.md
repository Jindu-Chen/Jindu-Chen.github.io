---
title: 嵌入式ld链接脚本详解
date: 2023-12-20
tags:
categories:
- [MISC]
description: 记录链接脚本的学习内容，主要为链接脚本的要素简述、常用语法说明、常见片段的意义解析、示例代码的讲述及应用
---


## 前言

链接脚本的主要作用：描述如何将输入文件(.o目标文件)中的段 映射到 输出文件(.bin/.hex/...)中的段，并控制输出文件的内存布局(地址分配)。

注意：输入文件的段和输出文件的段是独立的，主要分为`.text`, `.data`, `.bss`, `heap`, `stack`段
    
输入文件的段是链接器从输入文件中读取的内容，输出文件的段是链接器将输入文件的段链接到一起后生成的内容。
    
链接脚本用于指导链接器如何将输入文件的段链接到一起，其主要由**MEMORY**和**SECTIONS**块组成
- `SECTIONS`是必须的
- `MEMORY`可选的，如果没有，链接器则认为所有输入文件都位于同一个内存区域，并且从0x0开始
<br>

## 基本知识点补充

.ld文件是GNU链接器(ld)使用的标准链接脚本文件，可使用GNU链接器的所有功能，包括符号解析、重定位、符号表生成等
<br>

.lds文件是旧式的链接脚本文件，其仅支持GNU链接器的部分功能，不推荐使用

## 块

### MEMORY

为链接器提供系统内存的布局信息，并确定内存区域的访问权限。链接器根据MEMORY的信息，将编译生成的.o目标文件中的代码、数据、符号等分配到不同的内存区域。

示例代码如下：
```
MEMORY
{
FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 128K
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 32K
}
```
代码描述了两个内存区域：
- 标号为`FLASH`的内存区域权限为：可读只执行，起始地址为0x8000000，长度为128K；
- 标号为`RAM`的内存区域权限为：可读可写可执行，起始地址为0x20000000，长度为32K。<br>

**而后，链接脚本可声明将指定的代码放到对应的memory区域——以下链接脚本代码将`.text`段存放到标号为`RAM`的内存区域，将`.data`段存放到标号为`FLASH`的内存区域**
```
SECTIONS
{
  .text >RAM
  .data >FLASH
}
```

---

### SECTIONS

用于定义目标文件中各个段，包括名称、存储位置、对齐方式、内容，并且可以确定其访问权限。
```
SECTIONS
{
  // 将可执行代码段（.text 段）存放到 RAM 区域，对齐到 4 字节边界
  .text >RAM : ALIGN(4)

  // 将全局变量段（.data 段）存放到 FLASH 区域，对齐到 4 字节边界
  .data >FLASH : ALIGN(4)

  // 将堆（heap）段存放到 RAM 区域，对齐到 8 字节边界
  .heap >RAM : ALIGN(8)
}
```
另，**链接脚本中的内存存放顺序是从上到下**

---


## 命令

**ENTRY**

将某个入口点函数指定为起始地址，如果没有指定`ENTRY`，链接器会默认使用第一个可执行section作为程序入口点

`ENTRY(Reset_Handler)`将Reset_Handler函数设为程序的入口点，链接器需要知道程序的入口点，才能正确地组织可执行文件的时序，才能正确地执行编译、链接、优化代码

**注意：在cortex-m4中，ENTRY(Reset_Handler)并没有将Reset_Handler实体链接到0x8000004地址，其地址由链接器根据实际链接**<br>

**链接脚本规定了起始段放置的是isr_vector，而启动文件 规定了向量中第一个字是指定存放 栈顶地址，第二个字即0x8000004存放的则是Reset_Handler的地址**

```
SECTIONS
{
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >FLASH

  .text :
  {
  _stext = .;
  . = ALIGN(4);
  *(.text)           /* .text sections (code) */
  xxx}}
```

---

**/DISCARD/**

用于指定链接器应丢弃的section
```
/DISCARD/ :
{
  libc.a ( * )
  libm.a ( * )
  libgcc.a ( * )
}
```
上述代码，指示链接器将C标准库、数学库、GCC库中的section丢弃，在生成的可执行文件中不需要保留这些段。

**如果有调用到以下函数，则需要保留：**
- libm.a 包含了用于实现数学函数的符号和数据，例如 sin()、cos()、sqrt() 等
- libgcc.a 包含了用于实现 GCC 扩展的符号和数据，例如 __builtin_add()、__builtin_div() 等
- libc.a 是 C 标准库的库文件，包含了许多实现 C 标准库函数的符号和数据，诸如 printf()、scanf()、malloc()、free() 等常用函数。如果使用了微库，也可以去掉该section


---

**KEEP**

用于防止链接器丢弃未被引用的段和符号
```
// 强制连接器保留 .text 和 .data 两个 section 中的所有符号
SECTIONS {
.text : { KEEP(*(.text.*)) }
.data : { KEEP(*(.data.*)) }
}
```
- `*`表示任意多个字符，`.text.*`表示所有以 .text 开头的 section
- `*(.text.*)` 表示所有以 .text 开头的 section 中的所有符号


---
**LOADADDR**
  用于指定段的加载地址
<br>

**PROVIDE**
用于在链接时提供一个符号，如果符号未被定义，则使用其提供的值
如：`PROVIDE ( end = . );`
<br>

**PROVIDE_HIDDEN**
作用类似于PROVIDE，但提供的符号不会被其它模块看到
如：`PROVIDE_HIDDEN (__fini_array_start = .);`
<br>


## 语句、片段示例解释

```
SECTIONS
{
. = 0x10000;  // 将地址指针设为0x10000，表示.text段从地址0x10000开始
.text : { *(.text) }  // 表示将输入文件的.text段复制到输出文件的.text段中
. = 0x8000000;        //将地址指针设为0x8000000，表示.data段从地址0x8000000开始
.data : { *(.data) }
.bss : { *(.bss) }
}
```

```
    ._user_heap_stack :
    {
        . = ALIGN(4);       // 将当前地址调整为4字节对齐，确保堆和栈数据的正确访问
        PROVIDE ( end = . );
        PROVIDE ( _end = . );
        . = . + _Min_Heap_Size;
        . = . + _Min_Stack_Size;
        _estack = .;      // 设置 _estack 符号的值为当前地址
        . = ALIGN(4);
    } >RAM
```



## 链接脚本完整示例解释

```
ENTRY(Reset_Handler)

_Min_Heap_Size = 0x200;      /* required amount of heap  */
_Min_Stack_Size = 0x1500; /* required amount of stack */

MEMORY
{
FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 128K
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 32K
}

SECTIONS
{
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector)) /* Startup code */
    . = ALIGN(4);
  } >FLASH

  .text :
  {
    . = ALIGN(4);
    *(.text)           /* .text sections (code) */
    *(.text*)          /* .text* sections (code) */
    *(.glue_7)         /* glue arm to thumb code */
    *(.glue_7t)        /* glue thumb to arm code */
    *(.eh_frame)

    KEEP (*(.init))
    KEEP (*(.fini))
	
	. = ALIGN(4);
	__fsymtab_start = .;
	KEEP(*(FSymTab))
	__fsymtab_end = .;

	. = ALIGN(4);
	__vsymtab_start = .;
	KEEP(*(VSymTab))
	__vsymtab_end = .;

	. = ALIGN(4);
	__rt_init_start = .;
	KEEP(*(SORT(.rti_fn*)))
	__rt_init_end = .;

    . = ALIGN(4);
    _etext = .;        /* define a global symbols at end of code */
    _exit = .;
  } >FLASH

  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)         /* .rodata sections (constants, strings, etc.) */
    *(.rodata*)        /* .rodata* sections (constants, strings, etc.) */
    . = ALIGN(4);

  _shell_command_start = .;
  KEEP (*(shellCommand))
  _shell_command_end = .;
  } >FLASH

  .ARM.extab   : { *(.ARM.extab* .gnu.linkonce.armextab.*) } >FLASH
  .ARM : {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } >FLASH

  .preinit_array     :
  {
    PROVIDE_HIDDEN (__preinit_array_start = .);
    KEEP (*(.preinit_array*))
    PROVIDE_HIDDEN (__preinit_array_end = .);
  } >FLASH
  .init_array :
  {
    PROVIDE_HIDDEN (__init_array_start = .);
    KEEP (*(SORT(.init_array.*)))
    KEEP (*(.init_array*))
    PROVIDE_HIDDEN (__init_array_end = .);
  } >FLASH
  .fini_array :
  {
    PROVIDE_HIDDEN (__fini_array_start = .);
    KEEP (*(SORT(.fini_array.*)))
    KEEP (*(.fini_array*))
    PROVIDE_HIDDEN (__fini_array_end = .);
  } >FLASH

  _sidata = LOADADDR(.data);

  .data : 
  {
    . = ALIGN(4);
    _sdata = .;        /* create a global symbol at data start */
    *(.data)           /* .data sections */
    *(.data*)          /* .data* sections */

    . = ALIGN(4);
    _edata = .;        /* define a global symbol at data end */
  } >RAM AT> FLASH
  
  . = ALIGN(4);
  .bss :
  {
    /* This is used by the startup in order to initialize the .bss secion */
    _sbss = .;         /* define a global symbol at bss start */
    __bss_start__ = _sbss;
    *(.bss)
    *(.bss*)
    *(COMMON)

    . = ALIGN(4);
    _ebss = .;         /* define a global symbol at bss end */
    __bss_end__ = _ebss;
  } >RAM

  /* User_heap_stack section, used to check that there is enough RAM left */
  ._user_heap_stack :
  {
    . = ALIGN(4);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    _estack = .;
    . = ALIGN(4);
  } >RAM

  /* Remove information from the standard libraries */
  /DISCARD/ :
  {
    libc.a ( * )
    libm.a ( * )
    libgcc.a ( * )
  }

  .ARM.attributes 0 : { *(.ARM.attributes) }
}
```

## 疑惑解答及注意项

### 如何将指定的函数/数据/变量链接到指定的内存地址上？

讲道理来说，原则上是可实现的，一般用在特殊情况需求上，比如两个互相独立的工程分别下载到同一个Flash的不同区域，但它们又需要共享用到一个非常大的常量数组，这时候或许需要这个技巧。

在编译时就指定变量的存放地址，程序直接指向强制访问该地址即可。

由于时间关系，或者本人意愿关系，暂且不想写下这个详细步骤，但先铺垫点草稿如下：
- 使用`__attribute__`关键字，如`int value __attribute__ ((section (".ARM.__at_0x20000000"))) = 0x33;`
- 给变量声明一个段，而后在链接脚本中指定这个段的链接或加载地址



## 参考站点

- [简单的ld链接脚本学习](https://blog.51cto.com/u_15060508/3869831)

