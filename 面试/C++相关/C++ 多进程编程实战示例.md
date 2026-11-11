我来给你一个完整的 C++ 多进程编程实战示例，覆盖了进程创建、进程间通信（多种 IPC 方式）、进程同步和守护进程等常见场景。

这个示例会模拟一个**简单的任务调度系统**：父进程创建多个工作进程，通过不同的 IPC 机制分配任务并收集结果。

---

## 完整代码示例

cpp

// multi_process_demo.cpp
// C++ 多进程编程完整示例 (POSIX)
// 编译: g++ -std=c++17 -pthread multi_process_demo.cpp -o multi_process_demo
#include <iostream>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>
#include <sys/sem.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <signal.h>
#include <cstring>
#include <vector>
#include <string>
#include <chrono>
#include <thread>
#include <random>
// ==================== 1. 信号处理 ====================
class SignalHandler {
public:
    static void setupSignalHandlers() {
        // 忽略 SIGPIPE，防止写已关闭的管道时崩溃
        signal(SIGPIPE, SIG_IGN);
        // 注册 SIGCHLD 处理，避免僵尸进程
        struct sigaction sa;
        sa.sa_handler = handleSigChild;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
        sigaction(SIGCHLD, &sa, nullptr);
    }
private:
    static void handleSigChild(int signo) {
        // 非阻塞回收所有已终止的子进程
        while (waitpid(-1, nullptr, WNOHANG) > 0) {}
    }
};
// ==================== 2. 管道通信 (Pipe) ====================
class PipeCommunicator {
public:
    PipeCommunicator() {
        if (pipe(pipefd) == -1) {
            perror("pipe");
            exit(EXIT_FAILURE);
        }
    }
    ~PipeCommunicator() {
        close(pipefd[0]);
        close(pipefd[1]);
    }
    // 父进程写入数据给子进程
    void writeToChild(const std::string& data) {
        write(pipefd[1], data.c_str(), data.size() + 1);
    }
    // 子进程读取父进程发送的数据
    std::string readFromParent() {
        char buffer[256] = {0};
        read(pipefd[0], buffer, sizeof(buffer));
        return std::string(buffer);
    }
    int getReadFd() const { return pipefd[0]; }
    int getWriteFd() const { return pipefd[1]; }
private:
    int pipefd[2];
};
// ==================== 3. 共享内存 + 信号量同步 ====================
class SharedMemory {
public:
    SharedMemory(key_t key, size_t size, bool create) 
        : shmid(-1), semid(-1), data(nullptr), size(size) {
        
        // 创建或获取共享内存
        shmid = shmget(key, size, IPC_CREAT | 0666);
        if (shmid == -1) {
            perror("shmget");
            exit(EXIT_FAILURE);
        }
        // 附加到进程地址空间
        data = shmat(shmid, nullptr, 0);
        if (data == (void*)-1) {
            perror("shmat");
            exit(EXIT_FAILURE);
        }
        // 创建或获取信号量集（用于同步）
        semid = semget(key, 1, IPC_CREAT | 0666);
        if (semid == -1) {
            perror("semget");
            exit(EXIT_FAILURE);
        }
        if (create) {
            // 初始化信号量为 1（互斥锁）
            semctl(semid, 0, SETVAL, 1);
            // 清空共享内存
            memset(data, 0, size);
        }
    }
    ~SharedMemory() {
        if (data != nullptr && data != (void*)-1) {
            shmdt(data);
        }
    }
    // 加锁（P操作）
    void lock() {
        struct sembuf op = {0, -1, SEM_UNDO};
        semop(semid, &op, 1);
    }
    // 解锁（V操作）
    void unlock() {
        struct sembuf op = {0, 1, SEM_UNDO};
        semop(semid, &op, 1);
    }
    // 写入数据（带锁保护）
    void write(const std::string& value) {
        lock();
        strncpy((char*)data, value.c_str(), size - 1);
        unlock();
    }
    // 读取数据（带锁保护）
    std::string read() {
        lock();
        std::string result((char*)data);
        unlock();
        return result;
    }
    // 清理资源（由父进程调用）
    void cleanup() {
        shmctl(shmid, IPC_RMID, nullptr);
        semctl(semid, 0, IPC_RMID);
    }
private:
    int shmid, semid;
    void* data;
    size_t size;
};
// ==================== 4. 消息队列通信 ====================
struct Message {
    long mtype;          // 消息类型（必须 > 0）
    char mtext[256];     // 消息数据
};
class MessageQueue {
public:
    MessageQueue(key_t key, bool create) {
        msqid = msgget(key, IPC_CREAT | 0666);
        if (msqid == -1) {
            perror("msgget");
            exit(EXIT_FAILURE);
        }
    }
    ~MessageQueue() = default;
    // 发送消息
    void send(long type, const std::string& data) {
        Message msg;
        msg.mtype = type;
        strncpy(msg.mtext, data.c_str(), sizeof(msg.mtext) - 1);
        if (msgsnd(msqid, &msg, sizeof(msg.mtext), 0) == -1) {
            perror("msgsnd");
        }
    }
    // 接收指定类型的消息（阻塞）
    std::string receive(long type) {
        Message msg;
        if (msgrcv(msqid, &msg, sizeof(msg.mtext), type, 0) == -1) {
            perror("msgrcv");
            return "";
        }
        return std::string(msg.mtext);
    }
    // 清理资源
    void cleanup() {
        msgctl(msqid, IPC_RMID, nullptr);
    }
private:
    int msqid;
};
// ==================== 5. 工作进程函数 ====================
// 使用管道处理简单任务
void workerWithPipe(int readFd, int workerId) {
    char buffer[256] = {0};
    ssize_t n = read(readFd, buffer, sizeof(buffer) - 1);
    if (n > 0) {
        std::string task(buffer);
        std::cout << "[Worker " << workerId << "] Received task: " << task 
                  << " (PID: " << getpid() << ")" << std::endl;
        
        // 模拟任务处理
        std::this_thread::sleep_for(std::chrono::milliseconds(500 + rand() % 500));
        
        // 通过管道返回结果（这里简单输出）
        std::cout << "[Worker " << workerId << "] Completed task: " << task 
                  << " -> result: " << task << "_processed" << std::endl;
    }
    close(readFd);
    exit(0);
}
// 使用共享内存处理任务
void workerWithShm(SharedMemory* shm, int workerId) {
    // 等待父进程通过共享内存发送任务
    for (int i = 0; i < 3; i++) {
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        std::string task = shm->read();
        if (!task.empty() && task != "IDLE") {
            std::cout << "[Worker " << workerId << " (SHM)] Processing: " << task 
                      << " (PID: " << getpid() << ")" << std::endl;
            // 处理完成后清空任务
            shm->write("DONE_" + std::to_string(workerId));
        }
    }
    exit(0);
}
// 使用消息队列处理任务（实现双向通信）
void workerWithMsgQueue(MessageQueue* mq, int workerId) {
    while (true) {
        // 接收类型为 workerId 的任务消息
        std::string task = mq->receive(workerId);
        if (task == "EXIT") {
            break;
        }
        
        std::cout << "[Worker " << workerId << " (MQ)] Processing: " << task 
                  << " (PID: " << getpid() << ")" << std::endl;
        
        // 模拟处理
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
        
        // 发送结果回父进程（类型 0）
        std::string result = "RESULT_" + std::to_string(workerId) + ": " + task;
        mq->send(0, result);
    }
    exit(0);
}
// ==================== 6. 守护进程创建 ====================
void createDaemon() {
    // 1. fork 并让父进程退出（脱离终端）
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    }
    if (pid > 0) {
        exit(EXIT_SUCCESS);  // 父进程退出
    }
    // 2. 创建新的会话，成为会话首进程
    if (setsid() < 0) {
        perror("setsid");
        exit(EXIT_FAILURE);
    }
    // 3. 再次 fork，确保不是会话首进程（防止获取终端）
    pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    }
    if (pid > 0) {
        exit(EXIT_SUCCESS);
    }
    // 4. 修改工作目录
    chdir("/tmp");
    // 5. 设置文件掩码
    umask(0);
    // 6. 关闭所有文件描述符
    for (int i = 0; i < sysconf(_SC_OPEN_MAX); i++) {
        close(i);
    }
    // 7. 重定向标准输入输出到 /dev/null
    open("/dev/null", O_RDWR);   // stdin
    dup(0);  // stdout
    dup(0);  // stderr
    // 8. 守护进程主循环
    while (true) {
        // 这里执行后台任务
        std::this_thread::sleep_for(std::chrono::seconds(5));
        // 可以写入日志文件（这里省略）
    }
}
// ==================== 7. 主程序 - 演示所有功能 ====================
int main(int argc, char* argv[]) {
    // 初始化随机数
    srand(time(nullptr));
    
    // 设置信号处理
    SignalHandler::setupSignalHandlers();
    std::cout << "=== POSIX 多进程编程完整示例 ===" << std::endl;
    std::cout << "主进程 PID: " << getpid() << std::endl << std::endl;
    // ======== 场景1: 管道通信 ========
    std::cout << "【场景1】使用管道进行父子进程通信" << std::endl;
    {
        PipeCommunicator pipe;
        pid_t pid = fork();
        if (pid == 0) {
            // 子进程：工作进程
            close(pipe.getWriteFd());  // 关闭写端
            workerWithPipe(pipe.getReadFd(), 1);
            return 0;
        } else if (pid > 0) {
            // 父进程：发送任务
            close(pipe.getReadFd());   // 关闭读端
            std::vector<std::string> tasks = {"task1", "task2", "task3"};
            for (const auto& task : tasks) {
                pipe.writeToChild(task);
                std::cout << "[Parent] Sent task: " << task << std::endl;
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            }
            close(pipe.getWriteFd());
            waitpid(pid, nullptr, 0);
        }
    }
    std::cout << std::endl;
    // ======== 场景2: 共享内存 + 信号量 ========
    std::cout << "【场景2】使用共享内存 + 信号量实现进程间数据共享" << std::endl;
    {
        key_t key = ftok("/tmp", 'A');
        SharedMemory shm(key, 256, true);
        
        // 创建多个工作进程
        const int NUM_WORKERS = 3;
        std::vector<pid_t> workerPids;
        
        for (int i = 1; i <= NUM_WORKERS; i++) {
            pid_t pid = fork();
            if (pid == 0) {
                // 子进程
                workerWithShm(&shm, i);
                return 0;
            } else if (pid > 0) {
                workerPids.push_back(pid);
            }
        }
        // 父进程分配任务
        for (int i = 0; i < 5; i++) {
            std::string task = "SharedTask_" + std::to_string(i);
            shm.write(task);
            std::cout << "[Parent] Wrote to shared memory: " << task << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(300));
        }
        
        // 等待所有子进程完成
        for (pid_t pid : workerPids) {
            waitpid(pid, nullptr, 0);
        }
        
        // 清理共享资源
        shm.cleanup();
    }
    std::cout << std::endl;
    // ======== 场景3: 消息队列 ========
    std::cout << "【场景3】使用消息队列实现多进程双向通信" << std::endl;
    {
        key_t key = ftok("/tmp", 'B');
        MessageQueue mq(key, true);
        
        // 创建多个工作进程
        const int NUM_WORKERS = 3;
        std::vector<pid_t> workerPids;
        
        for (int i = 1; i <= NUM_WORKERS; i++) {
            pid_t pid = fork();
            if (pid == 0) {
                workerWithMsgQueue(&mq, i);
                return 0;
            } else if (pid > 0) {
                workerPids.push_back(pid);
            }
        }
        // 父进程：分配任务给不同的工作进程
        std::vector<std::string> tasks = {"MathTask", "LogTask", "NetTask"};
        for (size_t i = 0; i < tasks.size(); i++) {
            int workerId = (i % NUM_WORKERS) + 1;
            mq.send(workerId, tasks[i] + "_" + std::to_string(i));
            std::cout << "[Parent] Sent task to Worker " << workerId 
                      << ": " << tasks[i] << std::endl;
        }
        // 接收工作进程返回的结果
        for (int i = 0; i < NUM_WORKERS; i++) {
            std::string result = mq.receive(0);  // 接收类型 0 的消息
            std::cout << "[Parent] Received result: " << result << std::endl;
        }
        // 通知所有工作进程退出
        for (int i = 1; i <= NUM_WORKERS; i++) {
            mq.send(i, "EXIT");
        }
        // 等待所有子进程结束
        for (pid_t pid : workerPids) {
            waitpid(pid, nullptr, 0);
        }
        // 清理消息队列
        mq.cleanup();
    }
    std::cout << std::endl;
    // ======== 场景4: 进程间同步（信号） ========
    std::cout << "【场景4】使用信号实现进程间同步" << std::endl;
    {
        pid_t pid = fork();
        if (pid == 0) {
            // 子进程：等待父进程发信号
            std::cout << "[Child] Waiting for SIGUSR1..." << std::endl;
            
            sigset_t mask;
            sigemptyset(&mask);
            sigaddset(&mask, SIGUSR1);
            
            int sig;
            sigwait(&mask, &sig);  // 阻塞等待信号
            
            std::cout << "[Child] Received SIGUSR1, proceeding!" << std::endl;
            exit(0);
        } else if (pid > 0) {
            // 父进程：做一些准备工作后发送信号
            std::cout << "[Parent] Preparing resources..." << std::endl;
            std::this_thread::sleep_for(std::chrono::seconds(1));
            
            std::cout << "[Parent] Sending SIGUSR1 to child..." << std::endl;
            kill(pid, SIGUSR1);
            
            waitpid(pid, nullptr, 0);
        }
    }
    std::cout << std::endl;
    // ======== 场景5: 孤儿进程与僵尸进程处理 ========
    std::cout << "【场景5】孤儿进程处理（子进程会由 init 收养）" << std::endl;
    {
        pid_t pid = fork();
        if (pid == 0) {
            // 子进程
            std::cout << "[Child] My parent PID: " << getppid() << std::endl;
            // 父进程退出后，子进程变成孤儿
            sleep(1);
            std::cout << "[Child] After parent exit, new parent PID: " << getppid() 
                      << " (init or systemd)" << std::endl;
            exit(0);
        } else if (pid > 0) {
            // 父进程快速退出
            std::cout << "[Parent] Exiting immediately..." << std::endl;
            sleep(2);  // 等待子进程输出
        }
    }
    std::cout << std::endl;
    std::cout << "=== 所有演示完成 ===" << std::endl;
    return 0;
}

