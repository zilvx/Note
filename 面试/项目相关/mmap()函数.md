`mmap()` 是 Linux/Unix 系统中一个非常核心的系统调用，全称是 **memory map**（内存映射）。它的作用是将**文件、设备或匿名内存区域映射到进程的虚拟地址空间**，让进程可以像访问普通内存一样读写这些内容，而无需使用 `read()`/`write()` 系统调用。

下面从作用、原理、参数、具体案例几个方面详细介绍。

---

## 一、核心作用

`mmap()` 主要有三大用途：

1. **文件映射**：把一个文件映射到内存，读写内存就是读写文件，由内核负责同步。
    
2. **匿名映射**：映射一块不与任何文件关联的内存，用于分配大块内存或实现共享内存。
    
3. **设备映射**：把设备（如显卡显存、DMA 缓冲区）映射到用户空间，实现零拷贝访问。
    

---

## 二、函数原型

c

#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

**参数含义：**

|参数|含义|
|---|---|
|`addr`|建议的映射起始地址，通常传 `NULL` 让内核自动选择|
|`length`|映射长度（字节）|
|`prot`|内存保护权限：`PROT_READ`、`PROT_WRITE`、`PROT_EXEC`、`PROT_NONE`|
|`flags`|映射类型：`MAP_SHARED`、`MAP_PRIVATE`、`MAP_ANONYMOUS`、`MAP_FIXED` 等|
|`fd`|文件描述符，匿名映射时传 `-1`|
|`offset`|文件偏移量，必须是页大小（通常 4KB）的整数倍|

**返回值：** 成功返回映射区域的起始地址，失败返回 `MAP_FAILED`（即 `(void *)-1`），并设置 `errno`。

**配套函数：**

c

int munmap(void *addr, size_t length);        // 解除映射
int msync(void *addr, size_t length, int flags); // 同步到文件

---

## 三、映射类型对比

|类型|flags 组合|特点|
|---|---|---|
|**共享文件映射**|`MAP_SHARED` + fd|修改会写回文件，多个进程可见|
|**私有文件映射**|`MAP_PRIVATE` + fd|修改不写回文件（写时复制），用于加载可执行文件|
|**共享匿名映射**|`MAP_SHARED \| MAP_ANONYMOUS`|用于父子进程间共享内存|
|**私有匿名映射**|`MAP_PRIVATE \| MAP_ANONYMOUS`|用于 malloc 大块内存|

---

## 四、具体案例

### 案例 1：用 mmap 读写文件

这是最经典的用法：把一个文件映射到内存，直接通过指针修改内容。

c

