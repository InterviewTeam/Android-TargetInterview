## Handler问题答疑

### 使用Handler如何有效避免内存泄漏
Messagequeue持有message，message的target持有Handler；如果Handler是非静态内部类的话，就会持有外部Activity的引用，因此导致Activity无法被释放。

解决方法：

1. 定义Handler为静态内部类，Activity以弱饮用的方式传入；
2. 推出Activity时，调用Handler.removeCallbacksAndMessages(null)；

### Handler的post和sendMessage方法的区别和应用场景？

### Handler的post发送的是同步消息还是异步消息

### Handler的post和postDelayed的区别？

### 子线程创建Handler会抛异常吗？
如果直接new Handler()会抛一个RuntimeException异常，需要先调用Looper.prepare()为当前创建一个Looper；或者直接new Handler(Looper.getMainLooper())。

### Handler实现线程切换
子线程enqueue到MessageQueue里的msg，会在主线程Looper的loop方法中取出来，然后执行msg.target.dispatchMessage方法。

### HandlerThread的使用场景与用法
HandlerThread继承了Thread，是单线程。run方法里调用了Looper.prepare()和Looper.loop()，提供了和主线程一样的消息处理机制。只要new一个Handler，把HandlerTread的Looper对象传进去，就可以通过post或者sendMessage等方法让这个子线程处理任务。

IntentService
### Activity中runOnUiThread的原理
往主线程的handler上post一个runnable

### Handler与Looper的关系
多对一

### 了解Android中有哪些是基于Handler来实现通信的
AMS

### 谈谈Handler的同步屏障和使用场景
ViewRootImpl的sheduleTraversals方法里调用postSyncBarrier，doTraversal方法里removeSyncBarrier。

## Looper
### 如何在子线程中创建Looper
Looper.prepare()
Looper.loop()是死循环，可以停止么

### Looper.loop()是死循环，可以停止么
可以，Looper提供quit和quitSafely两个方法。quit是直接把队列所有消息都移除，quitSafely将所有大于当前时间的消息回收掉。

## MessageQueue
### 消息是如何插入到MessageQueue中的
