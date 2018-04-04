---
title: Effective Java 读书笔记（2.5）
date: 2017-07-23 24:38:31
tags: [HandlerThrea, Handler, 源码]
categories: [Android]
---

## HandlerThread

### 简介
Handler的目标是解决子线程与主线程通信的问题，HandlerThread则是解决子线程和子线程之间的通信问题，官方给的子线程实现handler的机制。
UI线程中的MessageQueue以及Looper是默认创建好的，只要创建Handler对象实例并调用handleMessage方法即可进行消息处理，而子线程要自己去自定义Looper才能实现相同的功能。
HandlerThread 继承自Thread类，主要就是增加了一个looper变量（looper.loop()可以取消息进行处理）

### 源码分析
``` java
int mPriority;
int mTid = -1;
Looper mLooper;
public HandlerThread(String name) {
    super(name);
    mPriority = Process.THREAD_PRIORITY_DEFAULT; // 优先级默认是0
}
```

``` java
/**
 * Call back method that can be explicitly overridden if needed to execute some
 * setup before Looper loops.
 */
protected void onLooperPrepared() {
}
```

``` java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper(); // 创建Looper
        notifyAll();
    }
    Process.setThreadPriority(mPriority);--->D // 设置线程优先级
    onLooperPrepared(); // Looper创建后调用，可重写
    Looper.loop(); // 开始循环了
    mTid = -1;
}
```

getLooper时，如果这个thread 已经start了 （执行run）， Looper有可能还没有实例化结束，可能阻塞（有synchronized关键字），这也是为什么 创建Looper后，有notifyAll()的原因！
``` java
/**
 * This method returns the Looper associated with this thread. If this thread not been started
 * or for any reason is isAlive() returns false, this method will return null. If this thread 
 * has been started, this method will block until the looper has been initialized.  
 * @return The looper.
 */
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
    
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

D.
``` java
public static final native void setThreadPriority(int priority)
        throws IllegalArgumentException, SecurityException;
/**
 * Set the priority of the calling thread, based on Linux priorities.  See
 * {@link #setThreadPriority(int, int)} for more information.
 * 
 * @param priority A Linux priority level, from -20 for highest scheduling
 * priority to 19 for lowest scheduling priority.
```

用处：串行处理任务，主要是分担UI线程的一些耗时操作，不会干扰或阻塞UI线程。对于网络IO操作，HandlerThread并不适合，因为它只有一个线程，还得排队一个一个等着。