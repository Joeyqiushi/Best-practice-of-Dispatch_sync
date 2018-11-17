# 如何安全使用dispatch_sync

## 概述
iOS开发者在与线程打交道的方式中，使用最多的应该就是GCD框架了，没有之一。GCD将繁琐的线程抽象为了一个个队列，让开发者极易理解和使用。但其实队列的底层，依然是利用线程实现的，同样会有死锁的问题。本文将探讨如何规避`disptach_sync`接口引入的死锁问题。


## GCD基础

GCD最基础的两个接口
```
dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```
第一个参数`queue`为队列对象，第二个参数`block`为block对象。这两个接口可以将任务`block`扔到队列`queue`中去执行。

开发者使用最频繁的，就是在子线程环境下，需要做UI更新时，我们可以将任务扔到主线程去执行，
```
dispatch_sync(dispatch_get_main_queue(), block);
dispatch_async(dispatch_get_main_queue(), block);
```
而`dispatch_sync(dispatch_get_main_queue(), block)`有可能引入死锁的问题。

##### async _VS._ sync

`disptach_async`是异步扔一个`block`到`queue`中，即扔完我就不管了，继续执行我的下一行代码。实际上当下一行代码执行时，这个`block`还未执行，只是入了队列`queue`，`queue`会排队来执行这个`block`。

而`disptach_sync`则是同步扔一个`block`到`queue`中，即扔了我就等着，等到`queue`排队把这个`block`执行完了之后，才继续执行下一行代码。


## 为什么要使用sync
`disptach_sync`主要用于代码上下文对时序有强要求的场景。简单点说，就是下一行代码的执行，依赖于上一行代码的结果。例如说，我们需要在子线程中读取一个`image`对象，使用接口`[UIImage imageNamed:]`，但`imageNamed:`实际上在iOS9以后才是线程安全的，iOS9之前都需要在主线程获取。所以，我们需要从子线程切换到主线程获取`image`，然后再切回子线程拿到这个`image`，

    // ...currently in a subthread
    __block UIImage *image;
    dispatch_sync_on_main_queue(^{
        image = [UIImage imageNamed:@"Resource/img"];
    });
    attachment.image = image;

这里我们必须使用`sync`。

## 为什么会死锁

假设当前我们的代码正在`queue0`中执行。然后我们调用`disptach_sync`将一个任务`block1`扔到`queue0`中执行，
```
// ... currently in queue0 or queue0's corresponding thread.
dispatch_sync(queue0, block1);
```
这时，`dispatch_sync `将等待`queue0`排队执行完`block1`，然后才能继续执行下一行代码。But，当前代码执行的环境也是`queue0`。假设当前执行的任务为`block0`。也就是说，`block0`在执行到一半时，需要等到自己的下一个任务`block1`执行完，自己才能继续执行。而`block1`排队在后面，需要等`block0`执行完才能执行。这时死锁就产生了，`block0`和`block1`互相等待执行，当前线程就卡死在`dispatch_sync `这行代码处。

我们发现的卡死问题，一般都是主线程死锁。一种较为常见的情况是，本身就已经在主线程了，还同步向主线程扔了一个任务：
```
// ... currently in the main thread
dispatch_sync(dispatch_get_main_queue(), block);
```

## 安全方法

YYKit中提供了一个同步扔任务到主线程的安全方法：

    /**
     Submits a block for execution on a main queue and waits until the block completes.
    */
    static inline void dispatch_sync_on_main_queue(void (^block)()) {
        if (pthread_main_np()) {
            block();
        } else {
            dispatch_sync(dispatch_get_main_queue(), block);
        }
    }

其方式就是在扔任务给主线程之前，先检查当前线程是否已经是主线程，如果是，就不用调用GCD的队列调度接口`dispatch_sync`了，直接执行即可；如果不是主线程，那么调用GCD的`dispatch_sync`也不会卡死。

但事实上并不是这样的，`dispatch_sync_on_main_queue`也可能会卡死，**这个安全接口并不安全**。这个接口只能保证两个`block`之间不因互相等待而死锁。多于两个`block`的互相依赖就束手无策了。

举个例子，假设`queue0`是一个子线程的队列：
```
/* block0 */
// ... currently in the main thread.
dispatch_sync(queue0, ^{
    /* block1 */
    // ... currently in queue0's corresponding subthread.
    dispatch_sync_on_main_queue(^{
        /* block2 */
    });
});
```
在上述代码中，`block0`正在主线程中执行，并且同步等待子线程执行完`block1`。`block1`又同步等待主线程执行完`block2`。而当前主线程正在执行`block0`，即`block2`的执行需要等到`block0`执行完。这样就成了`block0`-->`block1`-->`block2`-->`block0`...这样一个循环等待，即死锁。由于`block1`的环境是子线程，所以安全API的线程判断不起任何作用。

另举一个例子：
```
/* block0 */
// ... currently in the main thread.
[[NSNotificationCenter defaultCenter] postNotificationName:@"aNotification" object:nil];
    
// ... in another context
[[NSNotificationCenter defaultCenter] addObserverForName:@"aNotification"
                                                  object:nil
                                                   queue:queue0
                                              usingBlock:^(NSNotification * _Nonnull note) {
                                                  /* block1 */
                                                  // ... currently in queue0's corresponding subthread.
                                                  dispatch_sync_on_main_queue(^{
                                                      /* block2 */
                                                  });
                                              }];
```

由于通知`NSNotification`的执行是同步的，这里会出现和上一例一样的死锁情况：`block0`-->`block1`-->`block2`-->`block0`...

---

## 如何定位死锁问题

##### 1.死锁监测和堆栈上报机制
要定位死锁的问题，我们需要知道在哪一行代码上死锁了，以及为什么会出现死锁。通常只要知道哪一行代码死锁了，我们就能通过代码分析出问题所在了。所以，如果死锁的时候，我们能够把堆栈上报上来，就能知道哪一行代码死锁了。这里需要有完善的**死锁监测和堆栈上报机制**。

##### 2.打印日志
如果暂时没有人力或者技术支撑你去搭建完善的死锁监测和堆栈上报机制，那么你可以做一件简单的事情以协助你定位问题，那就是**打印日志**。在`dispatch_sync`或者加锁之前，打印一条日志。这样在用户反馈问题，或者测试重现问题的时候，提取日志便可分析出卡死的代码处。

## 如何安全使用dispatch_sync

答案是，**尽量不要使用**。没有哪一个接口是可以保证绝对安全的。必须要使用`dispatch_sync`的时候，尽量使用`dispatch_sync_on_main_queue`这个API。

## 最后

若有发现问题，或是更好的建议，欢迎私信或者评论:)
