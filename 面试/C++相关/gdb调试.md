GDB（GNU Project Debugger）是 Linux 下 C/C++ 开发中最核心、最强大的命令行调试器[](https://cloud.tencent.com.cn/developer/article/2443029?policyId=1004)。它虽然主要通过命令行操作，但能让你精细地控制程序的执行，深入观察其内部状态，是排查程序“逻辑错误”的必备神器[](https://cloud.tencent.com.cn/developer/article/2443029?policyId=1004)。

### ⚙️ 准备：让你的程序“可调试”

要让 GDB 发挥作用，首先需要编译出一个“可调试”的程序。关键在于在编译时加入 `-g` 选项[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)。

```bash
gcc -g your_program.c -o your_program
```

这个选项会将**调试信息**（如变量名、行号、源文件路径等）嵌入到可执行文件中[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)。调试完成后，可以重新编译一个不带 `-g` 的版本用于正式发布[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)。

### 🚀 核心功能与基础操作

GDB 的命令虽多，但掌握以下核心功能就足以应对大部分调试场景。在 GDB 的 `(gdb)` 提示符下输入命令即可[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)。

1. **启动与退出程序**
    
    - **启动 GDB**：`gdb ./your_program`[](https://cloud.baidu.com/article/3584703)
        
    - **传递参数**：`gdb --args ./your_program arg1 arg2`[](https://cloud.baidu.com/article/3584703)
        
    - **附加到进程**：`gdb -p PID`（调试正在运行的进程）[](https://cloud.baidu.com/article/3584703)
        
    - **退出 GDB**：`quit` 或 `q`[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)[](https://cloud.baidu.com/article/3584703)
        
2. **控制程序执行**
    
    - `run` (`r`)：开始运行程序[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)
        
    - `continue` (`c`)：继续运行直到下一个断点[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)
        
    - `next` (`n`)：**单步步过**，执行下一行，不进入函数内部[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)[](https://bbs.huaweicloud.com/blogs/444285)
        
    - `step` (`s`)：**单步步入**，执行下一行，如果是函数则进入其内部[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)[](https://bbs.huaweicloud.com/blogs/444285)
        
    - `finish`：执行完当前函数并返回[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)[](https://cloud.baidu.com/article/3584703)
        
3. **设置与管理断点**
    
    - **设置断点**：`break` (`b`)[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)
        
        - `break main`：在 `main` 函数入口打断点[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)
            
        - `break 12`：在当前文件的第 12 行打断点[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)
            
        - `break main.c:15`：在 `main.c` 文件的第 15 行打断点[](https://cloud.baidu.com/article/3584703)
            
    - **条件断点**：`break 20 if x > 100`[](https://cloud.baidu.com/article/3584703)
        
    - **查看断点**：`info breakpoints`[](https://cloud.baidu.com/article/3584703)
        
    - **删除断点**：`delete 2`（删除编号为2的断点）[](https://mooc1.chaoxing.com/mooc-ans/ztnodedetailcontroller/visitnodedetail?courseId=83879182&knowledgeId=83879217&_from_=&_fromV2_=&rtag=)[](https://cloud.baidu.com/article/3584703)
        
    - **禁用/启用断点**：`disable 2` / `enable 2`[](https://cloud.baidu.com/article/3584703)
        
4. **查看与监控数据**
    
    - **打印变量/表达式**：`print` (`p`) `variable_name`[](https://cloud.baidu.com/article/3584703)
        
        - `print *ptr`：打印指针指向的内容[](https://cloud.baidu.com/article/3584703)
            
        - `print/x var`：以十六进制格式打印[](https://cloud.baidu.com/article/3584703)
            
    - **自动显示变量**：`display variable_name`，每次暂停时自动显示[](https://cloud.baidu.com/article/3584703)
        
    - **查看内存**：`x/4xw 0x地址`（以十六进制查看4个字）[](https://cloud.baidu.com/article/3584703)
        
    - **监视变量**：`watch variable_name`，当变量值变化时暂停[](https://bbs.huaweicloud.com/blogs/444285)
        
    - **查看调用栈**：`backtrace` (`bt`)[](https://bbs.huaweicloud.com/blogs/444285)
        

### 🧠 高级特性：应对复杂场景

GDB 的强大之处在于其对复杂调试场景的支持[](https://cloud.baidu.com/article/3584703)。

- **调试多线程程序**
    
    - **查看所有线程**：`info threads`[](https://bbs.huaweicloud.com/blogs/444285)
        
    - **切换线程**：`thread <thread-id>`[](https://bbs.huaweicloud.com/blogs/444285)
        
    - **查看所有线程调用栈**：`thread apply all bt`[](https://bbs.huaweicloud.com/blogs/444285)
        
    - **锁定线程调度**：`set scheduler-locking on`，防止其他线程干扰当前调试
        
- **调试动态链接库 (共享库)**
    
    - **查看已加载的库**：`info sharedlibrary`[](https://bbs.huaweicloud.com/blogs/444285)
        
    - **在库函数设置断点**：`break libname.so:function_name`[](https://bbs.huaweicloud.com/blogs/444285)
        
- **远程调试 (Remote Debugging)**  
    这主要用于**嵌入式开发**，调试运行在另一台设备（如 ARM 开发板）上的程序[](https://cloud.tencent.com.cn/developer/article/1823078?from=15425)。需要在目标设备上运行 `gdbserver` 程序，然后在本地 GDB 中通过 `target remote` 命令连接。
    

### ⚙️ 核心原理简述：GDB 如何工作？

GDB 是一个典型的**客户端-服务器（C/S）** 架构程序：

1. **本地调试**：GDB 自身作为客户端，通过操作系统内核提供的调试接口（如 `ptrace` 系统调用）来控制作为服务器端的被调试程序[](https://cloud.tencent.com.cn/developer/article/1823078?from=15425)。
    
2. **远程调试**：GDB 在宿主机（Host）运行，通过 **GDB远程串行协议（RSP）** 与目标机（Target）上的 `gdbserver` 通信，`gdbserver` 再通过 `ptrace` 控制目标程序[](https://cloud.tencent.com.cn/developer/article/1823078?from=15425)。
    

无论是哪种模式，其核心都是 **GDB 通过系统调用（如 `ptrace`）来控制和观测被调试进程**[](https://cloud.tencent.com.cn/developer/article/1823078?from=15425)。

### 💎 总结

GDB 是 Linux 开发者手中不可或缺的利器。虽然它是命令行工具，但通过掌握其**断点、单步执行、数据查看**等核心功能，就能应对绝大多数调试任务。对于更复杂的多线程、动态库等问题，GDB 同样提供了完备的解决方案。初次接触可能会觉得有些门槛，但它带来的调试效率和掌控感是无可替代的。