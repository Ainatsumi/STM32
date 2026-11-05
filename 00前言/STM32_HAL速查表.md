# STM32 HAL 速查表（F103C8T6）

> 配套工程：`C:\Users\a\Desktop\STM32_gongcheng\1st\demo1`
> 更新：2026-11-04

---

## 0. 先记住这一条

**HAL 库的命名是有规律的，不用背函数名，看懂规律就能猜出来。**

```
HAL_<外设>_<动作>[_<方式>]()
```

例：

```c
HAL_GPIO_WritePin()      // HAL + GPIO + 写引脚
HAL_UART_Transmit_IT()   // HAL + 串口 + 发送 + 中断方式
HAL_TIM_PWM_Start_DMA()  // HAL + 定时器 + PWM + 启动 + DMA方式
```

---

## 1. 命名规律（最重要的部分）

### 1.1 四种前缀，含义完全不同

| 前缀 | 例子 | 是什么 | 能不能改 |
|---|---|---|---|
| `HAL_` | `HAL_GPIO_Init()` | 普通函数，调用 ST 的库 | 不改，只调用 |
| `__HAL_` | `__HAL_RCC_GPIOA_CLK_ENABLE()` | **宏**，直接操作寄存器，全大写 | 不改，只调用 |
| `MX_` | `MX_GPIO_Init()` | **CubeMX 生成**的初始化函数 | 不要手改（会被覆盖） |
| `System` | `SystemClock_Config()` | CubeMX 生成的时钟配置 | 不要手改 |
| 无前缀 | `Error_Handler()` | 你自己要实现的 | 可以改 |

**`__HAL_` 双下划线开头的一律是宏**，展开后就是寄存器操作，所以全大写、带下划线。

### 1.2 三种后缀，决定"怎么传数据"

| 后缀 | 含义 | 阻塞? | 用法 |
|---|---|---|---|
| 无 | 轮询（Polling） | ✅ 阻塞 | `HAL_UART_Transmit(...)` 卡住直到发完 |
| `_IT` | 中断（Interrupt） | ❌ 非阻塞 | 发完触发中断，回调通知你 |
| `_DMA` | DMA 搬运 | ❌ 非阻塞 | 硬件搬运，CPU 不管 |

例：

```c
HAL_UART_Transmit(&huart1, buf, 10, 1000);      // 阻塞，发完才返回
HAL_UART_Transmit_IT(&huart1, buf, 10);         // 立刻返回，发完进回调
HAL_UART_Transmit_DMA(&huart1, buf, 10);        // 立刻返回，DMA 搬运
```

### 1.3 回调函数：`_Callback` 结尾

HAL 里预先写了**空的弱函数**（`__weak`），你重写同名函数就会覆盖它。

```c
HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)          // 外部中断触发
HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) // 串口收完一帧
HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) // 串口发完
HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) // 定时器溢出
HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)  // ADC 转换完成
HAL_I2C_MasterTxCpltCallback(I2C_HandleTypeDef *hi2c)
```

> 看到 `Callback` 就知道：**这是留给你填的，不用自己调用。**

### 1.4 中断入口：`_IRQHandler` 结尾

在 `Core/Src/stm32f1xx_it.c` 里，CubeMX 自动生成：

```c
void EXTI15_10_IRQHandler(void)  { HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13); }
void USART1_IRQHandler(void)     { HAL_UART_IRQHandler(&huart1); }
void TIM2_IRQHandler(void)       { HAL_TIM_IRQHandler(&htim2); }
```

**规律：CubeMX 生成的中断入口只做一件事——转交给 HAL 处理。** 你不要改这里，要写逻辑去写 `_Callback`。

### 1.5 常见动作词

