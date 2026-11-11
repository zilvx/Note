# iBMC 中的 Redfish 详解

## 一、Redfish 是什么

Redfish 是由 **DMTF（Distributed Management Task Force）** 制定的服务器平台管理标准（ISO/IEC 30115），基于 **HTTPS + RESTful API + JSON** 构建，旨在替代传统的 IPMI over LAN 作为新一代服务器带外管理接口。iBMC 对 Redfish 的实现遵循：

- **Redfish 规范版本**：1.20.1（300 系列文档）/ 1.0.2（早期 V23 文档）
- **Schema 版本**：2016.1
- **数据模型**：采用 **OData v4** + **JSON Schema** 双重定义

---

## 二、iBMC Redfish 架构

```
┌─────────────────────────────────────────────────┐
│          Redfish 客户端                          │
│  (Postman / curl / python-ibmcclient / 自研平台) │
└──────────────┬──────────────────────────────────┘
               │ HTTPS (443)
               ▼
┌──────────────────────────────────────────────────┐
│  iBMC Apache Web Server                          │
│  ┌────────────────────────────────────────────┐   │
│  │  Redfish Service (RESTful 引擎)             │   │
│  │  - 路由解析 & 认证鉴权                       │   │
│  │  - 请求分发到内部模块                         │   │
│  │  - JSON 序列化/反序列化                       │   │
│  │  - ETag / If-Match 乐观并发控制              │   │
│  └────┬──────────┬──────────┬────────────────┘   │
│       │          │          │                      │
│  ┌────▼───┐ ┌────▼───┐ ┌───▼────────────────┐   │
│  │Systems │ │Managers│ │Chassis / 其他资源    │   │
│  │模块    │ │模块    │ │模块                 │   │
│  └────────┘ └────────┘ └────────────────────┘   │
└──────────────────────────────────────────────────┘
```

---

## 三、认证与会话

iBMC Redfish 支持两种认证方式：

| 方式 | 说明 |
|---|---|
| **Session-based（推荐）** | POST `/redfish/v1/SessionService/Sessions` 创建会话，返回 `X-Auth-Token` 请求头，后续请求携带此 Token |
| **HTTP Basic Auth** | 每次请求携带 `Authorization: Basic base64(user:password)` |

会话管理要点：
- 会话有超时机制，超时后 Token 失效需重新创建
- 通过 `SessionService` 可管理会话超时时间、最大会话数
- 支持 SSO（单点登录）

---

## 四、资源树全景

iBMC Redfish 的资源树分为**公共固定资源**和**业务资源**两大类：

### 1. 公共固定资源（服务入口）

```
/redfish                              ← 协议版本信息
  └── /v1                             ← 根服务资源 (ServiceRoot)
        ├── RedfishVersion            ← "1.20.1"
        ├── UUID                      ← 设备全局唯一标识
        ├── Systems                   → 系统资源集合
        ├── Chassis                   → 机箱资源集合
        ├── Managers                  → 管理器资源集合
        ├── SessionService            → 会话服务
        ├── AccountService            → 账户服务
        ├── EventService              → 事件服务
        ├── UpdateService             → 固件升级服务
        ├── TaskService               → 任务服务
        ├── Registries                → 消息注册表
        ├── JSONSchemas               → Schema 文件
        ├── Fabrics                   → 拓扑编排资源
        ├── SmsUpdateService          → SMS 升级服务
        ├── DataAcquisitionService    → 数据采集服务
        ├── SystemOverview            → 系统概览
        ├── $metadata                 → OData 元数据文档
        └── odata                     → OData 服务文档
```

### 2. 核心业务资源（三大资源集合）

#### Systems（系统资源）

代表服务器计算系统，是 Redfish 最核心的资源：

