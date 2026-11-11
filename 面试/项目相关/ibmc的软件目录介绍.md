华为 iBMC 的软件目录结构，可以理解为**一个为“管理”而高度定制化的 Linux 文件系统**。它的布局遵循 Linux 的 Filesystem Hierarchy Standard (FHS)，但核心功能都集中在几个特定的目录中。

### 🗺️ 整体文件系统布局

从 iBMC 命令行的 `df` 命令输出，可以清晰地看到其分区和挂载点结构[](https://support.huawei.com/enterprise/zh/doc/EDOC1100121687/9121e7c4)：

|挂载点 (Mounted on)|文件系统 (Filesystem)|主要用途|
|---|---|---|
|`/`|`/dev/nvm_active`|**根文件系统**，存放核心系统程序|
|`/dev`|`tmpfs`|设备文件|
|`/dev/shm`|`tmpfs`|共享内存，用于进程间通信|
|`/tmp`|`none`|临时文件|
|`/data`|`/dev/mmcblk0p1`|**持久化数据分区**，存放配置和日志|
|`/opt/pme/pram`|`/dev/pramdisk0`|**关键配置分区**，存放 iBMC 运行配置|
|`/opt/pme/extern/profile`|`/dev/loop0`|外部配置文件，以回环设备形式挂载|
|`/opt/pme/extern/web`|`/dev/loop1`|WebUI 资源文件，以回环设备形式挂载|
|`/data/sp`|`/dev/mapper/mmcblksp`|另一个数据分区|

### 📂 核心目录详解

#### `/opt/pme`：管理功能的核心

这是 iBMC 软件架构中最重要的目录，几乎所有管理应用的二进制文件、脚本和配置都集中在这里。

- **`/opt/pme/bin`**：存放**可执行程序**。例如，`cliget` 和 `cliset` 分别用于查询和设置基础硬件信息，`upgrade_bmc`、`upgrade_bios`、`upgrade_cpld` 等则负责固件升级。此外，还有 `switch_cfg`、`uart_check`、`version_show` 等工具。[](https://support.huawei.com/enterprise/en/doc/EDOC1100318732/5811cb0d/descriptions-of-internal-files-in-the-bmc-system#en-us_topic_0202175013_table14983789#1)[](https://support.huawei.cn/enterprise/en/doc/EDOC1000181619/6554ec68/descriptions-of-internal-files-in-the-bmc-system#en-us_topic_0195457546_SR_interface_01)
    
- **`/opt/pme/sbin`**：存放**系统管理脚本**。这些脚本负责启动、监控和维护系统。例如，`start_pme.sh` 是启动 PME (Platform Management Engine) 的主脚本，`run_app.sh` 启动应用进程，`boardcfg` 和 `config.sh` 负责系统配置，`memory_monitor.sh` 监控内存使用，`ssh_agent_monitor.sh` 监控 SSH 服务等。[](https://support.huawei.com/enterprise/en/doc/EDOC1100318732/5811cb0d/descriptions-of-internal-files-in-the-bmc-system#en-us_topic_0202175013_table14983789#1)[](https://support.huawei.cn/enterprise/en/doc/EDOC1000181619/6554ec68/descriptions-of-internal-files-in-the-bmc-system#en-us_topic_0195457546_SR_interface_01)
    
- **`/opt/pme/pram`**：存放 iBMC 的**运行配置**。这是一个重要的持久化分区。
    
- **`/opt/pme/save`**：存放**配置备份与持久化文件**。这里的文件通常包含 `.md5` 或 `.sha256` 校验码以及 `.bak` 备份文件，用于配置的完整性保护和恢复。
    
- **`/opt/pme/extern`**：存放**外部资源**，如 WebUI 界面文件和额外的配置文件。
    

#### `/data`：持久化数据区

`/data` 是另一个关键的持久化分区，用于存放需要长期保存的数据。例如，iBMC 的配置持久化文件 `per_config.ini` 就位于此目录下，各项 iBMC 配置也保存在这里。

#### `/tmp` 与 `/dev/shm`

- **`/tmp`**：标准的临时目录，用于存放程序运行期间的临时文件。例如，配置文件导出时，会先存放到 `/tmp/config.xml`。
    
- **`/dev/shm`**：基于内存的临时文件系统，主要用于**进程间通信 (IPC)**，特别是 D-Bus 通信。iBMC 的 D-Bus 套接字文件就位于 `/dev/shm/dbus/` 下[](https://gitcode.com/leessential/rootfs_user/blob/main/conanfile.py)。
    

### 📁 openUBMC 源码目录结构

iBMC 的开源版本 openUBMC 采用**微组件架构**，其源码结构更具参考意义。核心构建由 `manifest` 仓库统一管理[](https://discuss.openubmc.cn/t/topic/6015/2)[](https://hwcomputing.csdn.net/6940d11b43d3770353bd95ff.html)。

- **`manifest/`**：**项目主控仓库**，通过 `manifest.yml` 文件定义所有组件的依赖关系和版本，是整个系统的构建入口[](https://discuss.openubmc.cn/t/topic/6015/2)。
    
- **`build/`**：构建过程的**临时输出目录**，例如存放编译日志和中间文件。
    
- **`board_openUBMC/`**：**板级配置目录**，存放针对不同硬件平台的适配文件。
    
- **`components/` 或 `app/`**：各个**业务组件**的源码目录，如存储管理组件 `storage`、网络组件等。每个组件内部通常包含 `src/` (源码)、`model.json` (模型定义)、`CMakeLists.txt` (编译脚本) 等[](https://discuss.openubmc.cn/t/topic/6015/2)。
    

在部署环境中，组件的 **Lua 脚本**会被编译成二进制形式后，直接放置在根目录 `/` 下，而 **C/C++ 编译的动态库** (`.so` 文件) 则通常位于 `/lib64` 目录中[](https://discuss.openubmc.cn/t/topic/1322)。

iBMC 的软件目录结构清晰地反映了其作为**专用管理设备**的定位：核心功能高度集中于 `/opt/pme`，配置和数据持久化在 `/data`，而 openUBMC 的源码结构则揭示了其**模块化、组件化**的现代软件工程实践。