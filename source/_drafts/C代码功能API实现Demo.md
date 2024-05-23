---
title: C语言功能代码API实现
date:
tags:
categories:
description: 
---


### hsv_to_rgb转换算法接口实现

```c
#include <math.h>

/*
* 将HSV颜色转换为RGB颜色
* hue,色调:0-360; saturation,纯度:0-1; value,明度:0-1
* r,g,b,RGB颜色，此值范围为0-255
*/
void hsv_to_rgb(float h, float s, float v, uint8_t *r, uint8_t *g, uint8_t *b)
{
   float f, x, y, z;
   int i;
   v *= 255.0;
   if (s == 0.0) {
      *r = *g = *b = (uint8_t)v;
   } else {
      while (h < 0)
      h += 360;
      h = fmod(h, 360) / 60.0;
      i = (int)h;
      f = h - i;
      x = v * (1.0 - s);
      y = v * (1.0 - (s * f));
      z = v * (1.0 - (s * (1.0 - f)));
      switch (i) {
         case 0: *r = v; *g = z; *b = x; break;
         case 1: *r = y; *g = v; *b = x; break;
         case 2: *r = x; *g = v; *b = z; break;
         case 3: *r = x; *g = y; *b = v; break;
         case 4: *r = z; *g = x; *b = v; break;
         case 5: *r = v; *g = x; *b = y; break;
      }
   }
}
```

使用Demo如下：
```c
void led_set_poll(void)
{    
    uint8_t red,green,blue;
    
    static float hue = 0;
    hue = fmodf(hue + 1.0, 360.0);  // 0-360 色调循环
    hsv_to_rgb(hue, 1.0, 1.0, &red, &green, &blue);
    display_board_rgb_color_set(red, green, blue);
}
```



### 按键检测代码


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


