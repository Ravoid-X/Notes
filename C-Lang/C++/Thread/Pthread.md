
## 创建线程
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
## 终止线程
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
## 连接线程
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
## 分离线程
   ````
   pthread_detach (pthread_t thread)  
   ````
   分离的线程在终止的时候，会自动释放资源返回给系统\
   不能多次分离，会产生不可预测的行为\
## 获取当前的线程的线程ID
   ````
   pthread_t pthread_self(void)
   ````
## 比较两个线程ID是否相等
   ````
   int pthread_equal(pthread_t t1,pthread_t t2);
   ````
   返回值：相等非0，不相等0