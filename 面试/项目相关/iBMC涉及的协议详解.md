# iBMC 涉及的协议详解

> 华为 iBMC（Hi1710/Hi1711 芯片平台）中使用的全部通信协议，按层次从硬件总线到应用层逐一详解。

---

## 协议全景索引

```
┌─────────────────────────────────────────────────────────────────┐
│                    iBMC 协议全景                                  │
├─────────────┬───────────────────────────────────────────────────┤
│  硬件总线层  │ I2C/SMBus · PMBus · IPMB · MCTP                  │
│             │ eSPI/LPC · PCIe · UART · JTAG · NC-SI/RMII       │
├─────────────┼───────────────────────────────────────────────────┤
│  传输协议层  │ RMCP · RMCP+ · SNMP · HTTPS/TLS · SSH           │
│             │ NTP · Syslog · SMTP · LDAP                       │
├─────────────┼───────────────────────────────────────────────────┤
│  管理协议层  │ IPMI 2.0 · Redfish · DCMI · PLDM · SPDM         │
├─────────────┼───────────────────────────────────────────────────┤
│  带内通信层  │ BT/KCS · SOL · KVM over IP · VMedia · NC-SI     │
├─────────────┼───────────────────────────────────────────────────┤
│  安全协议层  │ TLS 1.2 · AES-CTR · HMAC-SHA256 · RAKP · TCM   │
└─────────────┴───────────────────────────────────────────────────┘
```

---

## 第一部分：硬件总线协议

### 1. I2C / SMBus

#### 1.1 协议概述

| 属性 | I2C | SMBus |
|---|---|---|
| **全称** | Inter-Integrated Circuit | System Management Bus |
| **制定者** | Philips（现 NXP） | Intel |
| **物理层** | SCL + SDA 双线 | 兼容 I2C 物理层 |
| **拓扑** | 一主多从（Master-Slave） | 一主多从 |
| **寻址** | 7 位 / 10 位 | 7 位（含 ARP 动态分配） |
| **速率** | 100K / 400K / 1M / 3.4M | 10K ~ 100K（最低保证） |
| **规范** | I2C-bus Specification (NXP) | SMBus Spec 2.0 / 3.0 |

#### 1.2 报文结构

```
I2C 基本读写时序：

写操作：
S → [Addr+W] → ACK → [RegAddr] → ACK → [Data] → ACK → P

读操作：
S → [Addr+W] → ACK → [RegAddr] → ACK →
Sr → [Addr+R] → ACK → [Data] → NACK → P

S = START 条件    P = STOP 条件
Sr = REPEATED START
ACK = 应答（低电平）   NACK = 非应答（高电平）
```

#### 1.3 SMBus 增强特性

| 特性 | 说明 |
|---|---|
| **超时机制** | 从设备在 35ms 内必须释放 SCL，否则主设备可复位总线；防止总线死锁 |
| **PEC（Packet Error Checking）** | CRC-8 校验，多项式 x⁸+x²+x+1；附加在每个数据包末尾 |
| **SMBus Alert** | 从设备可通过 Alert# 引脚主动中断主设备，减少轮询开销 |
| **ARP（Address Resolution Protocol）** | 动态分配从设备地址，解决地址冲突 |
| **Host Notify** | 从设备向主设备主动发送 3 字节通知 |

#### 1.4 iBMC 中的使用

- Hi1710/Hi1711 提供 **16 路 I2C/SMBus 控制器**
- 典型器件：TMP75（温度）、INA226（电压/电流）、24C64（FRU EEPROM）、PCA9555（GPIO 扩展）
- 典型周期：传感器轮询 1~3 秒/次
- 告警机制：连续访问失败触发 `ALM-0x100000C9`（Failed to Access I2C Device）

---

### 2. PMBus（Power Management Bus）

#### 2.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Power Management Bus |
| **制定者** | SMIF（System Management Interface Forum） |
| **基于** | SMBus 物理层 + 扩展命令集 |
| **版本** | PMBus 1.2 / 1.3 |
| **适用** | 数字电源模块、DC-DC 转换器、VRM |

#### 2.2 核心命令集

| 命令代码 | 命令名 | 功能 |
|---|---|---|
| 0x8B | `READ_VOUT` | 读取输出电压 |
| 0x8C | `READ_IOUT` | 读取输出电流 |
| 0x96 | `READ_TEMPERATURE_1` | 读取电源模块温度 |
| 0x97 | `READ_POUT` | 读取输出功率 |
| 0x88 | `READ_VIN` | 读取输入电压 |
| 0x89 | `READ_IIN` | 读取输入电流 |
| 0x95 | `READ_PIN` | 读取输入功率 |
| 0x8D | `READ_FAN_SPEED_1` | 读取风扇转速 |
| 0x78 | `STATUS_BYTE` | 读取 8 位状态字 |
| 0x79 | `STATUS_WORD` | 读取 16 位状态字 |
| 0x21 | `VOUT_COMMAND` | 设置目标输出电压 |

#### 2.3 iBMC 中的使用

- iBMC 通过 PMBus 管理服务器 PSU（电源模块）
- 读取 PSU 输入/输出功率、效率、温度、风扇转速
- 实现 DCMI 1.5 功率封顶（Power Capping）
- PSU 故障时通过 PMBus STATUS_WORD 上报错误码

