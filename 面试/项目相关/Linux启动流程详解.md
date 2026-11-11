# Linux 启动流程详解

> 从 ROM → U-Boot → 加载内核+DTB → 内核初始化 → 挂载 rootfs → init → 服务

---

## 全景概览

```
┌─────┐    ┌────────┐    ┌──────────────────┐    ┌──────────────┐    ┌────────┐    ┌──────┐    ┌───────┐
│ ROM │───→│ U-Boot │───→│ 加载内核 + DTB   │───→│ 内核初始化    │───→│rootfs  │───→│ init │───→│ 服务  │
│(BL) │    │(BL2/3) │    │ bootm/booti      │    │ start_kernel │    │ 挂载   │    │(PID1)│    │ 启动  │
└─────┘    └────────┘    └──────────────────┘    └──────────────┘    └────────┘    └──────┘    └───────┘
 BootROM    Bootloader    加载阶段                内核阶段            文件系统      用户空间     应用层
```

---

## 一、ROM 阶段（BootROM）

### 1.1 什么是 BootROM

BootROM 是 SoC（System on Chip）内部固化的**只读存储器**代码，在上电（Power-On Reset）后由硬件直接执行。它是整个启动链的**信任根（Root of Trust）**。

### 1.2 BootROM 的职责

| 步骤 | 动作 | 说明 |
|---|---|---|
| ① | **硬件初始化** | 初始化 CPU 核心（设置 SVC 模式、关闭中断、初始化栈指针）、看门狗、基础时钟 |
| ② | **安全启动校验** | 若使能了安全启动（Secure Boot），从 eFuse/OTP 中读取公钥哈希，校验第一级 Bootloader 的数字签名 |
| ③ | **确定启动介质** | 读取 Boot Pin / Strap 引脚的电平组合，确定启动设备（SPI Flash / eMMC / SD / NAND / USB） |
| ④ | **加载第一级 Bootloader** | 从启动介质读取 BL1（或 SPL）到 SoC 内部 SRAM（OCM / TCM），然后跳转执行 |

### 1.3 关键细节

- **为什么加载到 SRAM 而不是 DDR？**
  因为此时 DDR 控制器尚未初始化，SRAM 是 SoC 内部自带的高速存储器（通常几十 KB ~ 几百 KB），上电即可用。

- **多阶段启动的原因**
  BootROM 空间极小（通常 32KB~64KB），无法包含完整的 Bootloader，因此采用**链式加载**：

```
BootROM ──加载──→ BL1/SPL (SoC SRAM)
   BL1/SPL ──初始化DDR──加载──→ BL2/U-Boot (DDR)
      BL2/U-Boot ──加载──→ Linux Kernel (DDR)
```

- **华为 iBMC 场景**
  Hi1710/Hi1711 芯片上电后，BootROM 执行安全启动校验链：
  ```
  BootROM → 校验 BL1 签名 → 加载 BL1 到 SRAM
  BL1 → 初始化 DDR → 校验 U-Boot 签名 → 加载 U-Boot 到 DDR
  ```

---

## 二、U-Boot 阶段（Bootloader）

### 2.1 U-Boot 概述

U-Boot（Universal Bootloader）是嵌入式 Linux 最常用的 Bootloader，分两个阶段：

### 2.2 SPL（Secondary Program Loader）/ BL2

| 步骤 | 动作 | 说明 |
|---|---|---|
| ① | **初始化 DDR 控制器** | 配置 DDR PHY、时序参数、训练（training），使 DDR 可用 |
| ② | **初始化串口** | 配置 UART，输出早期启动信息（`U-Boot SPL` 打印） |
| ③ | **加载 U-Boot 主体** | 从启动介质（SPI Flash / eMMC）读取完整的 U-Boot 到 DDR 高地址 |
| ④ | **跳转执行** | 跳转到 DDR 中的 U-Boot 入口 |

### 2.3 U-Boot 主体（BL3）

U-Boot 主体运行在 DDR 中，功能远比分阶段丰富：

#### 2.3.1 硬件初始化

```c
// U-Boot 初始化序列（简化）
board_init_f()          // 阶段1：使用 SRAM/DDR 低端
  ├── arch_cpu_init()   // CPU 架构相关初始化
  ├── timer_init()      // 定时器
  ├── serial_init()     // 串口（此时已有完整输出）
  ├── dram_init()       // DDR 大小检测
  └── reloc_dst         // 计算重定位目标地址

board_init_r()          // 阶段2：重定位后，使用 DDR 全空间
  ├── board_init()      // 板级初始化（GPIO、I2C、SPI 等）
  ├── env_init()        // 环境变量初始化
  ├── mmc_init()        // eMMC/SD 初始化
  ├── spi_init()        // SPI Flash 初始化
  ├── net_init()        // 网络初始化（可选）
  └── main_loop()       // 进入主循环
```

