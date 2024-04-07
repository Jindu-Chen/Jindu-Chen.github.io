---
title: 嵌入式MCU开发问题记录
date: 2023-12-19
updated: 2023-12-19
tags:
categories:
- [MCU]
description: 记录、总结嵌入式开发中遇到的一些小问题，包括但不限于现象记录、原因分析以及解决思路等。
---


### 自定义打印函数，通过命令行调用能打印，但在中断函数执行却没有打印

如下函数源码：
```c
#define logPrintln(format, ...) \
        logWrite(LOG_ALL_OBJ, LOG_NONE, format "\r\n", ##__VA_ARGS__)

#define logPrinttext(format, ...) \
        logWrite(LOG_ALL_OBJ, LOG_NONE, format, ##__VA_ARGS__)

void drv_adc_read_total(void)
{
    logPrintln("aaa %d", drv_adc_read(1));
    logPrinttext("bbb");
    logPrintln("aaa %d", drv_adc_read(1));
}
SHELL_EXPORT_CMD(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC)|SHELL_CMD_DISABLE_RETURN, drv_adc_read_total, drv_adc_read_total, drv_adc_read_total);

void DMA_Channel6_IRQHandler(void)
{
    DMA_ClearFlag(DMA_FLAG_GL6, DMA);
    logDebug("read start");
    drv_adc_read_total();
    logDebug("read end");
}
```
在DMA_Channel6_IRQHandler中执行，仅没有 logPrinttext("bbb"); 这个输出
通过命令行调用drv_adc_read_total函数则有 logPrinttext("bbb"); 打印
```
中断函数执行结果如下：
D [0:11:49,025] DMA_Channel6_IRQHandler [208]: read start
aaa 65533
aaa 65533
D [0:11:49,025] DMA_Channel6_IRQHandler [210]: read end


命令行执行结果如下：
root:/$ drv_adc_read_total
aaa 65533
bbbaaa 65533
```

**问题调试分析：**
- 如果将`"\r\n"`同样加到 `logPrinttext`函数的宏定义中，那么在中断中也能打印此函数内容
- 在命令行中调用，无论怎样更改宏定义，都能完整打印。区别在于一个在正常环境，一个在中断环境中；一个带了`\r\n`拼接字符串，一个没带
- 同时把中断服务函数引出命令行调用，执行结果对比如下：
```
root:/$ DMA_Channel6_IRQHandler // 手动执行命令行调用，能够完整打印
D [0:00:17,989] DMA_Channel6_IRQHandler [210]: read start
aaa 65533
bbb 65533bbb
aaa 65533
D [0:00:17,989] DMA_Channel6_IRQHandler [212]: read end

// 中断执行
D [0:00:18,040] DMA_Channel6_IRQHandler [210]: read start
aaa 65533
bbb
aaa 65533
D [0:00:18,040] DMA_Channel6_IRQHandler [212]: read end
```

**问题解决：**
为尾行模式问题

由于使能了尾行模式，所以每次在调用`logWrite`函数输出时，都会紧接着将光标移到行首，而后输出终端用户等字符，覆盖当前行。**如果当前内容后面没有接上换行符，则会被覆盖。**

而当用户通过命令行调用函数时，由于是当前shell状态为执行命令行函数，所以不会启用尾行输出，因而不会覆盖当前行的内容。

---

### Switch Case语句里面定义的静态变量会自己变化

在一个case里面定义了一个`{}`作用域，而后定义一个静态变量，但有时发现其中某个for循环函数，明明没有对此变量进行修改，但是此变量的值却一直在变化。

**排查：**
考虑编译器优化等级、数据类型 u8 u16转换不当、等原因

**原因：**
最后发现为在 for 循环函数中有 循环给一个 数组进行赋值，然后**该数组下标越界**了，正好内存越界到 此静态变量的地址中。在对数组进行赋值操作或者对数组指针进行操作时，一定要注意可能的内存越界问题。

---
### 32MCU库函数应用问题