```
iBMC ──PMBus──→ PSU #1 (地址 0x58)
                PSU #2 (地址 0x59)
                
读取流程：
iBMC → [0x58+W] → [0x8B(READ_VOUT)] → [0x58+R] → [电压数据] → PEC
```

---

### 3. IPMB（Intelligent Platform Management Bus）

#### 3.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Intelligent Platform Management Bus |
| **基于** | I2C 物理层 |
| **用途** | 多节点服务器中 BMC 之间的板间管理通信 |
| **速率** | 100 kHz |
| **拓扑** | 多主（Multi-Master），支持总线仲裁 |

#### 3.2 报文结构

```
IPMB 消息格式：

┌─────────┬──────────┬────────┬─────────┬──────────┬────────────┬─────┐
│ rsSA    │ NetFn/LUN│ rsSeq  │ rqSA    │ rqSeq/LUN│ rqCmd      │Data │
│ (8bit)  │ (8bit)   │ (8bit) │ (8bit)  │ (8bit)   │ (8bit)     │(N×8)│
└─────────┴──────────┴────────┴─────────┴──────────┴────────────┴─────┘

rsSA  = 响应者从地址（目标 BMC）
rqSA  = 请求者从地址（源 BMC）
NetFn = 网络功能码（标识消息类别）
rqCmd = 命令码
```

#### 3.3 iBMC 中的使用

- 多节点服务器（如华为 TaiShan 高密服务器）中，多个 iBMC 通过 IPMB 互联
- 交换模块（MM910/MM920）通过 IPMB 与各计算节点的 iBMC 通信
- 支持跨节点的传感器查询、电源管理、固件升级

---

### 4. MCTP（Management Component Transport Protocol）

#### 4.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Management Component Transport Protocol |
| **制定者** | DMTF（Distributed Management Task Force） |
| **规范** | DSP0236（Base Spec，当前 1.3.3 / 2.0） |
| **用途** | 管理控制器之间、管理控制器与管理设备之间的统一传输协议 |

#### 4.2 核心设计思想

MCTP 是一个**传输无关的通用消息协议**：

```
                    ┌──────────────────────────┐
                    │  上层消息类型              │
                    │  ┌──────────────────────┐ │
                    │  │ PLDM (平台级数据模型)  │ │
                    │  │ NCSI (网卡边带管理)    │ │
                    │  │ SPDM (安全协议)        │ │
                    │  │ OEM (厂商自定义)       │ │
                    │  └──────────────────────┘ │
                    ├──────────────────────────┤
                    │  MCTP 传输层 (DSP0236)    │  ← 统一的消息格式
                    │  分段/重组/确认/错误检测   │
                    ├──────────────────────────┤
                    │  物理传输绑定              │
                    │  ┌──────────────────────┐ │
                    │  │ PCIe (VDM/TLP)        │ │
                    │  │ SMBus/I2C             │ │
                    │  │ UART                  │ │
                    │  │ USB                   │ │
                    │  └──────────────────────┘ │
                    └──────────────────────────┘
```

#### 4.3 MCTP 报文结构

```
MCTP Base Header (4 bytes):
┌──────────┬───────────────┬──────────────┬──────────────────────┐
│ Reserved │ Header Version│ Endpoint ID  │ SOM/EOM/PktSeq/Tag  │
│ (4 bits) │ (4 bits)      │ (8 bits)     │ (各1-2 bits)         │
└──────────┴───────────────┴──────────────┴──────────────────────┘

SOM  = Start of Message（消息起始标记）
EOM  = End of Message（消息结束标记）
PktSeq = Packet Sequence Number（包序号，用于分段重组）
Tag  = 消息标签（匹配请求/响应）

Message Type Header (1 byte):
┌────────────────────┬──────────┐
│ Message Type (5bit)│ IC (1bit)│
└────────────────────┴──────────┘

消息类型 ID：
0x00 = MCTP Control
0x01 = SMBus/I2C
0x02 = PCIe VDM
0x3F = PLDM (DSP0240)
0x04 = NCSI (DSP0222)
0x05 = SPDM (DSP0274)
```

#### 4.4 MCTP 网络模型

```
MCTP Endpoint (EID = 8)         MCTP Endpoint (EID = 9)
    │                                  │
    │  MCTP Message                    │
    ▼                                  ▼
┌──────────┐                      ┌──────────┐
│ PCIe     │  ←─ MCTP Network ─→ │ SMBus    │
│ Binding  │     (Bus Owner)      │ Binding  │
└──────────┘                      └──────────┘
    │                                  │
    ▼                                  ▼
 RAID Controller                 Temperature Sensor
```

- **Bus Owner (BO)**：总线管理者，通常是 BMC，负责分配 EID、路由消息
- **Endpoint (EP)**：管理设备（RAID 卡、NVMe SSD、NPU、网卡等），每个 EP 有唯一的 EID（Endpoint ID，8 bit）
- **Bridge**：跨物理介质转发 MCTP 消息（如 PCIe ↔ SMBus）

#### 4.5 iBMC 中的使用

