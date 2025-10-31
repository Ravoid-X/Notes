## 概述
1. PWM(Pulse Width Modulation) 叫做 脉冲宽度调制，是一种利用数字输出来控制模拟电路的方法.
2. 核心思想是在一个固定的周期内，通过改变“高电平”持续的时间，来改变输出信号的平均电压。
## 关键参数
### 频率 (Frequency)
信号每秒钟重复的次数，单位是赫兹。在一个 PWM 应用中，频率通常是固定的。
### 占空比 (Duty Cycle)
在一个周期内，高电平持续时间占总周期的百分比，这是主要调节的参数。
## 关键寄存器
### 预分频器 (Prescaler, PSC)
1. 定时器有一个时钟源（例如来自 APB1 或 APB2 总线，假设为 84MHz）
2. PSC 是一个 16 位的值，它对这个时钟源进行分频
3. 定时器计数时钟 (Timer_Clock) = (APB_Clock) / (PSC + 1)
4. PSC + 1 是因为 PSC 值为 0 意味着 1 分频（不分频）
### 自动重载寄存器 (Auto-Reload Register, ARR)
1. 一个 16 位（或 32 位，取决于定时器）的值，定义了 PWM 的周期
2. 定时器的计数器会从 0 开始，一直计数到 ARR 设定的值，然后复位为 0（在向上计数模式下）
3. PWM 频率 = Timer_Clock / (ARR + 1)
4. ARR + 1 是因为从 0 计数到 ARR 实际上是 ARR + 1 个计数周期
### 捕获/比较寄存器 (Capture/Compare Register, CCR_x)
1. 定义脉冲宽度或占空比的关键
2. 定时器的 CNT 寄存器在计数时，会不断地与 CCR_x 寄存器（例如 CCR1 对应通道 1）的值进行比较
### PWM 模式
1. 最常用的是 PWM 模式 1
2. 在向上计数模式下，PWM 模式 1 的规则是：当 CNT < CCR_x 时，输出高电平；当 CNT >= CCR_x 时，输出低电平
3. PWM 模式 2 则相反
## 实践
### 
```
#include "stm32f4xx.h"
#include "stm32f4xx_gpio.h"
#include "stm32f4xx_rcc.h"
#include "stm32f4xx_tim.h"

void PWM_Init(void){
    GPIO_InitTypeDef GPIO_InitStruct;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
    TIM_OCInitTypeDef TIM_OCInitStruct;
    // 1. 使能时钟
    RCC_AHB1PeriphClockCmd(RCC_AHB1Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);
    // 2. 配置GPIO
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_6;              // PA6
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF;           // 复用功能
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_100MHz;     // 100MHz
    GPIO_InitStruct.GPIO_OType = GPIO_OType_PP;         // 推挽输出
    GPIO_InitStruct.GPIO_PuPd = GPIO_PuPd_UP;           // 上拉
    GPIO_Init(GPIOA, &GPIO_InitStruct);
    // 将PA6连接到TIM3_CH1
    GPIO_PinAFConfig(GPIOA, GPIO_PinSource6, GPIO_AF_TIM3);
    // 3. 配置定时器基本参数
    TIM_TimeBaseInitStruct.TIM_Prescaler = 84 - 1;       // 84MHz/84 = 1MHz
    TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInitStruct.TIM_Period = 1000 - 1;        // 1MHz/1000 = 1kHz PWM频率
    TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseInitStruct);
    // 4. 配置PWM模式
    TIM_OCInitStruct.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStruct.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStruct.TIM_OCPolarity = TIM_OCPolarity_High;
    TIM_OCInitStruct.TIM_Pulse = 500;     // 初始占空比50% (500/1000)
    TIM_OC1Init(TIM3, &TIM_OCInitStruct);
    TIM_OC1PreloadConfig(TIM3, TIM_OCPreload_Enable);
    // 5. 使能定时器
    TIM_ARRPreloadConfig(TIM3, ENABLE);
    TIM_Cmd(TIM3, ENABLE);
}
// 修改PWM占空比
void PWM_SetDutyCycle(uint16_t dutyCycle)
{
    // dutyCycle范围: 0-1000 (对应0-100%)
    TIM_SetCompare1(TIM3, dutyCycle);
}

int main(void){
    PWM_Init();
    while(1){
        for(int i = 0; i <= 1000; i += 10){
            PWM_SetDutyCycle(i);
            for(int j = 0; j < 100000; j++); // 简单延时
        }
        for(int i = 1000; i >= 0; i -= 10){
            PWM_SetDutyCycle(i);
            for(int j = 0; j < 100000; j++); // 简单延时
        }
    }
}
```