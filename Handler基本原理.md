### 基本原理
1. Handler负责消息的发送和处理：Handler不断发送消息给MessageQueue和接收Looper返回的消息并处理消息；
2. Looper负责管理MessageQueue：Looper会不断地从MessageQueue取出消息，交给Handler处理；
3. MessageQueue是消息队列（链表），负责存放Handler发送过来的消息；
4. 一个线程只有一个Looper，一个Looper对应一个线程。Loop的loop()方法运行所在的线程中，当Handler在线程A发送一条消息存放在MessageQueue时，Looper的loop()方法在线程B把消息取出来，交给Handler处理。一存一取之间实现了线程的切换。

### 代码实现逻辑
Looper的构造方法是私有的，对外提供了prepare()方法创建并且通过Looper.myLooper()获取对象，对象存放在ThreadLocal中。

```java
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

Looper的构造方法里创建了一个MessageQueue对象。
创建完Looper后，需要让Looper循环去取消息，靠的就是Looper的loop方法。

```java
public static void loop() {
        final Looper me = myLooper();
        ...
        final MessageQueue queue = me.mQueue;
        ...
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            ...
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            ...
        }
    }
```

looper方法的框架很简单，拿到构造方法里创建的MessageQueue，在死循环里调用MessageQueue的next方法取出Handler发送过来的Message，然后调用msg.target.dispatchMessage方法，也就是Handler的dispatchMessage方法。
所以重点来看MessageQueue的next方法。

```java
MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

Message的构造方法会调用native方法nativeInit()。
我们来看这个方法的实现：

> frameworks/base/core/jni/android_os_MessageQueue.cpp

```c++
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```
> /system/core/libutils/Looper.cpp

```c++
Looper::Looper(bool allowNonCallbacks)
      : mAllowNonCallbacks(allowNonCallbacks),
        mSendingMessage(false),
        mPolling(false),
        mEpollRebuildRequired(false),
        mNextRequestSeq(0),
        mResponseIndex(0),
        mNextMessageUptime(LLONG_MAX) {
        //构造唤醒事件的fd
      mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
      LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));
      AutoMutex _l(mLock);
      //重建Epoll事件
      rebuildEpollLocked();
  }
```
在Native层创建了一个NativeMessageQueue对象，并且将其句柄返回到 Java 层保存在 mPtr 中。同时在NativeMessageQueue的构造方法里，创建了一个Native层的Looper对象。

> Java层，在Looper的构造函数里创建了MessageQueue，而在Native层正好相反，是在NativeMessageQueue的构造方法里创建了Looper。

重点来看rebuildEpollLocked方法：
> /system/core/libutils/Looper.cpp

```c++
void Looper::rebuildEpollLocked() {
    void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    // 关闭老的epoll实例
    if (mEpollFd >= 0) {
#if DEBUG_CALLBACKS
        ALOGD("%p ~ rebuildEpollLocked - rebuilding epoll set", this);
#endif
        mEpollFd.reset();
    }

    // Allocate the new epoll instance and register the wake pipe.
    // 创建新的epoll实例并且注册wake管道
    mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd.get();
    // 将唤醒事件添加到epoll实例
    int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                        strerror(errno));
    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        //将request队列的事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, request.fd, &eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                  request.fd, strerror(errno));
        }
    }
}
```
至此，Looper对象中的mWakeEventFd以及mRequests也添加到epoll的监控范围内。接下来分析获取事件然后唤醒线程的过程。
Java层的Looper会调用Java层的MessageQueue的next方法获取Message，分析这个过程：

```java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            
            nativePollOnce(ptr, nextPollTimeoutMillis);
            ...
        }
    }
```

nativePollOnce方法的实现在Native层：

```c++
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
        // 强制类型转换
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}

// Looper.h
inline int pollOnce(int timeoutMillis) {
        return pollOnce(timeoutMillis, nullptr, nullptr, nullptr);
    }
// Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    // 对fd对应的Responses进行处理，后面发现Response里都是活动fd
    for (;;) {
        ...
        // 处理内部轮训
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
    ...
    // 阻塞直到超时或者消息到了
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    ...
    return result;
}
```