```
/redfish/v1/Systems
  └── /{system_id}                    ← 单台服务器系统
        ├── Status                    ← 健康状态 (OK/Warning/Critical)
        ├── PowerState                ← 电源状态 (On/Off)
        ├── HostName                  ← 主机名
        ├── UUID                      ← 系统 UUID
        ├── Boot                      ← 启动配置 (启动源/启动顺序)
        ├── Bios                      → BIOS 配置资源
        │     ├── Attributes          ← BIOS 配置项
        │     └── Settings            ← BIOS 待生效配置
        ├── Processors                → CPU 资源集合
        │     └── /{processor_id}     ← 单颗 CPU 信息 (型号/核心/频率)
        ├── Memory                    → 内存资源集合
        │     └── /{memory_id}        ← 单条 DIMM 信息 (容量/频率/厂商)
        ├── EthernetInterfaces        → 网卡资源集合
        ├── Storages                  → 存储资源集合
        │     └── /{storage_id}
        │           ├── Drives        → 硬盘资源
        │           └── StorageControllers → RAID 控制器
        ├── LogServices               → 日志服务
        │     ├── SEL                 ← 系统事件日志
        │     └── CrashDump           ← 崩溃转储
        ├── SecureBoot                ← 安全启动配置
        └── Actions                   ← 系统操作
              ├── #ComputerSystem.Reset       ← 上电/下电/重启
              └── 其他 Action
```

**关键操作**：
- `GET /Systems/{id}` — 查询系统信息
- `PATCH /Systems/{id}` — 修改启动配置、通电策略等
- `POST /Systems/{id}/Actions/ComputerSystem.Reset` — 电源控制

#### Managers（管理器资源）

代表 iBMC 自身：

```
/redfish/v1/Managers
  └── /{manager_id}                   ← iBMC 管理器
        ├── Model                     ← "iBMC"
        ├── FirmwareVersion           ← iBMC 固件版本
        ├── DateTime                  ← 当前时间
        ├── DateTimeLocalOffset       ← 时区偏移
        ├── Status                    ← 健康状态
        ├── UUID
        ├── NetworkProtocol           → 网络协议配置
        │     ├── SSH / HTTPS / SNMP / IPMI 等
        │     └── 各协议的使能状态和端口号
        ├── EthernetInterfaces        → iBMC 网口配置
        │     └── /{interface_id}     ← IPv4/IPv6/DHCP/DNS
        ├── VirtualMedia              → 虚拟媒体
        │     └── /{media_id}         ← ISO/IMG 挂载
        ├── SerialInterfaces          → 串口配置
        ├── HostInterfaces            → 带内通信接口
        ├── LogServices               → iBMC 日志
        │     ├── OperateLog          ← 操作日志
        │     ├── SecurityLog         ← 安全日志
        │     ├── RunLog              ← 运行日志
        │     └── MaintenanceLog      ← 维护日志
        └── Actions
              ├── #Manager.Reset      ← 重启 iBMC
              └── #Manager.ResetToDefaults ← 恢复出厂设置
```

**关键操作**：
- `GET /Managers/{id}` — 查询 iBMC 信息
- `PATCH /Managers/{id}` — 修改时间/UUID
- `PATCH /Managers/{id}/NetworkProtocol` — 开关 SSH/SNMP/IPMI 等服务，修改端口号
- `POST /Managers/{id}/Actions/Manager.Reset` — 重启 iBMC

#### Chassis（机箱资源）

代表服务器物理机箱：

```
/redfish/v1/Chassis
  └── /{chassis_id}                   ← 机箱
        ├── ChassisType               ← "RackMount"
        ├── Manufacturer / Model / SerialNumber / PartNumber
        ├── AssetTag                  ← 资产标签
        ├── IndicatorLED              ← 定位灯 (Lit/Blinking/Off)
        ├── Power                     → 电源信息
        │     ├── PowerControl        ← 功率数据
        │     ├── PowerSupplies       ← 电源模块列表
        │     └── Voltages            ← 电压传感器
        ├── Thermal                   → 散热信息
        │     ├── Fans                ← 风扇列表 (转速/状态)
        │     └── Temperatures        ← 温度传感器列表
        ├── Sensors                   → 传感器集合 (新版)
        ├── PCIeDevices               → PCIe 设备
        │     └── /{pciedevices_id}   ← 设备信息 + OEM 扩展
        ├── NetworkAdapters           → 网卡
        └── Links
              ├── ComputerSystems     ← 关联的系统
              └── ManagedBy           ← 关联的管理器
```