| 场景 | 传输绑定 | 管理对象 |
|---|---|---|
| RAID 控制器管理 | MCTP over PCIe | RAID 逻辑盘/物理盘状态 |
| NVMe SSD 管理 | MCTP over PCIe | 健康度、温度、寿命 |
| NPU/GPU 管理 | MCTP over PCIe | 温度、功耗、状态 |
| 网卡管理 | MCTP over PCIe（NCSI-over-MCTP） | 网卡配置、统计 |
| 智能网卡/DPU | MCTP over PCIe（PLDM-over-MCTP） | DPU 带外管理 |
| 黑匣子数据 | MCTP over PCIe | iBMA Agent 崩溃数据写入 |

---

### 5. eSPI / LPC

#### 5.1 协议对比

| 属性 | LPC (Low Pin Count) | eSPI (Enhanced SPI) |
|---|---|---|
| **制定者** | Intel | Intel |
| **数据线** | 4 位并行 (LAD[3:0]) | SPI 四线（MOSI/MISO/SCLK/CS） |
| **速率** | 33 MHz | 最高 66 MHz |
| **用途** | 传统 ISA 设备替代 | LPC 的下一代替代 |
| **通道** | 单通道 | 多通道（Peripheral / VW / OOB / Flash） |

#### 5.2 iBMC 中的使用

| 通道 | 功能 |
|---|---|
| **Peripheral 通道** | 80h Port（POST 码）：BIOS 启动过程中向 iBMC 上报诊断码 |
| **VW (Virtual Wire) 通道** | 虚拟 GPIO 信号（如 SMI#、SCI#、PLTRST#） |
| **OOB (Out-of-Band) 通道** | 带内 IPMI 通信（BT/KCS/SSIF 接口的物理承载） |
| **Flash 通道** | 共享 SPI Flash 访问（BIOS Flash 远程更新） |

**POST 码捕获流程**：

```
BIOS POST 过程：
  BIOS 写入 Port 80h → eSPI/LPC → iBMC 捕获
  → iBMC 记录 SEL（System Event Log）
  → WebUI 显示 POST 进度
  
典型 POST 码含义：
  0x62 = DXE 阶段开始
  0x92 = PCI 总线初始化
  0xA6 = SCSI 子系统初始化
  0xD6 = 控制台输出初始化
```

---

### 6. PCIe（Peripheral Component Interconnect Express）

#### 6.1 在 iBMC 中的角色

iBMC 通过 PCIe 通道实现：

| 功能 | 说明 |
|---|---|
| **MCTP over PCIe VDM** | 使用 PCIe Vendor Defined Message 传输 MCTP 消息，管理 RAID/NVMe/NPU 等 PCIe 设备 |
| **黑匣子** | 主机 OS 蓝屏/内核崩溃时，iBMA Agent 通过 PCIe 将 crash kernel 写入 iBMC 的 4MB 黑匣子存储 |
| **共享 Flash 访问** | iBMC 和 CPU 共享 BIOS SPI Flash 的访问权 |

#### 6.2 PCIe VDM 报文结构

```
PCIe TLP (Transaction Layer Packet):
┌──────┬────────┬──────────┬──────────────┬───────────┐
│ Fmt  │ Type   │ Length   │ Requester ID │ MCTP Data │
│(3bit)│(5bit)  │(10bit)   │(16bit)       │(variable) │
└──────┴────────┴──────────┴──────────────┴───────────┘

Fmt = 011 (Message with data)
Type = 11100 (Vendor Defined Type 0)
```

---

### 7. NC-SI（Network Controller Sideband Interface）

#### 7.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Network Controller Sideband Interface |
| **制定者** | DMTF |
| **规范** | DSP0222（当前 1.2.0） |
| **用途** | BMC 与网卡之间的边带管理通信 |
| **物理层** | RMII（Reduced MII）/ RGMII |

#### 7.2 NC-SI 核心价值

**管理平面与业务平面分离**：

```
┌──────────────────────────────────────────────┐
│              物理网卡 (NIC PHY)                │
│                                              │
│  ┌──────────┐          ┌──────────────────┐  │
│  │ 业务端口  │          │ NC-SI 边带端口    │  │
│  │ (Host)   │          │ (MC/BMC)         │  │
│  └────┬─────┘          └────────┬─────────┘  │
│       │                         │             │
│  ┌────▼─────────────────────────▼──────────┐  │
│  │         MAC 层（共享）                    │  │
│  │         VLAN 隔离管理/业务流量            │  │
│  └────────────────┬────────────────────────┘  │
│                   │                           │
└───────────────────┼───────────────────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    ┌────▼────┐          ┌────▼────┐
    │ 主机 OS │          │  iBMC   │
    │ (业务)  │          │ (管理)  │
    └─────────┘          └─────────┘
```

- iBMC 与业务网卡共享物理网口，节省独立管理网口
- 通过 VLAN 实现管理流量与业务流量隔离，互不可见
- 即使主机 OS 宕机，iBMC 的网络管理仍可用

#### 7.3 NC-SI 报文结构

