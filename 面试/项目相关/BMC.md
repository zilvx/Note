BMC，全称基板管理控制器（Baseboard Management Controller），是服务器等设备主板上一个**独立的小型嵌入式系统**[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。你可以把它理解为服务器的“管家”或“隐形守护者”，它拥有自己的处理器、内存和网络连接[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)，即使主系统（CPU、操作系统）关机或故障，它也能独立运行[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)，实现我们常说的**带外管理**（Out-of-Band Management）[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)。

### 🎯 BMC的核心功能

BMC的功能非常强大，它让管理员仿佛身临其境，可以远程完成几乎所有硬件管理任务。

- **远程控制（带外管理）**：
    
    - **电源管理**：无论服务器状态如何，都可以远程执行开机、关机、重启等操作。
        
    - **远程KVM**：提供类似本地操作的远程键盘、视频、鼠标控制台[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。
        
    - **安装操作系统与维护**：通过虚拟媒体或网络，远程安装操作系统、更新固件[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        
- **硬件状态监控**：
    
    - **环境监控**：实时监测主板各部件的温度、电压、风扇转速等[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        
    - **智能风扇调控**：根据温度智能调节风扇转速，平衡散热与能耗[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        
    - **主动告警**：当监控数据异常时，通过多种协议主动向管理员发送告警[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        
- **系统维护与管理**：
    
    - **详尽的日志记录**：记录所有硬件事件和操作日志，便于故障排查[](https://www.ctyun.cn/developer/article/763221166112837)。
        
    - **资产管理**：记录服务器的型号、固件版本等详细配置信息[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)。
        
    - **全生命周期管理**：从出厂检测、部署、运行维护到报废，BMC贯穿始终[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)。
        
- **故障自动处理**：
    
    - **智能容错**：部分BMC具备自动处理能力，例如在风扇故障时自动调整其他风扇转速[](https://www.openubmc.cn/docs/zh/development/quick_start/explore_openubmc.html)。
        

### ⚙️ 工作原理与技术架构

BMC之所以能独立工作，源于其特殊的软硬件架构。

- **硬件架构**[](https://www.ctyun.cn/developer/article/763221166112837)：
    
    - **独立处理器**：通常采用低功耗的**ARM架构**处理器[](https://www.ctyun.cn/developer/article/763221166112837)。
        
    - **独立存储**：拥有独立的Flash（存固件）和RAM[](https://www.ctyun.cn/developer/article/763221166112837)。
        
    - **独立网络**：拥有专用的管理网口，与业务网络物理隔离[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)[](https://www.ctyun.cn/developer/article/763221166112837)。
        
    - **独立供电**：由主板上的待机电源供电，只要电源线插入即可工作[](https://kb-cn-stage.netapp.com/on-prem/ontap/OHW/OHW-KBs/What_is_a_Baseboard_Management_Controller_BMC_and_how_do_I_use_it)。
        
    - **传感器总线**：通过I2C等总线连接各种传感器[](https://www.ctyun.cn/developer/article/763221166112837)。
        
- **软件与协议**：
    
    - **BMC固件**：运行在BMC处理器上的轻量级操作系统和应用程序[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        
    - **IPMI (智能平台管理接口)**：一个标准化的管理协议[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)。BMC是IPMI的**物理载体**[](https://www.ctyun.cn/developer/article/763221166112837)。
        
    - **其他协议**：现代BMC也支持**Redfish**（RESTful API）、**SNMP**、**SMTP**等协议[](https://cloud.tencent.com.cn/developer/article/2005506?from=15425)。
        

### 💻 如何与BMC交互

管理员可以通过多种方式与BMC交互：

- **Web管理界面**：通过浏览器访问BMC的IP地址，即可在图形化界面中进行几乎所有管理操作[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)。
    
- **命令行工具**：使用 `ipmitool`[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)等工具，通过命令行发送IPMI指令。
    
- **第三方管理平台**：大型数据中心使用集中管理平台（如Dell OpenManage，HPE OneView）通过BMC管理成千上万台服务器。
    

### 🔐 不容忽视的安全风险

由于BMC权限极高且独立运行，其安全性至关重要[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。BMC一旦被攻破，攻击者便能获得服务器的完全控制权，且难以被察觉[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。常见风险包括：

- **漏洞频发**：BMC固件可能存在各种安全漏洞[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。
    
- **凭证泄露**：存在通过侧信道攻击等方式获取BMC登录凭证的风险[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。
    
- **缺乏基础防护**：部分BMC固件可能未启用ASLR等基础安全机制[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。
    

因此，务必修改默认密码、定期更新固件、将管理网络与业务网络隔离，并严格限制BMC的访问来源[](https://developer.nvidia.cn/blog/analyzing-baseboard-management-controllers-to-secure-data-center-infrastructure/#classic_memory_exploits_no_modern_mitigations)。

### 🏢 主流BMC实现

不同服务器厂商对BMC有各自的产品化名称和实现[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)：

- **戴尔 (Dell)**：**iDRAC** (Integrated Dell Remote Access Controller)
    
- **惠普 (HPE)**：**iLO** (Integrated Lights-Out)
    
- **联想 (Lenovo)**：**XCC** (Lenovo XClarity Controller)
    
- **华为 (Huawei)**：**iBMC**
    
- **浪潮 (Inspur)**：**IPMI**
    
- **超微 (Supermicro)**：**HPMI** / **ASPEED** (BMC芯片厂商)[](https://cloud.tencent.com.cn/developer/article/2375158?from=15425&frompage=seopage)
    

### 💎 总结

总的来说，BMC是现代服务器不可或缺的核心组件。它通过“带外管理”的方式，为数据中心的远程运维、故障处理和自动化管理提供了可能。