#### 2.3.2 主循环（main_loop）

U-Boot 进入主循环后有两条路径：

```
main_loop()
  ├── [有 bootdelay 倒计时] → 等待用户按键中断
  │   ├── 用户按键 → 进入命令行模式 (U-Boot Shell)
  │   └── 无按键   → 自动执行 bootcmd（默认启动）
  └── [无 bootdelay] → 直接执行 bootcmd
```

#### 2.3.3 关键环境变量

| 变量 | 说明 | 示例 |
|---|---|---|
| `bootcmd` | 自动启动命令 | `run bootcmd_mmc` |
| `bootargs` | 传递给内核的命令行参数 | `console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait` |
| `bootdelay` | 自动启动倒计时（秒） | `3` |
| `loadaddr` | 内核加载目标地址 | `0x40080000` |
| `fdtaddr` | DTB 加载目标地址 | `0x43000000` |
| `bootm_size` | 启动镜像最大尺寸 | `0x10000000` |

#### 2.3.4 加载内核 + DTB

U-Boot 将内核镜像和 DTB（Device Tree Blob）从存储介质加载到 DDR 指定地址：

```bash
# 从 eMMC 加载内核和 DTB（典型命令）
mmc dev 0
fatload mmc 0:1 0x40080000 Image        # 加载内核到 DDR
fatload mmc 0:1 0x43000000 dtb.dtb      # 加载 DTB 到 DDR

# 或通过 SPI Flash 加载
sf probe 0
sf read 0x40080000 0x100000 0x800000    # 内核
sf read 0x43000000 0x900000 0x10000     # DTB

# 通过 TFTP 网络加载（开发阶段常用）
tftp 0x40080000 Image
tftp 0x43000000 dtb.dtb
```

#### 2.3.5 启动内核

```bash
# ARM64 使用 booti 命令
booti 0x40080000 - 0x43000000
#       ↑内核地址  ↑initrd(无)  ↑DTB地址

# 32位 ARM 使用 bootm 命令（uImage 格式）
bootm 0x40080000 - 0x43000000

# 或 bootz（zImage 格式）
bootz 0x40080000 - 0x43000000
```

**booti 执行过程**：

```
booti kernel_addr - fdt_addr
  ├── 1. 校验内核镜像头部（Image Header / ARM64 Image Header）
  ├── 2. 解压内核（如果是压缩镜像，如 gzip/lz4/zstd）
  ├── 3. 处理 DTB
  │     ├── 校验 FDT magic number (0xd00dfeed)
  │     ├── 修改 DTB 中的 bootargs（可选）
  │     └── 设置 /chosen 节点（内存信息、命令行等）
  ├── 4. 禁用中断（IRQ/FIQ）
  ├── 5. 设置寄存器
  │     ├── x0 = DTB 地址
  │     ├── 其他寄存器清零
  │     └── PC = 内核入口地址
  └── 6. 跳转执行内核入口
```

---

## 三、内核初始化阶段

### 3.1 内核入口（ARM64 为例）

内核被 U-Boot 跳转执行后，从汇编入口开始：

```
内核入口 (_stext / head.S)
  │
  ├── 1. 早期 CPU 设置
  │     ├── 设置异常向量表 (VBAR_EL1)
  │     ├── 配置 MMU 相关寄存器 (SCTLR_EL1)
  │     └── 检测 CPU 特性 (CPUID)
  │
  ├── 2. 创建初始页表
  │     ├── 临时恒等映射 (Identity Mapping)
  │     └── 内核线性映射 (PAGE_OFFSET 偏移)
  │
  ├── 3. 开启 MMU
  │     ├── 写入 TTBR0_EL1 / TTBR1_EL1
  │     ├── 使能 MMU (SCTLR_EL1.M = 1)
  │     └── 切换到虚拟地址执行
  │
  └── 4. 跳转到 C 代码入口
        └── start_kernel()
```

### 3.2 start_kernel() — 内核核心初始化

`start_kernel()` 是内核 C 代码的总入口，按严格顺序初始化所有子系统：

