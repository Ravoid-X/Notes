## 队列
最主要的任务间通信方式，是一种线程安全的先进先出缓冲区，可用于在任务之间或中断服务程序（ISR）与任务之间传递消息。
### 按值复制
1. 队列的一个关键设计决策。当数据被发送到队列时，是数据本身被逐字节地复制到队列的存储区，而不是仅仅传递一个指向数据的引用。
2. 极大地简化了内存管理。发送方和接收方无需协调缓冲区的生命周期和所有权，发送方在发送数据后可以立即重用其变量或缓冲区。
3. 对于体积较大的数据，可以在队列中传递指向数据的指针来实现按引用传递的效果。但此时，发送方和接收方必须协同管理该数据缓冲区的生命周期。
### 阻塞机制
1. 任务可以带超时地阻塞在队列上。
2. 当一个任务试图从空队列中读取数据（xQueueReceive）或向满队列中写入数据（xQueueSend）时，会自动进入阻塞态，不消耗任何 CPU 时间，直到条件满足或超时。
## 队列使用
### 创建
```C++
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize)
```
1. uxQueueLength 是队列能容纳的最大项目数
2. uxItemSize 是每个项目的大小（字节）
### 发送/接收
1. 使用 xQueueSend()（或 xQueueSendToFront()）发送数据
2. 使用 xQueueReceive() 接收数据
### ISR中使用
在中断服务程序中，必须使用对应的ISR安全版本：xQueueSendFromISR() 和 xQueueReceiveFromISR()
### 示例1：传递简单数据
一个生产者任务生成整数，并通过队列发送给一个消费者任务进行打印
```C++
QueueHandle_t xIntegerQueue;
void vProducerTask(void *pvParameters) {
    int32_t lValueToSend = 0;
    for (;;) {
        xQueueSend(xIntegerQueue, &lValueToSend, portMAX_DELAY);
        lValueToSend++;
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
void vConsumerTask(void *pvParameters) {
    int32_t lReceivedValue;
    for (;;) {
        if (xQueueReceive(xIntegerQueue, &lReceivedValue, portMAX_DELAY) == pdPASS) {
            printf("Received = %ld\n", lReceivedValue);
        }
    }
}
void main(void) {
    xIntegerQueue = xQueueCreate(10, sizeof(int32_t));
    // 创建任务...
    vTaskStartScheduler();
}
```
### 示例2：传递结构体
一个传感器任务将采集到的数据打包成结构体，通过队列发送给数据处理任务
```C++
typedef struct {
    float temperature;
    float humidity;
} SensorData_t;
QueueHandle_t xSensorQueue;
void vSensorTask(void *pvParameters) {
    SensorData_t xData;
    for (;;) {
        xData.temperature = read_temperature();
        xData.humidity = read_humidity();
        xQueueSend(xSensorQueue, &xData, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```
## 信号量
### 二进制信号量
1. 可以看作是长度为1的队列，只有空和满两种状态。主要用于任务间的同步或ISR与任务间的同步。
2. 最经典的应用场景是：一个任务通过获取信号量来等待一个事件，而一个 ISR 在事件发生时通过给予信号量来通知该任务
3. 示例：一个任务阻塞等待信号量，一个 GPIO 中断服务程序在按键按下时释放该信号量，从而唤醒任务处理按键事件
```C++
SemaphoreHandle_t xButtonSemaphore;

void vButtonHandlerTask(void *pvParameters) {
    for (;;) {
        if (xSemaphoreTake(xButtonSemaphore, portMAX_DELAY) == pdTRUE) {
            printf("Button pressed!\n");
        }
    }
}
void EXTI0_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    // 清除中断标志...
    xSemaphoreGiveFromISR(xButtonSemaphore, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```
### 计数信号量
1. 可以看作是长度大于 1 的队列，其计数值可以大于 1
2. 主要有两种用途\
（1）事件计数，ISR 每发生一次事件就给予一次信号量，任务每处理一个事件就获取一次，计数值代表了未处理的事件数量。\
（2）资源管理，计数值代表可用资源的数量，任务在访问资源前必须先获取信号量，使用完毕后给予回去 。
## 互斥锁
能上类似于二进制信号量，但其设计初衷是用于保护共享资源
### 优先级反转
一个高优先级任务因为等待一个被低优先级任务持有的资源而被迫阻塞，同时一个中等优先级的任务抢占了那个低优先级任务，导致高优先级任务的等待时间被无限延长。
### 优先级继承
互斥锁与二进制信号量最本质的区别，该机制用于缓解“优先级反转”问题
1. 当一个高优先级任务试图获取一个由低优先级任务持有的互斥锁时，调度器会自动、临时地将该低优先级任务的优先级提升到与高优先级任务相同。
2. 确保了持有锁的低优先级任务能尽快运行并释放资源，从而最大限度地减少高优先级任务的阻塞时间。
### 示例
两个任务竞争使用同一个串口打印信息。使用互斥锁确保每次只有一个任务可以访问串口，防止输出信息交错混乱.
```C++
SemaphoreHandle_t xUartMutex;
void vTask1(void *pvParameters) {
    for (;;) {
        if (xSemaphoreTake(xUartMutex, portMAX_DELAY) == pdTRUE) {
            printf("This is message from Task 1.\n");
            xSemaphoreGive(xUartMutex);
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
// vTask2 类似
```
## 事件组
1. 一个集合，其中包含多个独立的二进制标志位，被存储在一个整型变量中。任务可以等待这些标志位的特定组合变为 1
2. 相比使用多个二进制信号量来实现多事件同步，事件组在内存效率上更高
3. 任务可以等待指定的任意一个事件位被设置（逻辑或），或者等待所有指定的事件位都被设置（逻辑与）
### 复杂同步
1. 创建：EventGroupHandle_t xEventGroupCreate(void)   
2. 设置事件位：在任务中调用 xEventGroupSetBits()，在 ISR 中调用 xEventGroupSetBitsFromISR()   
3. 等待事件位：xEventGroupWaitBits()，通过参数可以指定等待逻辑（AND/OR）、是否在退出时清除事件位以及超时时间 
### 示例
一个应用任务需要等待网络连接成功（事件位 0）和 NTP 时间同步完成（事件位 1）后才能开始工作
```C++
#define BIT_0_WIFI_CONNECTED (1 << 0)
#define BIT_1_NTP_SYNCED     (1 << 1)

EventGroupHandle_t xAppEventGroup;

void vAppTask(void *pvParameters) {
    EventBits_t uxBits;
    const EventBits_t uxBitsToWaitFor = (BIT_0_WIFI_CONNECTED | BIT_1_NTP_SYNCED);
    uxBits = xEventGroupWaitBits(
        xAppEventGroup,
        uxBitsToWaitFor,
        pdTRUE,        // 退出时清除这些位
        pdTRUE,        // 等待所有位都被设置 (AND)
        portMAX_DELAY
    );
    if ((uxBits & uxBitsToWaitFor) == uxBitsToWaitFor) {
        // WiFi已连接且NTP已同步，开始主应用逻辑
    }
}
```
## 任务通知
1. 一种直接向特定任务发送信号的机制，无需创建队列、信号量等中间对象
2. 每个任务内部都有一个 32 位的通知值。发送通知可以解锁接收任务，并选择性地更新其通知值。
3. 与其它 IPC 机制相比，任务通知在速度和 RAM 占用上都有显著优势。
4. 局限：只能用于向一个特定的、已知的任务发送信号，不适用于多接收者或广播场景。