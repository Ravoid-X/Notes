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