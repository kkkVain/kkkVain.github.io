---

title: Handler解析
date: 2017-09-24 15:23:42
tags: [Android, Java, 源码]
categories: [Android]

---

主要为了解决不能在UI线程更新控件的问题。

由于Handler运行在主线程中(UI线程中)，  它与子线程可以通过Message对象来传递数据， 这个时候，Handler就承担着接受子线程传过来的(子线程用sendMessage()方法传递Message对象，里面包含数据)  ， 把这些消息放入主线程队列中，配合主线程进行更新UI。

### 作用

handler可以分发Message对象和Runnable对象到主线程中， 每个Handler实例，都会绑定到创建他的线程中(一般是位于主线程)，它有两个作用：

(1)安排消息或Runnable在某个主线程中某个地方执行；

post类方法允许你排列一个Runnable对象到主线程队列中

sendMessage类方法， 允许你安排一个带数据的Message对象到队列中，等待更新。

(2）将一个任务放入队列，以便它可以在另外的线程中执行。

![http://ovwunej09.bkt.clouddn.com/handler.png](handler)

### 源码分析

###### A.Handler

1）几个构造函数，最终主要是两个：

a.使用传入的looper

public Handler(Looper looper, Callback callback, boolean async) {

​    mLooper = looper;//指定该handler实例所绑定的looper实例

​    mQueue = looper.mQueue;

​    mCallback = callback; //给定回调接口进行消息处理。

​    mAsynchronous = async;

}



****思考：looper遍历queue中的Msg ，用msg的target handler进行处理，这是looper 关联handler，那么handler关联looper是什么情况？****

b.使用默认的looper. Handler最终的构造函数：（async默认是false）

public Handler(Callback callback, boolean async) {

​    if (FIND_POTENTIAL_LEAKS) {

​        final Class<? extends Handler> klass = getClass();

​        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&

​                (klass.getModifiers() & Modifier.STATIC) == 0) {

​            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +

​                klass.getCanonicalName());

​        }

​    }

​    mLooper = Looper.myLooper(); //静态函数，从sThreadLocal中取出变量

​    if (mLooper == null) {

​        throw new RuntimeException(

​            "Can't create handler inside thread that has not called Looper.prepare()");

​    }

​    mQueue = mLooper.mQueue;

​    mCallback = callback;

​    mAsynchronous = async;

}

mLooper的默认值是什么？只有Looper调用了prepare()才会有！

// sThreadLocal.get() will return null unless you've called prepare().

static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

也就是说，handler在构造前，要先在这个线程中执行Looper.prepare()，使Looper.myLooper()能成功返回当前线程threadLocal的looper！

也就是说handler关联looper，是一开始在构造函数中就关联起来的了，handler构造时，要么传入一个存在了的looper（looper.prepare()创造），要么当前线程已经调用looper.prepare()

终究就是要先有Looper.prepare()

2）所有的常用的send和post方法，最终调用的都是sendMessageAtTime(Message, long) 这个方法（如果传入的参数是runnable的话，会先调用getPostMessage(Runnable) 或getPostMessage(Runnable, Object)方法获取消息）

假设看看postDelayed：

A.public final boolean postDelayed(Runnable r, long delayMillis)

{

​    return sendMessageDelayed(getPostMessage(r), delayMillis);--->B

}

其中get函数用来设置runnable依附的message

private static Message getPostMessage(Runnable r) {

​    Message m = Message.obtain();

​    m.callback = r;

​    return m;

}

B.public final boolean sendMessageDelayed(Message msg, long delayMillis)

{

​    if (delayMillis < 0) {

​        delayMillis = 0;

​    }

​    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);--->C

}

重点函数：

// 将message入队，并在指定时间到之前将该消息放在所有挂起的消息之后。

C.public boolean sendMessageAtTime(Message msg, long uptimeMillis) {

​        MessageQueue queue = mQueue;

​        if (queue == null) {

​            RuntimeException e = new RuntimeException(

​                    this + " sendMessageAtTime() called with no mQueue");

​            Log.w("Looper", e.getMessage(), e);

​            return false;

​        }

​        return enqueueMessage(queue, msg, uptimeMillis);

​    }

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {

​    msg.target = this; //为message指定target为该handler实例（精髓）

​    if (mAsynchronous) {

​        msg.setAsynchronous(true);

​    }

​    return queue.enqueueMessage(msg, uptimeMillis);

}