```c
// init/main.c — start_kernel() 关键流程（简化）

start_kernel()
{
    // ── 阶段 1: 架构级早期初始化 ──
    setup_arch(&command_line);        // 解析 DTB，初始化架构相关硬件
                                      //   - 解析 /memory 节点 → 建立内存映射
                                      //   - 解析 /cpus 节点 → 初始化 CPU 拓扑
                                      //   - 解析 /timer 节点 → 注册时钟源
                                      //   - 解析 /interrupt-controller → 初始化中断控制器(GIC)
                                      //   - 初始化 I/O 映射（ioremap 区域）

    // ── 阶段 2: 内存管理初始化 ──
    setup_per_cpu_areas();            // 设置 per-CPU 变量区域
    build_all_zonelists();            // 构建内存区域链表（ZONE_NORMAL / ZONE_DMA 等）
    page_alloc_init();                // 伙伴系统（Buddy Allocator）初始化
    memblock_free_all();              // 释放 memblock 保留的内存到伙伴系统

    // ── 阶段 3: 中断与调度初始化 ──
    trap_init();                      // 初始化异常/陷阱处理（缺页、除零等）
    init_IRQ();                       // 初始化中断控制器（GIC-400/GIC-v3）
    sched_init();                     // 初始化进程调度器（CFS / RT / DL 调度类）

    // ── 阶段 4: 时间子系统 ──
    timekeeping_init();               // 初始化时间管理
    time_init();                      // 初始化时钟事件设备（定时器中断）
    //   → 注册 tick 定时器 → 启动周期性调度

    // ── 阶段 5: 启动其他 CPU ──
    smp_init();                       // 启动从核（Secondary CPUs）
    //   → 对每个从核发送 PSCI CPU_ON 命令
    //   → 从核执行 secondary_start_kernel()
    //   → 从核初始化完成后进入 idle

    // ── 阶段 6: 调度器激活 ──
    sched_init_smp();                 // SMP 调度器初始化完成

    // ── 阶段 7: 设备初始化 ──
    //   这里执行大量的 initcall（按优先级排序）
    do_initcalls();
    //   ├── early_initcall    → 最早的初始化（console 等）
    //   ├── core_initcall     → 核心子系统（VFS、块设备、网络栈）
    //   ├── postcore_initcall → 总线驱动（I2C、SPI、PCIe）
    //   ├── arch_initcall     → 架构相关驱动
    //   ├── subsys_initcall   → 子系统驱动（USB、MMC、DMA）
    //   ├── device_initcall   → 设备驱动（网卡、GPIO、传感器）
    //   └── late_initcall     → 延迟初始化

    // ── 阶段 8: 文件系统与 VFS ──
    vfs_caches_init();                // 初始化 VFS 缓存（dentry / inode cache）

    // ── 阶段 9: 解析内核命令行 ──
    //   从 bootargs 中解析 root=, console=, init=, loglevel= 等参数

    // ── 阶段 10: 创建 PID 0 和 PID 1 ──
    rest_init();                      // 创建两个初始进程
}
```

### 3.3 rest_init() — 创建 PID 0 和 PID 1

```c
rest_init()
{
    // 创建 PID 1: kernel_init 线程
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    //   → 这就是未来的 init 进程

    // 创建 PID 2: kthreadd 线程
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    //   → 所有内核线程的父进程

    // 当前线程成为 PID 0 (idle/swapper)
    cpu_startup_entry(CPUHP_ONLINE);
    //   → 进入 idle 循环，在 CPU 无事可做时执行 WFI (Wait For Interrupt)
}
```

### 3.4 kernel_init() — 挂载根文件系统

```c
kernel_init(void *unused)
{
    // ── 准备用户空间环境 ──
    kernel_init_freeable();
    //   ├── 等待所有异步驱动探测完成
    //   │     （如 eMMC 驱动可能还在初始化）
    //   ├── prepare_namespace()
    //   │     ├── 解析 root= 参数
    //   │     ├── 如果有 initrd/initramfs，先挂载为临时根文件系统
    //   │     └── 等待真正的 root 设备就绪
    //   └── 准备 /dev, /proc, /sys 等伪文件系统

    // ── 挂载根文件系统 ──
    //   尝试方式（按优先级）：
    //   1. 如果有 initramfs → 已在内存中，直接挂载
    //   2. 挂载 root= 指定的块设备
    //      ├── /dev/mmcblk0p2 (eMMC 分区)
    //      ├── /dev/sda1 (SATA/USB)
    //      ├── /dev/mtdblockN (NAND Flash + JFFS2/UBIFS)
    //      └── NFS root (nfsroot=server:/path)
    //
    //   挂载参数来自 bootargs:
    //   root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait

    // ── 执行 init 程序 ──
    //   按顺序尝试以下路径，找到第一个可执行的就运行：
    if (execute_command)                   // ① init= 参数指定的程序
        run_init_process(execute_command);
    run_init_process("/sbin/init");        // ② /sbin/init
    run_init_process("/etc/init");         // ③ /etc/init
    run_init_process("/bin/init");         // ④ /bin/init
    run_init_process("/bin/sh");           // ⑤ /bin/sh（最后的退路）

    // 如果以上全部失败 → Kernel Panic
    panic("No working init found.");
}
```