```
NC-SI Command Packet:
┌──────────┬──────────────┬────────┬──────────┬──────────────┐
│ MC ID    │ Header Rev   │ IID    │ Ctrl Pkt │ Channel ID   │
│ (8 bit)  │ (8 bit)      │ (8 bit)│ Type(16) │ (8 bit)      │
├──────────┼──────────────┼────────┼──────────┼──────────────┤
│ Reserved │ Payload Len  │             Payload              │
│ (8 bit)  │ (16 bit)     │          (variable)              │
├──────────┼──────────────┼──────────────────────────────────┤
│                    Checksum (32 bit)                       │
├─────────────────────────────────────────────────────────────┤
│                    Padding (0-3 bytes)                      │
└─────────────────────────────────────────────────────────────┘

核心命令类型：
0x0001 = Clear Initial State
0x0002 = Select Package
0x0003 = Deselect Package
0x0004 = Enable Channel
0x0005 = Disable Channel
0x0015 = Get Version ID
0x0016 = Get Capabilities
0x001A = Get Link Status
0x0050 = Enable VLAN (VLAN 过滤使能)
0x0051 = Set VLAN Filter
```

#### 7.4 iBMC 中的使用

- 华为 TaiShan 系列服务器默认支持 NC-SI 边带网络
- iBMC 通过 RMII 接口连接网卡 PHY 的 NC-SI 端口
- 管理 VLAN 与业务 VLAN 隔离
- 支持 NC-SI over MCTP（PCIe 通道传输 NC-SI 命令）

---

### 8. UART（Universal Asynchronous Receiver/Transmitter）

#### 8.1 iBMC 中的使用

| 串口 | 用途 |
|---|---|
| UART 0 | SOL（Serial Over LAN）—— 主机串口输出远程转发 |
| UART 1 | iBMC 自身调试串口 |
| UART 2-3 | 智能网卡串口通信 |
| UART 4-7 | 预留 / 扩展设备 |

**SOL 工作原理**：

```
主机 BIOS/OS 串口输出 → UART → iBMC 捕获
  → iBMC 通过 RMCP+ 会话转发给远程客户端
  → 客户端（如 ipmitool sol activate）显示输出

远程输入 → iBMC → UART → 主机 BIOS/OS
```

---

## 第二部分：北向网络传输协议

### 9. RMCP / RMCP+

#### 9.1 RMCP（Remote Management Control Protocol）

| 属性 | 说明 |
|---|---|
| **全称** | Remote Management Control Protocol |
| **规范** | ASF 2.0（Alert Standard Format，IPMI 1.5 时代） |
| **传输层** | UDP |
| **端口** | 623（默认主用）、664（备用） |
| **认证** | 无认证 / MD5（不安全） |

**RMCP 报文结构**：

```
RMCP Header (4 bytes):
┌──────────┬──────────┬──────────┬──────────┐
│ Version  │ Reserved │ Reserved │ Msg Class│
│ (8 bit)  │ (8 bit)  │ (8 bit)  │ (8 bit)  │
└──────────┴──────────┴──────────┴──────────┘

Version = 0x06
Msg Class = 0x07 (IPMI) / 0x06 (ASF)
```

> **安全警告**：RMCP 无加密，存在已知漏洞（CVE-2013-4786 密码哈希泄露），华为 iBMC 默认禁用 RMCP。

#### 9.2 RMCP+（RMCP Plus）

| 属性 | 说明 |
|---|---|
| **全称** | Remote Management and Control Protocol Plus |
| **规范** | IPMI 2.0 规范 |
| **传输层** | UDP |
| **端口** | 623 / 664 |
| **认证** | RAKP-HMAC-SHA256（推荐） |
| **加密** | AES-CBC-128 / AES-128 |

**RMCP+ 会话建立流程**：

```
客户端                                    iBMC (BMC)
  │                                         │
  │ ① Get Channel Auth Capabilities         │
  │────────────────────────────────────────→│
  │←────────────────────────────────────────│
  │   (返回支持的认证方式列表)                │
  │                                         │
  │ ② Open Session Request                   │
  │────────────────────────────────────────→│
  │   (提议 Cipher Suite ID)                 │
  │←────────────────────────────────────────│
  │   Open Session Response                  │
  │   (确认 Session ID + Cipher Suite)       │
  │                                         │
  │ ③ RAKP Message 1                         │
  │────────────────────────────────────────→│
  │   (用户名 + 随机数)                       │
  │←────────────────────────────────────────│
  │   RAKP Message 2                         │
  │   (BMC 随机数 + HMAC-SHA256 校验)        │
  │                                         │
  │ ④ RAKP Message 3                         │
  │────────────────────────────────────────→│
  │   (客户端 HMAC-SHA256 校验 + 权限级别)    │
  │←────────────────────────────────────────│
  │   RAKP Message 4                         │
  │   (确认认证成功)                          │
  │                                         │
  │ ⑤ 加密会话建立                            │
  │   后续所有 IPMI 命令均在加密通道中传输     │
  │═══════════════════════════════════════════│
```

**Cipher Suite ID 定义**：

| ID | 认证算法 | 完整性算法 | 加密算法 | 安全性 |
|---|---|---|---|---|
| 0 | None | None | None | 无安全 |
| 1 | HMAC-SHA1 | None | None | 弱 |
| 2 | HMAC-SHA1 | HMAC-SHA1-96 | None | 中 |
| 3 | HMAC-SHA1 | HMAC-SHA1-96 | AES-CBC-128 | 中 |
| 17 | HMAC-SHA256 | HMAC-SHA256-128 | AES-128 | **强（推荐）** |

