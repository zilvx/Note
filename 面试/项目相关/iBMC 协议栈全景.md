# iBMC 协议栈全景

iBMC 的协议栈可以分为**硬件层接口**、**带外网络协议**、**带内通信协议**和**应用层服务**四个层次。

---

## 一、硬件层总线协议

这是 iBMC 芯片（Hi1710/Hi1711）与服务器主板各器件之间的物理通信通道：

| 总线/接口 | 用途 |
|---|---|
| **I2C / SMBus** | 连接温度传感器、电压传感器、风扇转速传感器、EEPROM（FRU 信息）等低速外设，iBMC 通过 16 路 I2C/SMBus 总线轮询硬件状态 |
| **IPMB**（Intelligent Platform Management Bus） | 基于 I2C 的板间管理总线，用于多节点服务器中 iBMC 与交换模块（MM910/MM920）或其他 BMC 之间的通信 |
| **eSPI / LPC** | 连接 CPU 的 Low Pin Count 或 Enhanced SPI 总线，用于 BIOS POST 过程中向 iBMC 上报 Port 80 诊断码 |
| **PCIe Gen2.0 + MCTP** | 通过 PCIe 边带通道传输 MCTP（Management Component Transport Protocol）消息，用于与 RAID 控制卡、NVMe SSD、NPU 等 PCIe 设备的管理通信；也是黑匣子数据写入的通道 |
| **RMII / SGMII** | 连接网卡 PHY，实现 NC-SI（Network Controller Sideband Interface）边带网络共享 |
| **UART（8 路）** | 连接系统串口（SOL）、调试串口、智能网卡串口等 |
| **JTAG** | 调试接口，用于固件开发和生产测试 |
| **GPIO / ADC / PWM** | 通用 IO（LED 控制、按钮检测）、模拟量采集（电压/温度）、PWM 风扇调速 |
| **SPI Flash / eMMC** | iBMC 自身的固件存储和运行存储 |
| **Local Bus** | 连接 CPLD 等并行设备 |
| **CAN Bus** | 部分机型的板间通信 |

---

## 二、带外网络协议（北向接口）

这是 iBMC 对外暴露的管理接口，用户和上层网管通过这些协议访问 iBMC：

### 1. IPMI 2.0 / IPMI 1.5

- **规范**：完全兼容 IPMI 2.0（Intelligent Platform Management Interface）
- **传输协议**：
  - **RMCP**（Remote Management Control Protocol，UDP 623 端口）——IPMI 1.5 的远程传输
  - **RMCP+**（UDP 664 端口）——IPMI 2.0 的增强版本，支持更强的认证和加密（Cipher Suite 3/17 等）
- **本地通道**：**BT（Block Transfer）** 接口，OS 侧通过 `/dev/ipmi0` 直接发 IPMI 命令给 iBMC
- **工具**：ipmitool、ipmicmd、华为 Huawei-iBMC-Cmdlets（PowerShell）
- **扩展**：支持 **DCMI 1.5**（Data Center Manageability Interface）子集，用于数据中心功率封顶（Power Capping）、功耗读取等标准化命令
- **华为 OEM 命令**：NetFn `0x30`，覆盖电子标签读写、TPM/TCM 状态上报、BIOS 配置同步等私有功能

### 2. Redfish

- **规范**：DMTF Redfish 1.0.2+（ISO/IEC 30115），基于 RESTful API + JSON over HTTPS
- **端口**：443（HTTPS）
- **认证**：Session-based（`X-Auth-Token`）或 Basic Auth
- **覆盖资源**：Systems、Managers、Chassis、UpdateService、SessionService、AccountService、EventService 等完整资源树
- **能力**：硬件信息查询、BIOS 配置、固件升级、RAID 管理、OS 部署（高级版 License）、智能诊断数据收集
- **OEM 扩展**：`/redfish/v1/Oem/Huawei/` 下提供华为私有资源

### 3. SNMP（Simple Network Management Protocol）

- **版本**：SNMPv1 / SNMPv2c / SNMPv3
- **端口**：UDP 161（Agent）
- **MIB**：`HUAWEI-SERVER-IBMC-MIB`，覆盖系统信息、传感器、告警、电源统计、功率管理等
- **操作**：支持 GET / GETNEXT / GETBULK / WALK / SET
- **Trap**：支持 SNMP Trap 主动上报告警事件到网管系统

### 4. HTTPS / HTTP

- **端口**：443（HTTPS）/ 80（HTTP）
- **用途**：WebUI 管理界面、Redfish API 服务
- **Web 服务器**：Apache HTTP Server（运行在 iBMC 嵌入式 Linux 上）
- **证书**：支持自签名证书和用户自定义证书导入替换

### 5. SSH（Secure Shell）

- **端口**：TCP 22
- **加密算法**：AES128-CTR / AES192-CTR / AES256-CTR
- **用途**：iBMC CLI 命令行远程访问，最多支持 5 个并发 SSH 会话

### 6. NTP（Network Time Protocol）

- **用途**：同步网络时间，确保 iBMC 日志和 SEL 时间戳准确

### 7. Syslog

- **用途**：将 iBMC 告警和运行日志转发到远端 Syslog 服务器
- **配置**：通过 `remotelog.conf` 配置目标地址

### 8. SMTP（Simple Mail Transfer Protocol）

