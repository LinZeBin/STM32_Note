# GPIO与外设基础

## 1. GPIO基本原理

### 1.1 GPIO引脚模式

```
GPIO Mode
├── Input Mode
│   ├── Floating (无上下拉)
│   ├── Pull-Up (上拉)
│   └── Pull-Down (下拉)
├── Output Mode
│   ├── Push-Pull (推挽输出)
│   ├── Open-Drain (开漏输出)
│   ├── Speed: 2MHz / 10MHz / 50MHz
├── Alternate Function (复用功能)
│   ├── UART, SPI, I2C
│   ├── Timer, ADC
│   └── CAN, USB, etc.
└── Analog Mode
    └── 直接连接片内模块 (ADC/DAC)
```

### 1.2 GPIO寄存器

```c
typedef struct {
    volatile uint32_t MODER;        // 0x00 模式
    volatile uint32_t OTYPER;       // 0x04 输出类型
    volatile uint32_t OSPEEDR;      // 0x08 输出速率
    volatile uint32_t PUPDR;        // 0x0C 上下拉
    volatile uint32_t IDR;          // 0x10 输入数据
    volatile uint32_t ODR;          // 0x14 输出数据
    volatile uint32_t BSRR;         // 0x18 位操作 (原子)
    volatile uint32_t LCKR;         // 0x1C 锁定
    volatile uint32_t AFR[2];       // 0x20-0x24 复用功能
} GPIO_TypeDef;
```

#### MODER 模式寄存器

```
每个引脚占用2位:
00 = Input
01 = Output
10 = Alternate Function
11 = Analog

例子: PA1配置为输出
MODIFY_REG(GPIOA->MODER, 0x3 << 2, 0x1 << 2);
```

#### PUPDR 上下拉寄存器

```
每个引脚占用2位:
00 = No Pull
01 = Pull-Up
10 = Pull-Down
11 = Reserved

例子: PA1上拉
MODIFY_REG(GPIOA->PUPDR, 0x3 << 2, 0x1 << 2);
```

### 1.3 GPIO初始化

```c
#include "stm32f4xx_hal.h"

void GPIO_Init_LED(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    // 启用GPIOA时钟
    __HAL_RCC_GPIOA_CLK_ENABLE();
    
    // 配置PA1为输出
    GPIO_InitStruct.Pin = GPIO_PIN_1;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;      // 推挽输出
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;    // 50MHz
    
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    // 设置初始状态
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
}

void GPIO_Init_Button(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    __HAL_RCC_GPIOC_CLK_ENABLE();
    
    // 配置PC13为输入，下拉
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLDOWN;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
}
```

---

## 2. GPIO高级操作

### 2.1 BSRR寄存器原子操作

```c
// BSRR 寄存器: 原子置位/清零
// 低16位用来置位 (SET)
// 高16位用来清零 (RESET)

// 原子置位 PA1 (推荐!)
GPIOA->BSRR = (1 << 1);

// 原子清零 PA1
GPIOA->BSRR = (1 << (16 + 1));

// 快速翻转 (非原子)
GPIOA->ODR ^= (1 << 1);

// 读取输入
uint32_t pin_state = (GPIOA->IDR >> 1) & 1;
```

### 2.2 复用功能配置

```c
// AFR 寄存器: 每个引脚4位选择功能 (AF0-AF15)
// AFR[0]: PA0-PA7
// AFR[1]: PA8-PA15

void UART1_GPIO_Init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();
    
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    GPIO_InitStruct.Alternate = GPIO_AF7_USART1;     // AF7
    
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

### 2.3 配置锁定

```c
void GPIO_Lock_Config(void)
{
    // LCKR 寄存器可锁定GPIO配置直到下一次复位
    
    // 第1步: 设置LCKK和要锁定的引脚
    GPIOA->LCKR = GPIO_LCKR_LCKK | GPIO_PIN_1;
    
    // 第2步: 清零LCKK
    GPIOA->LCKR = GPIO_PIN_1;
    
    // 第3步: 再次设置LCKK
    GPIOA->LCKR = GPIO_LCKR_LCKK | GPIO_PIN_1;
    
    // 验证锁定成功
    uint32_t lock_status = GPIOA->LCKR & GPIO_LCKR_LCKK;
}
```

---

## 3. GPIO中断

### 3.1 外部中断初始化

```c
void GPIO_EXTI_Init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();
    
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;     // 下降沿触发
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    // 配置NVIC
    HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
}

// 中断服务程序
void EXTI0_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

