---
title: LiteOS适配及开发笔记
date: 2023-11-29 11:48:26
tags:
- [LiteOS]
- [OpenHarmony]
categories:
- [OpenHarmony]
- [LiteOS]
description: 基于码云OpenHarmony的kernel_liteos_m项目源码，进行国民技术N32G452芯片的移植开发，应用gcc+make命令行开发环境，本文主要介绍个人初次接触LiteOS的适配开发过程、心得，和不得不说的一些槽点。
---


## 环境搭建

暂略

通常用户对LiteOS的初始化调用如下：
```
main():
    LOS_KernelInit();       // 内核初始化
    LiteOS_Task_sample();   // 用户业务代码，创建初始任务等
    LOS_Start();            // 开启调度
```


## 开发适配

### LiteOS-M的shell组件适配

LiteOS-M添加shell组件，主要思路为：
- 适配对接指定的串口读写接口，并配置串口接收中断触发写指定的shell事件：`g_shellInputEvent`，使得shell线程能够接收事件读取uart数据
- 在配置文件中启用`LOSCFG_USE_SHELL`相关宏定义，等，使得相关源代码编译链接
- 调用指定的初始化接口`LosShellInit()`
<br>

**以下为本人之适配代码片段摘选，驱动接口参考实现即可，不拘泥于形式：**
```
// uart.h
extern EVENT_CB_S g_shellInputEvent;

VOID ShellUartInit(VOID);
INT32 UartPutc(INT32 c, VOID *file);
uint8_t UartGetc(VOID);

// uart_shell.c

// 适配接口
INT32 UartPutc(INT32 ch, VOID *file)
{
    char RL = '\r';
    if (ch == '\n')
    {
        drv_uart_write(DEV_UART1, (uint8_t *)&RL, 1);
    }
    drv_uart_write(DEV_UART1, (uint8_t *)&ch, 1);
    return 0;
}

// 适配接口实现，由用户自己实现
uint8_t UartGetc(void)
{
    uint8_t data;

    if (drv_uart_read(DEV_UART1, &data, 1) != 0)
    {
        return data;
    }
    else
    {
        return 0;
    }
}

// 注册uart1接收中断，当收到一帧数据时，向shell线程阻塞等的事件写标志，触发shell线程读取接收数据并处理
void uart1_receive_cb(enum uart_dev_name uart_num, uint16_t length)
{
    (void)LOS_EventWrite(&g_shellInputEvent, 0x1);
}

VOID ShellUartInit(VOID)
{
    drv_uart_set_rx_indicate(DEV_UART1, uart1_receive_cb);
    LosShellInit();     // 由用户调用，初始化shell线程
}
```

### LiteOS中断管理之SysTick

主要介绍LiteOS如何重建中断映射向量、自定义SysTick中断服务函数、初始化SysTick时基

#### LiteOS中断映射表初始化
- 函数调用顺序：`LOS_KernelInit() -> ArchInit() -> HalHwiInit()`

如下代码`SCB->VTOR = (UINT32)(UINTPTR)hwiForm;`将中断向量映射基地址设为自定义变量地址
```
# arch/arm/cortex-m4/iar/los_interrupt.c > HalHwiInit()

#if (LOSCFG_USE_SYSTEM_DEFINED_INTERRUPT == 1)
    UINT32 index;
    HWI_PROC_FUNC *hwiForm = (HWI_PROC_FUNC *)ArchGetHwiFrom();
    hwiForm[0] = 0;             /* [0] Top of Stack */
    hwiForm[1] = (HWI_PROC_FUNC)Reset_Handler; /* [1] reset */
    for (index = 2; index < OS_VECTOR_CNT; index++) { /* 2: The starting position of the interrupt */
        hwiForm[index] = (HWI_PROC_FUNC)HalHwiDefaultHandler;
    }
    /* Exception handler register */
    hwiForm[NonMaskableInt_IRQn + OS_SYS_VECTOR_CNT]   = (HWI_PROC_FUNC)HalExcNMI;
    hwiForm[HARDFAULT_IRQN + OS_SYS_VECTOR_CNT]        = (HWI_PROC_FUNC)HalExcHardFault;
    hwiForm[MemoryManagement_IRQn + OS_SYS_VECTOR_CNT] = (HWI_PROC_FUNC)HalExcMemFault;
    hwiForm[BusFault_IRQn + OS_SYS_VECTOR_CNT]         = (HWI_PROC_FUNC)HalExcBusFault;
    hwiForm[UsageFault_IRQn + OS_SYS_VECTOR_CNT]       = (HWI_PROC_FUNC)HalExcUsageFault;
    hwiForm[SVCall_IRQn + OS_SYS_VECTOR_CNT]           = (HWI_PROC_FUNC)HalExcSvcCall;
    hwiForm[PendSV_IRQn + OS_SYS_VECTOR_CNT]           = (HWI_PROC_FUNC)HalPendSV;
    hwiForm[SysTick_IRQn + OS_SYS_VECTOR_CNT]          = (HWI_PROC_FUNC)SysTick_Handler;

    /* Interrupt vector table location */
    SCB->VTOR = (UINT32)(UINTPTR)hwiForm;
#endif
```

