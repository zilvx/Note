# 华为 iBMC 系统架构

> Intelligent Baseboard Management Controller — Hi1710 / Hi1711 芯片平台

---

## 一、分层架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          用户 / 上层平台接入                                 │
│                                                                             │
│  WebUI(HTTPS)  SSH CLI  IPMI工具  Redfish客户端  SNMP网管  SmartKit/iMaster │
│  浏览器界面    TCP 22   ipmitool  REST/JSON      Zabbix   华为批量管理平台   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          北向网络协议栈                                      │
│                                                                             │
│  Redfish        IPMI 2.0        SNMP v1/v2c/v3   HTTPS/HTTP    SSH          │
│  RESTful/JSON   RMCP+           UDP 161+Trap     Apache        AES-CTR     │
│  HTTPS 443      UDP 623,664                      443,80        TCP 22      │
│                                                                             │
│  NTP(时间同步)  Syslog(日志外发)  SMTP(告警邮件)  LDAP/AD(域认证)             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     应用服务层（模块化设计）                                   │
│                                                                             │
│  sensor_alarm   BMC管理     CpuMem      BIOS管理     PowerMgnt              │
│  传感器监控/SEL FRU/网口/电源 CPU/内存信息 配置同步/SMBIOS 功率封顶/功耗统计  │
│                                                                             │
│  StorageMgnt    cooling_app  UPGRADE     kvm_vmm      redfish               │
│  RAID/硬盘管理  风扇调速     固件升级    KVM/虚拟媒体 Redfish服务引擎        │
│                                                                             │
│  ipmi_app       FDM诊断                                                     │
│  IPMI协议栈     智能故障诊断                                                 │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                嵌入式操作系统（RTOS / 嵌入式 Linux）                           │
│                                                                             │
│  Apache HTTP Server    TCP/IP网络栈     D-Bus/IPC     文件系统               │
│  Web服务+Redfish宿主   IPv4/IPv6/VLAN   进程间通信    JFFS2/UBIFS           │
│                                                                             │
│  PAM认证(sfcb/用户管理)              Watchdog(看门狗定时)                     │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            硬件驱动层                                        │
│                                                                             │
│  I2C/SMBus驱动   GPIO驱动   ADC驱动     PWM驱动      SPI/eMMC驱动           │
│  ×16路控制器     LED/按钮   模拟量采集  风扇转速控制 Flash存储              │
│                                                                             │
│  UART驱动        PCIe驱动   eSPI/LPC驱动  网络驱动     USB驱动   Video驱动  │
│  ×8路串口        MCTP消息   POST码捕获    RMII/SGMII   虚拟媒体  KVM帧缓冲  │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           硬件总线接口                                        │
│                                                                             │
│  I2C/SMBus    IPMB       PCIe+MCTP     eSPI/LPC    UART                     │
│  温度/电压    板间I2C    RAID/NVMe     BIOS POST   SOL/串口                 │
│  /FRU/CPLD    管理总线   /NPU          码捕获                               │
│                                                                             │
│  GPIO/PWM    ADC        NC-SI/RMII    JTAG                                   │
│  LED/风扇    电压采样   边带网络共享  调试接口                                │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      服务器硬件（被管理对象）                                  │
│                                                                             │
│  CPU(×2/4)    内存DIMM    温度传感器   电压传感器   电源模块PSU   风扇      │
│  PECI/MCTP   SPD EEPROM  TMP75/TMP112 INA226/LTC2977 PMBus        PWM+Tach │
│                                                                             │
│  CPLD         FRU EEPROM  RAID控制器   NVMe SSD     NPU/GPU      BIOS Flash│
│  上电时序/复位 资产信息    MCTP/PCIe    MCTP/PCIe    MCTP/PCIe    SPI访问   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、安全体系（贯穿全栈）