**关键操作**：
- `GET /Chassis/{id}` — 查询机箱硬件信息
- `PATCH /Chassis/{id}` — 设置定位灯、资产标签
- `GET /Chassis/{id}/Thermal` — 查看所有温度和风扇
- `GET /Chassis/{id}/Power` — 查看电源和功率

---

## 五、其他服务资源

### AccountService（账户服务）

```
/redfish/v1/AccountService
  ├── Accounts                        → 用户账户集合
  │     └── /{account_id}             ← 单个用户 (用户名/角色/密码)
  ├── Roles                           → 角色集合
  │     ├── Administrator             ← 管理员
  │     ├── Operator                  ← 操作员
  │     └── ReadOnly                  ← 只读用户
  ├── AccountLockoutThreshold         ← 登录失败锁定阈值
  ├── AccountLockoutDuration          ← 锁定时长
  └── MinPasswordLength               ← 密码最小长度
```

### EventService（事件服务）

```
/redfish/v1/EventService
  ├── Subscriptions                   → 事件订阅集合
  │     └── POST 创建订阅
  │           ├── Destination          ← 目标 URL
  │           ├── EventTypes           ← 订阅的事件类型
  │           │   (StatusChange/ResourceUpdated/ResourceAdded/
  │           │    ResourceRemoved/Alert)
  │           └── Protocol             ← "Redfish"
  └── 支持 SNMP Trap / Syslog / Email 多种告警外发
```

### UpdateService（固件升级服务）

```
/redfish/v1/UpdateService
  ├── FirmwareInventory               → 固件清单集合
  │     └── /{softid}                 ← 单个固件 (版本/状态)
  ├── Actions
  │     ├── #UpdateService.SimpleUpdate  ← 标准升级
  │     └── #Oem/UpdateService.ParallelUpdate ← 并行升级（华为 OEM）
  └── Oem/Huawei
        ├── DowngradeDisabled         ← 是否禁止降级
        └── AutoFirmwareActivationEnabled ← 自动激活
```

### TaskService（任务服务）

```
/redfish/v1/TaskService
  └── Tasks                           → 异步任务集合
        └── /{task_id}
              ├── TaskState            ← (New/Running/Completed/Exception)
              ├── TaskStatus           ← (OK/Warning/Critical)
              └── PercentComplete      ← 完成百分比
```

用于跟踪固件升级、BIOS 配置等长耗时操作。

---

## 六、华为 OEM 扩展

iBMC 在标准 Redfish 资源基础上通过 `Oem.Huawei` 节点提供大量私有扩展：

| 资源位置 | OEM 扩展能力 |
|---|---|
| `/Chassis/{id}/Oem/Huawei` | SuperPod 使能、液冷配置 |
| `/Systems/{id}/Oem/Huawei` | 智能网卡管理、NPU 信息、OS 部署 |
| `/Managers/{id}/Oem/Huawei` | iBMC 高级配置、License 管理 |
| `/PCIeDevices/{id}/Oem/Huawei` | MRC 日志级别、UEFI 日志级别、PXE 启动选项 |
| `/UpdateService/Oem/Huawei` | 并行升级、固件降级控制、自动激活 |
| `/Oem/Huawei/AssetService` | 资产管理服务 |
| `/DataAcquisitionService` | 数据采集服务（需 License），按部件类型/指标类型/时间范围过滤查询性能数据 |

OEM 扩展的 URI 模式统一为 `.../Oem/Huawei/{ResourceName}`。

---

## 七、HTTP 操作与协议规范

### 支持的 HTTP 方法

| 方法 | 用途 |
|---|---|
| **GET** | 查询资源属性 |
| **PATCH** | 修改资源属性（需携带 `If-Match` 头做乐观锁） |
| **POST** | 创建资源或执行 Action |
| **DELETE** | 删除资源（如事件订阅、用户账户） |

> 注意：iBMC Redfish 不支持 PUT 方法。

### 请求/响应头关键规范

| 头字段 | 说明 |
|---|---|
| `X-Auth-Token` | 会话认证 Token |
| `Content-Type` | `application/json` 或 `application/json;charset=utf-8` |
| `If-Match` | PATCH 请求必须携带，值为上次 GET 返回的 ETag，防止并发冲突 |
| `If-None-Match` | 条件 GET，ETag 匹配时返回 304 |
| `ETag` | 响应头，标识资源版本 |
| `@odata.type` | 资源类型标识 |
| `@odata.id` | 资源 URI |

