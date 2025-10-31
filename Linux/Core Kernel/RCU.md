## 核心问题
一个被频繁读取、但偶尔更新的数据结构（例如，路由表、系统中的进程列表）
### 使用读写锁
1. 读者: 在读取前，必须获取“读锁”。这涉及原子操作，在多核上会产生缓存一致性开销
2. 写者: 在更新前，必须获取“写锁”。这个写锁会完全阻塞所有新的读者和其它写者。
3. 读者的开销虽然不大，但并非为零。而写者的开销则非常大，可能导致系统卡顿
## RCU(Read-Copy-Update) 概述
1. 读者: 几乎没有开销。不需要获取任何锁，在最快的实现中，读操作的“锁”和“解锁”在编译后是完全的空操作 (no-op)。
2. 写者: 不会阻塞任何读者。读者可以继续在旧的数据上进行读操作，与此同时，写者正在创建新版本的数据。
3. 代价: 写者在更新完成后，不能马上释放旧数据的内存。必须等待一个“宽限期 (Grace Period)”，确保所有在它更新之前就开始的读操作全部完成。
4. 核心是一种空间换时间的策略：通过暂时保留旧版本的数据（空间），来换取读者近乎为零的同步开销（时间）
# 核心机制
三大机制 Read, Copy-Update, Grace Period，以链表删除举例
## Read
### 示例
```
// 1. 声明进入 RCU 读侧临界区
rcu_read_lock();
// 2. 安全地获取指针
// rcu_dereference() 确保能正确看到指针
struct my_data *p = rcu_dereference(g_head);
while (p != NULL) {
    // 3. 在临界区内，指针 p 指向的内存是绝对安全的
    do_something_with(p->data); 
    p = rcu_dereference(p->next);
}
// 4. 声明退出 RCU 读侧临界区
rcu_read_unlock();
```
## Copy-Update
写者的工作分为两部分：移除 和 回收
### 示例
假设从链表 A -> B -> C 中删除节点 B
```
// 假设 g_head 指向 A
struct my_data *node_b = find_node_b(); // 找到 B
struct my_data *node_a = find_node_a(); // 找到 B 的前驱 A

// --- 1. 复制-更新 (Copy-Update) / 移除 (Removal) ---
// 写者通常需要一个 "写锁" (如 spinlock) 来保证
// 多个写者之间是互斥的。注意：这个锁读者不需要！
spin_lock(&writer_lock);
// 核心操作：原子地将 A 的 next 指针指向 C
// list_del_rcu() 修改 A->next = C，但不会破坏 B->next 指针
list_del_rcu(&node_b->list); 
spin_unlock(&writer_lock);
// 1. 内存状态: A -> C， 同时 B -> C
// 2. 老读者: 已经拿到 B 指针，可以安全地顺着 B->next 访问 C。
// 3. 新读者: 从 g_head 开始的读者，只会看到 A -> C，永远不会访问到 B。

// --- 2. 回收 (Reclamation) ---
// B 节点已经从链表 "逻辑上" 移除了。
// 但不能马上 kfree(node_b)，因为可能有老读者还在用它。
// 必须等待所有这些 "老读者" 全部完成，这就是 "等待一个宽限期"。
// 方案A：阻塞等待 (性能较低)
synchronize_rcu();
kfree(node_b); // 宽限期结束，现在可以安全释放
// 方案B：异步回调 (高性能)
// 注册一个回调，RCU 会在宽限期结束后帮我们调用它
call_rcu(&node_b->rcu_head, free_node_b_callback);
// (free_node_b_callback 函数内部会调用 kfree(node_b))
```
## Grace Period (宽限期)
### 定义
1. 宽限期: 从写者发起（例如调用 synchronize_rcu()）开始，到所有在宽限期开始前就进入 RCU 读侧临界区的 CPU 都退出临界区的这一段时间。
2. 静止态 (Quiescent State - QS): 一个 CPU 处于可以被内核确认它不在 RCU 读侧临界区内的状态。
### RCU-sched 非抢占实现
1. rcu_read_lock() 仅仅是禁止了抢占
2. 意味如果一个 CPU 发生了上下文切换，内核就可以确定这个 CPU 已经退出了 RCU 读侧临界区
3. 在 RCU-sched 中，上下文切换、进入用户态、或进入 idle 循环都被视为一个静止态
### 宽限期的检测流程
内核不需要跟踪每一个读者，只需要知道每一个 CPU 是否都经历过至少一次 QS
1. 一个写者调用 synchronize_rcu()，这会启动一个新的宽限期
2. RCU 子系统（由 rcu_sched 内核线程管理）开始监视所有在线的 CPU
3. 使用一种高效的树形结构 (rcu_node) 来汇总状态
4. 当一个 CPU（例如 CPU 0）发生了一次上下文切换，它就报告了一个 QS
5. 当一个 rcu_node 节点下的所有子节点（代表一组 CPU）都报告了 QS，这个节点就向它的父节点报告 QS
6. 这个过程一直传递到树的根节点。当根节点收到了所有子节点的 QS 报告，就意味着所有 CPU 都至少发生了一次 QS
7. 此时，RCU 子系统宣布这个“宽限期”结束
8. synchronize_rcu() 调用返回，或者 call_rcu() 注册的回调被执行
## API
### rcu_read_lock()
1. 进入读侧临界区，RCU-sched: 编译为空操作。仅通过 preempt_disable() 禁止内核抢占。
2. RCU-preempt (可抢占): 增加一个 per-task 的嵌套计数器 (current->rcu_read_lock_nesting)
### rcu_read_unlock()
1. 退出读侧临界区，RCU-sched: 编译为空操作。仅通过 preempt_enable() 允许内核抢占
2. RCU-preempt: 减少 per-task 的嵌套计数器。
### rcu_dereference()
安全地读取 RCU 保护的指针，有两个关键作用
1. 内存屏障: 在 ARM、PowerPC 等弱序 CPU 上，它插入一个内存屏障，确保 CPU 先读取指针，再读取指针指向的内容，防止读到未初始化的数据
2. 编译器屏障: 告诉编译器不要重排访问顺序或缓存指针
### rcu_assign_pointer()
安全地发布一个新指针，也有两个关键作用
1. 内存屏障: 确保新数据的所有初始化都已完成，然后才把指针赋给全局变量。这保证了读者要么看到旧指针，要么看到一个完全初始化好的新指针。 
2. 原子赋值: 这是一个原子的指针写操作。
### synchronize_rcu()
1. 阻塞，直到一个宽限期结束。 调用它的线程会进入睡眠，直到 RCU 子系统确认所有老读者都已退出。
2. 简单易用，但会阻塞写者。适用于不频繁的操作（如模块卸载）
### call_rcu()
1. 注册一个回调函数，立即返回 (非阻塞)，高性能路径的首选
2. 写者提供一个 rcu_head 结构和回调函数。RCU 子系统在宽限期结束后，会从 softirq 上下文调用该回调函数（通常是 kfree）