华为 iBMC 默认启用 **Cipher Suite 17**，这是当前最安全的 IPMI 加密套件。

---

### 10. SNMP（Simple Network Management Protocol）

#### 10.1 协议概述

| 属性 | SNMPv1 | SNMPv2c | SNMPv3 |
|---|---|---|---|
| **认证** | Community String | Community String | USM（用户安全模型） |
| **加密** | 无 | 无 | DES / AES |
| **完整性** | 无 | 无 | HMAC-MD5 / HMAC-SHA |
| **端口** | UDP 161 | UDP 161 | UDP 161 |

#### 10.2 iBMC SNMP MIB 结构

```
HUAWEI-SERVER-IBMC-MIB
  ├── huaweiServerInfo          ← 服务器基本信息
  │   ├── productName           ← 产品型号
  │   ├── serialNumber          ← 序列号
  │   ├── bmcVersion            ← iBMC 固件版本
  │   └── biosVersion           ← BIOS 版本
  │
  ├── huaweiServerSensor        ← 传感器数据
  │   ├── sensorTable           ← 传感器列表
  │   │   ├── sensorName        ← 传感器名称
  │   │   ├── sensorReading     ← 当前读数
  │   │   ├── sensorUnit        ← 单位
  │   │   └── sensorStatus      ← 状态 (Normal/Warning/Critical)
  │   └── thresholdTable        ← 阈值配置
  │
  ├── huaweiServerPower         ← 电源管理
  │   ├── powerSupplyTable      ← 电源模块列表
  │   ├── powerConsumption      ← 当前功耗
  │   └── powerCap              ← 功率封顶值
  │
  ├── huaweiServerAlarm         ← 告警信息
  │   └── alarmTable            ← 当前告警列表
  │
  └── huaweiServerStorage       ← 存储信息
      ├── raidControllerTable   ← RAID 控制器
      └── diskTable             ← 硬盘列表
```

#### 10.3 SNMP Trap（主动告警）

```
iBMC 检测到硬件故障
  → 构造 SNMP Trap 报文
  → 发送到配置的 Trap 目标（网管系统）
  
Trap OID 示例：
  hwServerAlarmTrap    ← 硬件告警
  hwServerPowerTrap    ← 电源告警
  hwServerStorageTrap  ← 存储告警
  
iBMC 支持的 Trap 版本：
  - SNMPv1 Trap
  - SNMPv2c Inform（带确认）
  - SNMPv3 Trap（加密）
```

---

### 11. HTTPS / TLS

#### 11.1 iBMC 的 HTTPS 实现

| 属性 | 说明 |
|---|---|
| **Web 服务器** | Apache HTTP Server |
| **端口** | 443（HTTPS）、80（HTTP，自动跳转 HTTPS） |
| **TLS 版本** | TLS 1.2（推荐）/ TLS 1.3 |
| **证书** | 自签名证书（出厂默认）或用户导入 CA 证书 |
| **加密套件** | ECDHE-RSA-AES256-GCM-SHA384 等高强度套件 |
| **并发** | 最多 4 个 WebUI 用户同时登录 |

#### 11.2 HTTPS 承载的服务

```
HTTPS 443
  ├── WebUI（HTML5 管理界面）
  ├── Redfish RESTful API
  ├── 固件升级（上传固件镜像）
  └── 证书管理（导入/替换 SSL 证书）
```

---

### 12. SSH（Secure Shell）

| 属性 | 说明 |
|---|---|
| **端口** | TCP 22 |
| **版本** | SSH-2.0 |
| **加密算法** | AES128-CTR / AES192-CTR / AES256-CTR |
| **密钥交换** | Diffie-Hellman Group 14 (2048-bit) |
| **主机密钥** | RSA 2048 / ECDSA 256 |
| **并发** | 最多 5 个 SSH 会话 |

---

### 13. NTP / Syslog / SMTP / LDAP

| 协议 | 端口 | iBMC 用途 |
|---|---|---|
| **NTP** | UDP 123 | 同步网络时间，确保 SEL 时间戳准确；支持 NTP Server 配置 |
| **Syslog** | UDP 514 | 将 iBMC 运行日志、告警日志转发到远端 Syslog 服务器 |
| **SMTP** | TCP 25/465 | 告警邮件通知，iBMC 检测到硬件故障时自动发送邮件 |
| **LDAP** | TCP 389 | 轻量目录访问协议，集中化用户认证 |
| **LDAP over SSL** | TCP 636 | LDAPS，加密的 LDAP 通信 |
| **Active Directory** | TCP 389/636 | 微软 AD 域集成，Windows 域账号登录 iBMC |

---

## 第三部分：管理协议层

### 14. IPMI 2.0（Intelligent Platform Management Interface）

#### 14.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Intelligent Platform Management Interface |
| **制定者** | Intel / HP / Dell / NEC |
| **版本** | IPMI 1.5 / IPMI 2.0 |
| **规范** | IPMI v2.0 Specification + Errata |
| **传输** | LAN (RMCP/RMCP+) / BT / KCS / SSIF / Serial |

#### 14.2 IPMI 消息结构

