---
title: LiteOS中断管理之SysTick
date: 2023-11-29
tags:
categories:
- [LiteOS]
- [MCU]
description: 主要介绍LiteOS如何重建中断映射向量、自定义SysTick中断服务函数、初始化SysTick时基
---

通常用户对LiteOS的初始化调用如下：
```
main():
    LOS_KernelInit();       // 内核初始化
    LiteOS_Task_sample();   // 用户业务代码，创建初始任务等
    LOS_Start();            // 开启调度
```

# LiteOS的中断函数映射管理

## LiteOS中断映射表初始化
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

# 配置SysTick中断服务函数
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


# 初始化SysTick时基
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

# 初始化调用逻辑总结

1. 在`arch/arm/cortex-m4/iar/los_interrupt.c > HalHwiInit()`中执行中断向量表初始化
   >但在此处初始化的SysTick_Handler向量是弱函数，并且其在后面会被替换
2. 在`LOS_KernelInit() -> OsTickTimerInit() -> OsTickTimerInit()`重设SysTick中断服务函数
3. 在`LOS_Start() -> ArchStartSchedule() -> OsSchedStart()`中调用 重置SysTick中断时基为`系统时钟/调度间隔Tick值 （如：时钟源 / 1000 = 1ms时基）`
4. 最后开启系统调度

# 勘误
- 初读代码，凭个人理解，还有很多细节并没看，后续补充

