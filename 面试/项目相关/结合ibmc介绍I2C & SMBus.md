# 结合 iBMC 理解 I2C/SMBus

## 一、I2C/SMBus 协议基础

### I2C（Inter-Integrated Circuit）

I2C 是 Philips（现 NXP）开发的双线制串行总线，用于微控制器与低速外设之间的通信：

- **物理层**：仅两根信号线 —— **SCL（时钟线）** + **SDA（数据线）**，均需上拉电阻
- **拓扑**：一主多从（Master-Slave），主设备发起所有通信
- **寻址**：7 位或 10 位地址，每个从设备有唯一地址
- **速率**：标准模式 100 kHz、快速模式 400 kHz、快速模式+ 1 MHz、高速模式 3.4 MHz

### SMBus（System Management Bus）

SMBus 是 Intel 在 I2C 基础上制定的**系统管理总线**规范（SMBus 2.0 / 3.0），相比 I2C 增加了：

| 特性 | I2C | SMBus |
|---|---|---|
| 最低电压 | 无规定 | 最低 3V |
| 超时机制 | 无 | 有，防止总线死锁 |
| 数据包校验 | 无 | PEC（Packet Error Checking），CRC-8 校验 |
| SMBus Alert | 无 | 从设备可通过 Alert 引脚主动中断主设备 |
| 地址解析协议 | 无 | ARP（Address Resolution Protocol），动态分配地址 |
| 命令协议 | 无 | 定义了标准化的读/写命令格式 |

**关键区别**：I2C 是通用总线协议，SMBus 是面向系统管理场景的增强子集。在服务器领域，BMC 与传感器/EEPROM 之间的通信几乎都遵循 SMBus 规范。

### PMBus（Power Management Bus）

PMBus 是基于 SMBus 之上的**电源管理协议**，定义了标准化的电源管理命令（如 `READ_VIN`、`READ_POUT`、`VOUT_COMMAND` 等），用于 BMC 与数字电源模块之间的通信。

三者的层级关系：

```
PMBus (电源管理命令集)
  └── SMBus (系统管理总线规范，含超时/PEC/Alert)
        └── I2C (物理层双线串行总线)
```

---

## 二、iBMC 中的 I2C/SMBus 硬件架构

华为 iBMC 芯片（Hi1710/Hi1711）原生提供 **16 路 I2C/SMBus 控制器**，这些总线是 iBMC 与服务器主板上各种低速器件通信的"毛细血管网络"。

```
                    ┌──────────────────────────────────────┐
                    │     iBMC 芯片 (Hi1710 / Hi1711)       │
                    │                                      │
                    │  I2C Controller × 16                 │
                    │  (SMBus 2.0 兼容)                     │
                    └────┬───┬───┬───┬───┬───┬───┬────────┘
                         │   │   │   │   │   │   │
        ┌────────────────┘   │   │   │   │   │   └────────────────┐
        │                    │   │   │   │   │                     │
   ┌────▼────┐    ┌──────────▼┐  │   │  ┌▼─────────┐    ┌────────▼──┐
   │ Bus 0   │    │ Bus 1     │  │   │  │ Bus N    │    │ Bus 15    │
   │ 温度    │    │ 电压      │  │   │  │ CPLD/    │    │ FRU       │
   │ 传感器  │    │ 传感器    │  │   │  │ SMC      │    │ EEPROM    │
   └─────────┘    └───────────┘  │   │  └──────────┘    └───────────┘
                                 │   │
                    ┌────────────┘   └────────────┐
                    │                             │
              ┌─────▼──────┐              ┌───────▼──────┐
              │ PMBus 电源 │              │ IPMB 板间    │
              │ 模块       │              │ 管理总线     │
              └────────────┘              └──────────────┘
```

### 总线隔离设计

将 16 路总线分开使用有重要意义：
- **故障隔离**：某一路 I2C 总线故障（如短路、器件损坏）不会影响其他总线上的设备
- **并行访问**：iBMC 可同时轮询多路总线，提高采集效率
- **地址空间扩展**：每路总线有独立的 7 位地址空间，避免地址冲突

---

## 三、iBMC 通过 I2C/SMBus 管理的设备类型

### 1. 温度传感器

这是 I2C 在 BMC 中最典型的应用。常见的温度传感器芯片：

| 芯片 | 厂商 | I2C 地址 | 精度 | 用途 |
|---|---|---|---|---|
| LM75 / TMP75 | TI / NXP | 0x48-0x4F | ±1°C | 进出风口、主板环境温度 |
| TMP112 | TI | 0x48-0x4B | ±0.25°C | 高精度温度测量点 |
| TMP411 / TMP421 | TI | 0x4C-0x4E | ±1°C | 多通道远程温度传感器 |
| CPU 内部 DTS | Intel/华为 | 通过 PECI 访问 | ±1°C | CPU 核心/封装温度 |