```
IPMI Message:
┌──────────────┬─────────────────────┬───────────────┐
│ IPMI Header  │ Message Data        │ Checksum      │
├──────────────┼─────────────────────┼───────────────┤
│ rsAddr(8bit) │ NetFn/rsLUN(8bit)  │               │
│ rsSeq(8bit)  │ rqSeq/rqLUN(8bit)  │               │
│ rqAddr(8bit) │ Command(8bit)       │               │
│              │ Request Data(N×8)   │               │
└──────────────┴─────────────────────┴───────────────┘
```

#### 14.3 NetFn（网络功能码）

| NetFn | 名称 | 功能 |
|---|---|---|
| 0x00/0x01 | Chassis | 机箱控制（电源、重启、定位灯） |
| 0x02/0x03 | Bridge | 桥接消息 |
| 0x04/0x05 | Sensor/Event | 传感器和事件 |
| 0x06/0x07 | App | 应用层（BMC 设备信息、看门狗） |
| 0x08/0x09 | Firmware | 固件升级 |
| 0x0A/0x0B | Storage | 存储（SEL/SDR/FRU） |
| 0x0C/0x0D | Transport | 传输配置（LAN/Serial） |
| 0x2C/0x2D | Group Extension | 扩展功能 |
| **0x30/0x31** | **OEM (华为)** | **华为私有命令（电子标签、TPM 等）** |

#### 14.4 常用 IPMI 命令示例

```bash
# 电源控制
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 power on/off/cycle/reset/status

# 传感器列表
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 sensor list

# SEL 日志
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 sel list
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 sel elist

# FRU 信息
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 fru print

# SOL 串口
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 sol activate

# 风扇转速
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 raw 0x30 0x70 0x66 0x00

# 华为 OEM 命令：读取电子标签
ipmitool -H <iBMC_IP> -U admin -P xxx -C 17 raw 0x30 0x61 0x01 0x00 0x00
```

#### 14.5 IPMI 本地通道

| 接口 | 说明 |
|---|---|
| **KCS（Keyboard Controller Style）** | 传统 ISA 风格接口，OS 侧通过 I/O 端口 0xCA2-0xCA5 访问 |
| **BT（Block Transfer）** | 块传输接口，支持大数据量，OS 侧通过 I/O 端口 0xE4-0xE7 访问 |
| **SSIF（SMBus System Interface）** | 基于 SMBus 的 IPMI 接口，OS 侧通过 I2C 总线访问 |

---

### 15. Redfish

#### 15.1 协议概述

| 属性 | 说明 |
|---|---|
| **全称** | Redfish Scalable Platforms Management API |
| **制定者** | DMTF |
| **规范** | DSP0266（Redfish Specification 1.20.1） |
| **架构** | RESTful API + JSON + HTTPS |
| **数据模型** | OData v4 + JSON Schema |

#### 15.2 资源树（iBMC 实现）

```
/redfish/v1 (ServiceRoot)
├── Systems/{id}
│   ├── Boot, Bios, Processors, Memory
│   ├── EthernetInterfaces, Storages
│   ├── LogServices/SEL, LogServices/CrashDump
│   └── Actions: Reset, SecureBoot
│
├── Managers/{id}
│   ├── NetworkProtocol, EthernetInterfaces
│   ├── VirtualMedia, SerialInterfaces
│   ├── LogServices: OperateLog, SecurityLog, RunLog
│   └── Actions: Reset, ResetToDefaults
│
├── Chassis/{id}
│   ├── Power (PowerSupplies, Voltages)
│   ├── Thermal (Fans, Temperatures)
│   ├── PCIeDevices, NetworkAdapters
│   └── Sensors
│
├── AccountService (Accounts, Roles)
├── EventService (Subscriptions)
├── UpdateService (FirmwareInventory, SimpleUpdate)
├── TaskService (Tasks)
├── DataAcquisitionService (华为 OEM 数据采集)
└── Oem/Huawei (华为私有扩展)
```

#### 15.3 与 IPMI 的定位

| 维度 | Redfish | IPMI |
|---|---|---|
| 数据格式 | JSON，人类可读 | 二进制报文 |
| 安全 | TLS + Token + RBAC | RMCP+（较弱） |
| 可扩展 | OEM 扩展 + Schema 自描述 | OEM NetFn（无自描述） |
| 趋势 | 主力发展方向 | 保持兼容，逐步被替代 |

---

### 16. DCMI（Data Center Manageability Interface）

| 属性 | 说明 |
|---|---|
| **制定者** | Intel（后被 DMTF 纳入） |
| **版本** | DCMI 1.5 |
| **基于** | IPMI 扩展 |
| **用途** | 数据中心级功率管理和资产追踪 |

**核心功能**：

| 功能 | 说明 |
|---|---|
| **Power Reading** | 读取服务器实时功耗、平均功耗、峰值功耗 |
| **Power Capping** | 设置服务器功率上限（硬封顶/软封顶） |
| **Power Limiting** | 功率限制策略管理 |
| **Asset Tag** | 资产标签管理 |
| **Temperature Reading** | 进出风口温度读取（标准化传感器 ID） |

**iBMC 实现**：通过 IPMI 的 NetFn 0x2C（Group Extension）承载 DCMI 命令。

---

### 17. PLDM（Platform Level Data Model）

