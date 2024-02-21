---
title: C语言知识点记录
date: 2023-12-04 10:04:15
tags:
categories:
- [C语言]
description: 记录、总结在嵌入式开发中遇到的C知识点，以及一些C编程注意项
---


## 杂项

### 段的详解
- 常规的段：代码段、数据段、BSS段、堆、栈
- init段、符号表段、重定位段、调试信息段、通过attribute指定的段
  可通过编程将指定的函数链接到指定的段，如init段，实现在main前的初始化，等一系列自定义操作
- 链接器在链接过程中会自动生成特殊的符号，用于代表各段的首尾地址
  如：段名init_array，那么相应的会有`__init_array_start`或者`__start_init_array`表示该段的首地址，`__init_array_end`或`__stop_init_array`表示该段的末尾地址，不同编译器的表示方式有区别


## 预处理

重复宏定义？
- 宏定义的本质是替换，如果同一个宏名定义了两个不同的量，则在编译时可能会替换两次，最终输出的为最后一次宏定义处理的值

像`#if`、`#include`属于预处理指令

### C库集成的宏定义

`__DATE__`：字符串，为编译时的当前系统日期、时间
`__FILE__`：打印处的.c文件名
`__LINE__`：整型变量，为打印处的行号
`__TIME__`：字符串，编译时的系统时间
`__func__`：打印处的函数名

如下：
```c
#include <stdio.h>

int main()
{
  printf("[%s %s] %s: %s: %d\n", __DATE__, __TIME__, __FILE__, __func__, __LINE__);

  return 0;
}
```
编译运行，将会输出：`[2024-01-29 14:54:25] [14:54:25] main.c:main: 12`

<br>



### `#pragma pack(n)`指令

`#pragma pack(n)`是C语言中的预处理指令，用于设置结构体或者联合体的成员变量对齐的宽度都为`n`
- `__attribute__((packed))` 修饰符可以指定结构体成员按照最小的对齐方式，而 `#pragma pack()` 指令只能指定结构体成员按照指定的对齐方式
- 常规的变量，如`int`、`char`等由编译器默认设置，不受该指令影响
- `__attribute__((packed)) `修饰符的作用域仅限于其所在的语句块
```c
// 作用域位于#pragma pack(1) 与 #pragma pack()之间
#pragma pack(1)

// 以下结构体占用5个字节，而非8个字节
struct s {
    char ch;
    int i;
};

#pragma pack()
```


## 关键字

### `__attribute__` 关键字

`__attribute__((weak))`
`__attribute__((constructor))`
<br>

`__attribute__((packed))`属性用于指定结构体或联合体的成员变量按照实际占用字节数进行对齐（按照最小的对齐方式）
- 使用此属性可能会导致效率降低，因为 CPU 可能需要进行额外的操作来访问非对齐的数据
- 其可以减少结构体和联合体的大小，节省内存空间
- 如下代码所示，不加`__attribute__((packed))`属性的地址为 a:0 b:2 c:4，占用8个字节；加属性修饰后的地址为 a:0 b:1 c:3，占用7个字节
```c
typedef struct __attribute__((packed))
{
    uint8_t   a;
    uint16_t  b;
    uint32_t  c;
}
time_desc;
```
<br>

`__attribute__((__used__))` 告知编译器：即使修饰的对象在编译过程中没有被引用，也要保留
- 建议在C源文件中的汇编函数前加上,以避免高优化等级时其被优化掉
<br>

`__attribute__((__section__("my_section")))` 表示将修饰的对象放在名为`my_section`的段中
- 在C文件中编程，一般可以通过特殊的符号来获取attribute声明的段的首末地址，其由链接器在链接过程中自动定义
<br>

**示例1：**
```c
#define  __used  __attribute__((__used__))

typedef void (*initcall_t)(void);

#define __define_initcall(fn) \
    static const initcall_t __initcall_##fn##id __used \
    __attribute__((__section__("initcall_system_init"))) = fn; 

#define INIT_SYSTEM_EXPORT(fn)      __define_initcall(fn)

```

**应用示例2：** 定义系列 赋值为函数指针的静态常量，将其声明到指定的段中
```c
extern char __start_my_section[];
extern char __stop_my_section[];

char *start = __start_my_section;
char *end = __stop_my_section;

typedef void (*func_ptr)(void);

for (func_ptr *f = (func_ptr *)__start_my_section; f < (func_ptr *)__stop_my_section; ++f) {
    (*f)();
}
```

## 变量

静态局部变量，能被外界所访问吗？
>将其地址传递出去即可，因为静态变量的地址在编译时已经确定，其生存在整个程序运行期间


## 函数


### switch case
>当在case语句内定义变量时，需要在case语句内 加花括号限定变量的作用域，否则编译会报错


### main()类型函数

在单片机开发中，通常为`int main(void)`的定义形式，因为其启动源码文件已有汇编代码如下：
```
  bl  main
  bx  lr    
```

而常见的`main`函数定义方式为`int main(int argc, char *argv[])`，其用于处理命令行参数。在嵌入式中开发中，也可以引出`int function(argc, char *argv) { }`类型的命令行调试函数

其中，`argc`是一个整数，表示命令行参数的数量，并且该值至少为 1。（函数的名称算作一个参数）

`argv[0]`是函数名字符串，`argv[1]`到`argv[argc-1]`是命令行参数，当然了这些参数都是字符串


### 格式化输入输出

对于`printf`、`sprintf`等格式化输出函数
- 对于`%.2d %2d %02d`以及其它的数字指定宽度输出，其会在宽度不足时补足，但**若宽度超出，则会按照实际宽度输出**
- 对于`%s %2s %.2s`相关的字符串输出，当超出时，其中`%.2s`会截取前两位字符输出，其它按照实际宽度输出
- 部分测试代码及输出结果如下：
```c
int main()
{
        int a = 300;
        printf("a=%d\n", a);
        printf("a=%.2d\n", a);
        printf("a=%02d\n", a);

        a = 3;
        printf("a=%d\n", a);
        printf("a=%.2d\n", a);
        printf("a=%02d\n", a);

        char b[] = "hello, world";
        printf("b=%s\n", b);
        printf("b=%2s\n", b);
        printf("b=%.2s\n", b);

        char c[] = "he";
        printf("c=%s\n", c);
        printf("c=%4s\n", c);
        printf("c=%.4s\n", c);
        return 0;
}
```
输出结果如下：
```
a=300
a=300
a=300
a=3
a=03
a=03
b=hello, world
b=hello, world
b=he
c=he
c=  he
c=he
```



## 疑惑解答

### 如果强制转换调用函数指针的入参与实际函数入参的 数量或者类型 不一致，会如何？


函数参数的类型检查是在编译阶段进行的，如果传入的参数类型不正确，可能会导致未定义的行为错误。


函数指针的声明 int (*func)() 表示 func 是一个指向函数的指针，这个函数接受任意数量和类型的参数，并返回一个整数。这种声明方式在C89和C99标准中都是有效的。

在 C89/C99/GNU89/GNU99 编译标准中，可以通过声明`int (*func)()`函数指针，调用时传入可变的参数即可，但不建议这样做。建议使用可变参数列表来声明。


## 参考站点

- [c语言怎样设置一个函数，可以接受任何类型的参数并返回int型](https://www.zhihu.com/question/385731065)


