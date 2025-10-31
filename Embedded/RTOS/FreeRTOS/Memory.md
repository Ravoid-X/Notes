## 静态/动态内存分配
### 动态分配
1. 在 RTOS 对象创建 API 内部调用 pvPortMalloc() 来从 FreeRTOS 堆中分配内存
2. 这种方式灵活，允许在运行时创建和删除对象，但可能导致堆碎片化和不确定的执行时间。
### 静态分配
1. 由应用程序开发者在编译时提供内存缓冲区
2. 保证了对象创建总能成功，允许将对象放置在特定的内存地址，并且是许多禁止动态内存分配的安全关键型系统的必备选择。
## 堆
FreeRTOS 提供了 5 种堆内存管理方案（heap_1 到 heap_5），以替代标准 C 库中通常非线程安全、不可重入或在嵌入式系统中不可用的 malloc() 和 free()
### heap_1.c
1. 最简单的方案，只分配内存，不允许释放。行为是完全确定的，不会产生内存碎片。不支持 vPortFree()
2. 适用于那些在系统启动时一次性创建所有 RTOS 对象，并且在运行期间不再删除它们的应用程序。
### heap_2.c
1. 允许释放内存块，但不会合并相邻的空闲块。因此，在反复分配和释放不同大小的内存块时，容易产生严重的内存碎片。
2. 采用“最佳适配”算法，目前已被视为遗留方案，推荐使用 heap_4 替代。
### heap_3.c
1. 一个简单的线程安全包装器，直接调用标准 C 库的 malloc() 和 free()
2. 如果目标工具链提供了可靠的堆实现，这是一个便捷的选择。
3. 但会增加代码体积，并且其行为（执行时间、碎片情况）取决于底层库的实现，通常是不确定的。
### heap_4.c
1. 最常用和推荐的通用方案，允许内存的分配和释放。
2. 实现了空闲块合并算法，可以有效地防止内存碎片化，采用“首次适配”算法
### heap_5.c
1. 在 heap_4 的基础上进行了扩展，允许将堆内存分布在多个不连续的物理内存区域。
2. 这对于具有多种不同类型 RAM（如快速的片上 SRAM 和较慢的外部 SDRAM）的复杂微控制器非常有用
## pvPortMalloc()/vPortFree()
1. FreeRTOS 提供的底层的、线程安全的动态内存分配和释放接口，内核自身也使用它们，应用程序可以直接调用它们来动态管理内存。
2. 始终检查 pvPortMalloc() 的返回值是否为 NULL，以处理内存分配失败的情况。
3. 确保每次 pvPortMalloc() 调用都有对应的 vPortFree() 调用，以避免内存泄漏。
4. 避免在中断服务程序中调用这些函数
### 示例
一个任务动态分配一个缓冲区用于处理接收到的消息，处理完毕后释放该缓冲区，演示了正确的用法和错误检查
```
void vProcessMessageTask(void *pvParameters) {
    char *pcReceivedString;
    size_t xStringLength;
    for (;;) {
        // 等待从队列中接收一个指向字符串的指针
        if (xQueueReceive(xStringPointerQueue, &pcReceivedString, portMAX_DELAY) == pdPASS) {
            // 动态分配一个本地缓冲区来处理字符串
            xStringLength = strlen(pcReceivedString) + 1;
            char *pcLocalBuffer = (char *)pvPortMalloc(xStringLength);
            if (pcLocalBuffer!= NULL) {
                // 复制并处理字符串
                strcpy(pcLocalBuffer, pcReceivedString);
                //... 处理 pcLocalBuffer...
                printf("Processed: %s\n", pcLocalBuffer);
                // 处理完毕，释放本地缓冲区
                vPortFree(pcLocalBuffer);
            } else {
                // 内存分配失败处理
                printf("Error: Failed to allocate memory for processing.\n");
            }
            // 释放原始的字符串缓冲区（假设它也是动态分配的）
            vPortFree(pcReceivedString);
        }
    }
}
```