**通信过程**：iBMC 以固定周期（通常 1-3 秒）通过 I2C 轮询各温度传感器寄存器：

```
iBMC (Master) ──START──→ [传感器地址+W] ──ACK──→ [寄存器地址] ──ACK──→
              RESTART ──→ [传感器地址+R] ──ACK──→ [温度数据] ──NACK──→ STOP
```

当温度超过预设阈值时，iBMC 触发告警（如 `ALM-温度过高`），并可能联动调整风扇转速。

### 2. 电压/电流传感器

| 芯片 | 用途 |
|---|---|
| INA226 / INA219 | 电压 + 电流监控，通过 SMBus 读取 |
| ADM1272 | 热插拔控制器 + 功率监控 |
| LTC2977 | 多通道电源监控 |

iBMC 通过 PMBus 命令读取电源模块状态：

```bash
# 读取电源模块输出电压 (PMBus READ_VOUT 命令 0x8B)
i2cget -y 1 0x58 0x8B w

# 读取电源模块输出功率 (PMBus READ_POUT 命令 0x97)
i2cget -y 1 0x58 0x97 w
```

### 3. FRU EEPROM（现场可更换单元信息）

FRU（Field Replaceable Unit）EEPROM 存储了服务器和部件的**资产信息**：

- **内容**：产品序列号、型号、BOM 编码、制造商、生产日期、MAC 地址等
- **芯片**：通常是 24C32 / 24C64 等 I2C EEPROM
- **地址**：常见 0x50-0x57
- **数据格式**：遵循 IPMI FRU Information Storage Definition 规范

iBMC 在启动时扫描所有 FRU EEPROM，建立硬件资产清单。通过 IPMI 命令或 Redfish API 可查看：

```bash
# 通过 IPMI 读取 FRU 信息
ipmitool -H <iBMC_IP> -U Administrator -P xxx fru print

# 通过 Redfish 读取
curl -k https://<iBMC_IP>/redfish/v1/Chassis/1
# 返回的 SerialNumber、Model、Manufacturer 等字段来自 FRU
```

华为 iBMC 还支持通过 OEM IPMI 命令（NetFn `0x30`）读写**电子标签**（E-label），这是华为对 FRU 的扩展。

### 4. CPLD / SMC（卫星管理控制器）

在华为服务器中，CPLD（复杂可编程逻辑器件）通过 I2C 与 iBMC 通信，承担：

- **上电时序控制**：CPLD 负责按顺序开启各路电压
- **复位控制**：iBMC 通过 I2C 向 CPLD 发送复位命令
- **GPIO 扩展**：I2C IO 扩展芯片（如 PCA9555）扩展 iBMC 的 GPIO 数量
- **传感器信息采集**：部分传感器通过 CPLD 间接上报给 iBMC

华为的 SMC（Satellite Manager Controller）架构是 I2C 管理的高级形态：

```
iBMC ──I2C Root Bus──→ EEPROM (地址 0xAE，根链路标识)
                          │
                    ┌─────┴──────┐
                    │ SMC Hub    │
                    └─────┬──────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
      ┌─────▼─────┐ ┌────▼─────┐ ┌────▼─────┐
      │ CPLD      │ │ CPLD     │ │ CPLD     │
      │ (0x60)    │ │ (0x62)   │ │ (0x64)   │
      │ 主板组件  │ │ 背板组件 │ │ 扩展板   │
      └───────────┘ └──────────┘ └──────────┘
```

SMC 协议特点：
- 采用**命令 + 响应**方式交互
- 整个传输过程自带 **CRC 校验**
- 支持组件基本信息查询、CPU 信息、传感器信息、电源控制等

### 5. 风扇控制器

iBMC 通过 I2C/SMBus 向风扇控制芯片写入 PWM 占空比来调速，并读取风扇转速（tachometer）反馈：

- 写入目标转速 → 风扇控制芯片调整 PWM
- 读取实际转速 → 闭环控制 + 故障检测（转速异常低则告警）

### 6. 时钟芯片 / RTC / 其他

- **RTC 电池电压监控**：通过 ADC 或 I2C 电压监控芯片检测 CMOS 电池电压，低于阈值触发 `ALM-主板 RTC 电池电压低`
- **PCIe Retimer 寄存器配置**：部分机型通过 I2C 配置 Retimer 芯片
- **光模块信息读取**：通过 I2C 访问 SFP/QSFP 光模块的 EEPROM，获取型号、功率、温度等

---

## 四、I2C/SMBus 在 iBMC 中的软件实现

### 1. Linux I2C 子系统

iBMC 运行的嵌入式 Linux 内核提供了完整的 I2C 子系统：