| 词 | 含义 |
|---|---|
| `Init` / `DeInit` | 初始化 / 反初始化 |
| `Start` / `Stop` | 启动 / 停止 |
| `Transmit` / `Receive` | 发送 / 接收 |
| `Read` / `Write` | 读 / 写 |
| `Get` / `Set` | 获取 / 设置 |
| `Enable` / `Disable` | 使能 / 禁止 |
| `Toggle` | 翻转 |
| `Poll` | 轮询等待 |
| `Cplt` | Complete（完成） |
| `Err` | Error（错误） |
| `IRQ` | Interrupt Request（中断请求） |

---

## 2. 缩写对照表

### 2.1 系统 / 内核

| 缩写 | 全称 | 中文 | 说明 |
|---|---|---|---|
| **HAL** | Hardware Abstraction Layer | 硬件抽象层 | ST 现在主推的库 |
| **SPL** / StdPeriph | Standard Peripheral Library | 标准外设库 | 老库，江科大教程用的 |
| **LL** | Low Layer | 底层库 | 比 HAL 轻，接近寄存器 |
| **CMSIS** | Cortex Microcontroller Software Interface Standard | 内核软件接口标准 | ARM 定的 |
| **NVIC** | Nested Vectored Interrupt Controller | 嵌套向量中断控制器 | 管中断优先级 |
| **SysTick** | System Tick Timer | 系统滴答定时器 | 内核自带 24 位定时器，`HAL_Delay` 靠它 |
| **MSP** | MCU Support Package | 单片机支持包 | `HAL_XXX_MspInit()` 里的底层初始化 |
| **IRQ** | Interrupt Request | 中断请求 | |
| **ISR** | Interrupt Service Routine | 中断服务函数 | |
| **WFI** | Wait For Interrupt | 等待中断 | 低功耗指令 |
| **FPU** | Floating Point Unit | 浮点单元 | **F103 没有**，浮点靠软件模拟 |

### 2.2 时钟

| 缩写 | 全称 | 中文 | F103 典型值 |
|---|---|---|---|
| **RCC** | Reset and Clock Control | 复位与时钟控制 | |
| **HSE** | High Speed External | 外部高速晶振 | 8 MHz |
| **HSI** | High Speed Internal | 内部高速 RC | 8 MHz（精度差） |
| **LSE** | Low Speed External | 外部低速晶振 | 32.768 kHz（RTC 用） |
| **LSI** | Low Speed Internal | 内部低速 RC | 40 kHz（看门狗用） |
| **PLL** | Phase Locked Loop | 锁相环 | ×9 → 72 MHz |
| **SYSCLK** | System Clock | 系统时钟 | 72 MHz |
| **HCLK** | AHB Clock | AHB 总线时钟 | 72 MHz |
| **PCLK1** | APB1 Peripheral Clock | APB1 外设时钟 | **36 MHz（上限）** |
| **PCLK2** | APB2 Peripheral Clock | APB2 外设时钟 | 72 MHz |
| **AHB** | Advanced High-performance Bus | 高性能总线 | 高速外设 |
| **APB** | Advanced Peripheral Bus | 外设总线 | 低速外设 |

> ⚠️ **APB1 最高 36MHz**，这是 F103 的硬约束。串口、I2C、TIM2~7 挂在 APB1 上。

### 2.3 外设

| 缩写 | 全称 | 中文 |
|---|---|---|
| **GPIO** | General Purpose Input/Output | 通用输入输出 |
| **AFIO** | Alternate Function I/O | 复用功能 IO |
| **EXTI** | External Interrupt | 外部中断 |
| **TIM** | Timer | 定时器 |
| **USART** | Universal Synchronous/Asynchronous Receiver/Transmitter | 通用同步/异步收发器 |
| **UART** | Universal Asynchronous Receiver/Transmitter | 通用异步收发器 |
| **I2C / IIC** | Inter-Integrated Circuit | 集成电路总线 |
| **SPI** | Serial Peripheral Interface | 串行外设接口 |
| **ADC** | Analog-to-Digital Converter | 模数转换器 |
| **DAC** | Digital-to-Analog Converter | 数模转换器 |
| **DMA** | Direct Memory Access | 直接存储器访问 |
| **RTC** | Real Time Clock | 实时时钟 |
| **IWDG** | Independent Watchdog | 独立看门狗 |
| **WWDG** | Window Watchdog | 窗口看门狗 |
| **PWR** | Power Control | 电源控制 |
| **BKP** | Backup Registers | 备份寄存器 |
| **CAN** | Controller Area Network | 控制器局域网 |
| **FSMC** | Flexible Static Memory Controller | 静态存储器控制器 |