PS: 发现了两个强行插队，让runnable在message 的下一个Loop直接执行的函数：

其实就是把delaymills置为0= =

public final boolean postAtFrontOfQueue(Runnable r)

{

​    return sendMessageAtFrontOfQueue(getPostMessage(r));

}

public final boolean sendMessageAtFrontOfQueue(Message msg) {

​    MessageQueue queue = mQueue;

​    if (queue == null) {

​        RuntimeException e = new RuntimeException(

​            this + " sendMessageAtTime() called with no mQueue");

​        Log.w("Looper", e.getMessage(), e);

​        return false;

​    }

​    return enqueueMessage(queue, msg, 0);

}

D. loop()函数中，msg.target调用的函数：

执行优先级：1)msg中含有callback（用post方法时），用message的这个callback方法处理msg；2）构造这个handler时传入了callback，同样用callback处理Msg；3）直接用handler的重载函数handleMessage处理msg

/**

 \* Handle system messages here.

 */

public void dispatchMessage(Message msg) {

​    if (msg.callback != null) {

​        handleCallback(msg);

​    } else {

​        if (mCallback != null) {

​            if (mCallback.handleMessage(msg)) {

​                return;

​            }

​        }

​        handleMessage(msg);

​    }

}

######B.Message

extends Object

implements Parcelable

一个message对象包含一个自身的描述信息和一个可以发给handler的任意数据对象。

虽然Message的构造方法是public的，但实例化Message的最好方法是调用Message.obtain() 或 Handler.obtainMessage() ，因为这两个方法是从一个可回收利用的message对象回收池中获取Message实例（实际上最终调用的仍然是Message.obtain()）。该回收池用于将每次交给handler处理的message对象进行回收。 

##### C.MessageQueue

MessageQueue是用来存放Message的集合，并由Looper实例来分发里面的Message对象。同时，message并不是直接加入到MessageQueue中的, 而是通过与Looper对象相关联的MessageQueue.IdleHandler 对象来完成的。我们可以通过Looper.myQueue() 方法来获得当前线程的MessageQueue。 

MQ实际上没有采用队列，而是采用单链表的数据结构来存储消息列表。

##### D.Looper

Looper是线程用来运行消息循环(message loop)的类。默认情况下，线程并没有与之关联的Looper，可以通过在线程中调用Looper.prepare() 方法来获取，用来执行loop，并通过Looper.loop() 无限循环地获取并分发MessageQueue中的消息，直到所有消息全部处理。

\* <p>This is a typical example of the implementation of a Looper thread,

