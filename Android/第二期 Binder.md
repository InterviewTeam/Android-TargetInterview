# 第二期 Binder

- **IPC可进程间通信的本质？**

  内核空间可共享

- binder机制

- **简述你对AIDL的理解。**

  接口的跨进程调用。常说AIDL是安卓的一种跨进程通信的方式，其实不太严谨，应该说Binder才是一种跨进程通信的方式，而AIDL是Binder的一种具体应用，有点类似于网络中的传输层和应用层，IBinder有自己的通信协议，负责建立和维护连接，发送数据，而AIDL则定义了更上一层的传输内容，而像ContentProvider、广播同样是利用IBinder进行工作的。

- **为什么AMS与Zygote的通信不通过binder而是用socket**

  1. 猜测：Zygote是底层就有的，用socket可能涉及底层通信，binder只能android层通信

  2. **UNIX上C++程序设计守则3**

     **准则3：多线程程序里不准使用fork**

     但是Binder通讯是需要多线程操作的，多线程的fork子进程可能会死锁

     例子1：例如父进程的当前线程正在等待另外一个工作线程释放锁，而fork只会拷贝当前线程至子进程中。子进程启动后当前线程会一直等待工作线程的锁，但父进程的工作线程并没有拷贝到子进程，所以会造成死锁。

     **所以fork只能拷贝当前线程，不支持多线程的fork。**

  ​        如果zygote使用binder的多线程模型与system_server进程进行通讯的话，fork()出的App进程的binder     通讯没法用（binder维护了一个16个线程的线程池，Linux中fork只会拷贝当前线程到子进程，那么fork出的App进程中的binder就是个单线程了），那么只能再使用exec()启动一个新进程。

  但是exec()启动的新进程不再包含zygote进程的信息，那这样的就失去了fork的作用了，fork的原理就是copy-on-write机制，zygote进程中已经启动了虚拟机、进行资源和类的预加载以及各种初始化操作，App进程用时拷贝即可。

  所以最终zygote采用的方案就是socket + epoll，然后fork出子进程后再在子进程中启动binder线程池。

- **binder初始化时机**

- **mmap原理**

  ```cpp
  void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize);
  ```

  几个重要参数

  1. 参数start：指向欲映射的内存起始地址，通常设为 NULL，代表让系统自动选定地址，映射成功后返回该地址。
  2. 参数length：代表将文件中多大的部分映射到内存。
  3. 参数prot：映射区域的保护方式。可以为以下几种方式的组合：

  返回值是void *类型，分配成功后，被映射成虚拟内存地址。

  

  

- binder c/s架构简谈 

- 设计模式 

- binder与管道、内存比较

- AIDL有回调接口时，客户端针对该接口应该注意什么

- in out inout关键字

- `BINDER_VM_SIZE = (1*1024*1024) - (4096 *2)`, binder分配的默认内存大小为1M-8k。

- `DEFAULT_MAX_BINDER_THREADS = 15`，binder默认的最大可并发访问的线程数为16。

- ioctl

- ProcessState采用单例模式，保证每一个进程都只打开一次Binder Driver。

- **linux支持的IPC**
  1. 管道
  2. System V IPC
  3. 消息队列
  4. 共享内存
  5. 信号量
  6. Socket