在本例中使用到 PB4引脚 复用为 USART2 的 TX 引脚，但是配置却不生效。最后发现，查看芯片数据手册得知：该引脚默认使能功能为 SW-JTAG 的烧录调试引脚之一，需要关闭此功能，代码如下：
```c
    RCC_EnableAPB2PeriphClk(RCC_APB2_PERIPH_AFIO, ENABLE);
    GPIO_ConfigPinRemap(GPIO_RMP3_USART2, ENABLE);              
    GPIO_ConfigPinRemap(GPIO_RMP_SW_JTAG_SW_ENABLE, ENABLE);    // 需要配置引脚重映射，增加此项配置
    RCC_EnableAPB2PeriphClk(RCC_APB2_PERIPH_GPIOB, ENABLE);
    GPIO_InitCtlStruct.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitCtlStruct.Pin = GPIO_PIN_4;
    GPIO_InitPeripheral(GPIOB, &GPIO_InitCtlStruct);
    GPIO_InitCtlStruct.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_InitCtlStruct.Pin = GPIO_PIN_5;
    GPIO_InitPeripheral(GPIOB, &GPIO_InitCtlStruct);
```
另外，在进行库函数开发时，特别要注意查看函数功能注释，比如`GPIO_ConfigPinRemap`该库函数，并不支持一次同时输入多个值如`GPIO_RMP3_USART2 | GPIO_RMP_SW_JTAG_SW_ENABLE`。


---

### 配置ADC-DMA的发送完成中断触发无效

**现象：**
- DMA、ADC皆已配置完成，adc能够正常采集，dma也能够正常地将adc的数据采集存放到指定的缓冲区中。此时添加使能相应的DMA-ADC通道发送完成中断，发现无效。

**原因：**
为初始化顺序不符合，正常的配置顺序应为： 初始化ADC外设及相应的复用IO->初始化DMA外设通道->配置DMA-ADC通道映射->清除DMA标志位->配置使能DMA通道标志触发中断->使能ADC-DMA访问请求->使能相应的DMA通道->配置NVIC使能DMA通道中断

以上，特别注意：ADC-DMA使能、DMA通断使能、DMA通道使能中断要放到最后.

另外，编写中断服务函数时，须要在其内清除相应的中断标志位，否则可能会无限重入中断，程序运行出现问题。


### 添加了bootloader的LiteOS应用程序没能跑起来
**原因分析：**
- 首先检查bootloader和app的链接脚本是否有误，确认bootloader跳转地址为0x8004000，而app链接脚本的FLASH起始地址为0x8004000无误
- 确认app中断向量SCB->VTOR配置正确
- 使用此前的bootloader + LiteOS APP、 gcc环境下的bootloader + 重构前的rt-thread App分别观察现象；发现gcc bootloader + rt-thread App能够正常运行，说明问题出现在LiteOS的应用程序上
- **可能是中断向量仍然未配置好、可能是程序实际并未以0x8004000为起始地址链接、可能是App程序中关于IAP部分程序没配置好地址、可能是LiteOS自定义的中断向量没有8字节对齐、可能是App部分大小超过FLASH限制**
- 另外，不排除bootloader中有读取擦除指定地址数据的操作，当App程序过大越界可能会被擦除部分数据
- 不排除bootloader与app的系统时钟初始化可能存在不一致问题，如两者工程的启动文件都会调用到`system_stm32xxxx.c`里的系统时钟初始化函数，进入到main函数，通常用户会再次初始化系统时钟。需要保持其时钟源配置一致，避免出现初始化错误
- 另外发现，`Reset_handler`函数的hex文件地址解析 比 在map文件中的地址加1.
- 在检查链接脚本时，发现如下代码：
```
_estack = 0x20024000;    /* end of RAM */
_sstack = 0x20000000;    /* start of RAM */
```
- 突发奇想，将栈顶指针`_estack = 0x20024000;`删减掉，加到如下链接脚本处，发现bootloader可以正常跳转运行了
```
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
```

**问题最终分析：**
- app的栈顶指针指向芯片的最大RAM地址，在无bootloader时，程序能够正常运行；加上了bootloader后，存在问题
- 此两者区别是：一个是由链接器自动计算堆栈的大小，一个是自己指定栈顶地址
- 进一步分析二者对程序可能的影响：


---

### J-Link仿真器正常，但点击烧录就工作异常

- MCU及元件上电工作，J-Link供电不足 -- 插上电源
- 芯片在读写保护状态，需要用特定软件解除读保护后，才可烧录

---
### 移植TIMER-PWM驱动时, 发现电机出现高频异响

**原因分析:**
- 经检查,高频异响是电机驱动输入pwm频率过高引起的
- 而该PWM驱动默认了MCU主频是72MHz, PWM时钟分频为72分频, 以1MHz的定时器时钟来作为基准
- 所以当该驱动移植到芯片主频为144MHz时, 使得PWM的频率为之前的两倍

