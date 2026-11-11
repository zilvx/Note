`/dev/shm/` 是 Linux 系统中一个非常特殊且重要的目录，全称是 **shared memory**（共享内存）。它不是一个普通的磁盘目录，而是一个**基于内存的临时文件系统（tmpfs）的挂载点**，专门用于进程间共享内存和高速临时数据交换。

下面从本质、原理、使用、管理以及在 iBMC 中的角色几个方面详细介绍。

---

### 一、本质：tmpfs 挂载点

`/dev/shm/` 本身是一个目录，但系统启动时会在它上面挂载一个 **tmpfs** 文件系统。

- **数据在内存中**：文件内容存放在 RAM 和交换分区（swap）中，读写速度极快。
    
- **不持久化**：系统重启后，`/dev/shm/` 中的所有内容都会丢失。
    
- **动态大小**：tmpfs 按需使用内存，不会预先占用固定空间。默认大小通常是物理内存的 50%，但可以调整。
    
- **挂载选项**：通常包含 `rw,nosuid,nodev,noexec,relatime`，有时还有 `size=xxx` 限制。
    

你可以用以下命令查看：

bash

df -h /dev/shm
mount | grep /dev/shm
ls -l /dev/shm

典型输出：

text

tmpfs   3.9G   0   3.9G   0% /dev/shm

---

### 二、核心用途：POSIX 共享内存

`/dev/shm/` 最主要的用途是支持 **POSIX 共享内存**。POSIX 提供了一组 API：

- `shm_open()`：创建或打开一个共享内存对象
    
- `ftruncate()`：设置大小
    
- `mmap()`：映射到进程地址空间
    
- `shm_unlink()`：删除共享内存对象
    

当你调用 `shm_open("/my_shm", ...)` 时，系统实际会在 `/dev/shm/` 下创建一个文件 `/dev/shm/my_shm`。多个进程可以通过同一个名字打开它，并映射到各自的地址空间，从而实现**零拷贝的高速数据共享**。

**C 语言示例：**

c

#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// 使用 ptr 读写共享内存
munmap(ptr, 4096);
close(fd);
shm_unlink("/my_shm");

Python 的 `multiprocessing.shared_memory` 模块底层也使用 `/dev/shm/`。

---

### 三、与其他共享内存机制的区别

|机制|是否出现在 `/dev/shm/`|查看方式|特点|
|---|---|---|---|
|**POSIX 共享内存**|是，文件形式|`ls /dev/shm`|通过 `shm_open` 创建，有文件名|
|**System V 共享内存**|否|`ipcs -m`|通过 `shmget` 创建，内核标识符|
|**匿名 mmap 共享**|否|无文件|`MAP_ANONYMOUS \| MAP_SHARED`，父子进程共享|
|**memfd_create**|否|`/proc/<pid>/fd`|匿名内存文件，不显示在 `/dev/shm`|

所以，`/dev/shm/` 是 **POSIX 共享内存**的专属文件系统视图。

---

### 四、目录权限与大小管理

- **权限**：`/dev/shm/` 目录权限通常是 `1777`（`drwxrwxrwt`），带粘滞位。任何用户都可以创建文件，但只能删除自己的文件。
    
- **大小限制**：
    
    - 默认：物理内存的 50%（旧内核）或由 systemd 配置。
        
    - 容器中：Docker 默认只给 64MB，需要 `--shm-size` 调整。
        
    - 查看：`df -h /dev/shm`
        
    - 临时调整：`mount -o remount,size=2G /dev/shm`
        
    - 永久调整：在 `/etc/fstab` 中添加 `tmpfs /dev/shm tmpfs defaults,size=2G 0 0`
        
- **清理**：
    
    - 正常退出时，程序应调用 `shm_unlink()` 删除。
        
    - 如果进程崩溃，文件可能残留，占用内存直到重启或手动 `rm`。
        
    - 删除正在使用的文件不会立即释放内存，直到所有映射解除。
        

---

### 五、常见操作命令

bash

# 查看使用情况
df -h /dev/shm
ls -lh /dev/shm
# 查看哪些进程在使用
lsof /dev/shm/*
# 手动清理（确认无进程使用后）
rm /dev/shm/残留文件
# 查看 System V 共享内存（不在 /dev/shm）
ipcs -m

---

### 六、在 iBMC 中的角色

结合你之前对华为 iBMC 的关注，`/dev/shm/` 在 iBMC 这类嵌入式 Linux 中扮演着重要角色：

1. **D-Bus 通信**：iBMC 的微组件架构通过 D-Bus 总线互联。D-Bus 使用 Unix domain socket，而 iBMC 中这些套接字文件就位于 `/dev/shm/dbus/` 下。放在 tmpfs 上可以保证通信的高速和系统运行期间的有效性。
    
2. **进程间共享数据**：多个管理模块（如传感器监控、风扇控制、日志服务）可能需要共享实时数据，使用 `/dev/shm/` 可以避免数据复制，降低延迟。
    
3. **临时文件**：一些配置导出、固件升级的临时文件也可能放在这里，因为速度快且重启即清。
    

在 iBMC 的 `df` 输出中，你会看到 `/dev/shm` 是一个 tmpfs，大小通常受限于系统内存。由于 iBMC 内存有限（如 1GB），其 `/dev/shm` 大小可能被显式限制，防止某个进程耗尽内存。

---

### 七、注意事项与陷阱

- **易失性**：重启后数据全部丢失，不能存放需要持久化的配置。
    
- **内存耗尽**：如果程序不断往 `/dev/shm/` 写数据且不删除，会耗尽内存，导致系统或应用崩溃。
    
- **安全**：粘滞位防止用户删除他人文件，但文件权限设置不当可能导致敏感数据泄露。
    
- **容器限制**：在 Docker/K8s 中，`/dev/shm` 默认很小，运行数据库或浏览器等应用时需调整。
    
- **noexec**：通常挂载为 `noexec`，不能直接执行其中的二进制文件，但共享内存映射不受影响。
    

---

### 总结

`/dev/shm/` 是 Linux 的 **POSIX 共享内存文件系统**，基于 tmpfs，位于内存中，速度快、易失、可动态调整大小。它通过文件接口提供共享内存，是高性能进程间通信的重要机制。在 iBMC 等嵌入式系统中，它常被用于 D-Bus 套接字和模块间数据共享，是系统软件架构中不可或缺的一环。