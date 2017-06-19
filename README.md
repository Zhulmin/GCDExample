# GCDExample



##GCD 串行并发

dispatch_semaphore和dispatch_group可以实现自定义的并发操作, 在指定位置出栈

dispatch_semaphore设置信号量为1, 执行完耗时操作发送信号

dispatch_group也可以实现, 但是每次都要创建一个group



/**
 Group task
 */
- (void)groupTask {
    
    [self groupTaskWithTackId:0];
    
    // do other task on main thread
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"do task on main thread");
    });
}


- (void)groupTaskWithTackId:(NSInteger)taskId {
    
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_enter(group);
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            
            // do time-consuming operation
            unsigned int time = arc4random_uniform(2) + 1;
            sleep(time);
            
            NSLog(@"do group task %zd", taskId);
            
            dispatch_group_leave(group);
        });
    });
    
    dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
        [self groupTaskWithTackId:taskId + 1];
    });
    
}

/**
 Do async serial task
 */
- (void)barrierTask {
    
    
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.GCDDemo.concurrentQueue",DISPATCH_QUEUE_CONCURRENT);
    
    for (int i = 0; i < 9; i = i + 3) {
        dispatch_async(concurrentQueue, ^{
            [NSThread sleepForTimeInterval:i ];
            NSLog(@"download task %d successed!",i);
        });
    }
    
    dispatch_barrier_async(concurrentQueue, ^{
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"do barrier async task!");
        });
        
        //        [NSThread sleepForTimeInterval:3.0];
        //        NSLog(@"barrier blcok success!");
    });
    
    
    dispatch_async(concurrentQueue, ^{
        NSLog(@"async blcok success!");
    });
    
}


/**
 dispatch_semaphore create a sync singal
 */
- (void)signalTask {
    
    
    dispatch_queue_t serial = dispatch_queue_create("serial.queue", NULL);
    
    dispatch_async(serial, ^{
        
        dispatch_queue_t global = dispatch_get_global_queue(0, 0);
        
        dispatch_semaphore_t signal = dispatch_semaphore_create(1);
        
        self.signal = signal;
        
        dispatch_semaphore_wait(signal, DISPATCH_TIME_FOREVER);
        
        for (NSInteger i = 0; i < 100; i++) {
            
            dispatch_async(global, ^{
                
                // do time-consuming operation
                sleep(2);
                //                NSLog(@"%@", [NSThread currentThread]);
                
                // send signal when operation finished
                dispatch_semaphore_signal(signal);
            });
            
            dispatch_semaphore_wait(signal, DISPATCH_TIME_FOREVER);
            NSLog(@"signal:  %zd", i);
            
        }
    });
    
    // do other task on main thread
    for (NSInteger j = 1; j < 100; j++) {
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(j * 0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSLog(@"%zd", j);
        });
    }
    
}
