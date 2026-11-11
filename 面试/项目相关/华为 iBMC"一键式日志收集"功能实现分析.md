# 华为 iBMC"一键式日志收集"功能实现分析

## 一、iBMC 底层架构

iBMC（Intelligent Baseboard Management Controller）是华为自研的服务器带外管理系统，核心硬件形态为独立的 **Hi1710 / Hi1711 管理芯片**，物理上集成在服务器主板上，拥有独立的：

- **ARM 处理器 + 独立内存 + 独立 Flash 存储**（不依赖服务器硬盘和 CPU）
- **独立网络口**（管理网口，带外访问）
- 运行一套**嵌入式 Linux（RTOS）**，在上面跑 Apache Web 服务、IPMI 协议栈、Redfish 服务、SNMP 代理等

这意味着 iBMC 本身就是一台完整的微型计算机，"一键收集"本质上是在这台微型计算机上运行的一个**自动化脚本/守护进程**。

---

## 二、触发与交互流程

```
浏览器 (HTTPS) ──→ iBMC Apache Web 服务 ──→ 后端收集脚本 ──→ 打包压缩 ──→ HTTP 响应下载
```

具体步骤：

1. 用户通过浏览器访问 iBMC 管理 IP（HTTPS）
2. 进入 WebUI → **"快捷操作 > 一键收集"**（旧版本路径为 `信息 > 信息概况 > 常用操作快捷入口 > 一键收集信息`）
3. 点击按钮后，WebUI 向 iBMC 后端发起 REST/CGI 请求
4. 后端执行收集脚本，遍历各模块的数据目录，拷贝/导出日志文件
5. 将所有文件组织到 `dump_info/` 目录结构下
6. 打包为 `.tar.gz` 压缩包，通过 HTTP 响应推送给浏览器自动下载

命令行等效方式：通过 iBMC CLI（SSH 登录）执行 `ipmcget -t maintenance -d catlog` 也可以查询收集结果。

---

## 三、收集内容的分层结构（核心实现逻辑）

从收集的目录结构可以还原出 iBMC 的软件架构分层：

### 1. 应用层日志（AppDump）

iBMC 内部采用**模块化设计**，每个功能模块独立运行，各自写本地日志：

| 模块 | 收集内容 |
|---|---|
| `sensor_alarm` | 传感器数据、SEL（System Event Log）原始二进制文件、告警事件、LED 状态 |
| `BMC` | FRU 电子标签（序列号/型号）、网口配置、电源信息 |
| `CpuMem` | CPU 详细参数、内存 DIMM 信息 |
| `BIOS` | BIOS 配置项、变更列表、SMBIOS 信息 |
| `PowerMgnt` | 功率统计 CSV |
| `StorageMgnt` | RAID 控制器/逻辑盘/硬盘信息 |
| `cooling_app` | 风扇型号、转速 |
| `UPGRADE` | 各器件固件版本 |
| `kvm_vmm` | KVM 远程控制台模块日志 |
| `redfish` | Redfish API 模块日志 |
| `ipmi_app` | IPMI 协议栈日志 |

每个模块都有一个 `*_dfl.log`（默认日志文件），这是 iBMC 应用层的统一日志格式。

### 2. 操作系统层（RTOSDump）

这部分直接暴露了 iBMC 运行的是嵌入式 Linux，收集的内容就是一套标准的 Linux 系统信息：

- `dmesg` → 内核启动日志
- `ps -elf` → 进程列表
- `top -bn 1` → 实时进程快照
- `ifconfig`、`netstat -a`、`route` → 网络状态
- `meminfo`、`vmstat`、`slabinfo` → 内存与虚拟内存
- `cpuinfo`、`stat` → CPU 使用率
- `modules`、`lsmod` → 内核模块
- `interrupts`、`locks` → 中断和锁状态
- `df`、`partitions`、`mtd` → Flash 分区信息

这说明 iBMC 的收集脚本本质上就是在跑一组 **shell 命令集合**（类似 `dmesg > dmesg_info`、`ps -elf > ps_info`、`top -bn 1 > top_info`）。

### 3. Web 服务层（3rdDump）

iBMC WebUI 基于 **Apache HTTP Server**，收集：

- `error_log` / `access_log` → Apache 访问与错误日志
- `httpd.conf` / `httpd-ssl.conf` → Web 服务配置

### 4. 持久化配置（OptPme）

来自 `/opt/pme/pram` 和 `/opt/pme/save` 目录，这是 iBMC 的**持久化存储区**：

