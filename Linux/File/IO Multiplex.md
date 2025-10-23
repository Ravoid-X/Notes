## 概述
1. 阻塞 IO 一个线程处理一个连接（File Descriptor, FD），扩展性极差
2. 非阻塞 IO 一个线程处理所有连接，避免了线程阻塞，但引入了用户态的忙轮询，CPU 被浪费在大量空转的系统调用上。
3. IO 多路复用提供了一种同步 IO 模型，允许单个线程（或进程）同时监视多个 FD，核心思想是将“轮询”工作从用户态下放到内核态

## select
最早的、符合 POSIX 标准的多路复用实现
### API
```
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```
1. nfds：需要监视的最大文件描述符加1，即待监视的文件描述符的最大值加1。
2. readfds：可读性检查的文件描述符集合。
3. writefds：可写性检查的文件描述符集合。
4. exceptfds：异常条件的文件描述符集合。
5. timeout：最长等待时间（NULL表示一直阻塞直到有事件发生）
### 宏
```
FD_ZERO(fd_set *set): 清空集合
FD_SET(int fd, fd_set *set): 将 fd 添加到集合
FD_CLR(int fd, fd_set *set): 将 fd 从集合中移除
FD_ISSET(int fd, fd_set *set): 检查 fd 是否在集合中（即是否就绪）
```
### fd_set
1. select 的核心，本质上是位图，长度固定（通常由 FD_SETSIZE 决定，编译时固定，常见为1024）
2. 如果 fd_set 的第 n 位被置 1，表示程序关心第 n 号 FD
### 工作流程
1. 用户态（调用前）： 程序员必须手动创建并填充 fd_set，设置所有需要监视的位。
2. 拷贝（用户态 -> 内核态）： 调用 select 时，整个 fd_set 被从用户空间拷贝到内核空间。
3. 内核态（监视）： 内核线性遍历这个位图（从 0 到 maxfd），检查每个被标记的 FD 的当前状态。
4. 内核态（返回）： 如果没有 FD 就绪，内核让该线程休眠。当有 FD 就绪或超时，内核会修改拷贝到内核的 fd_set，将未就绪的 FD 位清零。
5. 拷贝（内核态 -> 用户态）： 内核将这个被修改过的 fd_set 再次拷贝回用户空间，覆盖原始的 fd_set。
6. 用户态（调用后）： select 返回。程序不知道具体是哪个FD就绪，必须再次线性遍历整个 fd_set（从 0 到 maxfd），使用 FD_ISSET() 宏来检查哪一位仍然是 1。
### 返回值
1. 大于 0：返回值为有事件发生的文件描述符的总数。
2. 0：表示超时，没有事件发生。
3. -1：出错，可以通过查看全局变量 errno 来获取错误码。
### 缺点
1. FD_SETSIZE 限制了监视的 FD 数量，默认 1024
2. 每次调用 select 都需要将 fd_set 在用户空间和内核空间之间来回拷贝，开销较大
3. 内核需要 O(n) 遍历所有传入的 FD 来检查就绪状态，用户空间在 select 返回后，也需要 O(n) 遍历来找出是哪些 FD 就绪了
4. fd_set 是“值-结果”参数，内核会修改它，导致用户每次循环都必须重新构建 fd_set
### 代码示例
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/select.h>

#define PORT 8080
#define MAX_CLIENTS 30
#define BUFFER_SIZE 1024