消息到了主要看MessageQueue的enqueueMessage方法：

```java
// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
        ...
        synchronized (this) {
            ...
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
// android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

> Looper.cpp

```c++
void Looper::wake() {
    ...
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
    ...
}
```
Looper的wake函数会往mWakeEventFd中写入一些内容，最后把epoll_wait唤醒。


### Handler消息池
Message类有一个静态变量sPool，通过next成员变量维护一个消息池，消息池默认大小为50。消息池的作用是提高效率，提供一个大小为50的Message缓存链表，减少对象不断创建与销毁的过程。
obtain方法，从消息池中取数据：
```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null; //从sPool中取出一个Message对象，并消息链表断开
            m.flags = 0; // 清除in-use flag
            sPoolSize--; //消息池的可用大小进行减1操作
            return m;
        }
    }
    return new Message(); // 当消息池为空时，直接创建Message对象
}
```
recycle方法，把不再使用的消息加入消息池：

```java
public void recycle() {
    if (isInUse()) { //判断消息是否正在使用
        if (gCheckRecycle) { //Android 5.0以后的版本默认为true,之前的版本默认为false.
            throw new IllegalStateException("This message cannot be recycled because it is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

//对于不再使用的消息，加入到消息池
void recycleUnchecked() {
    //将消息标示位置为IN_USE，并清空消息所有的参数。
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) { //当消息池没有满时，将Message对象加入消息池
            next = sPool;
            sPool = this;
            sPoolSize++; //消息池的可用大小进行加1操作
        }
    }
}
```

### addFd方法
> /system/core/libutils/Looper.cpp

```c++
int Looper::addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data) {
    return addFd(fd, ident, events, callback ? new SimpleLooperCallback(callback) : nullptr, data);
}

int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ addFd - fd=%d, ident=%d, events=0x%x, callback=%p, data=%p", this, fd, ident,
            events, callback.get(), data);
#endif
    // 如果callback不为空，则把ident置为POLL_CALLBACK，表示监听的这个文件符需要执行回调方法
    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }

        if (ident < 0) {
            ALOGE("Invalid attempt to set NULL callback with ident < 0.");
            return -1;
        }
    } else {
        ident = POLL_CALLBACK;
    }

    { // acquire lock
        AutoMutex _l(mLock);
        // 根据传参，构造一个Request对象
        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.seq = mNextRequestSeq++;
        // callback的handleevent函数会在唤醒后执行，具体见下文
        request.callback = callback;
        request.data = data;
        if (mNextRequestSeq == -1) mNextRequestSeq = 0; // reserve sequence number -1

        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        // 去mRequests数组里找是否已经有该文件符的reqeust，找不到的话返回-2；mReqeusts是一个KeyedVector对象，indexOfKey用的二分查找。
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            // 如果没查到，就用epoll_ctl的EPOLL_CTL_ADD命令注册fd到mEpollFd中
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, fd, &eventItem);
            if (epollResult < 0) {
                ALOGE("Error adding epoll events for fd %d: %s", fd, strerror(errno));
                return -1;
            }
            // 然后以fd为key，request为value，添加到mRequests中
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_MOD, fd, &eventItem);
            ...
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}
```
根据上面的内容，会在Looper.cpp的pollInner函数中阻塞。
> system/core/libutils/Looper.cpp

```c++
int Looper::pollInner(int timeoutMillis) {
    ...
    // 阻塞，唤醒后之后下面的代码
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();

    // Rebuild epoll set if needed.
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif
    // 根据之前的
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd.get()) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            // 通过addFd接口加入到epoll监听队列的fd，会把reqeust取出来，填充到responds里，后续处理
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        // 只处理ident为POLL_CALLBACK的事件
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            // 调用request的callback的handleEvent方法，真正处理逻辑的地方
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}

void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;
    mResponses.push(response);
}
```