- **用途**：告警邮件通知，iBMC 检测到硬件故障时自动发送邮件给管理员

### 9. LDAP / Active Directory

- **用途**：域账号认证和目录服务集成，支持集中化用户管理

### 10. CIM / SFCB（Small Footprint CIM Broker）

- **用途**：实现 WBEM（Web-Based Enterprise Management）接口，兼容 DMTF CIM 标准
- **认证**：通过 PAM 模块 `sfcb` 进行登录认证

---

## 三、带内通信协议（iBMC 与主机 OS 之间）

| 协议/通道 | 用途 |
|---|---|
| **BT（Block Transfer）** | IPMI 本地通道，OS 侧通过 KCS/BT 接口发送 IPMI 命令 |
| **SOL（Serial Over LAN）** | 将主机串口输出通过 iBMC 网络口远程转发，实现文本控制台重定向 |
| **KVM over IP** | 键盘/视频/鼠标远程重定向，端口 2198（KVM）/ 2199（Video）；支持 HTML5 和 Java 两种客户端 |
| **虚拟媒体（Virtual Media）** | 将本地 ISO/IMG 镜像挂载到服务器，用于远程 OS 安装 |
| **NC-SI（Network Controller Sideband Interface）** | iBMC 与业务网卡共享物理网口，管理流量和业务流量通过 VLAN 隔离，管理平面与业务平面分离 |
| **iBMA（iBMC Agent）** | 安装在主机 OS 上的代理软件，通过 PCIe 通道向 iBMC 上报 OS 侧硬件状态（内存故障隔离、硬盘 SMART、性能数据等），并配合黑匣子写入崩溃信息 |
| **MCTP over PCIe** | 用于 iBMC 与 PCIe 设备（RAID 卡、NVMe SSD、NPU 等）的带外管理通信 |

---

## 四、协议栈分层架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                        上 层 网 管 / 用 户                        │
│  SmartKit / iMaster / Zabbix / 自研平台 / 浏览器 / ipmitool       │
└──────┬──────────┬──────────┬──────────┬──────────┬───────────────┘
       │          │          │          │          │
   Redfish     SNMP      IPMI       SSH/HTTPS    SMTP/Syslog
   (REST/      (v1/v2c/  (RMCP/     (TCP 22/     (告警上报)
    JSON/       v3)       RMCP+/     443/80)
    HTTPS)               BT)
       │          │          │          │          │
┌──────┴──────────┴──────────┴──────────┴──────────┴───────────────┐
│                     iBMC 应用层 (Apache + 模块化服务)              │
│  ┌────────┐ ┌────────┐ ┌─────────┐ ┌──────┐ ┌──────┐ ┌───────┐ │
│  │Redfish │ │IPMI    │ │SNMP     │ │WebUI │ │KVM/  │ │FDM    │ │
│  │Service │ │Stack   │ │Agent    │ │(HTTP)│ │VMM   │ │诊断   │ │
│  └────┬───┘ └────┬───┘ └────┬────┘ └──┬───┘ └──┬───┘ └───┬───┘ │
│       │          │          │         │        │          │      │
│  ┌────┴──────────┴──────────┴─────────┴────────┴──────────┴───┐  │
│  │              嵌入式 Linux (RTOS) — 内核 + 驱动层             │  │
│  │  网络栈(TCP/IP) │ USB │ I2C/SMBus │ PCIe │ UART │ eSPI    │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
         │            │           │          │          │
    ┌────┴───┐  ┌─────┴────┐ ┌───┴────┐ ┌───┴────┐ ┌───┴──────┐
    │ 管理网口 │  │ NC-SI    │ │ I2C    │ │ PCIe   │ │ eSPI/LPC │
    │ (专用GE)│  │ 边带网口 │ │ 传感器 │ │ MCTP   │ │ Port 80  │
    └─────────┘  └──────────┘ └────────┘ └────────┘ └──────────┘
```

---

## 五、安全特性相关的协议机制

| 层次 | 安全措施 |
|---|---|
| **硬件层** | 自研 Hi1710/Hi1711 芯片集成安全核 + TCM（可信密码模块），安全启动链 |
| **传输层** | HTTPS/TLS 加密、SSH AES-CTR 加密、IPMI RMCP+ 加密（Cipher Suite 3/17） |
| **认证层** | LDAP/AD 域认证、本地用户密码复杂度策略、弱口令字典检查 |
| **访问控制** | 基于 IP/MAC 的登录白名单（最多 3 条规则）、协议最小化原则（默认关闭不安全端口） |
| **完整性** | 关键配置文件和日志附带 MD5 + SHA256 校验码 |
| **可信链** | 芯片 → 固件 → OS → 应用的分层可信度量传递 |

---

## 六、协议栈演进趋势

- **IPMI → Redfish 迁移**：Redfish 作为新一代标准，提供 RESTful API + JSON 的现代化接口，华为在 Redfish 之上持续扩展 OEM 能力（OS 部署、智能诊断等），逐步替代传统的 IPMI raw 命令
- **MCTP 扩展**：随着 PCIe 设备（NVMe SSD、GPU/NPU、智能网卡）增多，MCTP over PCIe 成为 iBMC 管理外设的主要通道
- **可信计算**：基于自研芯片的 TPCM（可信平台控制模块）满足国标要求，构建全栈可信链