### 状态码

iBMC 支持完整的 HTTP 状态码体系，常用：200（成功）、201（创建成功）、202（异步任务创建）、204（成功无内容）、304（未修改）、400（非法请求）、401（未认证）、403（禁止）、404（不存在）、405（方法不允许）、409（冲突）、412（先决条件失败，如 ETag 不匹配）、500（内部错误）。

---

## 八、典型使用场景

### 1. 查询服务器信息

```bash
# 登录获取 Token
curl -k -X POST https://<iBMC_IP>/redfish/v1/SessionService/Sessions \
  -H "Content-Type: application/json" \
  -d '{"UserName":"Administrator","Password":"xxx"}'
# 响应头返回 X-Auth-Token: <token_value>

# 查询系统信息
curl -k https://<iBMC_IP>/redfish/v1/Systems/1 \
  -H "X-Auth-Token: <token_value>"

# 查询 CPU 信息
curl -k https://<iBMC_IP>/redfish/v1/Systems/1/Processors

# 查询温度和风扇
curl -k https://<iBMC_IP>/redfish/v1/Chassis/1/Thermal
```

### 2. 电源控制

```bash
# 强制重启
curl -k -X POST https://<iBMC_IP>/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "X-Auth-Token: <token_value>" \
  -H "Content-Type: application/json" \
  -d '{"ResetType":"ForceRestart"}'
```

### 3. BIOS 配置

```bash
# 查询 BIOS 配置项
curl -k https://<iBMC_IP>/redfish/v1/Systems/1/Bios

# 修改 BIOS 配置（写入 Settings，重启后生效）
curl -k -X PATCH https://<iBMC_IP>/redfish/v1/Systems/1/Bios/Settings \
  -H "X-Auth-Token: <token_value>" \
  -H "Content-Type: application/json" \
  -H "If-Match: <etag>" \
  -d '{"Attributes":{"BootMode":"UEFI"}}'
```

### 4. 固件升级

```bash
# 查询固件清单
curl -k https://<iBMC_IP>/redfish/v1/UpdateService/FirmwareInventory

# 执行升级
curl -k -X POST https://<iBMC_IP>/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -H "X-Auth-Token: <token_value>" \
  -d '{"ImageURI":"https://<server>/firmware.bin","TransferProtocol":"HTTPS"}'
```

### 5. 事件订阅

```bash
# 创建事件订阅
curl -k -X POST https://<iBMC_IP>/redfish/v1/EventService/Subscriptions \
  -H "X-Auth-Token: <token_value>" \
  -H "Content-Type: application/json" \
  -d '{"Destination":"https://<callback_url>","EventTypes":["Alert"],"Protocol":"Redfish"}'
```

---

## 九、开源工具生态

| 工具 | 语言 | 用途 |
|---|---|---|
| **Huawei-iBMC-Cmdlets** | PowerShell | 华为官方 iBMC Redfish PowerShell Cmdlets |
| **python-ibmcclient** | Python | 社区维护的 Python SDK |
| **Postman** | GUI | Chrome 插件，调试 Redfish API |
| **Redfish Mockup Server** | — | DMTF 提供的模拟服务器，用于开发测试 |

---

## 十、与 IPMI 的定位对比

| 维度 | Redfish | IPMI |
|---|---|---|
| **协议基础** | HTTPS + REST + JSON | UDP + RMCP + 二进制 |
| **数据格式** | JSON，人类可读 | 二进制报文，需解析 |
| **安全** | TLS 加密、Token 认证、RBAC | RMCP+ 加密较弱 |
| **可扩展性** | OEM 扩展 + OData Schema 自描述 | OEM NetFn 扩展，缺乏自描述 |
| **适用场景** | 系统集成、自动化编排、云平台对接 | 底层硬件控制、紧急故障处理 |
| **iBMC 中的趋势** | 主力发展方向，持续扩展新能力 | 保持兼容，逐步被 Redfish 替代 |

华为在 iBMC 中将 Redfish 定位为**北向集成的主接口**，SmartKit、iMaster NCE 等上层管理平台均优先通过 Redfish 与 iBMC 交互。
