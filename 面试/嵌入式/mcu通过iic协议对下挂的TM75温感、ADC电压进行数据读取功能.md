好的，我们来梳理一下MCU（以你关注的STM32L451CE为例）通过I2C总线读取TM75温度传感器和外部ADC（模数转换器）电压数据的流程。

实现这个功能，核心是掌握两个层面的操作：一是**I2C通信协议本身**，二是**具体传感器芯片的操作方法**。

### 💡 整体思路：通信流程概览

无论是读取温度还是电压，本质上都是MCU作为I2C主机，按照以下流程与作为从机的传感器芯片进行数据交换：

1. **初始化I2C外设**：配置好MCU的I2C引脚（SCL、SDA）、时钟速度、寻址模式等。
    
2. **向从机写数据（可选）**：对于需要配置的芯片（如ADC），需要先写入配置寄存器，设定工作模式、通道、量程等。
    
3. **向从机读数据**：发送读取指令，从芯片的特定寄存器中获取原始数据。
    
4. **数据处理**：将读取到的原始数据（如ADC的二进制值）根据芯片手册的公式换算成实际的电压值或温度值。
    

下面分别说明TM75和ADC芯片的具体操作差异。

### 🌡️ 读取TM75温度传感器

TM75是一类数字温度传感器的典型代表，它内部有可以直接读取温度值的寄存器，操作相对简单。

- **芯片特点**：TM75使用I2C/SMBus接口，支持7位寻址，一条总线上最多可挂载8个设备[](https://training-dev.ti.com/tool/TMP75BEVM)。它的温度数据通常以12位或更高分辨率存储在只读寄存器中。
    
- **读取流程**：
    
    1. **初始化**：配置STM32的I2C外设。
        
    2. **读取温度寄存器**：通常只需一步，即向TM75发送读取命令。有些型号可能先要写入寄存器指针地址（如0x00指向温度寄存器），然后开始读取。
        
    3. **数据解析**：读取到的2个字节（高字节和低字节）按照芯片手册规定的格式（如高字节是整数部分，低字节是小数部分）组合，再通过公式换算成温度值。例如，一个常见的12位分辨率数据，转换公式可能是`温度(°C) = 原始值 * 0.0625`[](https://training-dev.ti.com/tool/TMP75BEVM)[](https://www.ti.com.cn/tool/cn/TMP75BEVM)。
        

### ⚡ 读取外部ADC电压

读取ADC芯片稍复杂一些，因为通常需要先通过I2C指令配置芯片，再读取结果。这里以一个常见的16位ADC芯片（如ADS1115系列）的流程为例说明[](https://www.iczoom.com/t-news/78478-industry.html#2)：

- **芯片特点**：ADS1115有多个输入通道，可通过I2C配置其工作模式（单次/连续）、通道选择、量程（PGA）和数据速率。
    
- **读取流程**：
    
    1. **初始化并配置芯片**：通过I2C向ADS1115的**配置寄存器（Config Register）** 写入一个配置字。这个配置字决定了使用哪个通道、量程范围、转换模式等[](https://www.iczoom.com/t-news/78478-industry.html#2)。这是一个典型的“主机写”操作。
        
    2. **启动转换并等待**：配置完成后，芯片开始进行模数转换。如果是单次模式，需要等待转换完成（或查询状态引脚），通常需要几毫秒的延时[](https://www.iczoom.com/t-news/78478-industry.html#2)。
        
    3. **读取转换结果**：通过I2C从ADS1115的**转换寄存器（Conversion Register）** 中读取16位的结果数据[](https://www.iczoom.com/t-news/78478-industry.html#2)。这是一个“主机读”操作。
        
    4. **电压换算**：将读取到的16位有符号整数，根据配置的量程（例如±4.096V）换算成实际电压值。计算公式为：`电压(V) = 原始ADC值 * 满量程电压 / (2^15 - 1)`[](https://www.iczoom.com/t-news/78478-industry.html#2)。
        

### 🛠️ STM32 HAL库代码示例（核心操作）

在你使用的STM32L4系列MCU上，使用HAL库可以很方便地实现上述流程。以下是根据搜索到的文档总结的几个关键函数[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)：

|操作类型|核心HAL函数|功能说明|
|---|---|---|
|**主机发送数据**|`HAL_I2C_Master_Transmit(&hi2c1, DeviceAddr << 1, pData, Size, Timeout)`|用于向从机（如ADC）发送配置命令和寄存器地址[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。|
|**主机接收数据**|`HAL_I2C_Master_Receive(&hi2c1, DeviceAddr << 1, pData, Size, Timeout)`|用于从从机读取数据，如读取ADC转换结果[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。|
|**写寄存器（简便）**|`HAL_I2C_Mem_Write(&hi2c1, DeviceAddr<<1, RegAddr, MemAddSize, pData, Size, Timeout)`|常用于向有内部寄存器的设备（如ADC）写入配置，操作更直观[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。|
|**读寄存器（简便）**|`HAL_I2C_Mem_Read(&hi2c1, DeviceAddr<<1, RegAddr, MemAddSize, pData, Size, Timeout)`|常用于从有内部寄存器的设备（如TM75、ADC）读取数据[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。|

> **注意**：HAL库函数的设备地址需要**左移一位**，因为HAL库的地址是8位格式，低1位用于表示读写方向[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。

### ⚠️ 关键注意事项

1. **设备地址确认**：必须查阅你所用芯片的数据手册，确认其7位设备地址。同时，确认芯片的地址引脚（如TM75的A0, A1, A2）接法，以确定其在总线上的具体地址。
    
2. **GPIO配置**：STM32的I2C引脚（SCL, SDA）必须配置为**复用开漏（Alternate Function Open-Drain）** 模式，并在外部连接上拉电阻[](https://github.com/Compile-N-Run/Compile-N-Run/blob/main/docs/iot/stm32/5-stm32-communication-protocols/5-stm32-i2c-basics.mdx#1)。
    
3. **时序与延时**：不同芯片的转换速度不同，特别是ADC转换需要时间。代码中要加入适当的延时（如`HAL_Delay()`），确保读取时转换已完成[](https://www.iczoom.com/t-news/78478-industry.html#2)。此外，还要关注I2C总线速度是否符合芯片规格（通常为100kbps或400kbps）。
    
4. **数据手册是根本**：**温度换算公式**和**ADC换算公式**完全取决于具体芯片，务必仔细阅读其数据手册。