---

## 代码详解

### 1. 信号处理 (`SignalHandler` 类)

- **SIGCHLD**：当子进程终止时，父进程收到此信号。我们使用 `SA_NOCLDSTOP` 标志只关注终止事件，并通过 `waitpid(-1, WNOHANG)` 非阻塞地回收所有已终止的子进程，防止僵尸进程堆积。
    
- **SIGPIPE**：写入已关闭的管道会触发，我们直接忽略它，避免程序意外崩溃。
    

### 2. 管道通信 (`PipeCommunicator` 类)

- `pipe()` 创建一对文件描述符：`pipefd[0]` 读端，`pipefd[1]` 写端。
    
- 父子进程通信前，各自关闭不需要的一端（父进程关闭读端，子进程关闭写端）。
    
- 管道是**单向**的，适合简单的数据流传输。
    

### 3. 共享内存 + 信号量 (`SharedMemory` 类)

- **共享内存** (`shmget` + `shmat`)：多个进程可以访问同一块物理内存，是最快的 IPC 方式。
    
- **信号量** (`semget` + `semop`)：用作互斥锁，防止多个进程同时修改共享内存造成数据竞争。
    
- `semctl(..., SETVAL, 1)` 将信号量初始化为 1，实现二元信号量（互斥锁）。
    