**解决:**
- 修改该PWM驱动, 以使其更具通用性, 将时钟分频修改为变量形式 = 系统时钟频率 /1MHz,即可实现在不同主频芯片中的定时器基准频率为1MHz

---

### 应用LiteOS的定时器，出现bug：在软件定时器回调函数中给一个句柄写入消息，其中涉及到阻塞1s申请互斥量操作，但却一直无法写入

**原因分析：**
- 首先检查日志，发现每次回调函数能被正常执行，但写入操作一直失败。其中为延时获取互斥量失败，但实际上没有延时
- 怀疑可能是软件定时器本身就设计不支持阻塞接口操作，所以直接返回；可能是软件设计不当导致的互斥量一直被其它任务所占用
- 排查缩小范围

**解决：**
- 最终确认为在软件定时器线程中，不支持事件、信号量、互斥量相关的操作，相关内核组件运行时，会检查当前线程是否为定时器线程，是——则直接返回错误。


---

### 应用LiteOS信号量，阻塞100tick 等待，程序却卡死在阻塞等待处，另，串口UART4中断函数异常跑飞
部分源代码节选如下：
```c
  static uint32_t comm_board_sem;
  
  LOS_SemCreate(0, &comm_board_sem);
  
  LOS_SemPost(comm_board_sem);        // 在串口接收空闲中断处调用

  logVerbose("init");
      result = LOS_SemPend(comm_board_sem, LOS_MS2Tick(100));
  logVerbose("init"); // 未能执行到此处
```
通过断点调试，程序卡死在如下处：
```c
  /* ****************************************************************************
  Function    : ArchSysExit
  Description : Task exit function
  **************************************************************************** */
  LITE_OS_SEC_TEXT_MINOR VOID ArchSysExit(VOID)
  {
      (VOID)LOS_IntLock();
      while (1) {
      }
  }
```

**原因分析与解决步骤：**
1. 将该线程注释掉，系统能够正常运作，说明是此线程的设计问题
2. 考虑可能是此处的设计导致了系统线程调度出现问题，在该线程入口函数的for循环内加上断点打印，注射掉`LOS_SemPend(comm_board_sem, LOS_MS2Tick(100));`，再行调试
3. 发现程序又卡死在`LOS_TaskDelay()`中，此时精简后的线程入口函数源码如下：
   ```c
   static void comm_board_rx_task(void *parameter)
    {
        logVerbose("xxx");
        uint8_t recv_from_rb[256];
        int16_t recv_ret;
        uint8_t interpret_buf[256];
        err_enum unpack_ret = err_ok;
        for (;;)
        {
            logVerbose("xxx");
            LOS_TaskDelay(100);
            logVerbose("xxx");      // 无法执行到此处，程序卡死
        }
    }
   ```
4. 此前该段入口函数在rt-thread中运行无误，在LiteOS中却无法运行，推测可能是在for循环之前定义了几个大数组，其存储位置在线程上下文即 线程栈中
5. 而系统崩掉的原因应该是线程调度时没有考虑到如此大的数据入栈出栈，从而导致出错
6. 经过排查和不断的代码注释运行，最终发现是串口的DMA收发初始化，导致程序无法运行 -- 判断是uart的驱动问题
7. 再次排查此前已经验证通过的uart驱动，代码对比无误，进行常规分析：可能是LiteOS的中断注册存在问题、可能是代码的驱动存在冲突、可能中断服务函数并没正确对应注册、可能是初始化步骤存在问题
8. 经过验证，发现是UART4可能存在问题，另外发现一个奇葩现象，如果开启了UART1驱动的宏定义，UART4能够正常运行；直接现象是USART1的中断服务函数影响到了UART4的中断，但USART1中断函数从没被调用
9.  毫不相关的USART1函数，却直接导致了UART4的使用问题；
10. 突发奇想，把编译优化等级从O1改为O0，程序就能正常运行了，有可能是**这部分驱动程序编写不够健壮，也可能是编译选项存在应用不当问题**
11. 继续缩小代码范围，发现了注释与调用某一个变量，会导致程序能运行与否，另外查看其map文件对比，只一个变量的区别 -- **怀疑可能是字节对齐的问题**
12. 此外，发现编译选项 **-fdata-sections** 的删减，可使程序正确运行，通过对比删减前后的map文件，发现在函数`drv_uart_init()`的text段大小差了4个字节；同时在当前文件bss段变量大小也差了4个字节
13. 考虑代码可能存在以下几个问题：
    全局变量与外部变量重名，导致的链接错误；
    变量可能未被正确初始化，或存在一些漏洞


