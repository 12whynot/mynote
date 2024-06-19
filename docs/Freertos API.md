- FreeRTOS  
    
    - 任务状态  
        
        - 运行态  
            
        
        - 就绪态  
            
        
        - 阻塞 态  
            
        
        - 挂起态  
            
    
    - 任务相关API  
        
        - 任务创建（动态方法xTaskCreate()、静态方法xTaskCreateStatic()）  
            
        
        - 任务删除（xTaskDelete()）  
            
        
        - 查询任务优先级(uxTaskPriorityGet())  
            
        
        - 改变任务优先级（vTaskPrioritySet()）  
            
    
    - 延时函数API  
        
        - 相对延时（vTaskDelay()）  
            
        
        - 绝对延时（vTaskDelayUntil()）  
            
    
    - 队列相关API  
        
        - 队列创建（xQueueCreate()）  
            
        
        - 任务级入队函数  
            
            - 后向入队（xQueueSend()、xQueueSendToBack()）  
                
            
            - 前向入队（xQueuendSendToFront()）  
                
            
            - 带复写功能入队（xQueueOverwrite()）  
                
        
        - 中断级入队函数  
            
            - xQueueSendFromISR()  
                
            
            - xQueueSendToBackFromISR()  
                
            
            - xQueueSendToFrontFromISR()  
                
            
            - xQueueOverwriteFromISR()  
                
        
        - 任务级出队函数  
            
            - 读取队列信息，读取后删除（xQueueReceive()）  
                
            
            - 读取队列信息，读取后不删除（xQueuePeek()）  
                
        
        - 中断级出队函数  
            
            - xQueueReceviceFromISR()  
                
            
            - xQueuePeekFromISR()  
                
    
    - 信号量相关API  
        
        - 创建二值信号量（BinarySemaphore）  
            
            - 静态创建二值信号量（xSemaphoreCreate()）  
                
            
            - 动态创建二值信号量（xSemaphoreStatic()）  
                
        
        - 释放信号量  
            
            - 任务级释放（xSemaphoreGive()）  
                
            
            - 中断级释放（xSemaphoreGiveFromISR()）  
                
        
        - 获取信号量  
            
            - 任务级获取（xSemaphoreTake()）  
                
            
            - 中断级获取（xSemaphoreTakeFromISR()）  
                
        
        - 创建计数型信号量  
            
            - 动态创建（xSemaphoreCreateCounting()）  
                
            
            - 静态创建（xSemaphoreCreateCountingStatic()）  
                
        
        - 创建互斥信号量  
            
            - 动态创建（xSemaphoreCreateMutex()）  
                
            
            - 静态创建（xSemaphoreCreateMutexStatic()）  
                
        
        - 创建递归互斥信号量  
            
            - 动态创建（xSemaphoreCreateRecuisiveMutex()）  
                
            
            - 静态创建(xSemaphoreCreateRecursiveMutexStatic())  
                
    
    - 软件定时器相关API  
        
        - 创建定时器  
            
            - 动态创建（xTimerCreate()）  
                
            
            - 静态创建（xTimerCreateStatic()）  
                
        
        - 复位定时器  
            
            - 任务级复位（xTimerReset()）  
                
            
            - 中断级复位（xTimerResetFromISR()）  
                
        
        - 开启定时器  
            
            - 任务级开启（xTimerStart()）  
                
            
            - 中断级开启(xTimerStartFromISR())  
                
        
        - 停止定时器  
            
            - 任务级停止（xTimerStop()）  
                
            
            - 中断级停止(xTimerStopFromISR())  
                
    
    - 事件标志组相关API  
        
        - 创建事件标志组  
            
            - 动态创建（xEventGroupCreate()）  
                
            
            - 静态创建（xEventGroupCreateStatic()）  
                
        
        - 设置事件位  
            
            - 任务级清零（xEventGroupClearBits()）  
                
            
            - 任务级置位（xEventGroupSetBits()）  
                
            
            - 中断级清零（xEventGroupClearBitsFromISR()）  
                
            
            - 中断级置位（xEventGroupSetBitsFromISR()）  
                
        
        - 获取事件标志组值  
            
            - 任务级获取（xEventGroupGetBits()）  
                
            
            - 中断级获取（xEventGroupGetBitsFromISR()）  
                
        
        - 等待指定事件标志组  
            - xEventGroupWaitBits()  
                
    
    - 任务通知  
        -