\* using the separation of {@link #prepare} and {@link #loop} to create an

\* initial Handler to communicate with the Looper.

*

\* <pre>

\*  class LooperThread extends Thread {

\*      public Handler mHandler;

*

\*      public void run() {

\*          Looper.prepare(); // 创建一个looper，来循坏获取处理MessageQueue里面的消息

*

\*          mHandler = new Handler() { // 那么这个handler，就不是在主线程处理msg了，而是在looperThread上处理

\*              public void handleMessage(Message msg) {

\*                  // process incoming messages here

\*              }

\*          };

*

\*          Looper.loop();  // 处理messagequeue的关键函数

\*      }

\*  }</pre>

1）public static void prepare() {

​    prepare(true);

}

private static void prepare(boolean quitAllowed) { // quitAllowed：是否允许调用quit()函数

​    if (sThreadLocal.get() != null) {

​        throw new RuntimeException("Only one Looper may be created per thread");

​    }

​    sThreadLocal.set(new Looper(quitAllowed));

}

可以看到，调用prepare()方法时，会执行prepare(boolean quitAllowed) 方法，其中得先判断sThreadLocal是否为空，不为空才将新建的Looper实例放进sThreadLocal对象中。那么这个sThreadLocal是什么呢？它是一个本地线程存储类，所有线程共享这个对象，但是这个对象对每一个线程而言却具有不同的值，且每个线程对这个对象的访问或修改都不会影响到其他线程，即它的值对于每个线程来说都是独立的。 不同线程的Looper相互独立，之所以能做到这一点，就是借助ThreadLocal来实现的。

sThreadLocal.set(new Looper(quitAllowed))则是把一个新建的Looper实例放进sThreadLocal对象中，而Looper的构造函数内部实现如下,创建一个新的MessageQueue对象并绑定当前线程:

private Looper(boolean quitAllowed) {

​    mQueue = new MessageQueue(quitAllowed);

​    mThread = Thread.currentThread();// 这个mThread在Looper类中毫无用处....

}

2）public static void loop() {}代码很长

public static void loop() {

​    final Looper me = myLooper(); ---->  A. 从threadLocal.get()取looper

​    if (me == null) {

​        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");

​    }

​    final MessageQueue queue = me.mQueue;

​    // Make sure the identity of this thread is that of the local process,

​    // and keep track of what that identity token actually is.

​    Binder.clearCallingIdentity();

​    final long ident = Binder.clearCallingIdentity();

​    for (;;) {

​        Message msg = queue.next(); // might block

​        if (msg == null) {

​            // No message indicates that the message queue is quitting.

​            return;

​        }

​        // This must be in a local variable, in case a UI event sets the logger

​        final Printer logging = me.mLogging;

​        if (logging != null) {

​            logging.println(">>>>> Dispatching to " + msg.target + " " +

​                    msg.callback + ": " + msg.what);

​        }

​        final long traceTag = me.mTraceTag;

​        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {

​            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));

​        }

​        try {

​            msg.target.dispatchMessage(msg); // handler不断把msg入队，这里从队列取出msg，然后用msg的handler进行消息处理

​        } finally {

​            if (traceTag != 0) {

​                Trace.traceEnd(traceTag);

​            }

​        }

​        if (logging != null) {

​            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);

​        }

​        // Make sure that during the course of dispatching the

​        // identity of the thread wasn't corrupted.

​        final long newIdent = Binder.clearCallingIdentity();

​        if (ident != newIdent) {

​            Log.wtf(TAG, "Thread identity changed from 0x"

​                    \+ Long.toHexString(ident) + " to 0x"

​                    \+ Long.toHexString(newIdent) + " while dispatching to "

​                    \+ msg.target.getClass().getName() + " "

​                    \+ msg.callback + " what=" + msg.what);

​        }

​        msg.recycleUnchecked();

​    }

}

这个方法，主要是通过myLooper()获取当前线程的Looper实例，如果这个实例存在，就取出这个looper的MQ，然后通过一个死循环，不断的调用queue.next();取出message，如果msg没有target（想想enqueueMessage里面为msg指定了target），就退出循环。

如果有target:msg.target.dispatchMessage(msg);

把真正的处理工作交给msg的target，即对应的handler，直到队列为空。

该函数在每次msg的循环的最后都会调用msg.recycle();回收msg资源。

A.public static @Nullable Looper myLooper() {

​    return sThreadLocal.get();

}

3）Message next() {} 代码也很长，不贴了。

主要功能就是：取出单链表的头节点，对相应指针进行操作，最后返回这个头节点。



### 灵魂拷问

**思考一：loop() 这里采用的是无限循环，该循环会不会特别消耗CPU资源？？**

其实并不会，如果MQ里面有消息，next（）自然能不停的取到msg去交给handler进行处理，如果取不到，那么在nex()的nativePollOnce(ptr, nextPollTimeoutMillis);  就会阻塞该线程，主线程会释放CPU资源进入休眠状态，当有下一个工作到来时，才通过往pipe管道写端写入数据来唤醒主线程工作。这里涉及到的是Linux的pipe/epoll机制，epoll机制是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

4）public void dispatchMessage(Message msg) {

​    if (msg.callback != null) {

​        handleCallback(msg); ①

​    } else {

​        if (mCallback != null) {

​            if (mCallback.handleMessage(msg)) {②

​                return;

​            }

​        }

​        handleMessage(msg);③

​    }

}

...

① private static void handleCallback(Message message) {

​    message.callback.run();

}