### 2.4 CubeMX 注释段

| 段名 | 全称 | 放什么 |
|---|---|---|
| `Header` | — | 文件头 |
| `Includes` | — | `#include` |
| `PTD` | Private Typedef | 私有类型定义 |
| `PD` | Private Define | 私有宏定义 |
| `PM` | Private Macro | 私有宏函数 |
| `PV` | Private Variables | 私有全局变量 |
| `PFP` | Private Function Prototypes | 私有函数声明 |
| `0` | — | `main()` 之前的代码 |
| `1` | — | `main()` 开头 |
| `Init` | — | `HAL_Init()` 之后 |
| `SysInit` | — | 时钟配置之后 |
| `2` | — | 外设初始化之后、`while` 之前 |
| `WHILE` | — | 进入 `while` 之前 |
| `3` | — | **`while(1)` 循环体内** ← 最常用 |
| `4` | — | `main()` 之后 |

---

## 3. 常用函数速查

### 3.1 系统

```c
HAL_Init();                  // HAL 初始化（必须最先调用）
HAL_Delay(ms);               // 毫秒延时（阻塞）
HAL_GetTick();               // 获取开机以来的毫秒数
HAL_IncTick();               // 增加 tick（SysTick 中断里自动调）
Error_Handler();             // 错误处理（死循环）
```

### 3.2 GPIO

```c
HAL_GPIO_Init(port, &init);                    // 初始化引脚
HAL_GPIO_DeInit(port, pin);                    // 反初始化
HAL_GPIO_WritePin(port, pin, state);           // 写电平
HAL_GPIO_ReadPin(port, pin);                   // 读电平
HAL_GPIO_TogglePin(port, pin);                 // 翻转
HAL_GPIO_LockPin(port, pin);                   // 锁定配置（防误改）

// 中断回调（自己重写）
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin);
```

### 3.3 时钟

```c
HAL_RCC_OscConfig(&osc);                       // 配置振荡器 / PLL
HAL_RCC_ClockConfig(&clk, latency);            // 配置总线分频

// 以下都是宏
__HAL_RCC_GPIOA_CLK_ENABLE();
__HAL_RCC_GPIOB_CLK_ENABLE();
__HAL_RCC_GPIOC_CLK_ENABLE();
__HAL_RCC_USART1_CLK_ENABLE();
__HAL_RCC_TIM2_CLK_ENABLE();
__HAL_RCC_ADC1_CLK_ENABLE();
__HAL_RCC_DMA1_CLK_ENABLE();
__HAL_RCC_AFIO_CLK_ENABLE();
```

### 3.4 串口（USART）

```c
HAL_UART_Init(&huart1);                                    // 初始化
HAL_UART_Transmit(&huart1, buf, len, timeout);             // 阻塞发送
HAL_UART_Receive(&huart1, buf, len, timeout);              // 阻塞接收
HAL_UART_Transmit_IT(&huart1, buf, len);                   // 中断发送
HAL_UART_Receive_IT(&huart1, buf, len);                    // 中断接收
HAL_UART_Transmit_DMA(&huart1, buf, len);                  // DMA 发送
HAL_UART_Receive_DMA(&huart1, buf, len);                   // DMA 接收
HAL_UART_Abort(&huart1);                                   // 中止当前传输

// 回调（自己重写）
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart);
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart);
```

### 3.5 定时器（TIM）

