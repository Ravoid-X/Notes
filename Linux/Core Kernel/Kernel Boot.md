## 概述
可以将整个启动过程分为四个主要阶段：
1. BIOS/UEFI 固件阶段：硬件自检，加载引导加载程序
2. Bootloader 阶段 (GRUB2)：加载内核镜像和 initramfs 到内存
3. Kernel 启动阶段：内核的“自解压”与初始化
4. User Space Init 阶段：内核启动第一个用户进程 (PID 1)，系统进入可用状态
## BIOS / UEFI 固件 & Bootloader
见 /Subject/Computer 启动部分
#  Kernel 启动
Bootloader（如 GRUB）把 vmlinuz（压缩的内核映像）和 initramfs（初始 RAM 文件系统）加载到了内存中，然后将 CPU 的控制权交给了 vmlinuz 的入口点。
## 内核解压与CPU模式切换（汇编阶段）
Bootloader 跳转到的地址并不是 vmlinux（最终的内核）的地址，而是 vmlinuz 中包含的一个小小的“引导存根”（Boot Stub）
### 相关文件 (x86架构)
1. arch/x86/boot/header.S
2. arch/x86/boot/compressed/head_64.S
3. arch/x86/boot/compressed/misc.c
### 入口 (实模式)
1. CPU 此时处于 16 位的实模式，这是 x86 CPU 加电后的默认状态，只能寻址 1MB 内存
2. header.S 中的代码是第一个执行的。它是一个 16 位的引导扇区，做一些非常基础的设置，然后跳转到 arch/x86/boot/main.c 中的 main() 函数
3. 注意：这不是内核的 main，而是解压存根的 main
### 解压内核 (保护模式)
1. misc.c 的主要工作是：找到内存中被 GRUB 加载进来的 vmlinuz 的压缩部分（它知道自己和压缩负载在内存中的相对位置）。
2. 它调用 decompress_kernel() 函数（例如，使用 zlib 或 lzma 算法）将压缩的 vmlinux 内核解压到内存中的一个高地址（例如物理地址 0x1000000，即 16MB 处，这是个历史约定）。
3. 解压完成后，它会跳转到 arch/x86/boot/compressed/head_64.S（假设是 64 位系统）。
### CPU模式切换
1. 此时，代码仍在 32 位保护模式下运行
2. 设置早期页表\
（1）问题：vmlinux 被链接在 一个非常高的虚拟地址（例如 0xffffffff81000000）上运行。但现在被解压到了一个很低的物理地址（例如 0x1000000）。CPU如何执行一个地址和它实际位置不符的程序？\
（2）解决：head_64.S 建立了一个临时的身份映射的页表\
（3）将虚拟地址 0x1000000 -> 映射到物理地址 0x1000000（1:1 映射）\
（4）将虚拟地址 0xffffffff81000000 -> 也映射到物理地址 0x1000000\
（5）无论CPU是按物理地址（跳转前）还是虚拟地址（跳转后）寻址，都能找到同一份内核代码
3. 开启64位长模式\
（1）设置 CR4 寄存器的 PAE（物理地址扩展）位\
（2）加载临时页表的地址到 CR3 寄存器\
（3）设置 EFER 寄存器的 LME（Long Mode Enable）位\
（4）开启 CR0 寄存器的 PG（Paging）位，正式启用分页
4. 跳转到 6 4位：head_64.S 执行一个长跳转（ljmp），CPU 正式进入 64 位长模式，并开始从高虚拟地址（0xffffffff81000000）执行代码
### 跳转到 C 语言
汇编代码（head_64.S）清空 BSS 段（未初始化的全局变量），设置好第一个内核堆栈，并最后跳转到内核的通用 C 语言入口：start_kernel()
## start_kernel()（C语言阶段）
1. 内核架构无关的 C 语言入口点。内核的 main() 函数。一旦进入这里，就有了堆栈和 C 语言环境
2. 唯一工作就是按顺序调用大量的初始化函数，唤醒内核的几乎所有子系统。
### 相关文件
init/main.c
### 初始化流程
1. pr_notice("Booting Linux ..."): 打印第一条内核日志信息（能在 dmesg 中看到）。
2. setup_arch(&command_line_data): 极其重要。这是一个架构相关的函数（如 arch/x86/kernel/setup.c）\
（1）解析 Bootloader 传递的内核命令行参数（如 root=/dev/sda1）\
（2）初始化控制台（console），让 printk 可以真正输出\
（3）探测 CPU 特性\
（4）探测和初始化内存布局（基于 BIOS/UEFI 提供的 E820 图），建立最终的内核页表，并抛弃临时的早期页表
3. trap_init(): 初始化中断描述符表（IDT），设置好各种 CPU 异常（如缺页中断、一般保护性故障）的处理函数。这是内核稳定运行的基础。
4. mm_init(): 初始化内存管理。此时，Buddy 伙伴系统和 Slab/Slub/Slob 分配器才真正开始工作。
5. sched_init(): 初始化进程调度器（例如 CFS）。初始化每个 CPU 的运行队列。
6. vfs_caches_init(): 初始化虚拟文件系统的缓存层，主要是 dcache（目录项缓存）和 icache（inode 缓存）。
7. init_timers(): 初始化内核定时器和 jiffies（时钟节拍）
8. ... (省略大量其他初始化：RCU, IRQ, ACPI, Ftrace, BPF, ...) ...
9. rest_init(): 这是调用的最后一个函数。start_kernel 本身的执行流到此结束
### rest_init()
1. 创建第一个进程 (PID 1)\
（1）rest_init() 通过 kernel_thread() 函数创建了内核的第一个进程（PID 1），并指定它去执行 kernel_init 函数。\
（2）这就是未来将成为所有用户进程“祖先”的 init 进程。
2. 创建第二个进程 (PID 2)\
（1）rest_init() 还创建了 PID 2，即 kthreadd。\
（2）这个进程是内核线程的“父进程”，未来所有其他的内核线程（如 kworker、flush）都由它派生
3. PID 0 空闲态\
（1）rest_init() 在创建完 PID 1 和 PID 2 后，PID 0（swapper 进程）会调用 schedule() 并进入空闲（Idle）循环。\
（2）当 CPU 上没有任何其他任务可运行时，就会执行 PID 0。
## kernel_init()（PID 1）
start_kernel 的执行流（称为 swapper 进程或 PID 0）在调用 rest_init 后，就完成了使命
### 相关文件
init/main.c
### do_basic_setup()
调用 driver_init() 来初始化驱动模型，并执行 do_initcalls()——这个函数会加载所有编译时静态链接进内核的驱动程序（例如基础的总线驱动）
### 挂载 initramfs
1. kernel_init 调用 prepare_namespace()。这个函数在内存中找到 GRUB 加载的 initramfs（一个 cpio 压缩包）。
2. 将这个 initramfs 解压到一个临时的、基于内存的文件系统（rootfs，也叫 tmpfs）中。
3. 内核将这个临时的 rootfs 挂载为根文件系统（/）。
4. 此时内核可能还不认识硬盘，根文件系统可能在 NVMe、LVM、RAID 或加密卷上，这些都需要驱动（.ko模块）。
5. initramfs 中就包含了这些驱动模块和加载它们的工具（如 modprobe）
### 执行 /init
1. initramfs 挂载成功后，kernel_init 函数会在这个临时的根文件系统中查找并执行 /init 程序（通常是一个 shell 脚本或一个小型 C 程序，如 systemd 的 systemd-stub）。
2. 注意：此时 PID 1 仍然是内核线程，但它即将执行用户空间的程序。
### initramfs 脚本 (用户空间)
1. /init 脚本（现在由 PID 1执行）开始运行。加载必要的驱动模块（如 nvme.ko, ext4.ko）。
2. 等待真实的根分区设备（如 /dev/nvme0n1p2）出现。
3. 将真实的根文件系统挂载到一个临时目录，例如 /sysroot。
4. 执行 pivot_root（或 switch_root）命令，将文件系统的根从临时的initramfs 切换到 /sysroot上。
### execve
1. initramfs 脚本的最后一步，是执行 exec /sbin/init (或者 exec /usr/lib/systemd/systemd)。
2. exec 系统调用（execve）是内核启动的分水岭。
3. 内核响应这个系统调用\
（1）从真实的硬盘文件系统加载 /sbin/init （即 systemd）程序\
（2）用 systemd 的程序代码替换掉 PID 1 进程当前（即 shell 脚本）的内存空间\
（3）将 CPU 的特权级别从内核态（Ring 0）切换到用户态（Ring 3）\
（4）CPU 开始执行 systemd 的程序入口
# User Space Init
1. 目标是将一个内核已就绪，但什么服务都没运行的最小系统，转变为一个用户可以登录并使用的、功能完备的系统。
2. 在现代 Linux 发行版中，这个过程几乎完全由 systemd 接管
## systemd 概述
### 特殊职责
1. 所有进程的祖先：系统上未来启动的每一个用户进程，都将是 systemd 的子进程或后代进程。
2. 孤儿进程的收养者：当一个父进程退出，而它的子进程仍在运行时，内核会自动将这些孤儿进程的父进程 ID 设置为 1
### 工作方式
不是基于脚本，而是基于单元
1. Unit（单元）：systemd 管理的基本对象。每一个服务、每一个设备、每一个挂载点，都是一个 Unit。这些 Unit 由.unit 配置文件（例如 sshd.service）定义。
2. Target（目标）：一种特殊的 Unit（以.target 结尾）。它本身不做什么，而是作为一组 Unit 的集合或同步点。例如，multi-user.target 代表所有“多用户模式”下应该运行的服务都已启动。
## 启动流程
### 解析依赖，确定目标
1. 加载配置：扫描文件系统中的特定目录（如 /usr/lib/systemd/system/ 和 /etc/systemd/system/），读取所有 .unit文件。
2. 构建依赖树：解析每个 Unit 文件中的依赖关系（如 Wants=，Requires=，After=）。例如，nginx.service 可能会声明 After=network.target（必须在网络就绪后启动）。systemd 在内存中构建一个庞大且复杂的依赖关系图。
3. 确定最终目标：查找 default.target。这通常是一个指向 graphical.target（图形桌面环境）或 multi-user.target（多用户命令行环境）的符号链接。这个 default.target 就是 systemd 本次启动的最终目标。
### 启动“事务”与并行化
1. 创建事务（Transaction）：以 default.target 为终点，反向遍历依赖树，计算出需要启动的所有 Unit 的列表。这个列表被称为一个“事务”。
2. 启动执行：systemd 开始执行这个事务。
3. 最大限度并行（核心）：检查事务列表中的所有 Unit。立即启动所有那些没有依赖（或依赖已满足）的 Unit，而且是同时启动（并行）。
4. 事件驱动：当一个 Unit（例如 NetworkManager.service）启动完成后，它会通知 systemd。systemd 随即更新依赖树，检查现在有哪些新的 Unit（例如 sshd.service）的依赖已经满足，然后立即并行启动所有这些新的 Unit。
5. 这个“并行启动，事件驱动”的模式会一直持续，直到最终的 default.target 所依赖的所有 Unit 都成功启动
### 关键的早期启动任务
在并行启动服务的同时，systemd 会优先处理一些非常基础的系统任务
1. 挂载文件系统：分析 /etc/fstab 文件（以及 systemd 自己的.mount 单元），将所有需要的文件系统（如 /home, /var, /boot/efi）挂载到指定位置。
2. 启动日志服务 (systemd-journald)：非常早地启动日志守护进程 systemd-journald。确保从启动初期开始的所有服务日志都能被捕获和管理。
3. 启动设备管理器 (systemd-udevd)：udevd 服务被启动，它负责监听来自内核的“uevents”（硬件插拔事件）。
### 按需启动 (套接字激活)
在启动过程中，systemd 会大量使用套接字激活技术来提高效率
1. 原理：以 SSH 服务为例。systemd 会启动 sshd.socket 单元，而不是 sshd.service。
2. 谁在监听？：systemd (PID 1) 打开 TCP 22 端口并开始监听。此时，sshd 服务进程根本没有运行，节省了内存。
3. 按需启动：当第一个 SSH 连接请求到达 22 端口时，systemd 会感知到这个连接。
4. 启动并交接：systemd 此刻才去启动 sshd.service 进程，然后将这个已经建立好的连接“递给”刚启动的 sshd 进程，让它去处理。
5. 系统启动时不必启动大量可能用不到的服务，大大加快了启动到可用状态的速度，同时也减少了空闲时的资源占用。
### 使用控制组进行进程跟踪
systemd 在启动和管理服务时，会深度利用内核的控制组功能
1. 原理：当启动一个服务（如 nginx.service）时，会先为这个服务创建一个专属的控制组。
2. 自动归类：nginx 的主进程被放入这个控制组。之后，nginx（例如 fork 出的worker 子进程）创建的所有后代进程，都会被内核自动地、强制地放入同一个控制组中。
3. 完美控制：这给了 systemd 对服务的绝对控制权。当执行 systemctl stop nginx.service 时，只需简单地向内核请求 kill 某个控制组。这保证了服务被彻底、干净地停止，不会有任何进程逃逸或残留。
### 启动完成，进入维护模式
1. 达成目标：当 default.target（例如 graphical.target）的所有依赖都已满足，systemd 就认为系统启动已完成。
2. 启动登录管理器：作为 graphical.target 的一部分，gdm.service（或lightdm.service 等）会作为最后一个主要服务被启动，它负责拉起图形界面并显示登录窗口。
3. 进入主循环：systemd 完成了启动任务，进入其事件驱动的主循环。它现在转变为一个服务管理器，在后台安静运行，等待并响应事件，例如\
（1）用户的systemctl start ...命令\
（2）某个服务的崩溃，根据配置重启\
（3）套接字激活请求\
（4）定时器（.timer单元）到期\
（5）收养并清理孤儿进程\