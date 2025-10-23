## 概述
1. 在 Linux 中，网络连接被抽象成文件描述符（非负整数），称为 Socket (套接字)。
2. Socket 在传输层创建和管理连接，实现跨进程/跨主机的数据传输
## 流程
### 服务端
1. 创建 (socket())： 创建一个套接字用于通信。
2. 绑定地址 (bind())： 将套接字与一个 IP 地址和 端口号 关联起来。这样客户端才能找到并连接到它。
3. 监听连接 (listen())： 将套接字置于被动监听模式，准备接受客户端的连接请求。
4. 接受连接 (accept())： 阻塞等待客户端的连接。一旦有客户端连接，就会创建一个新的套接字用于与该客户端通信。
5. 数据收发 (read()/recv(), write()/send())： 通过新的套接字与客户端进行数据交换。
6. 关闭 (close())： 通信结束后关闭套接字。
### 客户端
1. 创建 (socket())： 创建一个套接字。
2. 连接服务器 (connect())： 使用服务器的 IP 地址和 端口号，向服务器发起连接请求。
3. 数据收发 (write()/send(), read()/recv())： 与服务器进行数据交换。
4. 关闭 (close())： 通信结束后关闭套接字。
## 类型
### 流式套接字（SOCK_STREAM）
1. 提供一个面向连接、可靠的、基于字节流的服务，数据不会丢失、不会重复、且按发送顺序到达。
2. 数据传输前必须建立连接（如 TCP 的三次握手）。
3. 典型代表：TCP。适用于网页浏览(HTTP)、文件传输(FTP)、邮件(SMTP)等要求高可靠性的场景。
### 数据报套接字（SOCK_DGRAM）
1. 提供一个无连接、不可靠的服务，不保证数据送达，不保证顺序，可能会重复。
2. 每个数据包（报文）都是独立路由的，大小受限。
3. 典型代表：UDP。适用于视频直播、在线游戏、DNS查询等能容忍少量丢包但对实时性要求高的场景。
### 原始套接字（SOCK_RAW）
1. 允许直接访问网络层的协议，可以读写IP数据包，需要管理员权限才能创建。
2. 常用于网络嗅探、协议分析等特殊应用。
## socket()
### 代码
返回一个整数，即套接字描述符。创建失败，则返回-1。
```
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```
### domain (地址族)
1. AF_INET： 这是最常用的，表示使用IPv4协议。
2. AF_INET6： 表示使用IPv6协议。
3. AF_UNIX 或 AF_LOCAL： 用于同一台主机上的进程间通信（IPC）。使用文件系统路径作为地址。
### type (套接字类型)
1. SOCK_STREAM (流式套接字)
2. SOCK_DGRAM (数据报套接字)
3. SOCK_RAW (原始套接字)
### protocol (协议)
1. 通常设置为 0，表示由系统根据 domain 和 type 自动选择默认的协议。
2. 当 domain 为 AF_INET，type 为 SOCK_STREAM 时，默认协议是 TCP。
3. 当 domain 为 AF_INET，type 为 SOCK_DGRAM 时，默认协议是 UDP。
4. 如果需要指定特定的协议，可以使用 IPPROTO_TCP 或 IPPROTO_UDP 等常量。
## 服务端
```
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/un.h>

// 1. 选择协议: PROTOCOL_TCP 或 PROTOCOL_UDP
constexpr int PROTOCOL_TCP = 1;
constexpr int PROTOCOL_UDP = 2;
const int SERVER_PROTOCOL_TYPE = PROTOCOL_TCP;
// 2. 选择通信模式: COMM_INTER_HOST (不同主机) 或 COMM_LOCAL_IPC (同主机)
constexpr int COMM_INTER_HOST = 1;
constexpr int COMM_LOCAL_IPC = 2;
const int SERVER_COMM_MODE = COMM_INTER_HOST; 
// 3. 网络配置 (COMM_INTER_HOST 模式)
const int SERVER_PORT = 8080;
// 4. 本地IPC配置 (COMM_LOCAL_IPC 模式)
const char* SERVER_SOCKET_PATH = "/tmp/my_server_socket";

// 私有辅助函数声明
void handle_tcp_connection(int client_socket);
void run_udp_loop(int server_socket);

//此函数会阻塞，持续监听和处理请求
void run_server() {
    int server_fd;
    int socket_type = (SERVER_PROTOCOL_TYPE == PROTOCOL_TCP) ? SOCK_STREAM : SOCK_DGRAM;
    int domain = (SERVER_COMM_MODE == COMM_INTER_HOST) ? AF_INET : AF_UNIX;
    // 1. 创建套接字
    if ((server_fd = socket(domain, socket_type, 0)) == -1) {
        perror("server: socket failed");
        return;
    }
    int opt = 1;
    if (SERVER_COMM_MODE == COMM_INTER_HOST) {
        setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    }
    // 2. 绑定地址
    if (SERVER_COMM_MODE == COMM_INTER_HOST) {
        struct sockaddr_in address;
        address.sin_family = AF_INET;
        address.sin_addr.s_addr = INADDR_ANY;
        address.sin_port = htons(SERVER_PORT);
        if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
            perror("server: bind failed");
            close(server_fd);
            return;
        }
    } else { // COMM_LOCAL_IPC
        unlink(SERVER_SOCKET_PATH);
        struct sockaddr_un address;
        address.sun_family = AF_UNIX;
        strncpy(address.sun_path, SERVER_SOCKET_PATH, sizeof(address.sun_path) - 1);
        if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
            perror("server: bind failed");
            close(server_fd);
            return;
        }
    }
    // 3. 根据协议类型处理连接
    if (SERVER_PROTOCOL_TYPE == PROTOCOL_TCP) {
        if (listen(server_fd, 5) < 0) {
            perror("server: listen failed");
            close(server_fd);
            return;
        }
        while (true) {
            int client_socket = accept(server_fd, nullptr, nullptr);
            if (client_socket < 0) {
                perror("server: accept failed");
                continue;
            }
            cout << "Server: Accepted a new connection." << endl;
            handle_tcp_connection(client_socket);
            close(client_socket);
            cout << "Server: Connection closed." << endl;
        }
    } else { // PROTOCOL_UDP
        run_udp_loop(server_fd);
    }

    close(server_fd);
}
// TCP: 循环接收来自特定客户端的消息并回显
void handle_tcp_connection(int client_socket) {
    char buffer[4096];
    ssize_t bytes_received;
    while ((bytes_received = recv(client_socket, buffer, sizeof(buffer) - 1, 0)) > 0) {
        buffer[bytes_received] = '\0';
        cout << "Received (TCP): " << buffer << endl;
        send(client_socket, buffer, bytes_received, 0); // Echo
    }
}
// UDP: 循环接收来自任何客户端的消息并回显
void run_udp_loop(int server_socket) {
    char buffer[4096];
    struct sockaddr_storage client_addr;
    socklen_t client_len = sizeof(client_addr);
    while (true) {
        ssize_t bytes_received = recvfrom(server_socket, buffer, sizeof(buffer) - 1, 0, (struct sockaddr*)&client_addr, &client_len);
        if (bytes_received < 0) {
            perror("recvfrom failed");
            continue;
        }
        buffer[bytes_received] = '\0';
        cout << "Received (UDP): " << buffer << endl;
        sendto(server_socket, buffer, bytes_received, 0, (struct sockaddr*)&client_addr, client_len); // Echo
    }
}
```
## 客户端
```
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/un.h>

// 1. 选择协议: PROTOCOL_TCP 或 PROTOCOL_UDP
constexpr int PROTOCOL_TCP = 1;
constexpr int PROTOCOL_UDP = 2;
const int CLIENT_PROTOCOL_TYPE = PROTOCOL_TCP; // <--- 修改这里
// 2. 选择通信模式: COMM_INTER_HOST (不同主机) 或 COMM_LOCAL_IPC (同主机)
constexpr int COMM_INTER_HOST = 1;
constexpr int COMM_LOCAL_IPC = 2;
const int CLIENT_COMM_MODE = COMM_INTER_HOST; // <--- 修改这里
// 3. 网络配置 (COMM_INTER_HOST 模式)
const char* TARGET_SERVER_IP = "127.0.0.1";
const int TARGET_SERVER_PORT = 8080;
// 4. 本地IPC配置 (COMM_LOCAL_IPC 模式)
const char* TARGET_SOCKET_PATH = "/tmp/my_server_socket"; // 必须与服务器的路径一致

void run_client() {
    int sock;
    int socket_type = (CLIENT_PROTOCOL_TYPE == PROTOCOL_TCP) ? SOCK_STREAM : SOCK_DGRAM;
    int domain = (CLIENT_COMM_MODE == COMM_INTER_HOST) ? AF_INET : AF_UNIX;
    
    if ((sock = socket(domain, socket_type, 0)) < 0) {
        perror("client: socket creation error");
        return;
    }

    struct sockaddr_storage server_addr;
    socklen_t server_len;
    if (CLIENT_COMM_MODE == COMM_INTER_HOST) {
        struct sockaddr_in* addr = (struct sockaddr_in*)&server_addr;
        addr->sin_family = AF_INET;
        addr->sin_port = htons(TARGET_SERVER_PORT);
        inet_pton(AF_INET, TARGET_SERVER_IP, &addr->sin_addr);
        server_len = sizeof(*addr);
    } else {
        struct sockaddr_un* addr = (struct sockaddr_un*)&server_addr;
        addr->sun_family = AF_UNIX;
        strncpy(addr->sun_path, TARGET_SOCKET_PATH, sizeof(addr->sun_path) - 1);
        server_len = sizeof(*addr);
    }

    if (CLIENT_PROTOCOL_TYPE == PROTOCOL_TCP) {
        if (connect(sock, (struct sockaddr *)&server_addr, server_len) < 0) {
            perror("client: connection failed");
            close(sock);
            return;
        }
        cout << "Client: Connected to server." << endl;
    } else {
         cout << "Client: Ready to send UDP messages." << endl;
    }
    
    cout << "You can start sending messages ('exit' to quit)." << endl;
    
    string line;
    char buffer[4096];
    while (getline(cin, line)) {
        if (line == "exit") break;
        if (line.empty()) continue;
        
        ssize_t sent_bytes;
        if (CLIENT_PROTOCOL_TYPE == PROTOCOL_TCP) {
            sent_bytes = send(sock, line.c_str(), line.length(), 0);
        } else {
            sent_bytes = sendto(sock, line.c_str(), line.length(), 0, (struct sockaddr*)&server_addr, server_len);
        }
        if (sent_bytes < 0) {
            perror("send error");
            break;
        }
        ssize_t received_bytes = recv(sock, buffer, sizeof(buffer) - 1, 0);
        if (received_bytes <= 0) {
            cout << "Server closed connection or error." << endl;
            break;
        }
        buffer[received_bytes] = '\0';
        cout << "Echo from server: " << buffer << endl;
    }
    close(sock);
}
```
## 客户端 + 服务端
会启动一个后台线程进行监听，同时主线程可以用于发送消息。
```
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/un.h>
#include <pthread.h>

// 1. 选择协议: PROTOCOL_TCP 或 PROTOCOL_UDP
constexpr int PROTOCOL_TCP = 1;
constexpr int PROTOCOL_UDP = 2;
const int PEER_PROTOCOL_TYPE = PROTOCOL_TCP; 
// 2. 选择通信模式: COMM_INTER_HOST (不同主机) 或 COMM_LOCAL_IPC (同主机)
constexpr int COMM_INTER_HOST = 1;
constexpr int COMM_LOCAL_IPC = 2;
const int PEER_COMM_MODE = COMM_INTER_HOST; 
// 3. 网络配置 (COMM_INTER_HOST 模式)
// 注意：P2P中，一个 peer 监听的端口就是另一个 peer 要连接的目标端口
const int MY_LISTEN_PORT = 8080;
const char* TARGET_PEER_IP = "127.0.0.1";
const int TARGET_PEER_PORT = 8081; // 假设连接到另一个 peer 的8081端口
// 4. 本地IPC配置 (COMM_LOCAL_IPC 模式)
const char* MY_SOCKET_PATH = "/tmp/peer_socket_A";
const char* TARGET_SOCKET_PATH = "/tmp/peer_socket_B";

void* receiver_thread_func(void* arg) {
    int server_sock = *((int*)arg);
    char buffer[4096];
    
    if (PEER_PROTOCOL_TYPE == PROTOCOL_TCP) {
        listen(server_sock, 1);
        cout << "[Receiver Thread] Listening for a peer..." << endl;
        int peer_sock = accept(server_sock, nullptr, nullptr);
        if (peer_sock < 0) { perror("peer: accept failed"); return nullptr; }
        
        cout << "[Receiver Thread] Peer connected." << endl;
        while (true) {
            ssize_t n = recv(peer_sock, buffer, sizeof(buffer) - 1, 0);
            if (n <= 0) { cout << "[Receiver Thread] Peer disconnected." << endl; break; }
            buffer[n] = '\0';
            cout << "\n[Received Message]: " << buffer << endl;
        }
        close(peer_sock);
    } else { // UDP
        cout << "[Receiver Thread] Listening for UDP messages..." << endl;
        while (true) {
            ssize_t n = recvfrom(server_sock, buffer, sizeof(buffer) - 1, 0, nullptr, nullptr);
            if (n < 0) continue;
            buffer[n] = '\0';
            cout << "\n[Received Message]: " << buffer << endl;
        }
    }
    return nullptr;
}

void run_peer() {
    // ---- Part 1: 设置监听 (服务器角色) ----
    int listen_sock;
    int socket_type = (PEER_PROTOCOL_TYPE == PROTOCOL_TCP) ? SOCK_STREAM : SOCK_DGRAM;
    int domain = (PEER_COMM_MODE == COMM_INTER_HOST) ? AF_INET : AF_UNIX;
    
    listen_sock = socket(domain, socket_type, 0);
    
    int opt = 1;
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    if (PEER_COMM_MODE == COMM_INTER_HOST) {
        struct sockaddr_in listen_addr;
        listen_addr.sin_family = AF_INET;
        listen_addr.sin_addr.s_addr = INADDR_ANY;
        listen_addr.sin_port = htons(MY_LISTEN_PORT);
        if (bind(listen_sock, (struct sockaddr*)&listen_addr, sizeof(listen_addr)) < 0) {
            perror("peer: bind failed"); return;
        }
    } else {
        unlink(MY_SOCKET_PATH);
        struct sockaddr_un listen_addr;
        listen_addr.sun_family = AF_UNIX;
        strncpy(listen_addr.sun_path, MY_SOCKET_PATH, sizeof(listen_addr.sun_path) - 1);
        if (bind(listen_sock, (struct sockaddr*)&listen_addr, sizeof(listen_addr)) < 0) {
            perror("peer: bind failed"); return;
        }
    }

    pthread_t receiver_tid;
    pthread_create(&receiver_tid, nullptr, receiver_thread_func, &listen_sock);
    
    // ---- Part 2: 主线程发送 (客户端角色) ----
    cout << "Peer application started. Listening in background." << endl;
    cout << "Type messages to send to the target peer." << endl;
    
    int send_sock;
    if ((send_sock = socket(domain, socket_type, 0)) < 0) {
        perror("peer: send socket creation failed");
        return;
    }
    
    // 准备目标地址
    struct sockaddr_storage target_addr;
    socklen_t target_len;
    if (PEER_COMM_MODE == COMM_INTER_HOST) {
        struct sockaddr_in* addr = (struct sockaddr_in*)&target_addr;
        addr->sin_family = AF_INET;
        addr->sin_port = htons(TARGET_PEER_PORT);
        inet_pton(AF_INET, TARGET_PEER_IP, &addr->sin_addr);
        target_len = sizeof(*addr);
    } else {
        struct sockaddr_un* addr = (struct sockaddr_un*)&target_addr;
        addr->sun_family = AF_UNIX;
        strncpy(addr->sun_path, TARGET_SOCKET_PATH, sizeof(addr->sun_path) - 1);
        target_len = sizeof(*addr);
    }
    // TCP需要先连接
    if (PEER_PROTOCOL_TYPE == PROTOCOL_TCP) {
        cout << "Press Enter to attempt connection..." << endl;
        cin.get(); // 等待用户确认
        if (connect(send_sock, (struct sockaddr*)&target_addr, target_len) < 0) {
            perror("peer: connect failed. Is the other peer running?");
        } else {
            cout << "Connection established. You can now send messages." << endl;
        }
    }
    string line;
    while (getline(cin, line)) {
        if (line == "exit") break;
        if (line.empty()) continue;

        if (PEER_PROTOCOL_TYPE == PROTOCOL_TCP) {
            send(send_sock, line.c_str(), line.length(), 0);
        } else {
            sendto(send_sock, line.c_str(), line.length(), 0, (struct sockaddr*)&target_addr, target_len);
        }
    }
    close(send_sock);
    pthread_cancel(receiver_tid);
    pthread_join(receiver_tid, nullptr);
    close(listen_sock);
}
```