```c
HAL_TIM_Base_Start(&htim2);                     // 启动计数
HAL_TIM_Base_Start_IT(&htim2);                  // 启动 + 更新中断
HAL_TIM_Base_Stop(&htim2);

HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);       // 启动 PWM
HAL_TIM_PWM_Stop(&htim2, TIM_CHANNEL_1);
__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, 500);  // 设置占空比（比较值）

HAL_TIM_IC_Start(&htim3, TIM_CHANNEL_1);        // 输入捕获
HAL_TIM_IC_Start_IT(&htim3, TIM_CHANNEL_1);

__HAL_TIM_GET_COUNTER(&htim2);                  // 读当前计数值
__HAL_TIM_SET_COUNTER(&htim2, 0);               // 写计数值
__HAL_TIM_SET_AUTORELOAD(&htim2, 999);          // 改重装值

// 回调（自己重写）
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim);   // 溢出
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim);      // 捕获
```

### 3.6 ADC

```c
HAL_ADC_Start(&hadc1);                              // 启动转换
HAL_ADC_Start_IT(&hadc1);
HAL_ADC_Start_DMA(&hadc1, buf, len);
HAL_ADC_PollForConversion(&hadc1, timeout);         // 等待转换完成
HAL_ADC_GetValue(&hadc1);                           // 读结果（0~4095）
HAL_ADC_Stop(&hadc1);

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc);   // 回调
```

### 3.7 I2C

```c
HAL_I2C_Master_Transmit(&hi2c1, addr, buf, len, timeout);
HAL_I2C_Master_Receive(&hi2c1, addr, buf, len, timeout);
HAL_I2C_Mem_Write(&hi2c1, addr, mem_addr, mem_size, buf, len, timeout);
HAL_I2C_Mem_Read(&hi2c1, addr, mem_addr, mem_size, buf, len, timeout);
HAL_I2C_IsDeviceReady(&hi2c1, addr, trials, timeout);    // 探测设备
```

### 3.8 SPI

```c
HAL_SPI_Transmit(&hspi1, buf, len, timeout);
HAL_SPI_Receive(&hspi1, buf, len, timeout);
HAL_SPI_TransmitReceive(&hspi1, txbuf, rxbuf, len, timeout);
```

### 3.9 DMA

```c
HAL_DMA_Start(&hdma, src, dst, len);
HAL_DMA_Start_IT(&hdma, src, dst, len);
HAL_DMA_Abort(&hdma);
```

---

## 4. 常量 / 枚举

### 4.1 GPIO

```c
// 电平原状态
GPIO_PIN_RESET        // 低电平 0
GPIO_PIN_SET          // 高电平 1

// 模式
GPIO_MODE_INPUT             // 输入
GPIO_MODE_OUTPUT_PP         // 推挽输出
GPIO_MODE_OUTPUT_OD         // 开漏输出
GPIO_MODE_AF_PP             // 复用推挽
GPIO_MODE_AF_OD             // 复用开漏
GPIO_MODE_ANALOG            // 模拟（ADC 用）
GPIO_MODE_IT_RISING         // 上升沿中断
GPIO_MODE_IT_FALLING        // 下降沿中断
GPIO_MODE_IT_RISING_FALLING // 双边沿中断

// 上下拉
GPIO_NOPULL
GPIO_PULLUP
GPIO_PULLDOWN

// 速度（F103）
GPIO_SPEED_FREQ_LOW       // 2 MHz
GPIO_SPEED_FREQ_MEDIUM    // 10 MHz
GPIO_SPEED_FREQ_HIGH      // 50 MHz
```

### 4.2 返回值

```c
HAL_OK          // 0  成功
HAL_ERROR       // 1  错误
HAL_BUSY        // 2  忙
HAL_TIMEOUT     // 3  超时
```

### 4.3 定时器通道

```c
TIM_CHANNEL_1 / TIM_CHANNEL_2 / TIM_CHANNEL_3 / TIM_CHANNEL_4
```

---

## 5. 寄存器缩写（看参考手册时用）