// 回调函数
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if(GPIO_Pin == GPIO_PIN_0)
    {
        // 处理中断
        LED_Toggle();
        
        // 防止抖动
        HAL_Delay(20);
    }
}
```

---

## 4. 外设基本概念

### 4.1 STM32外设框架

```
ARM Cortex-M4 Core (168MHz)
         │
    ┌────┴────┐
    │ AHB总线 │
    └────┬────┘
         │
    ┌────┴─────────────┬─────────┐
    │                  │         │
 GPIO             DMA1,DMA2    Flash
 ├─GPIOA          
 ├─GPIOB          ┌────────────────────┐
 ├─GPIOC          │   APB总线系统     │
 └─... (8个)      ├─APB1 (84MHz)      │
                  │ └─TIM2/3/4/5/12-14│
                  │ └─UART2/3/4/5     │
                  │ └─SPI2/3          │
                  │ └─I2C1/2/3        │
                  │ └─CAN1/2          │
                  ├─APB2 (168MHz)     │
                  │ └─TIM1/8-11       │
                  │ └─UART1/6         │
                  │ └─SPI1            │
                  │ └─ADC1/2/3        │
                  └────────────────────┘
```

### 4.2 外设时钟配置

```c
// 启用常用外设时钟

// AHB1 外设
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;      // GPIOA
RCC->AHB1ENR |= RCC_AHB1ENR_DMA1EN;       // DMA1

// APB1 外设
RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;       // TIM2
RCC->APB1ENR |= RCC_APB1ENR_USART2EN;     // USART2
RCC->APB1ENR |= RCC_APB1ENR_SPI2EN;       // SPI2
RCC->APB1ENR |= RCC_APB1ENR_I2C1EN;       // I2C1

// APB2 外设
RCC->APB2ENR |= RCC_APB2ENR_TIM1EN;       // TIM1
RCC->APB2ENR |= RCC_APB2ENR_USART1EN;     // USART1
RCC->APB2ENR |= RCC_APB2ENR_SPI1EN;       // SPI1
RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;       // ADC1
```

### 4.3 定时器时钟特殊规则

```c
// 重要: 定时器时钟与APB分频的关系

// 规则: 当 APB预分频 > 1 时，定时器时钟自动 × 2

// 示例配置:
// SYSCLK = 168MHz
// AHB = 168MHz (分频系数 1)
// APB1 = 84MHz (分频系数 2)
// APB2 = 168MHz (分频系数 1)

// 因此:
// TIM2/3/4/5 (APB1定时器) = 2 × 84MHz = 168MHz
// TIM1/8 (APB2定时器) = 168MHz (不倍频)

// 计算定时器周期
#define TIM_CLK         168000000          // 168MHz
#define TIM_PSC         1680                // 预分频
#define TIM_ARR         10000               // 自动重装值

// 计数频率 = TIM_CLK / (PSC+1) = 168M / 1680 = 100KHz
// 溢出周期 = (ARR+1) / 计数频率 = 10001 / 100K = 100.01ms
```

---

## 5. 推挽 vs 开漏输出

### 5.1 特性对比

| 特性 | 推挽输出 | 开漏输出 |
|------|---------|---------|
| **高电平** | VCC (主动驱动) | 高阻 (需外接上拉) |
| **低电平** | GND (主动驱动) | GND (主动驱动) |
| **应用** | LED, 普通IO | I2C, 1-Wire, 共线 |
| **上拉电阻** | 不需要 | 必需 |
| **功耗** | 较高 | 较低 |

### 5.2 初始化示例

```c
// 推挽输出: LED驱动
GPIO_InitStruct.Pin = GPIO_PIN_1;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;

// 开漏输出: I2C SDA/SCL
GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_9;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;
GPIO_InitStruct.Pull = GPIO_PULLUP;
```

---

## 6. 常见问题

### Q1: 为什么需要 volatile 关键字?

```c
// 防止编译器优化，确保每次都读取寄存器
volatile uint32_t *reg = (volatile uint32_t *)0x40020014;

uint32_t val1 = *reg;
uint32_t val2 = *reg;  // 没有volatile会被优化，val2不会重新读取
```

### Q2: GPIO初始化后为什么IO无反应?

✅ 检查时钟是否启用
✅ 检查模式配置是否正确
✅ 检查引脚复用冲突
✅ 检查电源连接

### Q3: 如何进行原子操作?

```c
// 使用BSRR寄存器替代ODR
GPIOA->BSRR = (1 << 1);         // 原子置位
GPIOA->BSRR = (1 << 17);        // 原子清零

// 避免这样做 (非原子)
GPIOA->ODR |= (1 << 1);         // 危险!
GPIOA->ODR &= ~(1 << 1);        // 危险!
```

