---
title: 嵌入式自动初始化
date: 2023-12-22
tags:
categories:
description: 主要讲述MCU如何具体实现自动初始化的几种方式，基本原理为：给所有需要初始化的函数声明一个段，在段的首末设定锚点，通过执行首末地址之间的所有函数指针，从而实现隐式的函数初始化。
---


## 背景

### 普通驱动初始化方式

常见的编写方式为 每个模块在其文件内部编写初始化函数实例，然后通过头文件导出初始化函数接口；主任务包含各模块的头文件，然后逐一调用其初始化函数<br>

这种初始化操作有以下特点：
- 显式的初始化调用较为直观，可以清楚地掌握程序的初始化流程顺序
- 头文件处理繁琐、不利于模块间的解耦合
- 编码难度较低

### 自动初始化方式

与常用初始化相对的即是隐式初始化，隐式初始化不用暴露模块的初始化接口，可以更好地解耦合。<br>

但相对应的，其初始化函数的调用、以及执行顺序不好把握<br>

另外，这种隐式的声明给审阅代码、移植工程也带来一定不便，开发者需要花费时间掌握工程的隐式初始化接口。<br>

#### Linux下的驱动初始化

Linux使用**模块机制**来实现驱动程序的自动初始化<br>
- 模块是Linux内核中的一个可加载组件，其可以包含驱动程序、函数库或其它代码
- 驱动程序模块加载到内核中时，内核会调用其初始化函数执行初始化工作，如：注册驱动程序的设备节点、初始化驱动程序的资源等
<br>

Linux提供了两个宏：`module_init()`、`module_exit()`分别用于注册驱动程序的初始化和卸载函数
- 如`module_init(init_function);`
- 如：
  ```
  static int __init my_driver_init(void)
  {
      // 完成驱动程序的初始化工作

      return 0;
  }
  module_init(my_driver_init);
  ``` 
- `module_init()`**宏将驱动初始化函数的地址存储到一个全局变量中**
- 内核启动时会调用`kernel_init()->init_module()->遍历执行所有通过module_init()注册的初始化函数`


## 前提知识

- `__attribute__((used))`是一个函数属性，其告诉编译器保留函数在目标文件中，即使其没被调用
- `__attribute__((section("name")))`属性用于将变量或函数放置在指定的段

## Keil环境下的自动初始化

- Keil的 ARM链接器会生成用于标识段首末地址的变量，如`initcall0init$$Base`为段名`initcall0init`的首地址；`initcall0init$$Limit`表示末地址

- 段的声明会将函数指针存储在输出文件的Data段中，属性为RO只读


**应用代码示例如下：**
- 以下为实现自动初始化的头文件
  - `act_auto_initialize()`是用以实现自动初始化的调用
  - 4个宏定义是对需要被调用的函数指针定义了4个显式执行次序
```
    #define  __used  __attribute__((__used__))

    typedef void (*initcall_t)(void);

    #define __define_initcall(fn, id) \
        static const initcall_t __initcall_##fn##id __used \
        __attribute__((__section__("initcall" #id "init"))) = fn; 

    #define INIT_SYSTEM_EXPORT(fn)      __define_initcall(fn, 0)
    #define INIT_BOARD_EXPORT(fn)       __define_initcall(fn, 1)
    #define INIT_DEVICE_EXPORT(fn)      __define_initcall(fn, 2)
    #define INIT_APP_EXPORT(fn)         __define_initcall(fn, 3)


    void act_auto_initialize(void);
```
- 以下为自动初始化的实现
```  
    void act_auto_initialize(void)
    {
        extern initcall_t initcall0init$$Base[];
        extern initcall_t initcall0init$$Limit[];
        extern initcall_t initcall1init$$Base[];
        extern initcall_t initcall1init$$Limit[];
        extern initcall_t initcall2init$$Base[];
        extern initcall_t initcall2init$$Limit[];
        extern initcall_t initcall3init$$Base[];
        extern initcall_t initcall3init$$Limit[];
        
        initcall_t *fn;
        
        for (fn = initcall0init$$Base;
                fn < initcall0init$$Limit;
                fn++)
        {
            if(fn)
                (*fn)();
        }
        
        for (fn = initcall1init$$Base;
                fn < initcall1init$$Limit;
                fn++)
        {
            if(fn)
                (*fn)();
        }
        
        for (fn = initcall2init$$Base;
                fn < initcall2init$$Limit;
                fn++)
        {
            if(fn)
                (*fn)();
        }
        
        for (fn = initcall3init$$Base;
                fn < initcall3init$$Limit;
                fn++)
        {
            if(fn)
                (*fn)();
        }
    }
  ```