| 属性 | 说明 |
|---|---|
| **全称** | Platform Level Data Model |
| **制定者** | DMTF |
| **规范** | DSP0240（PLDM Base Specification） |
| **承载** | MCTP 传输层 |
| **用途** | 平台级硬件数据模型，标准化传感器/固件/库存信息 |

**PLDM 子规范**：

| 规范 | 功能 |
|---|---|
| DSP0245 | PLDM for Platform Monitoring and Control（传感器监控） |
| DSP0248 | PLDM for Firmware Update（固件升级） |
| DSP0246 | PLDM for FRU Data（FRU 库存数据） |
| DSP0249 | PLDM for BIOS Configuration（BIOS 配置） |

iBMC 通过 **PLDM-over-MCTP** 与 PCIe 设备（RAID 卡、NVMe SSD、网卡）进行标准化的管理通信。

---

### 18. SPDM（Security Protocol and Data Model）

| 属性 | 说明 |
|---|---|
| **全称** | Security Protocol and Data Model |
| **制定者** | DMTF |
| **规范** | DSP0274 |
| **承载** | MCTP 传输层 |
| **用途** | 设备间安全认证和密钥交换 |

**核心能力**：

- 设备身份认证（基于 X.509 证书）
- 固件完整性度量（固件哈希验证）
- 安全会话密钥协商
- 支持 SPDM-over-MCTP 绑定

iBMC 使用 SPDM 对 PCIe 设备进行安全认证，确保只有可信设备才能接入服务器管理平台。

---

## 第四部分：带内通信协议

### 19. SOL（Serial Over LAN）

| 属性 | 说明 |
|---|---|
| **承载** | IPMI RMCP+ 会话 |
| **端口** | UDP 623/664 |
| **功能** | 将主机串口输出通过 iBMC 网络口远程转发 |

```
工作流程：
1. 客户端：ipmitool -I lanplus -H <iBMC> -U admin -P xxx sol activate
2. iBMC 建立 RMCP+ 加密会话
3. 主机 UART 输出 → iBMC 捕获 → RMCP+ 封装 → UDP 发送给客户端
4. 客户端键盘输入 → iBMC → UART → 主机
5. 实现完全的文本控制台远程访问
```

---

### 20. KVM over IP

| 属性 | 说明 |
|---|---|
| **KVM 端口** | TCP 2198 |
| **Video 端口** | TCP 2199 |
| **VMM 端口** | TCP 8208 |
| **客户端** | HTML5（浏览器原生）/ Java（传统） |
| **分辨率** | 最高 1920×1200 |
| **帧率** | 30fps（典型） |

```
远程 KVM 数据流：
主机 VGA/HDMI 输出 → iBMC Video 捕获引擎 → JPEG 压缩
  → KVM 协议封装 → TCP 2199 → 客户端

客户端键盘/鼠标 → TCP 2198 → iBMC → USB HID → 主机
```

---

### 21. Virtual Media（虚拟媒体）

| 属性 | 说明 |
|---|---|
| **端口** | TCP 8208 (VMM) |
| **协议** | 自定义协议（iBMC VMM Protocol） |
| **功能** | 将本地 ISO/IMG 镜像远程挂载为服务器 USB 设备 |

```
远程 OS 安装流程：
1. 客户端选择本地 ISO 文件
2. iBMC 通过 VMM 协议从客户端读取 ISO 数据
3. iBMC 模拟 USB Mass Storage 设备挂载到主机
4. 主机 BIOS 识别为 USB CD-ROM → 从虚拟媒体启动
5. 实现远程 OS 安装
```

---

