# **第一期 handler**

### Handler

- 使用Handler如何有效的避免内存泄漏问题？

- Handler的post和sendMessage方法的区别和应用场景？
- Handler的post发送的是同步消息还是异步消息？
- Handler的post和postDelayed的区别？
- 子线程创建Handler会抛异常吗？
- Handler实现线程切换？
- HandlerThread的使用场景与用法？
- Activity中runOnUiThread的原理？
- Handler与Looper对应的关系，一对一还是一对多？
- 了解Android中有哪些是基于Handler来实现通信的？
- 在子线程发送消息，主线程是怎样收到的？为什么handleMessage的message是在主线程？
- 谈谈Handler的同步屏障和使用场景？



### Lopper

- 如何在子线程中创建Looper?

- Looper.loop()是死循环，可以停止么？

- Looper.loop死循环，为什么没有阻塞主线程，原理是什么？
- 非UI线程真的不能更新UI吗？
- Looper和Handler一定要处于一个线程吗？子线程中可以用MainLooper去创建Handler吗？
- 子线程可以和主线程通信吗？主线程可以和子线程通信吗？

### Message

- Handler机制中哪一块有消息池？这一块消息池的作用？
- Message消息池的容量？
- Message中可传输消息的对象类型有哪些？
- 每个Message都必须有target吗？为什么？
- Message传输最大值是多少？

### MessageQueue

- 消息是如何插入到MessageQueue中的？
- MessageQueue没有消息时，它的next方法是阻塞的，会导致App ANR么？