---

## 四、挂载 rootfs 详解

### 4.1 两种 rootfs 方案

#### 方案 A：initramfs（内存根文件系统）

```
内核启动
  → 解压 initramfs.cpio.gz 到 tmpfs
  → 挂载 tmpfs 为临时根文件系统
  → 执行 initramfs 中的 /init 脚本
  → /init 脚本加载必要驱动（如 eMMC 驱动）
  → /init 脚本挂载真正的根文件系统
  → switch_root 切换到真实根文件系统
  → 执行真实根文件系统中的 /sbin/init
```

**适用场景**：根文件系统在 eMMC/SD/NAND 上，需要先加载存储驱动。

#### 方案 B：直接挂载块设备

```
内核启动
  → 驱动已编译进内核（非模块）
  → 直接挂载 root=/dev/mmcblk0p2
  → 执行 /sbin/init
```

**适用场景**：存储驱动已编译进内核（built-in），无需 initramfs 过渡。

### 4.2 根文件系统内容

```
/                          # 根文件系统
├── bin/                   # 基本命令 (busybox / coreutils)
├── sbin/                  # 系统命令 (init, mount, ifconfig)
├── lib/                   # 共享库
├── usr/                   # 用户空间程序
├── etc/                   # 配置文件
│   ├── init.d/            # SysVinit 启动脚本
│   ├── fstab              # 文件系统挂载表
│   ├── inittab            # SysVinit 配置
│   ├── passwd / shadow    # 用户信息
│   └── network/           # 网络配置
├── dev/                   # 设备节点 (devtmpfs)
├── proc/                  # 进程信息 (procfs)
├── sys/                   # 系统信息 (sysfs)
├── tmp/                   # 临时文件
├── var/                   # 可变数据 (log, run)
├── mnt/                   # 挂载点
├── home/                  # 用户目录
└── opt/                   # 可选软件
```

---

## 五、init 阶段（PID 1）

### 5.1 init 是什么

init 是用户空间的**第一个进程（PID 1）**，它的职责是：
- 启动所有系统服务和守护进程
- 作为所有孤儿进程的父进程
- 处理进程信号（如 SIGTERM 关机等）
- 永不退出（退出 = 系统关机/reboot）

### 5.2 三种常见的 init 实现

#### 5.2.1 BusyBox init（嵌入式最常用）

```
/sbin/init (BusyBox)
  ├── 解析 /etc/inittab
  │     # 格式: <id>:<runlevels>:<action>:<process>
  │     ::sysinit:/etc/init.d/rcS          # 系统初始化脚本
  │     ::askfirst:/bin/sh                  # 启动 shell
  │     ::ctrlaltdel:/sbin/reboot           # Ctrl+Alt+Del 处理
  │     ::shutdown:/bin/umount -a -r        # 关机时卸载
  │
  ├── 执行 /etc/init.d/rcS
  │     ├── 挂载 /proc, /sys, /dev/pts
  │     ├── 设置主机名
  │     ├── 配置网络接口 (ifconfig / ip)
  │     ├── 启动 syslog (syslogd)
  │     ├── 启动 klogd (内核日志)
  │     ├── 启动 crond (定时任务)
  │     └── 启动应用服务
  │
  └── 启动 shell / getty (登录终端)
```

#### 5.2.2 systemd（桌面/服务器主流）

```
/sbin/init (systemd)
  ├── 1. 早期初始化
  │     ├── 挂载 /proc, /sys, /dev (API VFS)
  │     ├── 挂载 /run (tmpfs)
  │     ├── 加载内核模块 (udev 规则)
  │     └── 读取 /etc/machine-id
  │
  ├── 2. 解析 Unit 文件（目标依赖图）
  │     ├── sysinit.target       ← 系统初始化
  │     │   ├── local-fs.target  ← 本地文件系统挂载
  │     │   ├── swap.target      ← Swap 激活
  │     │   └── sockets.target   ← 套接字激活
  │     ├── basic.target         ← 基本服务就绪
  │     │   ├── syslog.service
  │     │   ├── dbus.service
  │     │   └── network.service
  │     └── multi-user.target    ← 多用户模式
  │         ├── sshd.service
  │         ├── crond.service
  │         ├── docker.service
  │         └── 自定义应用.service
  │
  ├── 3. 并行启动服务
  │     └── 根据依赖关系图并行启动所有 Unit
  │
  └── 4. 达到默认目标
        └── default.target (通常是 multi-user.target 或 graphical.target)
```