int main() {
    int server_fd, new_socket, client_socket[MAX_CLIENTS], max_clients = MAX_CLIENTS;
    int activity, valread, sd, max_sd;
    struct sockaddr_in address;
    int opt = 1;
    socklen_t addrlen = sizeof(address);
    char buffer[BUFFER_SIZE];

    fd_set readfds; // 监视读事件的 fd_set
    // 初始化所有 client_socket 为0
    for (int i = 0; i < max_clients; i++) {
        client_socket[i] = 0;
    }
    // 创建监听socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, 3);
    printf("Server listening on port %d\n", PORT);
    while (1) {
        // 1. 每次循环都必须重新构建 fd_set
        FD_ZERO(&readfds);
        // 2. 将 server_fd (监听socket) 添加到集合
        FD_SET(server_fd, &readfds);
        max_sd = server_fd; // max_sd 是 select 需要检查的FD的最大值+1
        // 3. 将所有已连接的 client_socket 添加到集合
        for (int i = 0; i < max_clients; i++) {
            sd = client_socket[i];
            if (sd > 0) { // 如果是有效的fd
                FD_SET(sd, &readfds);
            }
            if (sd > max_sd) { // 更新 max_sd
                max_sd = sd;
            }
        }
        // 4. 调用 select()，阻塞等待
        // nfds 参数是 max_sd + 1
        activity = select(max_sd + 1, &readfds, NULL, NULL, NULL);
        if (activity < 0) {
            perror("select error");
            break;
        }
        // 5. 检查 server_fd 是否就绪 (即有新连接)
        if (FD_ISSET(server_fd, &readfds)) {
            new_socket = accept(server_fd, (struct sockaddr *)&address, &addrlen);
            // 将新连接添加到 client_socket 数组
            for (int i = 0; i < max_clients; i++) {
                if (client_socket[i] == 0) {
                    client_socket[i] = new_socket;
                    printf("New connection, socket fd is %d\n", new_socket);
                    break;
                }
            }
        }
        // 6. 遍历所有 client_socket，检查它们是否就绪 (即有数据)
        for (int i = 0; i < max_clients; i++) {
            sd = client_socket[i];
            if (FD_ISSET(sd, &readfds)) {
                // 读取数据
                valread = read(sd, buffer, BUFFER_SIZE - 1);
                if (valread == 0) {
                    // 客户端断开连接
                    getpeername(sd, (struct sockaddr*)&address, &addrlen);
                    printf("Client disconnected, fd %d\n", sd);
                    close(sd);
                    client_socket[i] = 0; // 从数组中移除
                } else if (valread > 0) {
                    // 正常收到数据
                    buffer[valread] = '\0';
                    printf("Received from fd %d: %s\n", sd, buffer);
                    // (示例：回显数据)
                    // send(sd, buffer, strlen(buffer), 0);
                } else {
                    perror("read error");
                    close(sd);
                    client_socket[i] = 0;
                }
            }
        }
    }
    return 0;
}
```
### 源码解释
1. 拷贝 fd_set：内核首先分配内核空间的 fd_set，然后使用 copy_from_user() 将用户传入的 readfds, writefds, exceptfds 拷贝到内核空间。
2. 获取 file 结构：对每个 fd，内核通过 fdget() 获取其对应的 struct file 对象
3. 调用 f_op->poll\
（1）这是最核心的一步。内核调用 file->f_op->poll (文件操作集中的 poll 方法)。对于 socket，这会调用到 sock_poll\
（2）sock_poll 会检查 socket 的状态（是否有数据、是否可写等）\
（3）关键：sock_poll 还会调用 poll_wait()
4. poll_wait() 将正在执行 select 的进程的等待项添加到这个 FD 对应的驱动程序（如 socket）的等待队列上，像是在 FD 上设定了一个回调
5. 内核遍历完所有 nfds 后，如果没有 FD 就绪，do_select 就会调用 schedule_timeout() 让当前进程进入睡眠
6. 当某个 FD收到数据时，网卡驱动的中断处理程序会触发，最终调用 wake_up() 唤醒注册在该 socket 等待队列上的所有进程
7. select 进程就在其中，于是它被唤醒，从 schedule_timeout() 返回
## poll
1. 可以监视任意数量的 FD，只要内存允许（受限于系统的 RLIMIT_NOFILE）
2. events（输入）和 revents（输出）分离，使得 fds 数组可以被重用
3. 两次拷贝和两次遍历依旧没有解决
### API
```
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
struct pollfd {
    int   fd;       // 文件描述符
    short events;   // [输入] 请求监视的事件 (e.g., POLLIN, POLLOUT)
    short revents;  // [输出] 实际发生的事件 (由内核填充)
};
```
### 流程
1. （用户空间）创建一个 struct pollfd 数组，并填充 fd 和 events 字段，调用 poll()
2. （内核）将整个 fds 数组从用户空间拷贝到内核空间，并遍历内核空间的 fds 数组
3. （内核）对每个 fd，执行与 select 几乎完全相同的操作：调用 f_op->poll，然后 poll_wait() 将进程注册到 FD 的等待队列上。
4. （内核）如果遍历完都没有就绪的，调用 schedule_timeout() 睡眠。
5. （内核）被唤醒后（无论是数据到了还是超时），再次遍历整个 fds 数组，检查哪些 FD 就绪了。
6. （内核）将就绪状态填入对应 struct pollfd 的 revents 字段，将整个 fds 数组拷贝回用户空间
7. （用户空间）poll() 返回，返回值是就绪 FD 的总数。
8. （用户空间）遍历 fds 数组，检查每个元素的 revents 字段，找出就绪的 FD。
9. （用户空间）下次循环不需要重新构建数组（fd 和 events 字段不变），但通常需要清空 revents（虽然不清空也行，但检查时要小心）。
### 代码示例
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <poll.h>

#define PORT 8080
#define MAX_FDS 200 // poll 可以设置得很大
#define BUFFER_SIZE 1024

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int opt = 1;
    socklen_t addrlen = sizeof(address);
    char buffer[BUFFER_SIZE];
    struct pollfd fds[MAX_FDS];
    int nfds = 1; // fds 数组中当前的FD数量, 初始只有 server_fd
    // 创建监听socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, 3);
    printf("Server listening on port %d\n", PORT);
    // 1. 初始化 fds 数组
    memset(fds, 0, sizeof(fds));
    // 2. 添加 server_fd
    fds[0].fd = server_fd;
    fds[0].events = POLLIN; // 关心读事件 (新连接)
    while (1) {
        // 3. 调用 poll()，阻塞等待
        // timeout = -1 表示无限等待
        int activity = poll(fds, nfds, -1); 
        if (activity < 0) {
            perror("poll error");
            break;
        }
        // 4. 检查 server_fd (总是在 fds[0])
        if (fds[0].revents & POLLIN) {
            new_socket = accept(server_fd, (struct sockaddr *)&address, &addrlen);
            // 5. 将新连接添加到 fds 数组
            if (nfds < MAX_FDS) {
                fds[nfds].fd = new_socket;
                fds[nfds].events = POLLIN;
                nfds++;
                printf("New connection, socket fd is %d\n", new_socket);
            } else {
                printf("Max connections reached, rejecting\n");
                close(new_socket);
            }
        }
        // 6. 遍历所有 client_socket (从 fds[1] 开始)
        for (int i = 1; i < nfds; i++) {
            if (fds[i].revents & POLLIN) {
                int sd = fds[i].fd;
                int valread = read(sd, buffer, BUFFER_SIZE - 1);
                if (valread == 0) {
                    // 客户端断开
                    printf("Client disconnected, fd %d\n", sd);
                    close(sd);
                    // 7. (重要) 从 fds 数组中移除
                    // 将最后一个元素移到当前位置，然后 nfds--
                    // (这是O(1)的移除方式)
                    fds[i] = fds[nfds - 1]; 
                    nfds--;
                    i--; // 因为 fds[i] 已经被替换了, 需要重新检查
                } else if (valread > 0) {
                    buffer[valread] = '\0';
                    printf("Received from fd %d: %s\n", sd, buffer);
                } else {
                    perror("read error");
                    close(sd);
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;
                }
            }
        }
    }
    return 0;
}
```
## epoll
1. 抛弃了 select/poll 的被动检查思想，转而采用主动的回调机制
2. 彻底解决了 select 和 poll 的 O(n) 问题，是实现高性能网络服务器的基石（如 Nginx, Redis）
### API
```
#include <sys/epoll.h>

// 1. 创建一个 epoll 实例 (返回一个 epoll fd)
int epoll_create(int size); // 'size' 在现代内核中已被忽略，>0 即可
int epoll_create1(int flags); // 推荐使用, flags=0 等同于 epoll_create
// 2. 控制 epoll 实例 (添加/修改/删除 关心的 fd)
int epoll_ctl(int epfd,      // epoll_create 返回的 fd
              int op,        // 操作: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL
              int fd,        // 目标 fd
              struct epoll_event *event); // 事件描述
// 3. 等待事件发生
int epoll_wait(int epfd,          // epoll_create 返回的 fd
               struct epoll_event *events, // [输出] 承载就绪事件的数组
               int maxevents,     // events 数组的大小
               int timeout);      // 超时时间
```
### 核心数据结构
```
struct epoll_event {
    uint32_t     events;    // 关心的事件 (e.g., EPOLLIN, EPOLLOUT, EPOLLET)
    epoll_data_t data;      // 用户数据 (非常重要)
};
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;
```
当 epoll_wait 返回时，data 会被带回，立即知道是哪个 FD（或哪个对象）就绪了，无需再次遍历
### 工作流程
1. （用户空间）调用 epoll_create1(0)，内核会创建一个 epoll 实例，并返回一个 epfd。
2. （内核）创建一个 struct eventpoll 对象。这个对象内部维护着两个核心结构：\
（1）红黑树：用于存储所有通过 epoll_ctl 添加的 FD（称为“兴趣列表”）。这保证了 ADD/MOD/DEL 的效率为 O(log N)。\
（2）就绪列表：一个双向链表，用于存储那些已经就绪的 FD。
3. （用户空间）对每一个关心的 FD，调用 epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev)。
4. （内核）创建一个 struct epitem（epoll item）来代表这个 fd，并将其插入到 eventpoll 对象的红黑树中。
5. （内核）关键：注册一个回调函数（ep_poll_callback）到这个 fd 的驱动程序等待队列上。
6. （用户空间）调用 epoll_wait(epfd, events, MAX_EVENTS, -1)。
7. （内核）epoll_wait 检查 eventpoll 对象的就绪列表。\
（1）如果为空，进程在 eventpoll 自身的等待队列上睡眠。\
（2）如果不为空，epoll_wait 会将就绪列表中的 epitem 对应的 epoll_event（包含用户 data）拷贝到用户空间的 events 数组中。\
（3）注意：这里只拷贝就绪的 FD，数量为 k，而不是全部 N 个 FD。\
（4）epoll_wait 返回就绪的 FD 数量 k。
8. 唤醒\
（1）当某个 fd 收到数据时，网卡驱动的中断处理程序被触发\
（2）它会调用 wake_up() 唤醒该 fd 等待队列上的进程\
（3）epoll 注册的回调函数 ep_poll_callback 被执行\
（4）ep_poll_callback 作用：将这个 fd 对应的 epitem 从红黑树移动（或添加）到 eventpoll 对象的就绪列表中\
（5）然后，ep_poll_callback 唤醒在 eventpoll 自身等待队列上睡眠的进程（即 epoll_wait 进程）。
9. epoll_wait 返回值 n。用户只需遍历 events 数组的前 n 个元素即可。
### LT (Level Triggered) vs ET (Edge Triggered)
1. LT 水平触发 (默认)\
（1）行为类似 poll。只要 FD 处于就绪状态（比如 socket 缓冲区里还有数据没读完），epoll_wait 每次都会返回这个 FD\
（2）编程简单，不易出错
2. ET 边缘触发 (EPOLLET)\
（1）只有当 FD 的状态发生变化时（比如 socket 缓冲区从“空”变为“非空”），epoll_wait 才会返回这个 FD，且只返回一次\
（2）如果 epoll_wait 返回了 EPOLLIN，必须使用非阻塞 I/O，循环 read() 这个 FD，直到 read() 返回 EAGAIN 或 EWOULDBLOCK（数据已读完）\
（3）效率极高，避免了 epoll_wait 被同一个未处理完的事件反复唤醒\
（4）编程复杂，容易出错（比如没有读完数据，导致后续数据永远无法触发通知）
### 缺点
1. Linux 独有：可移植性差（BSD 有 kqueue，Windows 有 IOCP，它们原理相似）
2. API 稍复杂：需要三个函数配合使用
### 代码示例
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <fcntl.h> // for fcntl
#include <errno.h> // for EAGAIN

