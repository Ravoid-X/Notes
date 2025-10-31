# 调度器
## 基于优先级的抢占式调度
1. 默认调度策略，在 FreeRTOSConfig.h 中设置 configUSE_PREEMPTION = 1 启用。
2. 调度器在任何时刻都确保当前正在运行的任务是所有处于 就绪 状态的任务中优先级最高的那个。
3. 抢占意味着，一个正在运行的任务可能在任何时间点被中断，只要有一个更高优先级的任务变为就绪状态。
## 协作式调度
1. configUSE_PREEMPTION = 0
2. 上下文切换不会自动发生。一个任务将一直运行，直到其主动放弃 CPU 控制权。
3. 通常通过调用 taskYIELD() 函数，或者进入阻塞状态（例如调用 vTaskDelay()或等待队列消息）来实现。
4. 这种模式的响应性较差，但对于共享资源的处理逻辑可能更简单，因为任务不会在非预期的代码点被中断。
## 同优先级任务的时间片轮转
1. configUSE_TIME_SLICING = 1（默认启用）
2. 在每个系统节拍中断时，如果当前运行任务的时间片已用尽，并且有其他同优先级的任务处于就绪状态，调度器就会触发一次上下文切换，将 CPU 交给下一个同优先级的任务。
## 上下文切换
1. ARM Cortex-M 等架构上，FreeRTOS 巧妙地利用了一个低优先级的软件中断（如 PendSV）来执行实际的上下文切换操作。
2. 好处是上下文切换的执行不会阻塞更高优先级的硬件中断，保证了系统的实时响应能力。
## 编程要求
1. FreeRTOS 调度器的严格、基于优先级的特性，实际上强制开发者采用事件驱动的编程模型，以避免“任务饥饿”并构建高效的系统。
2. 任务饥饿：如果一个高优先级任务在一个紧凑的循环中轮询某个事件，这个任务将永远处于就绪状态。因为它从不阻塞，调度器永远不会切换到任何低优先级的任务。
3. 符合 FreeRTOS 惯例的方法是让任务在一个同步原语（如信号量或队列）上阻塞，直到事件发生.
# 任务管理
## 任务
1. 任务是基本的调度单元，实现为一个永不返回的 C 函数，通常包含一个无限循环。
2. 如果任务需要终止，必须显式调用 vTaskDelete(NULL) 来安全地销毁自身。
3. 每个任务都拥有自己独立的栈空间，用于存储局部变量、函数调用信息以及在被切换出时保存的上下文。
### 任务状态
1. 运行态（Running）：当前正在 CPU 上执行。在单核处理器上，任何时刻最多只有一个任务处于此状态。
2. 就绪态（Ready）：已经准备好运行，但因为有更高优先级的任务正在运行，所以等待 CPU 时间。
3. 阻塞态（Blocked）：正在等待某个事件。这个事件可以是时间性的（如 vTaskDelay() 指定的延时到期）或同步性的（如等待队列中的数据）。处于阻塞态的任务不消耗任何 CPU 时间。
4. 挂起态（Suspended）：任务被显式地置于休眠状态（通过调用 vTaskSuspend()），并且不会被调度器考虑。只有通过显式调用 vTaskResume() 才能将其唤醒并移回就绪态。
## 任务创建
### 动态创建 (xTaskCreate)
```
BaseType_t xTaskCreate(TaskFunction_t pvTaskCode, const char * const pcName, const configSTACK_DEPTH_TYPE uxStackDepth, void *pvParameters, UBaseType_t uxPriority, TaskHandle_t *pxCreatedTask)
```
1. 从 FreeRTOS 堆中动态分配任务控制块（TCB）和任务栈所需的内存。因此，必须将 configSUPPORT_DYNAMIC_ALLOCATION 设置为 1
2. pvTaskCode：指向任务函数的指针
3. pcName：任务的文本名称，主要用于调试
4. uxStackDepth：任务栈的大小，单位是“字”，而不是字节。在32位架构上，如果此值为128，则分配的栈大小为 $128 \times 4 = 512$ 字节。
5. pvParameters：一个 void 指针，用于向任务函数传递参数。
6. uxPriority：任务的优先级，范围从 tskIDLE_PRIORITY（0）到 configMAX_PRIORITIES - 1。
7. pxCreatedTask：一个可选的输出参数，用于返回所创建任务的句柄（TaskHandle_t），后续可通过此句柄操作任务。
8. 返回值：成功时返回 pdPASS，如果堆内存不足则返回 errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY
### 静态创建 (xTaskCreateStatic)
```
TaskHandle_t xTaskCreateStatic(TaskFunction_t pxTaskCode,..., StackType_t * const puxStackBuffer, StaticTask_t * const pxTaskBuffer)
```
1. 要求应用程序开发者自己提供 TCB 和栈所需的内存，通常以静态或全局数组的形式。使用此函数需要将 configSUPPORT_STATIC_ALLOCATION 设置为 1 
```
// 定义任务栈和TCB的存储空间
#define BLINK_TASK_STACK_SIZE 128
static StackType_t xBlinkTaskStack;
static StaticTask_t xBlinkTaskTCB;

void main(void) {
    //...
    xTaskCreateStatic(
        vBlinkTask,             // 任务函数
        "Blink",                // 任务名称
        BLINK_TASK_STACK_SIZE,  // 栈大小（字）
        NULL,                   // 传递给任务的参数
        tskIDLE_PRIORITY + 1,   // 任务优先级
        xBlinkTaskStack,        // 任务栈的数组
        &xBlinkTaskTCB          // 任务TCB的变量
    );
    //...
}
```
### 比较
1. 动态和静态任务创建 API 并存并非冗余，反映了嵌入式系统设计中 运行时灵活性 与 编译时确定性 的权衡。
2. 动态分配方便灵活，非常适合任务数量可能变化的应用程序或快速原型开发。
3. 动态分配依赖于堆，这引入了不确定性（分配时间可变）和运行时失败的风险（内存不足、碎片化）。
## 任务控制 API
### 任务延时
1. vTaskDelay(xTicksToDelay)：将当前任务置于阻塞态，持续一个相对的时间长度（以系统节拍数为单位）。实现任务暂停、让出 CPU 的最有效方式。
2. vTaskDelayUntil(&xLastWakeTime, xTimeIncrement)：将任务阻塞至一个绝对的节拍计数值。非常适合创建固定频率的周期性任务，因为会自动补偿任务执行本身所花费的时间，从而避免了周期性延时累积的误差。
```
void vPeriodicTask(void *pvParameters) {
    TickType_t xLastWakeTime;
    const TickType_t xFrequency = pdMS_TO_TICKS(100); // 100ms周期
    xLastWakeTime = xTaskGetTickCount(); // 初始化

    for (;;) {
        // 等待下一个周期
        vTaskDelayUntil(&xLastWakeTime, xFrequency);
        // 执行周期性任务代码...
    }
}
```
### 任务挂起与恢复
1. vTaskSuspend(xTaskToSuspend)：挂起指定的任务。如果参数为 NULL，则挂起调用该函数的任务自身。
2. vTaskResume(xTaskToResume)：将一个处于挂起态的任务恢复到就绪态
```
//一个按键检测任务，根据按键事件来挂起或恢复一个 LED 闪烁任务
TaskHandle_t xLedTaskHandle = NULL;
void vButtonCheckTask(void *pvParameters) {
    bool bLedTaskSuspended = false;
    for (;;) {
        if (button_is_pressed()) {
            if (bLedTaskSuspended) {
                vTaskResume(xLedTaskHandle);
                bLedTaskSuspended = false;
            } else {
                vTaskSuspend(xLedTaskHandle);
                bLedTaskSuspended = true;
            }
            vTaskDelay(pdMS_TO_TICKS(200)); // 按键去抖
        }
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}
```
### 任务删除
1. vTaskDelete(xTaskToDelete)：将一个任务从内核管理中彻底移除。如果参数为 NULL，则删除调用任务自身。
2. vTaskDelete() 只会释放由内核为该任务分配的内存（TCB和栈）。
3. 任务本身通过 pvPortMalloc() 或其他方式动态分配的任何内存，都必须在任务被删除前由应用程序代码负责释放，否则将导致内存泄漏。
4. 当一个任务删除自身时，其栈和 TCB 的内存回收工作会交由空闲任务来完成。意味着如果应用程序频繁地自删除任务，必须保证空闲任务有足够的 CPU 时间运行。
```
void vDataProcessorTask(void *pvParameters) {
    uint8_t *pucBuffer = (uint8_t *)pvPortMalloc(1024);
    if (pucBuffer!= NULL) {
        //... 使用缓冲区处理数据...

        // 在任务结束前，释放所有已分配的资源
        vPortFree(pucBuffer);
    }

    // 所有资源已清理，现在可以安全地删除任务自身
    vTaskDelete(NULL);
}
```