#### 配置SysTick中断服务函数
- 函数调用顺序：`LOS_KernelInit() -> OsTickTimerInit() -> OsTickTimerInit() -> g_sysTickTimer->init(OsTickHandler) -> SysTickStart(OsTickHandler)`

```
# kernel/src/los_tick.c
LITE_OS_SEC_TEXT VOID OsTickHandler(VOID)
{
#if (LOSCFG_BASE_CORE_TICK_WTIMER == 0)
    OsUpdateSysTimeBase();
#endif

    LOS_SchedTickHandler();
}
```

```
# arch/arm/cortex-m4/gcc/los_timer.c
STATIC UINT32 SysTickStart(HWI_PROC_FUNC handler)
{
    UINT32 ret;
    ArchTickTimer *tick = &g_archTickTimer;

    tick->freq = OS_SYS_CLOCK;

#if (LOSCFG_USE_SYSTEM_DEFINED_INTERRUPT == 1)
#if (LOSCFG_PLATFORM_HWI_WITH_ARG == 1)
    OsSetVector(tick->irqNum, handler, NULL);
#else
    OsSetVector(tick->irqNum, handler);     // 设置SysTick的中断向量
#endif
#endif

    ret = SysTick_Config(LOSCFG_BASE_CORE_TICK_RESPONSE_MAX);   // 在此处配置的SysTick中断无效，后续开启调度时会reload
    if (ret == 1) {
        return LOS_ERRNO_TICK_PER_SEC_TOO_SMALL;
    }

    return LOS_OK;
}
```

```
# arch/arm/common/los_common_interrupt.c
VOID OsSetVector(UINT32 num, HWI_PROC_FUNC vector)
{
    if ((num + OS_SYS_VECTOR_CNT) < OS_VECTOR_CNT) {
        g_hwiForm[num + OS_SYS_VECTOR_CNT] = HalInterrupt;
        g_hwiHandlerForm[num + OS_SYS_VECTOR_CNT] = vector;
    }
}
```


#### 初始化SysTick时基
- 函数调用顺序：`LOS_Start() -> ArchStartSchedule() -> OsSchedStart() -> OsSchedSetNextExpireTime() -> OsTickTimerReload() -> SysTickReload()`

```
# arch/arm/cortex-m4/gcc/los_timer.c
# 调用此函数，传入的值实际上是
STATIC UINT64 SysTickReload(UINT64 nextResponseTime)
{
    if (nextResponseTime > g_archTickTimer.periodMax) {
        nextResponseTime = g_archTickTimer.periodMax;
    }
    SysTick->CTRL &= ~SysTick_CTRL_ENABLE_Msk;
    SysTick->LOAD = (UINT32)(nextResponseTime - 1UL); /* set reload register */
    SysTick->VAL = 0UL; /* Load the SysTick Counter Value */
    NVIC_ClearPendingIRQ(SysTick_IRQn);
    SysTick->CTRL |= SysTick_CTRL_ENABLE_Msk;
    return nextResponseTime;
}
```

#### 初始化调用逻辑总结

1. 在`arch/arm/cortex-m4/iar/los_interrupt.c > HalHwiInit()`中执行中断向量表初始化
   >但在此处初始化的SysTick_Handler向量是弱函数，并且其在后面会被替换
2. 在`LOS_KernelInit() -> OsTickTimerInit() -> OsTickTimerInit()`重设SysTick中断服务函数
3. 在`LOS_Start() -> ArchStartSchedule() -> OsSchedStart()`中调用 重置SysTick中断时基为`系统时钟/调度间隔Tick值 （如：时钟源 / 1000 = 1ms时基）`
4. 最后开启系统调度

#### 补充说明
- LiteOS的内核初始化和系统启动都对Systick进行了设置，内核初始化时设置了Systick的重装载值为24位最大值，即0xFFFFFF
- 系统启动时，重置Systick累计的节拍，并设置重装载值为 `SystemCoreClock / 1000`，其中SystemCoreClock为系统时钟源频率。



## 槽点

#### shell交互-uart-dma中断服务函数的配置

- 芯片上电执行`LOS_KernelInit()`，其中会重新定义中断向量，但在这个过程之前和之后都会有日志打印
- 如果日志打印是用uart + DMA完成的，其中需要注册中断服务函数。
- 但是uart的初始化打开应该在系统初始化前执行，但中断服务函数要在系统初始化后才能注册。给编程带来不便