- **binder与其他IPC比较有哪些优势？**

  1. 管道 name pipe：任何进程都能通讯，但速度慢

  2. 消息队列 message queue：容量受到系统限制，且要注意第一次读的时候，要考虑上一次没有读完数据的问题

  3. 信号量 signal：不能传递复杂消息，只能用来同步

  4. 共享内存 shared memory：能够容易控制容量，速度快，但要保持同步，比如写一个进程的时候，另一个进程要注意读写的问题，相当于线程中的线程安全。当然，共享内存同样可以作为线程通讯，不过没有这个必要，线程间本来就已经共享了同一个进程内的一块内存。

  5. socket：本机进程之间可以利用socket通信，跨进程之间也可利用socket通信，通常RPC的实现最底层都是通过socket通信。socket通信是一种比较复杂的通信方式，通常客户端需要开启单独的监听线程来接受从服务端发过来的数据，客户端线程发送数据给服务端，如果需要等待服务端的响应，并通过监听线程接受数据，需要进行同步，是一件很麻烦的事情。socket通信速度也不快。

  6. binder：

     **性能优势**：

     首先说说性能上的优势。Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。Binder 只需要一次数据拷贝，性能上仅次于共享内存。

  ![image-20200528155012374](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200528155012374.png)

  ​        **稳定优势：**

  ​		Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责    		明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度		讲，Binder 机制是优于内存共享的。

  ​        **安全优势：**

  ​		Android为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志。Binder  		既支持实名 Binder，又支持匿名 Binder，安全性高。

  ​		![](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200528155852368.png) 

  ​        **binder主要提供的功能：**

​                1、用驱动程序来推进进程间的通信方式

​				2、通过共享内存来提高性能

​				3、为进程请求分配的每个进程的线程池，每个进程默认启动的两个Binder服务线程

​				4、针对系统中的对象引入技术和跨进程对象的引用映射

​				5、进程间同步调用。



- **putBinder为什么支持跨进程大图传输**

  1. 简单方法：把图片存到 SD 卡，然后把路径传过去，在别的进程读出来。

  2. Bitmap 实现了 Parcelable 接口，可以通过 Intent.putExtra(String name, Parcelable value) 方法直接放 Intent 里面传递。Bitmap 太大会抛 **TransactionTooLargeException** 异常，原因是：**底层判断只要 Binder Transaction 失败，且 Intent 的数据大于 200k 就会抛这个异常了**。（见：android_util_Binder.cpp 文件 signalExceptionForError 方法）

  3. 通过Bundle.putBinder。为什么通过 putBinder 的方式传 Bitmap 不会抛 TransactionTooLargeException 异常

     通过startActivity最终都会走到**execStartActivity**

     ![image-20200602104817136](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200602104817136.png)

     ![image-20200602104912910](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200602104912910.png)

     ![image-20200602104924430](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200602104924430.png)

     最终设置的是在**Bundle中禁用描述符fd**。

     ![image-20200602104656647](C:\Users\wang.xiaotong\AppData\Roaming\Typora\typora-user-images\image-20200602104656647.png)

     **putBinder方法通过IBinder通道传输**，就算Bundle禁用了但是根本没有走Bundle通道，所以不会报异常也不会有限制。

- **Binder的定向制导，如何找到目标Binder，唤起进程或者线程**

- **Binder中的红黑树，为什么会有两棵binder_ref红黑树**

- **Binder一次拷贝原理(直接拷贝到目标线程的内核空间，内核空间与用户空间对应)**

- Binder传输数据的大小限制（内核4M 上层限制1m-8k），传输Bitmap过大，就会崩溃的原因，Activity之间传输BitMap

- 系统服务与bindService等启动的服务的区别

- Binder线程、Binder主线程、Client请求线程的概念与区别

- Client是同步而Server是异步到底说的什么

- Android APP进程天生支持Binder通信的原理是什么

- Android APP有多少Binder线程，是固定的么

- Binder线程的睡眠与唤醒（请求线程睡在哪个等待队列上，唤醒目标端哪个队列上的线程）

- Binder协议中BC与BR的区别

- Binder在传输数据的时候是如何层层封装的--不同层次使用的数据结构（命令的封装**）**

- Binder驱动传递数据的释放（释放时机）

- 一个简单的Binder通信C/S模型

- ServiceManager addService的限制（并非服务都能使用ServiceManager的addService）

- bindService启动Service与Binder服务实体的流程

- Java层Binder实体与与BinderProxy是如何实例化及使用的，与Native层的关系是怎样的

- Parcel readStrongBinder与writeStrongBinder的原理

- 同一个线程的请求必定是顺序执行，即使是异步请求(oneway)

  