#define PORT 8080
#define MAX_EVENTS 64 // epoll_wait 一次最多处理的事件数
#define BUFFER_SIZE 1024

// 设置 fd 为非阻塞
void set_non_blocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int opt = 1;
    socklen_t addrlen = sizeof(address);
    char buffer[BUFFER_SIZE];

    int epoll_fd;
    struct epoll_event ev, events[MAX_EVENTS];
    // 1. 创建 epoll 实例
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }
    // 创建监听socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    set_non_blocking(server_fd); // server_fd 也设为非阻塞
    
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    
    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, SOMAXCONN);
    printf("Server listening on port %d\n", PORT);
    // 2. 将 server_fd 添加到 epoll
    ev.events = EPOLLIN | EPOLLET; // 监听 读事件 和 边缘触发
    ev.data.fd = server_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1) {
        perror("epoll_ctl: server_fd");
        exit(EXIT_FAILURE);
    }
    while (1) {
        // 3. 调用 epoll_wait()
        int n_ready = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (n_ready == -1) {
            perror("epoll_wait");
            break;
        }
        // 4. 遍历就绪的事件 (O(k) 复杂度)
        for (int i = 0; i < n_ready; i++) {
            if (events[i].data.fd == server_fd) {
                // 是 server_fd 就绪 (新连接)
                // ET 模式，必须循环 accept 直到 EAGAIN
                while (1) {
                    new_socket = accept(server_fd, (struct sockaddr *)&address, &addrlen);
                    if (new_socket == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            // 已经没有新连接了
                            break; 
                        } else {
                            perror("accept");
                            break;
                        }
                    }
                    set_non_blocking(new_socket); // **必须**设为非阻塞
                    ev.events = EPOLLIN | EPOLLET; // 关心 读事件 和 ET
                    ev.data.fd = new_socket;
                    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, new_socket, &ev) == -1) {
                        perror("epoll_ctl: new_socket");
                        close(new_socket);
                    } else {
                         printf("New connection, socket fd is %d\n", new_socket);
                    }
                }
            } else {
                // 是 client_fd 就绪 (有数据)
                int client_fd = events[i].data.fd;
                // ET 模式，必须循环 read 直到 EAGAIN
                while (1) {
                    int valread = read(client_fd, buffer, BUFFER_SIZE - 1);
                    if (valread == 0) {
                        // 客户端断开
                        printf("Client disconnected, fd %d\n", client_fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL); // 从 epoll 移除
                        close(client_fd);
                        break;
                    } else if (valread > 0) {
                        buffer[valread] = '\0';
                        printf("Received from fd %d: %s\n", client_fd, buffer);
                    } else {
                        // valread == -1
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            // 数据读完了
                            break;
                        } else {
                            perror("read error");
                            epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
                            close(client_fd);
                            break;
                        }
                    }
                }
            }
        } // end for loop
    } // end while(1)
    close(server_fd);
    close(epoll_fd);
    return 0;
}
```