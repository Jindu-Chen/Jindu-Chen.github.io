---
title: C功能代码实现
date:
tags:
categories:
description: 
---





### ADC采集音量大小

```c
void DMA1_Channel1_IRQHandler(void)
{
    DMA_ClearFlag(DMA1_FLAG_GL1, DMA1);
    // sustain get max and min data,  save to buf, when user read, then empty it
    uint32_t adc_value = drv_adc_read(7);
    
    if(adc_value > adc_amplifier_max)   adc_amplifier_max = adc_value;
    if(adc_value < adc_amplifier_min)   adc_amplifier_min = adc_value;
}

// 读取音量值：每当展示完一个音量值的灯效周期后，调用该函数获取下一个音量值，并做灯效展示
uint32_t drv_adc_read_volume(void)
{
    uint32_t volume_value = adc_amplifier_max - adc_amplifier_min;
    logPrintln("waveform max_value=[%d]  min_value=[%d], volume=[%d]", adc_amplifier_max, adc_amplifier_min, volume_value);
    adc_amplifier_max = 0;
    adc_amplifier_min = 4095;
    return volume_value;
}
SHELL_EXPORT_CMD(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC)|SHELL_CMD_DISABLE_RETURN, drv_adc_read_volume, drv_adc_read_volume, drv_adc_read_volume);

```