这里我们可以看到，在分发消息时三个方法的优先级分别如下：

①Message的回调方法优先级最高，即message.callback.run()；

②Handler的回调方法优先级次之，即Handler.mCallback.handleMessage(msg)；

③Handler的默认方法优先级最低，即Handler.handleMessage(msg)



**思考二、为什么looper.loop()要使用死循环？**

对于线程既然是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。

**思考三、哪里为Looper.loop（)创建了新线程？**

在代码ActivityThread.main()中：

public static void main(String[] args) {

​    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

​    SamplingProfilerIntegration.start();

​    // CloseGuard defaults to true and can be quite spammy.  We

​    // disable it here, but selectively enable it later (via

​    // StrictMode) on debug builds, but using DropBox, not logs.

​    CloseGuard.setEnabled(false);

​    Environment.initForCurrentUser();

​    // Set the reporter for event logging in libcore

​    EventLogger.setReporter(new EventLoggingReporter());

​    // Make sure TrustedCertificateStore looks in the right place for CA certificates

​    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());

​    TrustedCertificateStore.setDefaultUserDirectory(configDir);

​    Process.setArgV0("<pre-initialized>");

​    //创建Looper和MessageQueue对象，用于处理主线程的消息

​    Looper.prepareMainLooper();----->A

​    // 创建ActivityThread对象

​    ActivityThread thread = new ActivityThread();

​    //建立Binder通道 (创建新线程)

​    thread.attach(false);

​    if (sMainThreadHandler == null) {

​        sMainThreadHandler = thread.getHandler();

​    }

​    if (false) {

​        Looper.myLooper().setMessageLogging(new

​                LogPrinter(Log.DEBUG, "ActivityThread"));

​    }

​    // End of event ActivityThreadMain.

​    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

​    Looper.loop();

​    throw new RuntimeException("Main thread loop unexpectedly exited");

}

A.我们自己不能调用这个function，如果调用了的话- -会 怎么样呢 ？？

/**

 \* Initialize the current thread as a looper, marking it as an

 \* application's main looper. The main looper for your application

 \* is created by the Android environment, so you should never need

 \* to call this function yourself.  See also: {@link #prepare()}

 */

public static void prepareMainLooper() {

​    prepare(false);

​    synchronized (Looper.class) {

​        if (sMainLooper != null) {

​            throw new IllegalStateException("The main Looper has already been prepared.");

​        }

​        sMainLooper = myLooper();

​    }

}

thread.attach(false)；便会创建一个Binder线程（具体是指ApplicationThread，Binder的服务端，用于接收系统服务AMS发送来的事件），该Binder线程通过Handler将Message发送给主线程。

ActivityThread实际上并非线程，不像HandlerThread类，ActivityThread并没有真正继承Thread类，只是往往运行在主线程，该人以线程的感觉，其实承载ActivityThread的主线程就是由Zygote fork而创建的进程。

**思考四、最后，扩展到Activity的生命周期。Activity的生命周期是怎么实现在死循环体外能够执行起来的？**

ActivityThread的内部类H继承于Handler，通过handler消息机制，简单说Handler机制用于同一个进程的线程间通信。Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施：

在H.handleMessage(msg)方法中，根据接收到不同的msg，执行相应的生命周期。

比如收到msg=H.LAUNCH_ACTIVITY，则调用ActivityThread.handleLaunchActivity()方法，最终会通过反射机制，创建Activity实例，然后再执行Activity.onCreate()等方法；  

再比如收到msg=H.PAUSE_ACTIVITY，则调用ActivityThread.handlePauseActivity()方法，最终会执行Activity.onPause()等方法。

主线程的消息又是哪来的呢？当然是App进程中的其他线程通过Handler发送给主线程，

**思考五、handler怎么进行线程切换的？**

Handler类的介绍写了：在自己的子线程中，调用post或者sendMessage函数，最后都是调用sendMessageAtTime函数，反正就是把message（post的runnalble 会经过包装，成为message）放到queue中让looper循环获取。

You can create your own threads, and communicate back with

\* the main application thread through a Handler.  This is done by calling

\* the same <em>post</em> or <em>sendMessage</em> methods as before, but from

\* your new thread. 