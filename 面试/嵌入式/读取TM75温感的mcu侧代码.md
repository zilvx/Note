使用STM32 HAL库读取TM75（兼容TMP75/LM75）温度传感器的MCU侧代码。

这份代码主要依赖STM32CubeMX生成的硬件初始化配置，以及HAL库提供的`HAL_I2C_Mem_Read`函数来完成I2C通信[](https://www.digikey.cn/zh/forum/t/topic/4660?gad_source=7&dclid=CM-FoaK30JQDFYalYwYdc90IVQ#1)[](https://wiki.st.com/stm32mcu/wiki/Getting_started_with_I2C#1)。

### 1. 核心代码实现

#### `tm75.h` - 头文件定义
```c
#ifndef TM75_H
#define TM75_H
#include "main.h"
// TM75的I2C从机地址（7位地址）
// 地址由A2,A1,A0引脚电平决定，此处以全部接GND的0x4B为例
// 注：有些资料使用0x4F，实际地址需根据硬件连接确认[citation:2][citation:9][citation:12]
#define TM75_ADDR         0x4B
// 温度寄存器地址
#define TM75_TEMP_REG     0x00
// 配置寄存器地址
#define TM75_CONFIG_REG   0x01
// 分辨率设置：12位（0.0625°C/LSB）
#define TM75_RESOLUTION_12BIT  0x60
// 函数声明
void TM75_Init(void);
float TM75_ReadTemperature(void);
#endif
```

#### `tm75.c` - 源文件实现
```c
#include "tm75.h"
#include "i2c.h" // 包含CubeMX生成的I2C句柄
extern I2C_HandleTypeDef hi2c1; // 假设使用I2C1
/**
 * @brief 初始化TM75传感器
 * @note  配置为12位分辨率，连续转换模式
 */
void TM75_Init(void)
{
    uint8_t configData = TM75_RESOLUTION_12BIT; // 默认其他控制位为0
    // 向配置寄存器(0x01)写入配置值
    // 使用HAL_I2C_Mem_Write函数，它内部处理了写寄存器所需的复合时序[citation:4][citation:10]
    HAL_StatusTypeDef status = HAL_I2C_Mem_Write(&hi2c1,
                                                  TM75_ADDR << 1,        // 7位地址左移1位，符合HAL库8位地址格式[citation:4]
                                                  TM75_CONFIG_REG,       // 寄存器地址
                                                  I2C_MEMADD_SIZE_8BIT,  // 寄存器地址为8位
                                                  &configData,           // 要写入的数据
                                                  1,                     // 数据长度1字节
                                                  HAL_MAX_DELAY);        // 超时时间
    if (status != HAL_OK)
    {
        // 在此处添加错误处理，例如记录日志或点亮LED
        Error_Handler();
    }
}
/**
 * @brief 读取温度值
 * @return float 温度值，单位摄氏度。若读取失败，返回一个特定错误值（如-273.0）
 */
float TM75_ReadTemperature(void)
{
    uint8_t rawData[2] = {0};
    int16_t rawTemp = 0;
    float temperature = 0.0f;
    // 从温度寄存器(0x00)读取2字节原始数据
    // HAL_I2C_Mem_Read会先发送寄存器地址，再重启总线读取数据[citation:4][citation:10]
    HAL_StatusTypeDef status = HAL_I2C_Mem_Read(&hi2c1,
                                                TM75_ADDR << 1,
                                                TM75_TEMP_REG,
                                                I2C_MEMADD_SIZE_8BIT,
                                                rawData,
                                                2,
                                                HAL_MAX_DELAY);
    if (status != HAL_OK)
    {
        // I2C通信失败，返回一个错误值
        return -273.0f;
    }
    // 将两个字节组合成16位数据
    // 对于TM75/TMP75，温度数据为12位，左对齐，存储在16位寄存器的高12位[citation:11]
    rawTemp = (int16_t)((rawData[0] << 8) | rawData[1]);
    // 将12位有符号数据转换为实际温度值
    // 步骤：右移4位，得到12位有符号整数，然后乘以分辨率0.0625°C
    // 注意：需要先将数据转换为有符号数再进行移位
    rawTemp >>= 4; // 此时rawTemp为12位有符号数
    temperature = (float)rawTemp * 0.0625f;
    return temperature;
}
```

### 2. 使用示例 (`main.c`)

在主程序中调用这些函数非常简单：
```c
#include "tm75.h"
#include <stdio.h>
int main(void)
{
    HAL_Init();
    SystemClock_Config(); // 由CubeMX生成
    MX_GPIO_Init();
    MX_I2C1_Init();      // 由CubeMX生成
    // 初始化TM75传感器
    TM75_Init();
    float currentTemp = 0.0f;
    while (1)
    {
        // 读取温度
        currentTemp = TM75_ReadTemperature();
        if (currentTemp > -100.0f) // 检查是否读取成功
        {
            // 通过串口打印温度值（假设已初始化UART）
            printf("Current Temperature: %.2f °C\r\n", currentTemp);
        }
        else
        {
            printf("Temperature Read Error!\r\n");
        }
        HAL_Delay(1000); // 每秒读取一次
    }
}
```

### 3. 关键点说明

1. **设备地址确认**：TM75的I2C地址由引脚A2、A1、A0决定[](https://blog.csdn.net/qq_40008325/article/details/124646693?utm_term=75)[](https://app2.alldatasheet.com/datasheet-pdf/pdf/2186632/TI2/TMP75CIDGKT.html#1)。头文件中默认的`0x4B`是一个常见配置（三个地址引脚接GND）。请务必根据你的硬件原理图确认实际地址。例如，其他搜索结果中也出现了`0x4F`的用法[](https://blog.csdn.net/weixin_53403301/article/details/136230557#1)。
    
2. **HAL库函数地址格式**：STM32 HAL库的`HAL_I2C_Mem_Read/Write`函数要求传入的`DevAddress`是**8位地址**，即需要将芯片的7位地址左移一位，并在最低位补0（写）或1（读）。代码中直接使用了`TM75_ADDR << 1`来满足这个要求[](https://www.digikey.cn/zh/forum/t/topic/4660?gad_source=7&dclid=CM-FoaK30JQDFYalYwYdc90IVQ#1)[](https://wiki.st.com/stm32mcu/wiki/Getting_started_with_I2C#1)。
    
3. **分辨率与转换公式**：代码中将TM75配置为12位分辨率（`0x0625°C/LSB`）[](https://app2.alldatasheet.com/datasheet-pdf/pdf/2186632/TI2/TMP75CIDGKT.html#1)。`TM75_ReadTemperature`函数中的转换逻辑（右移4位后乘以0.0625）正是针对这种12位、左对齐的数据格式[](https://git.kcir.pwr.edu.pl/kmichal2/projekt/-/blob/better/main.c?ref_type=heads#1)。
    
4. **错误处理**：代码包含了基础的错误处理（返回错误值），在实际产品中，你可能需要根据应用需求增加更复杂的重试机制或故障恢复逻辑[](https://git.kcir.pwr.edu.pl/kmichal2/projekt/-/blob/better/main.c?ref_type=heads#1)。


`MX_I2C1_Init`函数和你之前看到的`HAL_I2C_MspInit`函数，共同构成了I2C外设的完整初始化，它们在STM32CubeMX生成的代码中分工明确。我为你详细拆解一下这个完整的过程。

### 🧬 函数关系与分工

1. **`MX_I2C1_Init` (核心配置)**：这个函数由STM32CubeMX根据你在图形界面中的设置自动生成，主要完成I2C通信协议本身的核心参数配置。通俗地说，它决定了I2C总线“以什么规矩说话”[](https://bbs.huaweicloud.com/blogs/422765#1)[](https://bbs.huaweicloud.com/blogs/422766#1)。
    
2. **`HAL_I2C_MspInit` (底层硬件初始化)**：这个函数同样由CubeMX生成，但它负责的是与I2C外设相关的具体硬件资源初始化，比如：**开启I2C外设的时钟、配置SCL和SDA引脚为复用开漏模式、设置中断优先级和使能NVIC**等[](https://strade.fit.vutbr.cz/git/adam_prochazka/ProfilometerProbe/raw/commit/110d89d2221029f8dbe6c359ecbaf4fbd7202ac2/Firmware/Core/Src/stm32l4xx_hal_msp.c)[](https://os.mbed.com/teams/Lancaster-University/code/mbed-src//file/a1ed5b41f74f/targets/cmsis/TARGET_STM/TARGET_STM32L1/stm32l1xx_hal_i2c.c/)。它更像是为I2C通信“铺路搭桥”，准备好物理层面的连接。
    

`MX_I2C1_Init`函数会调用`HAL_I2C_Init`，而`HAL_I2C_Init`内部又会调用`HAL_I2C_MspInit`，从而完成从协议到硬件的全部初始化工作[](https://os.mbed.com/teams/Lancaster-University/code/mbed-src//file/a1ed5b41f74f/targets/cmsis/TARGET_STM/TARGET_STM32L1/stm32l1xx_hal_i2c.c/)[](http://mirrors.aliyun.com/android.googlesource.com/external/ms-tpm-20-ref/Samples/Nucleo-TPM/L4A6RG/Drivers/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_i2c.c)。

### 📄 `MX_I2C1_Init`函数代码示例 (适用于STM32L451CE)

下面是一个典型的、由CubeMX为STM32L4系列芯片生成的`MX_I2C1_Init`函数代码。其中的配置参数（如时钟速度）是你在CubeMX界面上选择的。
```c
/**
  * @brief I2C1 Initialization Function
  * @param None
  * @retval None
  */
static void MX_I2C1_Init(void)
{
  hi2c1.Instance = I2C1;                      // 使用I2C1外设
  hi2c1.Init.Timing = 0x00707CBB;             // I2C时序配置值（由CubeMX根据时钟和速度自动计算）
  hi2c1.Init.OwnAddress1 = 0;                 // 本机地址1，如果作为主机通常设为0
  hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT; // 7位寻址模式
  hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE; // 不使能双地址模式
  hi2c1.Init.OwnAddress2 = 0;                 // 本机地址2
  hi2c1.Init.OwnAddress2Masks = I2C_OA2_NOMASK; // 地址2掩码
  hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE; // 不使能广播呼叫
  hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE; // 允许从设备时钟延展
  if (HAL_I2C_Init(&hi2c1) != HAL_OK)         // 调用HAL库的初始化函数
  {
    Error_Handler();                          // 初始化失败则进入错误处理
  }
}
```

### 🔩 `HAL_I2C_MspInit`函数代码示例

这个函数负责具体的引脚和时钟配置。以下是一个针对使用`PA9`作为SCL，`PA10`作为SDA的STM32L4代码示例[](https://strade.fit.vutbr.cz/git/adam_prochazka/ProfilometerProbe/raw/commit/110d89d2221029f8dbe6c359ecbaf4fbd7202ac2/Firmware/Core/Src/stm32l4xx_hal_msp.c)：
```c
void HAL_I2C_MspInit(I2C_HandleTypeDef* hi2c)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};
  if(hi2c->Instance == I2C1) // 判断是否为I2C1
  {
    // 1. 配置I2C1的时钟源（可选，通常使用PCLK1）
    PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_I2C1;
    PeriphClkInit.I2c1ClockSelection = RCC_I2C1CLKSOURCE_PCLK1;
    HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit);
    // 2. 使能I2C1外设时钟
    __HAL_RCC_I2C1_CLK_ENABLE();
    // 3. 使能GPIOA时钟
    __HAL_RCC_GPIOA_CLK_ENABLE();
    // 4. 配置I2C的SCL和SDA引脚
    GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;      // 复用开漏模式
    GPIO_InitStruct.Pull = GPIO_NOPULL;          // 无上下拉
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
    GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;   // 复用功能为I2C1
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  }
}
```

### 🚀 如何使用

在你的`main.c`文件中，只需在初始化阶段调用一次`MX_I2C1_Init()`即可。它会自动完成所有配置工作，之后你就可以使用`HAL_I2C_Mem_Read/Write`等函数与TM75通信了。
```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_I2C1_Init();  // <--- 调用此函数完成全部初始化
  // 此时I2C1已经准备就绪，可以调用TM75驱动函数
  TM75_Init();
  // ... 后续代码
}

```

这两个函数（`MX_I2C1_Init`和`HAL_I2C_MspInit`）通常由STM32CubeMX自动生成，无需手动编写。核心在于通过CubeMX进行正确的图形化配置，并理解生成的代码结构，以便在需要时进行调整