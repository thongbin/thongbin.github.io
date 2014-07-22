---
layout: post
title: "Grand Central Dispatch(GCD)学习"
date: 2014-07-21 14:18:09 +0800
comments: true
categories: iOS
---
Grand Central Dispatch 简称GCD，苹果在Mac OS X 10.6 ，iOS 4平台首次发布，后续平台也可用。关于GCD的更多信息，参考[官方文档](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html#//apple_ref/doc/uid/TP40008079-CH1-SW1)。
<br >

###特性
* 基于队列工作 dispatch queue，严格按照FIFO（first in first out）
* 平行排队特定任务
* 利用任何可用核心资源（多核处理器）处理任务
* 任务：函数（function）、block

<br >
##Functions by Task
<!--more-->
#### Main dispatch queue
* 主线程 queue ，一般用来更新ui，串行地在主线程上执行任务
* main thread 有系统自动创建，并会自动关联上你的应用的主线程，若想在应用内使用，可使用以下三种方式：
<br >
```	objc
	 调用 dispatch_main
	 
	 调用 UIApplicationMain(iOS)或者NSApplicationMain(OS X)
	 
	 使用 CFRunLoopRef
```

####Concurrent - Global dispatch queue
* 并行队列，任务从队列被列出是按覅佛规则的，但是回并行执行，执行完成的顺序是随机的。适合并行处理数量庞大的任务，GCD可以自动生成四种此类并行队列，只通过各自的优先级不同去区别开来，当然你可以自己根据需要定义自己的并行队列。因为此队列对于你得应用程序来说是全局的，所以你并不需要关心它的引用计数，即使给它retain或者release，都会被忽略。
<br >
``` objc
	dispatch_get_global_queue(<#dispatch_queue_priority_t priority#>, <#unsigned long flags#>)
	
	DISPATCH_QUEUE_PRIORITY_HIGH 2
	
	DISPATCH_QUEUE_PRIORITY_DEFAULT 0
	
	DISPATCH_QUEUE_PRIORITY_LOW (-2)
	
	DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
	
```

####Serial - private dispatch queues
* 通常用于访问特定资源和数据，多个serial queue之间同步并发执行，串行fifo队列，适合用于按指定顺序执行的任务，保持线程安全，串行同步安全地访问资源，在应用程序里面，必须明确指定串行队列，可以已根据需要创建足够多的，但不要用serial queue 代替 concurrent queue 处理任务庞大的任务。
<br >
``` objc
	dispatch_queue_create(<#const char *label#>, <#dispatch_queue_attr_t attr#>)
	
```

<br >
##使用
###dispatch_once
* dispatch_once执行一个区块对象，在整个应用的生命周期只执行一次，非常适合用于已单列模式创建全局的对象。
<br >
``` objc
Executes a block object once and only once for the lifetime of an application.//官方解释

   void dispatch_once(
   dispatch_once_t *predicate,
   dispatch_block_t block);
   
//   example
	+(instancetype)shareDefaultManager
	{
    static TBReachabilityManager *_tbDefaultManager = nil;
    static dispatch_once_t once_t;
    dispatch_once(&once_t, ^(){
         _tbDefaultManager = [[self alloc]init];
    });
    
    return _tbDefaultManager;
	}
```

###dispatch_async
* 提交一个block到一个异步执行的队列，执行完成后马上返回
<br >
``` objc
Submits a block for asynchronous execution on a dispatch queue and returns immediately.

   void dispatch_async(
   dispatch_queue_t queue,
   dispatch_block_t block);
```
##### queue
* block提交得目标队列，在block执行完成之前，系统会保留这个queue，值不能为Null
<br >

##### block
* 队列执行的目标block，方法会自动执行Block_copy和Block_release，值不能为Null

提交bolck到dispatch_queue最基本的方法，队列决定block执行的方式，串行或者并行，当然了，是异步的。
####常用的方法
我们在处理一些耗时操作时，比如网络读取数据、预服务器交互、数据库读写、io等，有时执行时间过长会导致主界面有卡顿的甚至卡死的情况出现，当然这种情况可以使用NSThread、NSOperation解决，但使用dispatch_async更简单：
``` objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // 耗时的操作，读取数据、数据处理等操作
    dispatch_async(dispatch_get_main_queue(), ^{
        // 更新界面，dispatch_get_main_queue()是切换回主线程队列的方法
    });
});

```

###dispatch_group_async
* 以组的方式关联一组block，等block都执行完成，就发起完成的通知，`group里面的block是并行执行的`。
<br >
```objc
Submits a block to a dispatch queue and associates the block with the specified dispatch group.

   void dispatch_group_async(
   dispatch_group_t group,
   dispatch_queue_t queue,
   dispatch_block_t block);
```

####使用例子
```
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:1];
        NSLog(@"group1");
    });
    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:2];
        NSLog(@"group2");
    });
    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:3];
        NSLog(@"group3");
    });
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"updateUi");
    });
    dispatch_release(group);//释放group
```
这里面最后还调用了dispatch_group_notify，因为前面提交的block都被关联到同一个group里面，当group里面的block都执行完成后，就需要调用dispatch_group_notify做收尾。
<br >
###dispatch_apply
* 多次执行传入的block
<br >
```objc
Submits a block to a dispatch queue for multiple invocations.

   void dispatch_apply(
   size_t iterations,
   dispatch_queue_t queue,
   void (^block)(
   size_t));
```

```objc
dispatch_apply(10, global, ^(size_t index) {
    // 执行10次
});
```