| 缩写 | 全称 | 含义 |
|---|---|---|
| `CRL` / `CRH` | Configuration Register Low/High | GPIO 配置寄存器（F1） |
| `IDR` | Input Data Register | 输入数据 |
| `ODR` | Output Data Register | 输出数据 |
| `BSRR` | Bit Set/Reset Register | 位设置/清除 |
| `BRR` | Bit Reset Register | 位清除 |
| `SR` | Status Register | 状态寄存器 |
| `DR` | Data Register | 数据寄存器 |
| `CR1` / `CR2` | Control Register 1/2 | 控制寄存器 |
| `ARR` | Auto Reload Register | 自动重装值 |
| `PSC` | Prescaler | 预分频 |
| `CCR` | Capture/Compare Register | 捕获/比较寄存器 |
| `CNT` | Counter | 计数器 |

> F4/F7 之后的 GPIO 寄存器换成了 `MODER` / `OTYPER` / `OSPEEDR` / `PUPDR`，F1 用 `CRL`/`CRH`。

---

## 6. 自己查文档的方法

### 方法 1：看头文件（最快）

```
Drivers/STM32F1xx_HAL_Driver/Inc/stm32f1xx_hal_gpio.h
Drivers/STM32F1xx_HAL_Driver/Inc/stm32f1xx_hal_uart.h
Drivers/STM32F1xx_HAL_Driver/Inc/stm32f1xx_hal_tim.h
```

里面每个函数上面都有一段注释，说明参数和返回值。

### 方法 2：VS Code 里跳转

把光标放在函数名上，按 **F12**（转到定义），能看到实现。

**Ctrl + 鼠标左键** 同样效果。

### 方法 3：看函数原型

```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart,
                                    const uint8_t *pData,
                                    uint16_t Size,
                                    uint32_t Timeout);
```

- 返回 `HAL_StatusTypeDef` → 要检查 `HAL_OK`
- `huart` → 句柄指针
- `pData` → `const` 说明只读
- `Size` → `uint16_t`，**一次最多 65535 字节**
- `Timeout` → 毫秒，`HAL_MAX_DELAY` 表示永久等待

### 方法 4：查参考手册

```
RM0008（STM32F10xxx 参考手册）—— 寄存器级细节
```

配合 CubeMX 的芯片图看引脚复用表。

---

## 7. 一个记忆技巧

把函数名当成**英文短语**读，而不是当字符串背：

```
HAL_GPIO_TogglePin   →  "HAL，GPIO，翻转引脚"
HAL_UART_Receive_IT  →  "HAL，串口，接收，用中断"
HAL_TIM_PWM_Start    →  "HAL，定时器，PWM，启动"
```

读顺了自然就记住了，不用背。

---

## 8. 和标准库（江科大）的对照

| 功能 | 标准库 | HAL |
|---|---|---|
| 开 GPIO 时钟 | `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE)` | `__HAL_RCC_GPIOC_CLK_ENABLE()` |
| 配引脚 | `GPIO_InitTypeDef` + `GPIO_Init()` | `GPIO_InitTypeDef` + `HAL_GPIO_Init()` |
| 输出高 | `GPIO_SetBits(GPIOC, GPIO_Pin_13)` | `HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET)` |
| 输出低 | `GPIO_ResetBits(GPIOC, GPIO_Pin_13)` | `HAL_GPIO_WritePin(..., GPIO_PIN_RESET)` |
| 翻转 | `GPIO_WriteBit(...)` 手动取反 | `HAL_GPIO_TogglePin(...)` |
| 延时 | 自己写 `Delay_ms()` | `HAL_Delay(ms)` |
| 串口发送 | `USART_SendData()` + 等标志位 | `HAL_UART_Transmit()` |
| 定时器启动 | `TIM_Cmd(TIM2, ENABLE)` | `HAL_TIM_Base_Start(&htim2)` |
| ADC 读值 | `ADC_GetConversionValue()` | `HAL_ADC_GetValue()` |

**核心概念完全一样，只是 API 换了名字。**