// mmap_file.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
int main() {
    const char *path = "test.txt";
    // 1. 创建并写入初始内容
    int fd = open(path, O_RDWR | O_CREAT, 0666);
    ftruncate(fd, 4096);  // 确保文件至少有一页大小
    write(fd, "Hello mmap", 11);
    // 2. 映射文件到内存
    struct stat st;
    fstat(fd, &st);
    char *addr = mmap(NULL, st.st_size, PROT_READ | PROT_WRITE,
                      MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    // 3. 像操作内存一样读写文件
    printf("读取内容: %s\n", addr);
    memcpy(addr, "Hello World!", 12);  // 直接修改内存
    // 4. 同步到磁盘（可选，munmap 时也会同步）
    msync(addr, st.st_size, MS_SYNC);
    // 5. 解除映射
    munmap(addr, st.st_size);
    close(fd);
    return 0;
}

**运行结果：**

text

读取内容: Hello mmap

之后 `test.txt` 的内容变成了 `Hello World!`。

**关键点：**

- `MAP_SHARED` 保证修改会写回文件。
    
- 不需要 `read()`/`write()`，直接指针操作，性能更高。
    
- 适合大文件随机访问，避免频繁系统调用。
    

---

### 案例 2：匿名映射分配大块内存

`malloc()` 在分配大块内存（通常 >128KB）时，底层就是调用 `mmap()` 做匿名映射。

c

// mmap_anon.c
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
int main() {
    size_t size = 1024 * 1024;  // 1MB
    // 匿名映射，不关联文件
    char *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    // 像普通内存一样使用
    strcpy(addr, "This is anonymous memory");
    printf("%s\n", addr);
    munmap(addr, size);
    return 0;
}

**关键点：**

- `fd = -1`，`MAP_ANONYMOUS` 表示不关联文件。
    
- 内核分配零初始化的物理页，按需分配（写时才真正分配）。
    
- 这就是 `malloc` 大块内存的实现方式。
    

---

### 案例 3：父子进程共享内存

匿名映射 + `MAP_SHARED` 可以实现父子进程间共享内存，比管道、消息队列更高效。

c

// mmap_shm.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/wait.h>
#include <unistd.h>
int main() {
    size_t size = 4096;
    char *shared = mmap(NULL, size, PROT_READ | PROT_WRITE,
                        MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (shared == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    strcpy(shared, "Parent wrote this");
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程读取并修改
        printf("子进程读到: %s\n", shared);
        strcpy(shared, "Child modified it");
        exit(0);
    } else {
        wait(NULL);
        printf("父进程看到: %s\n", shared);
    }
    munmap(shared, size);
    return 0;
}

**运行结果：**

text

子进程读到: Parent wrote this
父进程看到: Child modified it

**关键点：**

- `fork()` 后，子进程继承映射。
    
- `MAP_SHARED` 让父子进程看到同一块物理内存。
    
- 如果用 `MAP_PRIVATE`，子进程的修改不会影响父进程。
    

---

### 案例 4：映射设备内存（驱动开发）

在嵌入式 Linux 中，`mmap` 常用于把设备的物理地址映射到用户空间，让用户程序直接访问寄存器。

c

// mmap_device.c
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#define GPIO_BASE 0x4804C000   // 假设的 GPIO 物理地址
#define MAP_SIZE  0x1000
int main() {
    int fd = open("/dev/mem", O_RDWR | O_SYNC);
    if (fd < 0) {
        perror("open /dev/mem");
        return 1;
    }
    // 映射物理地址到用户空间
    volatile unsigned int *gpio = mmap(NULL, MAP_SIZE,
                                       PROT_READ | PROT_WRITE,
                                       MAP_SHARED, fd, GPIO_BASE);
    if (gpio == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }
    // 直接读写寄存器
    unsigned int val = gpio[0x13C / 4];  // 读某个寄存器
    printf("GPIO register value: 0x%X\n", val);
    munmap((void *)gpio, MAP_SIZE);
    close(fd);
    return 0;
}

**关键点：**

- `open("/dev/mem")` 需要 root 权限。
    
- `offset` 是物理地址，必须是页对齐。
    
- 用户态直接操作硬件寄存器，避免了 `ioctl` 的开销。
    
- 在 iBMC 这类系统中，某些硬件监控模块也会用类似方式访问芯片寄存器。
    

---

### 案例 5：动态库加载

Linux 加载可执行文件和动态库（`.so`）时，就是用 `mmap` 的 `MAP_PRIVATE` 方式映射到进程地址空间。

bash

# 查看某个进程的内存映射
cat /proc/self/maps

输出示例：

text

55d4f8a00000-55d4f8a01000 r-xp  /usr/bin/cat
7f8b2c000000-7f8b2c021000 r-xp  /lib/x86_64-linux-gnu/libc.so.6
7f8b2c200000-7f8b2c201000 rw-p  [anon]

- `r-xp` 表示可读可执行，代码段。
    
- `rw-p` 表示可读写，数据段。
    
- 多个进程可以共享同一个 `.so` 的代码段，节省物理内存。
    

---

## 五、mmap 的优势

|优势|说明|
|---|---|
|**零拷贝**|文件内容直接映射到内存，无需 `read`/`write` 的数据复制|
|**随机访问**|可以像数组一样随机访问文件任意位置|
|**共享内存**|多进程可映射同一文件或匿名区域，实现高效 IPC|
|**按需分页**|只有访问到的页才会加载，节省内存|
|**大内存分配**|`malloc` 大块内存时底层用 mmap，避免堆碎片|

---

## 六、常见陷阱与注意事项

1. **偏移量必须页对齐**：`offset` 必须是 `sysconf(_SC_PAGESIZE)` 的整数倍，否则返回 `EINVAL`。
    
2. **长度不必页对齐**：内核会自动向上取整到页边界，但 `munmap` 时要传原始长度。
    
3. **越界访问会崩溃**：映射区域外访问会触发 `SIGSEGV`，和访问野指针一样。
    
4. **`MAP_SHARED` 的写回时机**：修改不会立即写磁盘，由内核延迟回写，可用 `msync` 强制同步。
    
5. **文件大小变化**：映射后如果文件被截断，访问超出部分会触发 `SIGBUS`。
    
6. **内存泄漏**：`mmap` 后忘记 `munmap`，进程退出前会一直占用虚拟地址空间。
    
7. **容器中的 `/dev/mem`**：在 Docker 等容器中通常不可用，需要特权模式。
    

---

## 七、mmap 与其他机制对比

|机制|数据拷贝|适用场景|
|---|---|---|
|`read`/`write`|2 次（内核↔用户）|小文件、顺序读写|
|`mmap` 文件映射|0 次（页缓存直接映射）|大文件、随机访问|
|`mmap` 匿名共享|0 次|父子进程 IPC|
|System V `shmget`|0 次|传统共享内存|
|POSIX `shm_open`|0 次|底层也是 mmap|
|管道/消息队列|2 次|小数据量通信|

---

## 八、在 iBMC 中的实际应用

结合你之前对 iBMC 的关注，`mmap` 在 iBMC 这类嵌入式管理系统中很常见：

1. **D-Bus 通信**：D-Bus 底层使用 Unix domain socket，但大数据传输时可能配合共享内存。
    
2. **硬件寄存器访问**：通过 `/dev/mem` 映射芯片寄存器，直接读写。
    
3. **固件升级**：把固件文件映射到内存，校验和写入 Flash 更高效。
    
4. **日志缓冲**：多个模块共享一块内存区域记录日志，避免频繁写磁盘。
    
5. **malloc 大块内存**：glibc 的 `malloc` 对大于 `MMAP_THRESHOLD`（默认 128KB）的请求直接用 `mmap` 匿名映射。
    

---

## 九、总结

`mmap()` 的本质是**在进程虚拟地址空间和文件/设备/物理内存之间建立映射关系**。它让进程可以用访问内存的方式访问文件或设备，实现了零拷贝、按需分页和高效共享。无论是文件读写、大内存分配、进程间通信，还是驱动开发中的寄存器访问，`mmap` 都是 Linux 系统编程中最基础也最强大的工具之一。

记住核心口诀：**`mmap` 建映射，指针当文件用；`munmap` 解映射，`msync` 保同步。**