#### 软件定时器的使用

- 鸡肋的软件定时器，只能在初始化时定好时间周期，后续不可更改。
- 其提供的API过少，只有初始化、开始、结束的有限几个接口。用户无法获取当前定时器剩余tick，无法重新设置定时周期，无法在使用过程中更改定时器的 单次或者周期定时等属性
- 该软件定时器中，不支持申请互斥量
  - 在写私有消息队列过程中，涉及临界区操作，需要申请互斥量，但在定时器的回调函数中申请互斥量会直接返回错误。

- 定时器任务创建源码如下:
    如下所示：其中系统在创建定时器线程时会或上`OS_TASK_FLAG_SYSTEM_TASK`这个标志，然后在申请互斥量、信号量、事件时，都会检查当前是否运行在定时器任务中，如果为真，则直接返回错误
  ```
    LITE_OS_SEC_TEXT_INIT UINT32 OsSwtmrTaskCreate(VOID)
    {
        UINT32 ret;
        TSK_INIT_PARAM_S swtmrTask;

        // Ignore the return code when matching CSEC rule 6.6(4).
        (VOID)memset_s(&swtmrTask, sizeof(TSK_INIT_PARAM_S), 0, sizeof(TSK_INIT_PARAM_S));

        swtmrTask.pfnTaskEntry    = (TSK_ENTRY_FUNC)OsSwtmrTask;
        swtmrTask.uwStackSize     = LOSCFG_BASE_CORE_TSK_SWTMR_STACK_SIZE;
        swtmrTask.pcName          = "Swt_Task";
        swtmrTask.usTaskPrio      = 0;
        ret = LOS_TaskCreate(&g_swtmrTaskID, &swtmrTask);
        if (ret == LOS_OK) {
            OS_TCB_FROM_TID(g_swtmrTaskID)->taskStatus |= OS_TASK_FLAG_SYSTEM_TASK;
        }
        return ret;
    }
  ```
  互斥量操作时，会进行以下检查：
  ```
    STATIC_INLINE UINT32 OsMuxValidCheck(LosMuxCB *muxPended)
    {
        if (muxPended->muxStat == OS_MUX_UNUSED) {
            return LOS_ERRNO_MUX_INVALID;
        }

        if (OS_INT_ACTIVE) {
            return LOS_ERRNO_MUX_IN_INTERR;
        }

        if (g_losTaskLock) {
            PRINT_ERR("!!!LOS_ERRNO_MUX_PEND_IN_LOCK!!!\n");
            return LOS_ERRNO_MUX_PEND_IN_LOCK;
        }

        if (g_losTask.runTask->taskStatus & OS_TASK_FLAG_SYSTEM_TASK) {
            return LOS_ERRNO_MUX_PEND_IN_SYSTEM_TASK;
        }

        return LOS_OK;
    }
  ```

#### 空闲任务函数
    查看LiteOS之原版api，是存在创建空闲任务钩子函数的，但最新版代码已经删除，代码如下：
由以下代码可知，空闲任务作用仅为回收结束任务资源，以及可选的电源管理。
    - 个人操作：增设一个专门的低优先级任务来执行一些空闲操作
```
LITE_OS_SEC_TEXT VOID OsIdleTask(VOID)
{
    while (1) {
        OsRecycleFinishedTask();

        if (PmEnter != NULL) {
            PmEnter();
        } else {
            (VOID)ArchEnterSleep();
        }
    }
}

LITE_OS_SEC_TEXT_INIT UINT32 OsIdleTaskCreate(VOID)
{
    UINT32 retVal;
    TSK_INIT_PARAM_S taskInitParam;
    // Ignore the return code when matching CSEC rule 6.6(4).
    (VOID)memset_s((VOID *)(&taskInitParam), sizeof(TSK_INIT_PARAM_S), 0, sizeof(TSK_INIT_PARAM_S));
    taskInitParam.pfnTaskEntry = (TSK_ENTRY_FUNC)OsIdleTask;
    taskInitParam.uwStackSize = LOSCFG_BASE_CORE_TSK_IDLE_STACK_SIZE;
    taskInitParam.pcName = "IdleCore000";
    taskInitParam.usTaskPrio = OS_TASK_PRIORITY_LOWEST;
    retVal = LOS_TaskCreateOnly(&g_idleTaskID, &taskInitParam);
    if (retVal != LOS_OK) {
        return retVal;
    }

    OsSchedSetIdleTaskSchedParam(OS_TCB_FROM_TID(g_idleTaskID));
    return LOS_OK;
}

```


## 参考站点