- `per_config.ini` / `per_def_config.ini` → 配置持久化文件
- `sel.db` → SEL 数据库
- `sensor_alarm_sel.bin` → SEL 原始二进制记录
- 每个关键文件都附带 `.md5` 和 `.sha256` **完整性校验码**

### 5. 带外侧数据（OSDump）

通过 **PCIe 通道 + SOL（Serial Over LAN）** 从主机侧采集：

- `systemcom.tar` → SOL 串口数据（主机 BIOS/OS 启动过程的串口输出）
- `img*.jpeg` → 主机最后一屏截图（通过 KVM 帧缓冲抓取）
- `*.rep` → 屏幕自动录像

### 6. 硬件寄存器层（DeviceDump / Register）

直接读取硬件寄存器：

- `i2c_info` → I2C 总线上的设备寄存器/存储区数据（温度传感器、EEPROM 等）
- `cpld_reg_info` → CPLD 寄存器快照

### 7. 崩溃转储（CoreDump）

- `core-*` → 应用程序 core dump，当 iBMC 上的进程异常崩溃时由内核自动生成

---

## 四、关键技术实现细节

### 1. 独立于主机运行

iBMC 有独立的 Flash 存储（日志空间为 200KB～1024KB 不等，视版本而定），日志循环写入，满了自动压缩旧日志。即使服务器宕机、OS 崩溃，iBMC 的日志依然完整。

### 2. 黑匣子（Black Box）机制

iBMC 内置 4MB 独立存储芯片，通过 PCIe 通道配合主机侧的 iBMA（Huawei iBMC Agent）驱动，在 OS 蓝屏/内核崩溃时将 crash kernel 信息写入黑匣子，这部分数据在一键收集中也能导出。

### 3. SEL 完整性保护

SEL 文件同时提供 `.md5` 和 `.sha256` 校验码，并有 `.bak` 备份文件，确保证据链不被篡改——这是面向企业级故障取证的设计。

### 4. 模块化日志框架

每个 iBMC 功能模块（sensor_alarm、BMC、CpuMem、BIOS...）各自维护独立的日志文件和 dfl（default log），收集时按模块目录结构打包，便于华为 TAC（技术支持中心）按模块快速定位故障。

### 5. 批量收集扩展

单台服务器用 iBMC WebUI 一键收集；批量场景华为提供 **SmartKit Computing（原 Computing ToolKit）** 工具，通过 Redfish/IPMI 协议并发连接多台 iBMC，统一拉取日志。

---

## 五、实现原理总结

```
┌─────────────────────────────────────────────────┐
│              iBMC 芯片 (Hi1710/Hi1711)           │
│  ┌─────────────────────────────────────────────┐ │
│  │  嵌入式 Linux (RTOS)                        │ │
│  │  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │ │
│  │  │ Apache   │ │ IPMI栈   │ │ Redfish服务  │ │ │
│  │  │ WebUI    │ │          │ │             │ │ │
│  │  └────┬─────┘ └──────────┘ └─────────────┘ │ │
│  │       │                                     │ │
│  │  ┌────▼──────────────────────────────────┐  │ │
│  │  │  一键收集脚本 (Shell/守护进程)          │  │ │
│  │  │  - 遍历各模块日志目录                   │  │ │
│  │  │  - 执行系统命令 (dmesg/ps/top/ifconfig)│  │ │
│  │  │  - 读取 I2C/CPLD 寄存器                │  │ │
│  │  │  - 导出 SEL 数据库 + 校验码             │  │ │
│  │  │  - 抓取 SOL 串口 / KVM 截图             │  │ │
│  │  │  - 打包 tar.gz                          │  │ │
│  │  └────────────────────────────────────────┘  │ │
│  │                                              │ │
│  │  /opt/pme/pram  /opt/pme/save  (持久化存储)  │ │
│  └─────────────────────────────────────────────┘ │
│                                                  │
│  PCIe ──→ 主机黑匣子数据 / SOL 串口 / KVM 帧缓冲 │
└─────────────────────────────────────────────────┘
```

简单说，**iBMC 的一键收集就是一个运行在带外管理芯片上的自动化脚本**，它利用 iBMC 独立操作系统的权限，一次性汇总应用层模块日志、Linux 内核状态、硬件寄存器快照、主机侧串口/屏幕数据，打包后通过 HTTPS 推送给用户。整个过程不依赖主机 CPU、硬盘和操作系统，即使服务器完全宕机也能工作（除了 OSDump 部分）。