```
┌──────────────────────────────────────────────────┐
│                安 全 体 系                         │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  自研安全芯片                                │  │
│  │  Hi1710 / Hi1711                            │  │
│  │  集成安全核 + TCM 可信密码模块                │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  安全启动链                                  │  │
│  │  BootROM → 签名校验 → 固件加载               │  │
│  │  → 可信度量传递                              │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  传输加密                                    │  │
│  │  TLS / HTTPS                                │  │
│  │  SSH AES-CTR                                │  │
│  │  IPMI RMCP+ Cipher Suite 3/17               │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  访问控制                                    │  │
│  │  LDAP / AD 域认证                            │  │
│  │  RBAC 角色权限                               │  │
│  │  IP/MAC 白名单                               │  │
│  │  弱口令检测                                  │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  数据完整性                                   │  │
│  │  SEL + MD5/SHA256 校验                       │  │
│  │  固件签名校验                                │  │
│  │  配置防篡改                                  │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  可信链传递                                   │  │
│  │  芯片 → 固件 → OS → 应用                     │  │
│  │  分层可信度量                                │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 三、带内通信通道（iBMC ↔ 主机 OS）

| 通道 | 说明 |
|---|---|
| **BT / KCS 接口** | IPMI 本地通道；OS 侧通过 `/dev/ipmi0` 访问；SSIF（SMBus 系统接口） |
| **SOL 串口重定向** | Serial Over LAN；文本控制台远程转发；TCP 端口映射 |
| **KVM over IP** | 远程键盘/视频/鼠标；HTML5 / Java 客户端；端口 2198 / 2199 |
| **虚拟媒体** | ISO/IMG 远程挂载；USB Mass Storage 模拟；远程 OS 安装 |
| **NC-SI 边带网络** | 与业务网卡共享物理口；VLAN 隔离管理/业务流量；管理平面与业务平面分离 |
| **iBMA Agent** | 主机侧代理软件；PCIe 通道上报 OS 信息；黑匣子崩溃数据写入 |

---

## 四、各层详细说明

### Layer 1 · 用户/上层平台接入

| 接入方式               | 协议/端口       | 说明                                       |
| ------------------ | ----------- | ---------------------------------------- |
| WebUI              | HTTPS 443   | 浏览器图形化管理界面                               |
| SSH CLI            | TCP 22      | 命令行远程管理，最多 5 并发                          |
| IPMI 工具            | UDP 623/664 | ipmitool / ipmicmd / Huawei-iBMC-Cmdlets |
| Redfish 客户端        | HTTPS 443   | RESTful API + JSON，Postman / curl / SDK  |
| SNMP 网管            | UDP 161     | Zabbix / Prometheus / NMS 平台对接           |
| SmartKit / iMaster | HTTPS       | 华为批量管理平台，Redfish + IPMI 并发               |

### Layer 2 · 北向网络协议栈

| 协议 | 传输层 | 端口 | 用途 |
|---|---|---|---|
| Redfish | HTTPS | 443 | RESTful 管理 API |
| IPMI 2.0 | RMCP / RMCP+ | UDP 623 / 664 | 传统带外管理 |
| SNMP v1/v2c/v3 | UDP | 161 | 网管系统对接 + Trap 告警 |
| HTTPS/HTTP | TCP | 443 / 80 | Web 服务 |
| SSH | TCP | 22 | 安全命令行 |
| NTP | UDP | 123 | 时间同步 |
| Syslog | UDP | 514 | 日志外发 |
| SMTP | TCP | 25/465 | 告警邮件 |
| LDAP/AD | TCP | 389/636 | 域认证 |

### Layer 3 · 应用服务层

| 模块 | 功能 | 数据来源 |
|---|---|---|
| sensor_alarm | 传感器轮询、阈值判断、SEL 记录、告警触发 | I2C/SMBus 传感器 |
| BMC 管理 | FRU 信息、网口配置、电源状态 | FRU EEPROM、网口 PHY |
| CpuMem | CPU 型号/核心/频率、内存 DIMM 信息 | PECI / SPD EEPROM |
| BIOS 管理 | BIOS 配置同步、SMBIOS 信息 | eSPI/LPC ↔ BIOS |
| PowerMgnt | 功率封顶、功耗统计、DCMI 1.5 | PMBus 电源模块 |
| StorageMgnt | RAID 控制器/逻辑盘/硬盘管理 | MCTP over PCIe |
| cooling_app | 风扇调速（MPC-PID 智能算法） | I2C 风扇控制器 + 温度传感器 |
| UPGRADE | iBMC/BIOS/CPLD 固件升级 | SPI Flash |
| kvm_vmm | KVM 远程控制台、虚拟媒体 | Video 控制器 + USB |
| redfish | Redfish RESTful 服务引擎 | 聚合所有模块数据 |
| ipmi_app | IPMI 协议栈处理 | 内部消息总线 |
| FDM 诊断 | 智能故障诊断（20+ 年专家经验库） | 全模块数据 |

### Layer 4 · 嵌入式操作系统

| 组件 | 说明 |
|---|---|
| Apache HTTP Server | Web 服务 + Redfish API 的 HTTP 宿主 |
| TCP/IP 网络栈 | IPv4 / IPv6 / VLAN / DHCP / DNS |
| D-Bus / IPC | 模块间进程通信总线 |
| 文件系统 | JFFS2 / UBIFS（Flash 友好） |
| PAM 认证 | 统一认证框架（sfcb / SSH / FTP） |
| Watchdog | 硬件看门狗，防止 iBMC 自身死机 |

### Layer 5 · 硬件驱动层

| 驱动 | 控制器数量 | 连接对象 |
|---|---|---|
| I2C / SMBus | × 16 路 | 温度/电压传感器、FRU EEPROM、CPLD、电源模块 |
| GPIO | 多路 | LED 指示灯、按钮、定位灯 |
| ADC | 多通道 | 电压模拟量采集（电池电压等） |
| PWM | 多通道 | 风扇转速控制 |
| SPI / eMMC | — | iBMC Flash 存储、BIOS Flash |
| UART | × 8 路 | SOL 串口、调试串口、智能网卡串口 |
| PCIe | — | MCTP 消息（RAID/NVMe/NPU）、黑匣子 |
| eSPI / LPC | — | BIOS POST 码捕获、BT/KCS 接口 |
| 网络（RMII/SGMII） | — | NC-SI 边带网络共享 |
| USB | — | 虚拟媒体（USB Device 模拟） |
| Video | — | KVM 帧缓冲捕获 |

### Layer 6 · 硬件总线接口

| 总线 | 速率 | 典型设备 |
|---|---|---|
| I2C / SMBus | 100 kHz / 400 kHz | 温度/电压传感器、FRU EEPROM、CPLD、电源模块 |
| IPMB | 100 kHz（基于 I2C） | 板间 BMC 通信（多节点服务器） |
| PCIe + MCTP | Gen2.0 带宽 | RAID 控制器、NVMe SSD、NPU/GPU |
| eSPI / LPC | 33 MHz / 25 MHz | CPU/PCH、BIOS Flash |
| UART | 115200 bps 典型 | 系统串口（SOL）、调试串口 |
| GPIO / PWM | — | LED、风扇控制 |
| ADC | — | 电压采样（RTC 电池等） |
| NC-SI / RMII | 100 Mbps | 网卡 PHY 边带管理 |
| JTAG | — | 调试接口（生产/研发） |

### Layer 7 · 服务器硬件（被管理对象）

| 硬件 | 连接总线 | 管理内容 |
|---|---|---|
| CPU (× 2/4) | PECI / MCTP | 温度、核心状态、MCE/AER 错误 |
| 内存 DIMM | SPD EEPROM (I2C) | 容量、频率、厂商、CE/UE 错误 |
| 温度传感器 | I2C/SMBus | 进出风口、CPU、主板各区域温度 |
| 电压传感器 | I2C/SMBus | 各路供电轨电压 |
| 电源模块 PSU | PMBus (I2C) | 输入/输出功率、效率、状态 |
| 风扇 | PWM + Tach (I2C) | 转速控制与反馈 |
| CPLD | I2C / Local Bus | 上电时序、复位控制、GPIO 扩展 |
| FRU EEPROM | I2C (0x50-0x57) | 序列号、型号、BOM、制造商、生产日期 |
| RAID 控制器 | MCTP over PCIe | 逻辑盘状态、硬盘 SMART |
| NVMe SSD | MCTP over PCIe | 健康度、温度、寿命 |
| NPU / GPU | MCTP over PCIe | 温度、功耗、状态 |
| BIOS Flash | SPI | 固件版本、升级 |

---

## 五、数据流向

```
                    外部管理入口
         ┌──────────────────────────────┐
         │                              │
    浏览器   SSH   Redfish/IPMI/SNMP   SmartKit
         │      │        │                │
         └──────┴────────┴────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  网络协议层    │
              │               │
              │  HTTPS/TLS    │
              │  SSH          │
              │  IPMI RMCP+   │
              │  SNMP Agent   │
              │  Redfish Svc  │
              └───────┬───────┘
                      │
                      ▼
         ┌────────────────────────────┐
         │       应用服务层            │
         │                            │
         │  sensor_alarm → D-Bus/IPC  │
         │  BMC管理      → D-Bus/IPC  │
         │  KVM/VMM      → D-Bus/IPC  │
         │  UPGRADE      → D-Bus/IPC  │
         │  FDM诊断      → D-Bus/IPC  │
         │  StorageMgnt  → D-Bus/IPC  │
         └─────────────┬──────────────┘
                       │
                       ▼
         ┌────────────────────────────┐
         │      嵌入式 Linux          │
         │                            │
         │  Apache  D-Bus  文件系统   │
         │  TCP/IP栈                  │
         └─────────────┬──────────────┘
                       │
                       ▼
         ┌──────────────────────────────────────────┐
         │              硬件驱动层                    │
         │                                          │
         │  I2C/SMBus  GPIO/PWM  UART  PCIe  eSPI  │
         └──┬──────────┬────────┬──────┬─────┬─────┘
            │          │        │      │     │
            ▼          ▼        ▼      ▼     ▼
     ┌──────────────────────────────────────────────┐
     │            服务器硬件                          │
     │                                              │
     │  传感器  FRU  PSU  风扇  CPLD  CPU  RAID     │
     └──────────────────────────────────────────────┘

     ┌──────────────────────────────────────────────┐
     │           主机 OS（带内通信）                   │
     │                                              │
     │  iBMA Agent ──PCIe+MCTP──→ PCIe 驱动          │
     │  IPMI 驱动  ──BT/KCS/SSIF─→ eSPI/LPC 驱动    │
     └──────────────────────────────────────────────┘
```