```
┌─────────────────────────────────────┐
│       iBMC 应用层                    │
│  sensor_alarm 模块 / FDM 诊断模块    │
├─────────────────────────────────────┤
│       硬件抽象层 (hwmon / sysfs)     │
│  /sys/class/hwmon/hwmon0/temp1_input │
│  /sys/bus/i2c/devices/1-0048/       │
├─────────────────────────────────────┤
│       I2C 总线驱动层                 │
│  hi1710-i2c 驱动 (适配 Hi1710/1711) │
├─────────────────────────────────────┤
│       硬件 (I2C Controller)          │
└─────────────────────────────────────┘
```

### 2. 传感器轮询机制

iBMC 的 `sensor_alarm` 模块以固定周期轮询所有传感器：

```
轮询循环:
  for each 传感器 in 传感器列表:
    1. 通过 I2C 读取原始数据 (raw value)
    2. 根据 SDR (Sensor Data Record) 中的转换公式计算物理值
       物理值 = M × raw + B
    3. 与阈值比较 (Upper Critical / Upper Non-Critical /
                   Lower Non-Critical / Lower Critical)
    4. 如果越限:
       a. 记录 SEL (System Event Log)
       b. 触发告警 (SNMP Trap / Syslog / Email)
       c. 联动风扇调速 / 功率封顶
    5. 等待下一个轮询周期
```

### 3. I2C 故障检测与告警

华为 iBMC 专门定义了 I2C 访问失败的告警：

> **ALM-0x100000C9**：`Failed to access I2C device`（主板，严重告警）

当 iBMC 连续多次无法访问某个 I2C 设备时触发此告警。可能原因：
- I2C 总线物理故障（短路、断路）
- 器件损坏（传感器、EEPROM 失效）
- 板卡未正确连接（背板线缆松动）

处理流程：检查电源线连接 → 重新插拔线缆 → 更换线缆 → 更换故障板卡。

### 4. 一键收集中的 I2C 数据

在 iBMC "一键收集"的日志包中，I2C 相关数据分布在：

- `dump_info/DeviceDump/i2c_info/` — I2C 设备的寄存器/存储区原始 dump
- `dump_info/AppDump/sensor_alarm/sensor_info.txt` — 所有传感器当前值
- `dump_info/AppDump/BMC/fruinfo.txt` — FRU 电子标签信息
- `dump_info/RTOSDump/driver_info/` — I2C 驱动信息

---

## 五、I2C/SMBus 与 iBMC 其他总线的协作

```
               iBMC 芯片
          ┌──────────────────────────────────────────────┐
          │                                              │
  I2C/SMBus │  温度/电压/风扇/FRU EEPROM/CPLD/电源模块     │ ←── 硬件监控主通道
          │                                              │
   IPMB   │  板间管理（多节点服务器中与其他 BMC 通信）      │ ←── I2C 的扩展应用
  (I2C)   │                                              │
          │                                              │
  eSPI/LPC │  BIOS POST 码、带内 IPMI (BT/KCS 接口)       │ ←── 与 CPU 通信
          │                                              │
  PCIe+MCTP│  RAID 卡/NVMe SSD/NPU 带外管理、黑匣子       │ ←── 高速设备管理
          │                                              │
   UART   │  SOL 串口重定向、智能网卡串口                   │ ←── 文本控制台
          │                                              │
   GPIO   │  LED 控制、按钮检测、开机信号                   │ ←── 离散信号
          │                                              │
   ADC    │  电压模拟量直接采集（电池电压等）               │ ←── 模拟量采集
          │                                              │
   PWM    │  风扇转速控制                                  │ ←── 模拟量输出
          │                                              │
          └──────────────────────────────────────────────┘
```

I2C/SMBus 在 iBMC 中的定位是**低速硬件监控的主通道**，承担数据量最大、设备种类最多的采集任务。高速设备管理（RAID/NVMe/NPU）交给 PCIe+MCTP，与 CPU 的通信交给 eSPI/LPC，I2C 专注做它最擅长的事 —— 以极低的硬件成本连接大量低速传感器和管理器件。

---

## 六、I2C/SMBus 的局限性

| 局限 | 影响 |
|---|---|
| **带宽极低** | 标准模式 100 kHz，快速模式 400 kHz；不适合传输大数据量 |
| **不支持中断驱动** | iBMC 必须轮询读取（SMBus Alert 机制部分缓解，但非所有器件支持） |
| **地址冲突风险** | 多个器件共享总线时地址可能冲突，需在设计阶段规划 |
| **总线长度受限** | I2C 适合 PCB 板内通信（通常 < 30cm），不适合板间长距离通信（板间用 IPMB） |
| **单点故障** | 一条总线上的短路可能导致整条总线不可用（iBMC 通过分 16 路缓解） |

这些局限也正是 iBMC 采用多总线（16 路 I2C + PCIe MCTP + eSPI + IPMB）混合架构的原因 —— 不同速率、不同距离、不同可靠性的设备使用不同的总线。