#### 5.2.3 SysVinit（传统）

```
/sbin/init
  ├── 读取 /etc/inittab → 确定默认运行级别 (runlevel 2/3/5)
  ├── 执行 /etc/init.d/rcS (系统初始化)
  ├── 执行 /etc/rc2.d/S* (按编号顺序串行启动)
  │     S01syslog → S02network → S10sshd → S20cron → ...
  └── 启动 getty (终端登录)
```

---

## 六、服务启动阶段

### 6.1 服务启动的本质

init 进程按照配置文件（inittab / Unit 文件）的定义，fork + exec 启动各个服务进程：

```
init (PID 1)
  ├── fork() → exec("/usr/sbin/sshd")     → sshd (PID 100)
  ├── fork() → exec("/usr/sbin/crond")    → crond (PID 101)
  ├── fork() → exec("/usr/bin/dockerd")   → dockerd (PID 102)
  ├── fork() → exec("/opt/app/server")    → 应用进程 (PID 103)
  └── ...
```

### 6.2 典型服务启动顺序（嵌入式场景）

| 顺序 | 服务 | 说明 |
|---|---|---|
| 1 | 挂载文件系统 | `/proc`, `/sys`, `/dev`, `/tmp`, `/run` |
| 2 | 加载内核模块 | 通过 udev / modprobe 自动加载 |
| 3 | 设备节点创建 | udev / mdev 自动创建设备节点 |
| 4 | 设置主机名 | 读取 `/etc/hostname` |
| 5 | 配置网络 | DHCP / 静态 IP / 路由表 |
| 6 | syslog / klogd | 日志系统 |
| 7 | NTP 时间同步 | 校准时钟 |
| 8 | SSH 服务 | sshd 远程登录 |
| 9 | 应用服务 | BMC Web 服务、IPMI 栈、Redfish 服务等 |
| 10 | 看门狗 | 启动用户态 watchdog daemon |

### 6.3 iBMC 场景下的服务启动

在华为 iBMC 中，init 启动后需要拉起的服务包括：

```
init (PID 1)
  ├── 系统基础服务
  │   ├── 网络配置 (IP/DHCP/VLAN)
  │   ├── SSH 服务 (sshd)
  │   ├── NTP 客户端
  │   └── Syslog 服务
  │
  ├── iBMC 核心服务
  │   ├── sensor_alarm    ← 传感器轮询守护进程
  │   ├── ipmi_app        ← IPMI 协议栈
  │   ├── redfish         ← Redfish RESTful 服务
  │   ├── Apache (httpd)  ← Web 服务器
  │   ├── kvm_vmm         ← KVM 远程控制台
  │   └── FDM             ← 故障诊断模块
  │
  ├── 管理辅助服务
  │   ├── SNMP Agent (snmpd)
  │   ├── SMTP 告警服务
  │   ├── LDAP 客户端
  │   └── 固件升级服务
  │
  └── 应用层服务
      ├── SmartKit 对接接口
      └── License 管理服务
```

---

## 七、完整启动时间线

以一个典型的嵌入式 ARM64 服务器（如 iBMC）为例：

```
时间(ms)  阶段                    关键动作
─────────────────────────────────────────────────────────────────
0         上电复位                CPU 开始执行 BootROM 代码
~10       BootROM                 校验 BL1 签名，加载 BL1 到 SRAM
~50       BL1/SPL                 初始化 DDR 控制器
~200      U-Boot (BL3)           DDR 可用，初始化串口/SPI/eMMC
~500      U-Boot 环境加载         读取环境变量，确定启动介质
~800      加载内核 + DTB          从 eMMC/SPI 读取 Image + dtb 到 DDR
~1000     booti 跳转              校验镜像，跳转内核入口
~1200     内核汇编入口            设置页表，开启 MMU
~1500     start_kernel()          解析 DTB，初始化内存/中断/调度
~2000     SMP 初始化              启动从核
~3000     do_initcalls()          按优先级执行驱动初始化
~5000     挂载 rootfs             挂载根文件系统
~5200     执行 /sbin/init         用户空间 PID 1 启动
~5500     系统初始化脚本          挂载 proc/sys/dev，配置网络
~7000     启动核心服务            sensor_alarm, ipmi, Apache 等
~10000    启动完成                所有服务就绪，可以接收管理请求
```

> 注：以上时间仅为典型参考值，实际取决于硬件配置、内核大小、服务数量等因素。

