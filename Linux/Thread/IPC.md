## 匿名管道
最简单的 IPC 形式，也是在 Shell 中使用 | 时的底层技术
### 原理
1. pipe() 系统调用会创建一个内核中的内存缓冲区，并返回两个文件描述符 
2. 一个用于读取 (fd[0])，一个用于写入 (fd[1])
### 特点
1. 单向：数据只能从写入端流向读取端。
2. 有亲缘关系：匿名管道只能在具有共同祖先的进程之间使用（通常是父子进程）。父进程创建管道后，通过 fork() 将文件描述符复制给子进程。
3. 生命周期：当所有持有管道文件描述符的进程都关闭了它们，管道及其数据就会被销毁。
### 适用场景
父子进程间简单的单向数据传递
### 代码
```
#include <iostream>
#include <string>
#include <unistd.h>    // pipe, fork, read, write, close
#include <sys/wait.h>  // waitpid
using namespace std；

int main() {
    int pipe_fd[2]; // pipe_fd[0] 读取, pipe_fd[1] 写入
    const int BUFFER_SIZE = 128;
    char buffer[BUFFER_SIZE];
    // 1. 创建管道
    if (pipe(pipe_fd) == -1) {
        perror("pipe");
        return 1;
    }
    // 2. 创建子进程
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }
    if (pid == 0) { //子进程 (读取者)
        // 关闭不需要的写入端
        close(pipe_fd[1]); 
        // 从管道读取数据 (read 会阻塞，直到有数据)
        ssize_t bytes_read = read(pipe_fd[0], buffer, BUFFER_SIZE - 1);
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0'; // 添加字符串结束符
            cout << "[Child] 收到消息: \"" << buffer << "\"" << endl;
        } else {
            perror("read");
        }
        // 关闭读取端
        close(pipe_fd[0]);
        exit(0);
    } else { // 父进程 (写入者) 
        // 关闭不需要的读取端
        close(pipe_fd[0]); 
        string message = "Hello from Parent!";
        cout << "[Parent] 向管道写入消息: \"" << message << "\"" << endl;
        // 向管道写入数据
        write(pipe_fd[1], message.c_str(), message.length());
        // 关闭写入端
        close(pipe_fd[1]); 
        // 等待子进程结束
        waitpid(pid, nullptr, 0); 
        cout << "[Parent] 子进程已结束." << endl;
    }
    return 0;
}
```
## 命名管道
匿名管道的缺点是它没有名字，只能用于相关进程。命名管道通过在文件系统中创建一个特殊文件来解决这个问题。
### 原理
使用 mkfifo() 在文件系统上创建一个可见的“管道文件”
### 特点
1. 无亲缘关系：任何知道该文件路径的进程都可以打开它进行读写。
2. 持久性：FIFO 文件会一直存在于文件系统中，直到被显式删除（unlink）。
3. 阻塞：默认情况下，打开 FIFO 进行读取会阻塞，直到有另一个进程打开它进行写入（反之亦然）。
### 适用场景
在系统中运行的两个完全不相关的进程（例如，一个守护进程和一个客户端工具）之间进行通信。
### 代码
1. fifo_reader.cpp(必须先运行)
```
#include <iostream>
#include <string>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h> // open
using namespace std；
const string FIFO_PATH = "/tmp/my_fifo";
const int BUFFER_SIZE = 128;

int main() {
    // 1. 创建 FIFO (如果已存在，mkfifo 会失败，但没关系)
    mkfifo(FIFO_PATH.c_str(), 0666);
    cout << "[Reader] 正在打开 FIFO 进行读取..." << endl;
    // 2. 打开 FIFO (O_RDONLY 会阻塞，直到有写入者)
    int fd = open(FIFO_PATH.c_str(), O_RDONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }
    cout << "[Reader] FIFO 已打开." << endl;
    char buffer[BUFFER_SIZE];
    // 3. 读取数据
    ssize_t bytes_read = read(fd, buffer, BUFFER_SIZE - 1);
    if (bytes_read > 0) {
        buffer[bytes_read] = '\0';
        cout << "[Reader] 收到消息: \"" << buffer << "\"" << endl;
    }
    // 4. 关闭和清理
    close(fd);
    unlink(FIFO_PATH.c_str()); // 删除 FIFO 文件
    return 0;
}
```
2. fifo_writer.cpp (后运行)
```
#include <iostream>
#include <string>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <chrono>
#include <thread>
using namespace std；
const string FIFO_PATH = "/tmp/my_fifo";

int main() {
    cout << "[Writer] 正在打开 FIFO 进行写入..." << endl;
    // 1. 打开 FIFO (O_WRONLY 会阻塞，直到有读取者)
    int fd = open(FIFO_PATH.c_str(), O_WRONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }
    cout << "[Writer] FIFO 已打开." << endl;
    string message = "Hello from Writer!";
    // 2. 写入数据
    cout << "[Writer] 写入消息..." << endl;
    write(fd, message.c_str(), message.length());
    // 3. 关闭
    close(fd);
    cout << "[Writer] 消息已发送." << endl;
    return 0;
}
```
## 共享内存
最快的 IPC 机制。允许两个或多个进程将同一块物理内存映射到它们各自的虚拟地址空间中。
### 原理
1. 进程 A 创建一块内存，进程 B "附加"到这块内存。
2. 进程 A 写入的数据可以被进程 B 立即 看到，反之亦然。数据交换完全不需要内核的介入（read/write）
### 特点
1. 极快：等同于访问本地内存。
2. 无同步：这是最大的缺点。共享内存本身不提供任何同步机制。如果两个进程同时写入，会导致数据错乱。
3. 需要同步：必须配合信号量或互斥锁来保证访问的原子性。
### 适用场景
需要高频率、大批量数据交换的场景，如视频处理、高性能计算、数据库引擎。
## 信号量
### 原理
身不是用来传输数据的，而是用来同步进程的。它通常与共享内存一起使用，以保护共享资源
### 概念
信号量是一个由内核维护的整数（计数器）
1. sem_wait() (P 操作)：如果计数器 > 0，则将其减 1 并立即返回。如果计数器 == 0，则阻塞，直到有其他进程将其增加。
2. sem_post() (V 操作)：将计数器加 1，并唤醒一个可能在等待的进程。
### 适用场景
保护共享内存的“临界区”，确保同一时间只有一个进程在访问。
### 代码（共享内存 + 信号量）
1. shm_synced_writer.cpp
```
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <sys/mman.h>  // shm_open, mmap
#include <sys/stat.h>  // ftruncate
#include <fcntl.h>     // O_CREAT, O_RDWR
#include <semaphore.h> // sem_open, sem_wait, sem_post
#include <thread>
#include <chrono>
using namespace std;
const char* SHM_NAME = "/my_shm";
const char* SEM_NAME = "/my_sem";
const int SHM_SIZE = 1024;

struct SharedData {
    int counter;
    char message[100];
};

int main() {
    // 1. 创建/打开信号量，初始值为 1 (像一个互斥锁)
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 1);
    if (sem == SEM_FAILED) {
        perror("sem_open");
        return 1;
    }
    // 2. 创建/打开共享内存
    int shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (shm_fd == -1) {
        perror("shm_open");
        return 1;
    }
    // 3. 设置共享内存大小
    ftruncate(shm_fd, sizeof(SharedData));
    // 4. 内存映射
    void* ptr = mmap(0, sizeof(SharedData), PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if (ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    SharedData *data = static_cast<SharedData*>(ptr);
    // 5. 写入共享内存 (受信号量保护)
    for (int i = 0; i < 5; ++i) {
        sem_wait(sem); // --- P 操作：获取锁 --- 
        // --- 临界区开始 ---
        data->counter = i;
        string msg = "Writer message " + to_string(i);
        strncpy(data->message, msg.c_str(), sizeof(data->message) - 1);
        data->message[sizeof(data->message) - 1] = '\0';
        cout << "[Writer] 写入: " << data->message << " (Counter: " << data->counter << ")" << endl;
        // --- 临界区结束 ---
        sem_post(sem); // --- V 操作：释放锁 ---
        this_thread::sleep_for(chrono::seconds(1));
    }
    // 6. 清理
    munmap(ptr, sizeof(SharedData));
    close(shm_fd);
    sem_close(sem);
    // 写入者负责最后的清理
    shm_unlink(SHM_NAME);
    sem_unlink(SEM_NAME);
    return 0;
}
```
2. shm_synced_reader.cpp
```
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <semaphore.h>
#include <thread>
#include <chrono>
using namespace std;
const char* SHM_NAME = "/my_shm";
const char* SEM_NAME = "/my_sem";
const int SHM_SIZE = 1024;

struct SharedData {
    int counter;
    char message[100];
};

int main() {
    this_thread::sleep_for(chrono::milliseconds(500)); // 确保 writer 先启动
    // 1. 打开信号量
    sem_t *sem = sem_open(SEM_NAME, 0);
    if (sem == SEM_FAILED) {
        perror("sem_open (reader)");
        return 1;
    }
    // 2. 打开共享内存
    int shm_fd = shm_open(SHM_NAME, O_RDONLY, 0666);
    if (shm_fd == -1) {
        perror("shm_open (reader)");
        return 1;
    }
    // 3. 内存映射 (只读)
    void* ptr = mmap(0, sizeof(SharedData), PROT_READ, MAP_SHARED, shm_fd, 0);
    if (ptr == MAP_FAILED) {
        perror("mmap (reader)");
        return 1;
    }
    SharedData *data = static_cast<SharedData*>(ptr);
    // 4. 读取共享内存 (受信号量保护)
    for (int i = 0; i < 5; ++i) {
        sem_wait(sem); // --- P 操作：获取锁 ---
        // --- 临界区开始 ---
        cout << "[Reader] 读到: " << data->message << " (Counter: " << data->counter << ")" << endl;
        // --- 临界区结束 ---
        sem_post(sem); // --- V 操作：释放锁 ---
        this_thread::sleep_for(chrono::seconds(1));
    }
    // 5. 清理
    munmap(ptr, sizeof(SharedData));
    close(shm_fd);
    sem_close(sem);
    return 0;
}
```
3. 编译
```
g++ shm_synced_writer.cpp -o writer -lrt -lpthread
g++ shm_synced_reader.cpp -o reader -lrt -lpthread
```
## 消息队列
内核维护的一个“消息链表”。进程可以向队列中添加消息，也可以从队列中读取消息。
### 原理
与管道（字节流）不同，消息队列是以“消息”为单位的，每条消息都有自己的类型和大小
### 特点
1. 消息边界：读取时总是获取一条完整的消息，而不是任意字节。
2. 异步：发送方不需要等待接收方，消息会暂存在内核中。
3. 多对多：多个进程可以读写同一个队列。
4. 优先级：POSIX 消息队列支持消息优先级。
### 适用场景
任务分发。例如，一个 Web 服务器（生产者）接收到请求，将其封装为一条消息放入队列，多个工作进程（消费者）从队列中取出任务并处理。
### 代码
见 C++ 多线程编程笔记中 Message 部分
## 套接字
目前在本地 IPC 中功能最强大、最灵活的机制。提供了与网络套接字几乎完全相同的 API，但通信完全在内核中进行，不经过网络协议栈。
### 原理
使用文件系统中的一个特殊文件（*.sock）作为“地址”，而不是 IP 和端口。
### 特点
1. 双向通信：客户端和服务端可以自由地双向收发数据。
2. 流式(SOCK_STREAM)：提供像 TCP 一样可靠、有序的字节流。
3. 数据报(SOCK_DGRAM)：提供像 UDP 一样不可靠的、以包为单位的通信。
4. 高性能：比 TCP/IP 回环（Loopback）快得多，因为它不涉及 TCP 的复杂状态机和 IP 路由。
5. 安全：可以通过文件系统的权限来控制哪些用户/进程可以访问该 socket 文件。
### 适用场景
几乎所有现代 Linux 守护进程（如 docker, systemd, X11）都使用 Unix Socket 作为其本地客户端的主要通信方式。
### 代码
见 C++ 多线程编程笔记中 Socket 部分
## 信号
最古老、最轻量级的形式。它不用于传输数据，而是用于异步通知。
### 概念
内核向一个进程发送一个“事件通知”（例如 SIGTERM, SIGHUP, SIGUSR1）
### 特点
1. 异步：它会中断进程的正常执行流程。
2. 无数据：几乎不携带信息（除了极少数例外）。
3. 预定义：大多数信号都有预定义的含义（如 SIGKILL 强制终止）。
### 适用场景
1. 父进程管理子进程（如 waitpid 等待 SIGCHLD）。
2. 通知守护进程（如 `kill -HUP <pid>` 通知 nginx 重新加载配置）。
3. Ctrl+C 发送 SIGINT 来中断前台进程。