### 4. 消息队列 (`MessageQueue` 类)

- `msgget` 创建消息队列，`msgsnd` 发送，`msgrcv` 接收。
    
- 消息具有 **类型** (`mtype`)，可以实现多对多的定向通信（如根据 worker ID 分发任务）。
    
- 支持**双向通信**：父进程发送任务给特定 worker，worker 处理完后将结果发回给父进程（类型 0）。
    

### 5. 守护进程 (`createDaemon` 函数)

- 通过双重 `fork()` 确保进程脱离终端控制，成为后台服务。
    
- 改变工作目录到 `/tmp`，设置文件掩码 `umask(0)`，关闭所有文件描述符并重定向到 `/dev/null`。
    

### 6. 进程间信号同步 (场景4)

- `sigwait()` 让子进程阻塞等待特定信号 (`SIGUSR1`)。
    
- 父进程准备好后通过 `kill(pid, SIGUSR1)` 发送信号唤醒子进程。
    

---

## 编译运行

bash

g++ -std=c++17 -pthread multi_process_demo.cpp -o multi_process_demo
./multi_process_demo

---

## 核心要点总结

|特性|核心 API|适用场景|
|---|---|---|
|**管道**|`pipe`, `read`, `write`|父子进程间简单数据流|
|**共享内存**|`shmget`, `shmat`, `shmdt`|高频大数据交换|
|**信号量**|`semget`, `semop`|进程间同步/互斥|
|**消息队列**|`msgget`, `msgsnd`, `msgrcv`|定向多进程通信，支持消息类型|
|**信号**|`signal`, `sigwait`, `kill`|异步事件通知|
|**守护进程**|`fork`, `setsid`, `chdir`|后台长期运行服务|

这个示例覆盖了 POSIX 多进程编程的绝大部分场景。如果你想深入了解某个具体的 IPC 机制（比如共享内存的详细同步策略），我们可以继续展开讨论。