## 适用于GCC环境下的自动初始化

GCC环境下，可以采用事先定义两个静态函数的段，将需要初始化的函数指针声明到这个段的中间，通过执行两个静态函数地址之间的所有函数指针即可。<br>

**以下源码示例片段摘选自rt-thread：**
```
#define SECTION(x)                  __attribute__((section(x)))
typedef int (*init_fn_t)(void);
#define INIT_EXPORT(fn, level)                                                       \
    RT_USED const init_fn_t __rt_init_##fn SECTION(".rti_fn." level) = fn      // 定义一个函数指针常量，并声明到指定的段名中

static int rti_start(void)
{
    return 0;
}
INIT_EXPORT(rti_start, "0");
static int rti_board_start(void)
{
    return 0;
}
INIT_EXPORT(rti_board_start, "0.end");
static int rti_board_end(void)
{
    return 0;
}
INIT_EXPORT(rti_board_end, "1.end");
static int rti_end(void)
{
    return 0;
}
INIT_EXPORT(rti_end, "6.end");

// 系统在初始化时调用此函数
void rt_components_init(void)  
{
  volatile const init_fn_t *fn_ptr;

  for (fn_ptr = &__rt_init_rti_board_start; fn_ptr < &__rt_init_rti_board_end; fn_ptr++)
  {
      (*fn_ptr)();
  }
  
  volatile const init_fn_t *fn_ptr;

  for (fn_ptr = &__rt_init_rti_board_start; fn_ptr < &__rt_init_rti_board_end; fn_ptr++)
  {
      (*fn_ptr)();
  }
}
```
**以下为初始化函数应用接口，宏定义声明传入函数指针(函数名)即可：**
```
#define INIT_BOARD_EXPORT(fn)           INIT_EXPORT(fn, "1")
#define INIT_PREV_EXPORT(fn)            INIT_EXPORT(fn, "2")
#define INIT_DEVICE_EXPORT(fn)          INIT_EXPORT(fn, "3")
#define INIT_COMPONENT_EXPORT(fn)       INIT_EXPORT(fn, "4")
#define INIT_ENV_EXPORT(fn)             INIT_EXPORT(fn, "5")
#define INIT_APP_EXPORT(fn)             INIT_EXPORT(fn, "6")

//应用示例
void function_need_to_init(void)
{
  // xxx
}
INIT_BOARD_EXPORT(function_need_to_init);
```
<br>

## 注意项

**不同的段名在内存映射中的排序：**
- 通常，map文件中的段是按照首字母顺序排序的
- 另，依照rt-thread方式所编译生成`.map`文件的段排序如下，供参考：
```
.rti_fn.0                                0x08018e6c   Section        4  components.o(.rti_fn.0)
__tagsym$$used                           0x08018e6c   Number         0  components.o(.rti_fn.0)
.rti_fn.0.end                            0x08018e70   Section        4  components.o(.rti_fn.0.end)
__tagsym$$used                           0x08018e70   Number         0  components.o(.rti_fn.0.end)
.rti_fn.1                                0x08018e74   Section        4  user_board_interface.o(.rti_fn.1)
__tagsym$$used                           0x08018e74   Number         0  user_board_interface.o(.rti_fn.1)
.rti_fn.1.end                            0x08018e78   Section        4  components.o(.rti_fn.1.end)
__tagsym$$used                           0x08018e78   Number         0  components.o(.rti_fn.1.end)
.rti_fn.6                                0x08018e7c   Section        4  shell.o(.rti_fn.6)
__tagsym$$used                           0x08018e7c   Number         0  shell.o(.rti_fn.6)
.rti_fn.6.end                            0x08018e80   Section        4  components.o(.rti_fn.6.end)
```
<br>

**初始化函数指针声明在同一个段中的顺序：**
- 实际上，此种自动初始化是函数指针的段声明，本质上是变量的声明
- 同一源文件中，其声明顺序按照它们在源代码中的布局顺序进行排列
- 不同源文件中，为按照源文件的顺序
  - 编译时可以使用`-M`选项，生成包含目标文件依赖关系的文件，通过该本文可以查看源文件顺序
  - 另外，可以通过`.map`文件查看段的声明顺序，从而知道初始化顺序
<br>

**编译选项之 函数分段 对初始化顺序的影响：**