---

## 八、流程图总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Linux 启动流程全景图                                │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────── 硬件层 ────────────────────────────────┐
│                                                                       │
│  ┌──────────┐    上电     ┌──────────────────────────────────────┐    │
│  │ 电源模块  │───────────→│ CPU 核心复位，PC = BootROM 入口地址   │    │
│  │ (PSU)    │            │ ① 设置 CPU 模式 (SVC/EL1)           │    │
│  └──────────┘            │ ② 初始化基础时钟/看门狗               │    │
│                          │ ③ 读取 Boot Pin → 确定启动介质        │    │
│                          │ ④ (安全启动) 校验 BL1 数字签名        │    │
│                          │ ⑤ 加载 BL1/SPL 到 SoC SRAM           │    │
│                          └──────────────┬───────────────────────┘    │
│                                         │ 跳转执行                     │
│                                         ▼                             │
│  ┌───────────────────────────── BL1 / SPL ─────────────────────────┐  │
│  │  ① 初始化 DDR 控制器 (PHY + Timing + Training)                  │  │
│  │  ② 初始化早期串口 (输出 "U-Boot SPL")                            │  │
│  │  ③ 从启动介质加载 U-Boot 主体到 DDR                              │  │
│  │  ④ 跳转到 DDR 中的 U-Boot 入口                                   │  │
│  └─────────────────────────────┬───────────────────────────────────┘  │
│                                │                                       │
│                                ▼                                       │
│  ┌───────────────────────── U-Boot 主体 ───────────────────────────┐  │
│  │                                                                   │  │
│  │  ┌── board_init_f() ──────────────────────────────────────────┐  │  │
│  │  │  CPU 初始化 → 定时器 → 串口 → DDR 大小检测                  │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  │                           │                                       │  │
│  │                           ▼ (代码重定位到 DDR 高地址)              │  │
│  │  ┌── board_init_r() ─────────────────────────────────────────┐  │  │
│  │  │  GPIO → I2C → SPI → eMMC → 网络 → 环境变量 → 命令行解析   │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  │                           │                                       │  │
│  │                           ▼                                       │  │
│  │  ┌── main_loop() ────────────────────────────────────────────┐  │  │
│  │  │                                                             │  │  │
│  │  │  bootdelay 倒计时                                          │  │  │
│  │  │  ├── 用户按键 → U-Boot Shell (调试/手动操作)               │  │  │
│  │  │  └── 超时     → 自动执行 bootcmd                           │  │  │
│  │  │                                                             │  │  │
│  │  │  ┌── 加载内核 + DTB ──────────────────────────────────┐   │  │  │
│  │  │  │  从 eMMC/SPI/TFTP 读取:                             │   │  │  │
│  │  │  │  • Image (内核镜像) → DDR loadaddr                  │   │  │  │
│  │  │  │  • dtb.dtb (设备树) → DDR fdtaddr                   │   │  │  │
│  │  │  │  (可选) initramfs.cpio.gz → DDR ramdiskaddr         │   │  │  │
│  │  │  └──────────────────────────────────────────────────────┘   │  │  │
│  │  │                           │                                   │  │  │
│  │  │                           ▼                                   │  │  │
│  │  │  ┌── booti 执行 ──────────────────────────────────────┐     │  │  │
│  │  │  │  ① 校验内核镜像头                                   │     │  │  │
│  │  │  │  ② 解压内核 (gzip/lz4/zstd → 原始 Image)           │     │  │  │
│  │  │  │  ③ 校验/修改 DTB (FDT magic, chosen 节点)          │     │  │  │
│  │  │  │  ④ 禁用中断 (IRQ/FIQ)                              │     │  │  │
│  │  │  │  ⑤ 设置寄存器 (x0=DTB地址, PC=内核入口)             │     │  │  │
│  │  │  │  ⑥ 跳转执行内核                                    │     │  │  │
│  │  │  └──────────────────────────────────────────────────────┘     │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────┬───────────────────────────────────────┘  │
│                              │                                           │
└──────────────────────────────┼───────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────── 内核层 ────────────────────────────────────────┐
│                                                                        │
│  ┌── 汇编入口 (head.S / _stext) ───────────────────────────────────┐  │
│  │  ① 设置异常向量表 (VBAR_EL1)                                     │  │
│  │  ② 检测 CPU 特性 (CPUID)                                         │  │
│  │  ③ 创建初始页表 (恒等映射 + 内核线性映射)                          │  │
│  │  ④ 开启 MMU → 切换到虚拟地址执行                                  │  │
│  └─────────────────────────────┬───────────────────────────────────┘  │
│                                │                                       │
│                                ▼                                       │
│  ┌── start_kernel() ───────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ┌─ setup_arch() ──────────────────────────────────────────┐    │  │
│  │  │  解析 DTB:                                               │    │  │
│  │  │  /memory → 内存映射                                      │    │  │
│  │  │  /cpus → CPU 拓扑                                        │    │  │
│  │  │  /timer → 时钟源                                         │    │  │
│  │  │  /interrupt-controller → GIC 中断控制器                   │    │  │
│  │  │  /soc/i2c, /soc/spi → 总线初始化                         │    │  │
│  │  └──────────────────────────────────────────────────────────┘    │  │
│  │                                                                   │  │
│  │  ┌─ 内存管理 ──────────────────────────────────────────────┐     │  │
│  │  │  伙伴系统 (Buddy Allocator) 初始化                       │     │  │
│  │  │  Slab/SLUB 分配器初始化                                  │     │  │
│  │  │  memblock 保留内存释放到伙伴系统                           │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  │                                                                   │  │
│  │  ┌─ 中断与调度 ────────────────────────────────────────────┐     │  │
│  │  │  trap_init()     → 异常处理 (缺页/除零/系统调用)          │     │  │
│  │  │  init_IRQ()      → GIC 中断控制器初始化                   │     │  │
│  │  │  sched_init()    → CFS/RT/DL 调度器初始化                 │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  │                                                                   │  │
│  │  ┌─ 时间子系统 ────────────────────────────────────────────┐     │  │
│  │  │  timekeeping_init() → 时间管理                            │     │  │
│  │  │  time_init()        → 定时器中断 → tick 调度启动           │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  │                                                                   │  │
│  │  ┌─ SMP 多核启动 ──────────────────────────────────────────┐     │  │
│  │  │  smp_init() → PSCI CPU_ON → 从核初始化 → 进入 idle       │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  │                                                                   │  │
│  │  ┌─ do_initcalls() 驱动初始化 ─────────────────────────────┐     │  │
│  │  │                                                          │     │  │
│  │  │  early_initcall   → console, earlycon                    │     │  │
│  │  │  core_initcall    → VFS, 块设备, 网络协议栈              │     │  │
│  │  │  postcore_initcall→ 总线驱动 (I2C, SPI, PCIe)           │     │  │
│  │  │  arch_initcall    → 架构相关驱动                          │     │  │
│  │  │  subsys_initcall  → USB, MMC, DMA, GPIO                  │     │  │
│  │  │  device_initcall  → 网卡, 传感器, 存储控制器              │     │  │
│  │  │  late_initcall    → 延迟初始化                             │     │  │
│  │  │                                                          │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  │                                                                   │  │
│  │  vfs_caches_init() → VFS 缓存 (dentry / inode cache)              │  │
│  │  解析 bootargs → root=, console=, init=, loglevel=                │  │
│  │                                                                   │  │
│  └───────────────────────────┬───────────────────────────────────────┘  │
│                              │                                           │
│                              ▼                                           │
│  ┌── rest_init() ───────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  ┌─ PID 0 (idle/swapper) ──┐  ┌─ PID 2 (kthreadd) ──────────┐  │  │
│  │  │  当前线程降级为空闲进程   │  │  内核线程管理器               │  │  │
│  │  │  CPU 无事时执行 WFI      │  │  所有内核线程的父进程          │  │  │
│  │  └─────────────────────────┘  └───────────────────────────────┘  │  │
│  │                                                                    │  │
│  │  ┌─ PID 1 (kernel_init → init) ─────────────────────────────┐   │  │
│  │  │                                                            │   │  │
│  │  │  kernel_init_freeable()                                    │   │  │
│  │  │  ├── 等待异步驱动探测完成 (eMMC/SD/NAND)                    │   │  │
│  │  │  └── prepare_namespace()                                   │   │  │
│  │  │                                                            │   │  │
│  │  │  ┌─ 挂载根文件系统 ──────────────────────────────────┐   │   │  │
│  │  │  │                                                     │   │   │  │
│  │  │  │  方案A: initramfs                                   │   │   │  │
│  │  │  │  解压 cpio → 挂载 tmpfs → 执行 /init 脚本          │   │   │  │
│  │  │  │  → 加载存储驱动 → 挂载真实 rootfs                   │   │   │  │
│  │  │  │  → switch_root 切换                                 │   │   │  │
│  │  │  │                                                     │   │   │  │
│  │  │  │  方案B: 直接挂载                                    │   │   │  │
│  │  │  │  mount root=/dev/mmcblk0p2 (ext4)                   │   │   │  │
│  │  │  │  mount root=/dev/mtdblockN (jffs2/ubifs)            │   │   │  │
│  │  │  │  mount nfsroot=server:/path (NFS)                   │   │   │  │
│  │  │  └─────────────────────────────────────────────────────┘   │   │  │
│  │  │                                                            │   │  │
│  │  │  ┌─ 执行 init 程序 ──────────────────────────────────┐    │   │  │
│  │  │  │  尝试顺序:                                         │    │   │  │
│  │  │  │  ① init= 参数指定的程序                            │    │   │  │
│  │  │  │  ② /sbin/init                                      │    │   │  │
│  │  │  │  ③ /etc/init                                       │    │   │  │
│  │  │  │  ④ /bin/init                                       │    │   │  │
│  │  │  │  ⑤ /bin/sh                                         │    │   │  │
│  │  │  │  (全部失败 → Kernel Panic)                          │    │   │  │
│  │  │  └──────────────────────────────────────────────────────┘    │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────┬─────────────────────────────────────┘  │
│                                 │                                        │
└─────────────────────────────────┼────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────── 用户空间层 ─────────────────────────────────────┐
│                                                                        │
│  ┌── /sbin/init (PID 1) ───────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ┌─ 系统初始化 (/etc/init.d/rcS) ──────────────────────────┐   │  │
│  │  │  mount /proc /sys /dev /tmp /run                          │   │  │
│  │  │  udev/mdev → 创建设备节点                                  │   │  │
│  │  │  设置主机名                                                │   │  │
│  │  │  配置网络 (DHCP / 静态IP / 路由)                           │   │  │
│  │  │  启动 syslog + klogd                                      │   │  │
│  │  │  NTP 时间同步                                              │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                   │  │
│  │  ┌─ 启动服务 ──────────────────────────────────────────────┐   │  │
│  │  │                                                           │   │  │
│  │  │  基础服务:                                                │   │  │
│  │  │  ├── sshd          ← SSH 远程登录                         │   │  │
│  │  │  ├── crond         ← 定时任务                              │   │  │
│  │  │  └── watchdog      ← 用户态看门狗                          │   │  │
│  │  │                                                           │   │  │
│  │  │  应用服务 (以 iBMC 为例):                                  │   │  │
│  │  │  ├── Apache (httpd) ← Web 服务器 + Redfish API           │   │  │
│  │  │  ├── sensor_alarm   ← 传感器轮询守护进程                   │   │  │
│  │  │  ├── ipmi_app       ← IPMI 协议栈                         │   │  │
│  │  │  ├── redfish        ← Redfish RESTful 服务                │   │  │
│  │  │  ├── kvm_vmm        ← KVM 远程控制台                       │   │  │
│  │  │  ├── FDM            ← 智能故障诊断                         │   │  │
│  │  │  ├── snmpd          ← SNMP Agent                          │   │  │
│  │  │  └── license        ← License 管理                        │   │  │
│  │  │                                                           │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                   │  │
│  │  ┌─ 启动完成 ──────────────────────────────────────────────┐   │  │
│  │  │  所有服务就绪 → 系统可接受管理请求                         │   │  │
│  │  │  init 进入事件循环 → 等待子进程信号 / 关机信号             │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌── 其他进程 ─────────────────────────────────────────────────────┐  │
│  │  PID 0 (idle)    → CPU 空闲时运行 WFI 指令                      │  │
│  │  PID 2 (kthreadd)→ 按需创建内核线程 (kworker, jbd2 等)          │  │
│  │  PID 3+          → 各种服务进程和用户进程                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键知识点总结

| 阶段 | 关键输出 | 常见问题 |
|---|---|---|
| BootROM | 安全校验 + 加载 SPL 到 SRAM | Boot Pin 配置错误导致找不到启动介质 |
| SPL | DDR 初始化 + 加载 U-Boot 到 DDR | DDR training 失败（时序不对） |
| U-Boot | 加载 Image + DTB 到 DDR，booti 跳转 | 环境变量错误、镜像损坏、地址冲突 |
| 内核入口 | 开 MMU，进入 start_kernel | DTB 与硬件不匹配、内核镜像格式错误 |
| start_kernel | 内存/中断/调度/驱动初始化 | 驱动 probe 失败、中断冲突 |
| 挂载 rootfs | 挂载根文件系统 | root= 参数错误、文件系统损坏、驱动未编入内核 |
| init (PID 1) | 启动系统服务 | init 程序不存在 → Kernel Panic |
| 服务启动 | 所有服务就绪 | 依赖关系错误、端口冲突、配置错误 |