---
### 从N32L406芯片的验证通过的 DMA-USART TX RX驱动到N32WB452上, 串口DMA发送没能跑起来

**原因分析:**
- 通过GDB调试发现DMA_GetCurrDataCounter()的数值一直没变,说明是DMA通道的数据没能发送出去
- 查看芯片手册确认DMA通道映射无误,对比确认DMA初始化流程一致
- 不同点在于N32WB452加上了LiteOS,在DMA RX驱动处加上了 添加DMA接收完成和UART空闲中断服务函数

**问题解决步骤:**
-  重复比对初始化步骤,以及断点调试,查看DMA的寄存器配置信息,发现DMA寄存器没有被正确初始化
-  确定为驱动初始化的原因, 再次重复确认DMA-UART的结构体初始化信息,发现为信息结构体的位置错位了
```c
struct drv_uart_dev uart2 =
{
    USART2,
    USART2_IRQn,
    {
        DMA1_CH6,
        DMA1,
        DMA1_FLAG_GL6,
        DMA1_Channel6_IRQn,
        0,                      // 此处缺少一个 0 变量,导致下面的变量错位
        DMA1_CH7,
        DMA1,
    },
};

struct drv_uart_dev
{
    USART_Module *uart_device;
    IRQn_Type irq;
    struct drv_uart_dma
    {
        DMA_ChannelType *rx_ch;
        DMA_Module *rx_dma_type;
        uint32_t rx_gl_flag;
        uint8_t rx_irq_ch;
        uint32_t setting_dma_flag;
        uint16_t last_recv_index;

        DMA_ChannelType *tx_ch;
        DMA_Module *tx_dma_type;
    } dma;
    void (*user_cb)(enum uart_dev_name uart_num, uint16_t length);
    structUsertData rx_buf_dev;
    structUserTxData tx_buf_dev;
    enum uart_dev_name uart_num;
};
```

---
### 移植letter_shell组件和LiteOS, 开机上电立即打印了letter_shell版本信息, 但其后的打印信息阻塞延时了几秒钟, 在这期间内, shell完全没任何反应
**现象分析:**
 - 系统时基是从延时后才开始累计, 说明在此前Systick没有跑起来
 - 一上电就能输出信息

**问题解决步骤:**
 - 在上电正常输出日志处, 和延时处添加日志打印,不断往中间逼近,查找是哪个函数导致的阻塞延时
 - 通过日志逼近打印,发现是上电初始化时与充电ic的iic通信导致的阻塞延时,并且iic没有通信成功
 - iic与SGM41528初始化时序时败,阻塞占用了接近2s的时间,接下来需要排查为何移植LiteOS的 iic通信初始化 超时失败
 - 通过比较排查,已知: iic通信在刚上电时通信失败,但上电一段时间后,通信正常; 但与移植LiteOS前相比,iic的初始化调用逻辑并没有改变, 接下来需要排查为什么刚上电时的iic初始化失败
 - 通过调试发现iic初始化时是有ACK回复的,但耗时高
 - 最后发现是iic驱动中的微秒级延时代码存在问题, 而LiteOS在初始化Systick会配置Systick->load = 0xFFFFFF, 之后启动系统时再次配置为 SystemCoreClock/1000, 从而使得问题暴露
 ```c
 static void n32_udelay(uint32_t us)
 {
     uint32_t ticks;
     uint32_t told, tnow, tcnt = 0;
     uint32_t reload = SysTick->LOAD;

     // ticks = us * reload / (1000000 / DRV_TICK_PER_SECOND);
     ticks = us * (SystemCoreClock / 1000000); //需要的节拍数
     told = SysTick->VAL;
     while (1)
     {
         tnow = SysTick->VAL;
         if (tnow != told)
         {
             if (tnow < told)
             {
                 tcnt += told - tnow;
             }
             else
             {
                 tcnt += reload - tnow + told;
             }
             told = tnow;
             if (tcnt >= ticks)
             {
                 break;
             }
         }
     }
 }
 ```
 其中`ticks = us * reload / (1000000 / DRV_TICK_PER_SECOND);`更新为:`ticks = us * (SystemCoreClock / 1000000);`

---


