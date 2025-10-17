# lunux 多线程
C++11 前的库中没有提供和线程相关的类或者接口，编程时 Windows 上需调用 CreateThread 创建线程，Linux下需调用 clone 或者 pthread 线程库的接口函数 pthread_create 来创建线程
是直接调用了系统相关的API函数，编写的代码无法跨平台编译运行
1. main 函数所在的线程称为主线程，其余创建的线程称为子线程；在Linux环境下线程的本质仍然是进程
2. 进程虽然拥有相同的虚拟地址，但页目录、页表、物理页面各不相同，映射到不同的物理页面内存单元，最终访问不同物理页面\
   两个线程虽然有独立的PCB，但是共享同一个页目录、页表、物理页面，所以两个PCB共享同一个地址空间。
3. 创建线程
   ````
   int pthread_create(pthread_t *thread,const pthread_attr_t *attr,void*(*start_routine)(void*),void *arg);
   ````
   thread:指向线程标识符的指针,也可以直接传递某个 pthread_t 类型变量的地址\
   attr:设置线程的属性，一般使用默认值NULL\
   start_routine:函数指针,形参和返回值必须为void*,需要先进行强制类型转换，才能访问指针指向的数据\
   arg:运行函数的唯一传入参数\
   返回值:成功,0; 失败,返回错误号\
&emsp;&emsp;&emsp;EAGAIN：系统资源不足，无法提供创建线程所需的资源\
&emsp;&emsp;&emsp;EINVAL：传递给 pthread_create() 函数的 attr 参数无效\
&emsp;&emsp;&emsp;EPERM：attr 参数中某些属性的设置为非法操作，程序没有相关的设置权限
4. 终止线程
   ````
   pthread_exit (void *retval) 
   ````
   retval指向的数据作为返回值，也可将 retval 参数置为NULL\
   main() 结束时自动被终止，除非 main() 通过 pthread_exit() 在其之前结束
   ````
   int pthread_cancel(pthread_t thread) 
   ````
   thread为接收Cancel信号的目标线程，成功发送返回0，否则返回非零数\
   不是立刻终止,当子线程执行到一个取消点，线程才会终止\
   若线程没有取消点，可通过pthread_testcancel()设置\
5. 连接线程
   ````
   pthread_join (pthread_t thread,void ** retval) 
   ````
   与一个已终止的线程连接,回收资源。返回值：0，成功；非0，失败，返回错误号
   ````
   pthread_attr_init(&attr);
   pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
   ````
   只有创建时定义为可连接的线程才可以被连接\
   当A线程调用线程B并 pthread_join() 时，A线程会处于阻塞状态\
   一个线程只能被一个线程所连接,一般在主线程中使用\
   对detach状态的线程调用pthread_join将返回EINVAL错误
6. 分离线程
   ````
   pthread_detach (pthread_t thread)  
   ````
   分离的线程在终止的时候，会自动释放资源返回给系统\
   不能多次分离，会产生不可预测的行为\
7. 获取当前的线程的线程ID
   ````
   pthread_t pthread_self(void)
   ````
8. 比较两个线程ID是否相等
   ````
   int pthread_equal(pthread_t t1,pthread_t t2);
   ````
   返回值：相等非0，不相等0
# 互斥锁
1. 临界区\
   访问某一共享资源的代码片段，并且这段代码的执行应为原子操作。
2. 互斥锁特性\
   确保同时仅有一个线程可以访问某项共享资源，其他线程将遭遇阻塞\
   试图对已经锁定的某一互斥量再次加锁将可能阻塞线程或者报错失败\
   所有者才能给互斥量解锁。一般对每一个共享资源会使用不同的互斥量
3. 初始化互斥量
   ````
   int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr)
   ````
   mutex：需初始化的互斥量变量；attr：互斥量相关属性，通常 NULL
   restrict:C语言的修饰符，被修饰的指针不能由另外一个指针进行操作
4. 定义互斥量变量
   ````
   pthread_mutex_t mutex;
   ````
   需为全局变量,不能定义为栈上的临时变量
5. 销毁锁
   ````
   int pthread_mutex_destroy(pthread_mutex_t *mutex);
   ````
6. 加锁，阻塞的
   ````
   int pthread_mutex_lock(pthread_mutex_t *mutex);
   ````
7. 尝试加锁，非阻塞
   ````
   int  pthread_mutex_trylock(pthread_mutex_t *mutex);
   ````
   如果加锁失败，不会阻塞，会直接返回
8. 解锁
   ````
   int pthread_mutex_unlock(pthread_mutex_t *mutex);
   ````
# C11 后多线程
1. 头文件
atomic：主要声明了两个类, std::atomic 和 std::atomic_flag，另外还声明了一套 C 风格的原子类型和与 C 兼容的原子操作的函数\
thread：主要声明了 std::thread 类，另外 std::this_thread 命名空间也在该头文件中\
mutex：主要声明了与互斥量(mutex)相关的类，包括 std::mutex 系列类，std::lock_guard, std::unique_lock, 以及其他的类型和函数\
condition_variable：主要声明了与条件变量相关的类，包括std::condition_variable 和 std::condition_variable_any\
future：主要声明了std::promise,std::package_task两个Provider类，以及std::future和std::shared_future两个Future类，一些与之相关的类型和函数，std::async() 函数就声明在此头文件中