## 第五部分：协议关系总图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           iBMC 协议关系全景图                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─── 北向管理接口 ────────────────────────────────────────────────────┐    │
│  │                                                                     │    │
│  │   Redfish          IPMI 2.0       SNMP         SSH CLI     WebUI   │    │
│  │   (REST/JSON)     (RMCP+)        (v1/v2c/v3)  (SSH-2)     (HTML5) │    │
│  │      │                │              │            │          │       │    │
│  │      └────────┬───────┴──────────────┴────────────┴──────────┘      │    │
│  │               │                                                     │    │
│  │          HTTPS/TLS (443)    UDP (623/664)    TCP (22)    UDP (161)  │    │
│  │               │                                                     │    │
│  └───────────────┼─────────────────────────────────────────────────────┘    │
│                  │                                                           │
│  ┌───────────────┼── iBMC 应用层 ──────────────────────────────────────┐    │
│  │               ▼                                                      │    │
│  │  ┌──────────────────────────────────────────────────────────────┐    │    │
│  │  │  sensor_alarm  BMC管理  CpuMem  BIOS管理  PowerMgnt  FDM   │    │    │
│  │  │  StorageMgnt   cooling  UPGRADE  kvm_vmm  redfish   ipmi  │    │    │
│  │  └──────────────────────────┬───────────────────────────────────┘    │    │
│  │                              │  D-Bus / IPC                          │    │
│  └──────────────────────────────┼───────────────────────────────────────┘    │
│                                  │                                            │
│  ┌───────────────────────────────┼── 驱动 + 总线层 ─────────────────────┐    │
│  │                               ▼                                      │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │    │
│  │  │I2C/SMBus│  │PCIe+MCTP│  │eSPI/LPC │  │ UART    │  │NC-SI    │   │    │
│  │  │ PMBus   │  │ VDM     │  │ POST码   │  │ SOL     │  │ RMII    │   │    │
│  │  │ IPMB    │  │         │  │ BT/KCS   │  │ 调试    │  │         │   │    │
│  │  └────┬────┘  └────┬────┘  └────┬─────┘  └────┬────┘  └────┬────┘   │    │
│  └───────┼────────────┼────────────┼──────────────┼────────────┼────────┘    │
│          │            │            │              │            │              │
│  ┌───────┼────────────┼────────────┼──────────────┼────────────┼────────┐    │
│  │       ▼            ▼            ▼              ▼            ▼        │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │    │
│  │  │ 传感器  │  │RAID/NVMe│  │ CPU/BIOS│  │ 主机串口│  │ 网卡PHY │   │    │
│  │  │ FRU/PSU │  │ NPU/GPU │  │ CPLD    │  │ 智能网卡│  │ (共享)  │   │    │
│  │  │ CPLD    │  │         │  │         │  │         │  │         │   │    │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │    │
│  │                        服务器硬件                                    │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─── 带内通信 ─────────────────────────────────────────────────────────┐   │
│  │  主机 OS ←──BT/KCS/SSIF──→ iBMC (IPMI 本地通道)                     │   │
│  │  主机 OS ←──PCIe+MCTP───→ iBMC (iBMA Agent / 黑匣子)               │   │
│  │  主机 OS ←──NC-SI────────→ iBMC (边带网络共享)                      │   │
│  │  远程用户←──SOL/KVM/VMedia→ iBMC (RMCP+ 会话)                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─── 安全协议栈 ──────────────────────────────────────────────────────┐    │
│  │  TLS 1.2/1.3     │ HTTPS / Redfish 传输加密                         │    │
│  │  AES-128-CTR     │ SSH 会话加密                                     │    │
│  │  AES-CBC-128     │ IPMI RMCP+ 数据加密                              │    │
│  │  HMAC-SHA256     │ RMCP+ 认证 + 完整性校验                          │    │
│  │  RAKP            │ IPMI 会话密钥协商协议                             │    │
│  │  TCM/TPM         │ 可信计算模块（硬件安全根）                        │    │
│  │  SPDM            │ PCIe 设备安全认证                                │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 附录：协议速查表

| 协议 | 层次 | 传输介质 | 端口 | 规范 | iBMC 用途 |
|---|---|---|---|---|---|
| I2C/SMBus | 硬件总线 | PCB 走线 | — | NXP / Intel | 传感器/FRU/CPLD |
| PMBus | 硬件总线 | I2C | — | SMIF | 电源模块管理 |
| IPMB | 硬件总线 | I2C | — | IPMI | 板间 BMC 通信 |
| MCTP | 传输协议 | PCIe/I2C/UART | — | DSP0236 | 统一设备管理传输 |
| eSPI/LPC | 硬件总线 | PCB 走线 | — | Intel | POST 码/BT/KCS |
| PCIe VDM | 硬件总线 | PCIe | — | PCI-SIG | MCTP 承载 |
| NC-SI | 边带接口 | RMII | — | DSP0222 | 网卡共享管理 |
| UART | 硬件总线 | PCB 走线 | — | — | SOL/调试 |
| RMCP | 传输协议 | UDP | 623 | ASF 2.0 | IPMI 1.5 传输（已弃用） |
| RMCP+ | 传输协议 | UDP | 623/664 | IPMI 2.0 | IPMI 2.0 加密传输 |
| IPMI 2.0 | 管理协议 | RMCP+/BT/KCS | 623/664 | Intel | 传统带外管理 |
| Redfish | 管理协议 | HTTPS | 443 | DSP0266 | 现代 RESTful 管理 |
| DCMI 1.5 | 管理协议 | IPMI | — | Intel/DMTF | 数据中心功率管理 |
| PLDM | 数据模型 | MCTP | — | DSP0240 | 平台级硬件数据模型 |
| SPDM | 安全协议 | MCTP | — | DSP0274 | 设备安全认证 |
| SNMP v3 | 网管协议 | UDP | 161 | IETF RFC 3411 | 网管系统对接 |
| HTTPS/TLS | 传输安全 | TCP | 443 | IETF RFC 5246 | Web 加密通信 |
| SSH | 远程登录 | TCP | 22 | IETF RFC 4253 | 安全命令行 |
| NTP | 时间同步 | UDP | 123 | IETF RFC 5905 | 时钟同步 |
| Syslog | 日志 | UDP | 514 | IETF RFC 5424 | 日志外发 |
| SMTP | 邮件 | TCP | 25/465 | IETF RFC 5321 | 告警邮件 |
| LDAP/LDAPS | 认证 | TCP | 389/636 | IETF RFC 4511 | 域认证 |
| SOL | 远程串口 | RMCP+ | 623/664 | IPMI 2.0 | 串口重定向 |
| KVM over IP | 远程桌面 | TCP | 2198/2199 | 私有 | 远程 KVM |
| Virtual Media | 远程存储 | TCP | 8208 | 私有 | 远程 OS 安装 |
