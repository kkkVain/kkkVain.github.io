---
title: EventBus源码解析(1)
date: 2017-09-10 20:19:30
tags: [eventbus,事件总线,观察者模式]
categories: Android
---

**3.EventBus**

由于没有用过，所以先在这里理解一下几个重点名词：

event：我们想监听的事件，比如点击按钮，网页请求返回了响应

eventType：我们想监听的事件的类型

subscrible：订阅者，订阅事件的对象，当订阅者订阅的事件发生后，就会执行订阅者的响应函数，onEvent。

**使用第一步：在接收消息的activity中(onStart())**

**EventBus.getDefault().register(this); ---\>B**

**字母开头和1字母开头的，都是rigister的延伸。**

**使用第二步：在发送消息的activity中，**

**EventBus.getDefault().post(new Event("Event btn clicked")); ------\>2A**

**2字母开头的，都是这个的延伸。**

**使用第三步：在接收消息的activity中（onStop())**

**EventBus.getDefault().unregister(this);-------\>3A**

**①EventBus.JAVA**

**A.**

单例模式，双重检查+volatile ，获取eventBus的实例。

```java
static volatile EventBus defaultInstance;
public static EventBus getDefault() {
	if (defaultInstance == null) {
		synchronized (EventBus.class) {
			if (defaultInstance == null) {
				defaultInstance = new EventBus();
			}
		}
	}
	return defaultInstance;
}
```
**3A.**

```java
/* Unregisters the given subscriber from all event classes. 这是通过订阅者找出订阅类型来注销这个订阅者的所有订阅*/
public synchronized void unregister(Object subscriber) {
//取出订阅者订阅的所有事件类型
List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
	if (subscribedTypes != null) {
	//遍历所有事件类型，注销订阅
	for (Class<?> eventType : subscribedTypes) {
		unsubscribeByEventType(subscriber, eventType);---->3B
	}
	typesBySubscriber.remove(subscriber);
	} else { //这个订阅者没有订阅过任何事件。
	但是，没有订阅过，有可能register啊...？？？？
	Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
	}
}
```
**3B.**

**更新subscriptionsByEventType，而不是typesBySubscriber，typesBySubscriber在3A中（unregister)更新了**

``` java
/* Only updates subscriptionsByEventType, not typesBySubscriber! Caller must
update typesBySubscriber. 这是通过事件类型找出所有订阅者来取消订阅*/

private void unsubscribeByEventType(Object subscriber, Class<?> eventType)
{
//通过事件类型取出订阅该类型的所有subscriptions，**注意，定义的时候，value是copyOnWriteList,这里就直接List了？？**
List<Subscription> subscriptions =
subscriptionsByEventType.get(eventType);
	if (subscriptions != null) {
		int size = subscriptions.size();
//遍历subscription,找到当前要注销的subcriber，更改active状态，并且从这个list中将它移出
// 这个find并且remove的过程- -真是清新自然，没有更简单的方法？？
		for (int i = 0; i < size; i++) {
			Subscription subscription = subscriptions.get(i);
			if (subscription.subscriber == subscriber) {
				subscription.active = false;
				subscriptions.remove(i);
				i--;
				size--;
			}
		}
	}
}
```

**2A.**

**从这里到2I，都是这个函数的延伸。**
``` java
public void post(Object event) {
	PostingThreadState postingState =
	currentPostingThreadState.get();--->2B
	List<Object> eventQueue = postingState.eventQueue; //postingThread的queue
	eventQueue.add(event); //向这个List添加事件
	if (!postingState.isPosting) {
		//这个boolean值，都没有被赋值过...所以是取默认值？？boolean的默认值是false，所以就说得通了。
		//判断进行post的线程是否主线程
		postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
		//当前线程的isPosting为真
		postingState.isPosting = true;
		//如果取消了，抛出异常。
		if (postingState.canceled) {
			throw new EventBusException("Internal error. Abort state was not reset");
		}
		try {
			while (!eventQueue.isEmpty()) {
				//如果当前线程要post的事件队列不为空，取出队首，执行了postSingleEventForEventType，然后在里面又执行了postToSubscription，完成了post
				postSingleEvent(eventQueue.remove(0), postingState);--->2D
			}
		} finally {
			//finally里面初始化成员变量
			postingState.isPosting = false;
			postingState.isMainThread = false;
		}
	}
}
```

**2D.**

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
	Class<?> eventClass = event.getClass();
	boolean subscriptionFound = false;
	if (eventInheritance) {
	List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);--->2E,得到这个eventClass的所有超类和父接口（也就是它可以对应的所有事件类型）
	int countTypes = eventTypes.size();
		for (int h = 0; h < countTypes; h++) {
			Class<?> clazz = eventTypes.get(h);
			subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);---->2G，在postSingleEventForEventType里面遍历所有监听了这个事件类型的subcription,通过postToSubscription去找线程调用订阅者的响应方法
		}
	} else { //没有进入else，就直接调用postSingleEventForEventType	
		subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);---->2G
	}
	if (!subscriptionFound) {//如果
postSingleEventForEventType没执行成功，也就是这个事件没有订阅者订阅这类事件！！
		if (logNoSubscriberMessages) {
			Log.d(TAG, "No subscribers registered for event " + eventClass);
		}
//这个判断有点奇怪。 eventClass 和 NoSubscriberEvent.class
,怎么可能相等...不过看到post(new...）就明白了，最开始的时候当然都是！=，不过重新new了NoSubscriberEvent去post，当然可能==了。
		if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class && eventClass != SubscriberExceptionEvent.class) {---->2H，2I
			post(new NoSubscriberEvent(this,event));//用当前类（调用post的类）和被post的event构造NoSubscriberEvent，所以有可能出现if中的第二种情况!!厉害了。但是第三种呢？？
		}

	}
}
```

**2H.**

```java
/* 被post的事件没有订阅者订阅 哈哈哈哈哈 好惨
 This Event is posted by EventBus when no subscriber is found for a posted event.
 @author Markus
*/
public final class NoSubscriberEvent {
/* The {\@link EventBus} instance to with the original event was posted to.
*/
	public final EventBus eventBus;
	/* The original event that could not be delivered to any subscriber. */
	public final Object originalEvent;
	public NoSubscriberEvent(EventBus eventBus, Object originalEvent) {
		this.eventBus = eventBus;
		this.originalEvent = originalEvent;
	}

}
```

**2I.**

``` java
/* 这个post的事件在订阅者响应函数中出现异常（需要理解下...
 This Event is posted by EventBus when an exception occurs inside a subscriber's event handling method.*
 @author Markus*
*/
	public final class SubscriberExceptionEvent {
/* The {\@link EventBus} instance to with the original event was posted to.
*/
	public final EventBus eventBus;
	/* The Throwable thrown by a subscriber.*/
	//订阅者抛出的异常
	public final Throwable throwable;

	/* The original event that could not be delivered to any subscriber. */
	//无法再被post给任何订阅者的event
	public final Object causingEvent;

	/* The subscriber that threw the Throwable. */
	//抛出异常的订阅者
	public final Object causingSubscriber;

	public SubscriberExceptionEvent(EventBus eventBus, Throwable throwable,Object causingEvent,Object causingSubscriber) {
		this.eventBus = eventBus;
		this.throwable = throwable;
		this.causingEvent = causingEvent;
		this.causingSubscriber = causingSubscriber;
	}
}
```

**2G.**
```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
	CopyOnWriteArrayList<Subscription> subscriptions;
	synchronized (this) {
	//取出这个事件class所涉及的所有subscription(里面存储了订阅者和处理这个事件的方法）
		subscriptions = subscriptionsByEventType.get(eventClass);
	}

	// 这个if里面，不是一个意思？？？
	// 存在订阅这种事件类型的订阅者
	if (subscriptions != null && !subscriptions.isEmpty()) {
		for (Subscription subscription : subscriptions) {
			//遍历所有的subscription,设置postingState的参数
			postingState.event = event;
			postingState.subscription = subscription;
			boolean aborted = false;
			try {
				postToSubscription(subscription, event,postingState.isMainThread);---->1D,在subscribe(..）粘性事件那里分析过了，所以那里的疑惑就解决了....那里的post是由于“粘性”立即post最近的event给订阅者，这里的postToSubscription是发送信息的一方直接调用post导致的
				aborted = postingState.canceled; //判断是否abort
			} finally {
				//当前状态的变量置为null，由于if可能结束循环，所以在finally中置空，避免内存泄漏。个人觉得...为什么不直接for循环外面置空，一定要置空了再赋值？？
				postingState.event = null;
				postingState.subscription = null;
				postingState.canceled = false;
			}
			if (aborted) {
				break;//如果abort了，那么这个事件的post就结束了
			}
		}
		//for循环结束，返回true
		return true;
	}
//没进入if，返回false
return false;
}
```

**2E.**
``` java
/* Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass){
	synchronized (eventTypesCache) {
		//从事件类的缓存中，根据事件class取出队列
		List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
		if (eventTypes == null) {
			//如果没有该类型的event，新建List
			eventTypes = new ArrayList<>();
			Class<?> clazz = eventClass;
			while (clazz != null) {
				eventTypes.add(clazz); // 把当前事件类加到这个list中
				//clazz.getInterfaces(),又是反射机制，可以获得这个class实现自哪些接口。
				addInterfaces(eventTypes,clazz.getInterfaces());--->2F，递归获得事件类的所有父接口，加到eventTypes这个list中
				clazz = clazz.getSuperclass();
			}
			//然后把这个eventClass以及它可以对应的所有事件类型（List集合保存)加入cache
			eventTypesCache.put(eventClass, eventTypes);
		}
		return eventTypes;
	}
}
```

**2F.**
``` java
/* Recurses through super interfaces. */
static void addInterfaces(List<Class<?>> eventTypes, Class<?>[]
interfaces) {
	for (Class<?> interfaceClass : interfaces) {
	//遍历事件类实现的接口，如果这个事件类型的List中不包含有这个接口
		if (!eventTypes.contains(interfaceClass)) {
			eventTypes.add(interfaceClass);// 把这个接口添加进list
		// 继续用interfaceClass.getInterfaces()得到interfaceClass实现的接口
		//迭代这个方法
			addInterfaces(eventTypes, interfaceClass.getInterfaces());
		}
	}
}
```

**2B**.一个成员变量，**PostingThreadState类型--->2C**的ThreadLocal
``` java
private final ThreadLocal<PostingThreadState>
currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
	@Override
	protected PostingThreadState initialValue() {
		//这个ThreadLocal的初始化就是new一个新的PostingThreadState();
		return new PostingThreadState();
	}
};
```

**2C.一个静态常量内部类**
``` java
/* For ThreadLocal, much faster to set (and get multiple values). */
//为了ThreadLocal可以更快的set和get？这里就是之前看threadLocal时提到的，postingThreadState用到了threadLocal
final static class PostingThreadState {
	final List<Object> eventQueue = new ArrayList<Object>();
	boolean isPosting;
	boolean isMainThread;
	Subscription subscription;
	Object event;
	boolean canceled;
}
```

**B**.
```java
public void register(Object subscriber) {
//首先通过反射，获取订阅者的类
Class<?> subscriberClass = subscriber.getClass();
// 然后通过类，找类的方法，不是简单的用getMethod()
List\<SubscriberMethod\> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);--->C，C~S都是这个函数的延伸，主要是两种方法来查找订阅者的方法
	synchronized (this) {
		for (SubscriberMethod subscriberMethod : subscriberMethods) {
			// 遍历订阅者的方法，然后执行subscribe--->1A，重点就是处理subscriberMethod，其实不知道放进去在哪一步被执行...
			subscribe(subscriber, subscriberMethod);
		}
	}	
}
```

**1A.**
**1开头的，都是这个函数延伸出去的调用。**
``` java
// 涉及到数据结构：private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
/* Must be called in synchronized block*
*订阅者subscriber*/
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod){
	//订阅者的方法要订阅的事件类
	Class<?> eventType = subscriberMethod.eventType;
	Subscription newSubscription = new Subscription(subscriber,
	subscriberMethod);--->1B,封装了这个订阅者和订阅者的响应函数以及是否订阅这三个信息
	CopyOnWriteArrayList<Subscription> subscriptions =
	subscriptionsByEventType.get(eventType); 
	//根据eventType取出订阅这个情况的所有订阅情况
	if (subscriptions == null) {
		//如果这个event还从未被订阅，自然新建一个它所对应的subscription
		subscriptions = new CopyOnWriteArrayList<>();	
		subscriptionsByEventType.put(eventType, subscriptions);
	} else {
		//如果这个订阅关系已经存在， 抛出异常。Subscriber..already registered to event，和自己理解的一样~~~~ 开心
		if (subscriptions.contains(newSubscription)) {
			throw new EventBusException("Subscriber " + subscriber.getClass() +"already registered to event "+ eventType);
		}
	}

	// 这个事件有多少个订阅方法（\@subscribe的方法或者onEvent..)
	int size = subscriptions.size();
	for (int i = 0; i <= size; i++) {
		// 如果当前这个订阅方法的优先级比当前这个订阅方法的优先级高，就把它加入到第i位（也就是倒数第二位），然后break。
		// 如果直接到了size，说明当前方法优先级最低，直接插入尾部。
		if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
			subscriptions.add(i, newSubscription);
			break;
		}
	}
	//涉及到private final Map<Object, List<Class<?>>> typesBySubscriber;
	//查看这个订阅者要订阅的事件类型
	List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
	if (subscribedEvents == null) {
		//原来这个subscriber没有订阅过事件，那么现在增加订阅者的 subscribedEvents(是一个list）
		subscribedEvents = new ArrayList<>();
		typesBySubscriber.put(subscriber, subscribedEvents);
	}
	//不管subscriber是否订阅过事件,都把这个事件类型加入到这个subscriber的订阅事件列表中
	subscribedEvents.add(eventType);
	//如果这个响应函数是粘性的。 如果不是粘性的？？哪里涉及到线程去执行？
	//个人理解：
	这是3.0增加的新特性：从代码可以看出stickyEvents的entry，其实是一个map，map的key是事件类型（class\<?\>），value是事件实例(Object),由于是粘性事件，那么，当这个subscriberMethod订阅这个事件类型，就把最近的事件类型的事件（Object)
通过checkPostStickyEventToSubscription立刻将这个事件找到对应的处理线程，进行排队（有可能排）后，由对应的subcriberMethod进行处理。
	if (subscriberMethod.sticky) {
		if (eventInheritance) {
		/* Existing sticky events of all subclasses of eventType have to be
		considered.这个事件类型的所有子类的粘性事件都要考虑
		Note: Iterating over all events may be inefficient with lots of sticky events, 如果粘性事件比较多，遍历所有的事件可能不够高效
		thus data structure should be changed to allow a more efficient lookup*应该更改数据结构，进行更高效的遍历  (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).*/
			Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
		    //取出保存粘性事件的set
			for (Map.Entry<Class<?>, Object> entry : entries) {
			//遍历set中每个元素，取出每个元素的key，也就是事件类型class<?>
				Class<?> candidateEventType = entry.getKey();
				//如果当前订阅方法监听的事件类型，是之前存在的事件的 父类
				if (eventType.isAssignableFrom(candidateEventType)) {
					// 取出value,也就是这个事件类型的实例
					Object stickyEvent = entry.getValue();
					checkPostStickyEventToSubscription(newSubscription, stickyEvent);---->1C
				}
			}
		} else {
			Object stickyEvent = stickyEvents.get(eventType);
			checkPostStickyEventToSubscription(newSubscription, stickyEvent);
		}
	}
}
```

**1C. 主要是调用了postToSubscription，选择了合适的线程，执行了post**

``` java
private void checkPostStickyEventToSubscription(Subscription
newSubscription, Object stickyEvent) {
	if (stickyEvent != null) {
		/*If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state) 如果订阅者想阻止这个事件，将会失败--> Strange corner case, which we don't take care of here.奇怪的case，我们不用在意？？？ 这注释有点迷...*/
		postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());--->1D,真正处理event的函数（终于涉及了线程），赞！
	}
}
```

**1D.根据订阅者方法的threadMode，决定这个方法由哪个线程执行。**

**由各个poster去处理pengdingPost，可能由handler执行，也可能是线程池去执行runnable**

```java
private void postToSubscription(Subscription subscription, Object event,
boolean isMainThread) {
	switch (subscription.subscriberMethod.threadMode) {
		case POSTING:
			invokeSubscriber(subscription,event);--->1E，通过java反射机制，使用method.invoke()调用了subscirber对象中的参数为event订阅方法
			break;
		case MAIN:
			if (isMainThread) {
				//如果当前就是主线程，那么就当前线程执行subscription.subscriberMethod
				invokeSubscriber(subscription, event);
			} else {
				//mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);有主线程的looper
				mainThreadPoster.enqueue(subscription, event);---->1F，插入给主线程的handler，会将这个事件和方法放入queue，等待主线程执行，如果主线程的queue之前为空，那么就直接sendMsg。
			}
			break;
		case BACKGROUND:
			if (isMainThread) {
				//backgroundPoster = new BackgroundPoster(this);同理，这是后台线程的handler
				backgroundPoster.enqueue(subscription, event);--->1I
			} else {
				//如果当前线程不是主线程，就直接用当前线程执行，invokeSubscriber是当前的eventBus中的函数
				invokeSubscriber(subscription, event);
			}
			break;
		case ASYNC:
			asyncPoster.enqueue(subscription, event);--->1J
			break;
		default:
			throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
	}
}
```

**1E.**
``` java
void invokeSubscriber(Subscription subscription, Object event) {
	try {
		//method.invoke(Object obj, Object... args) 是反射机制的一个函数
		//对这个指定的对象obj，调用方法method，args是这个方法需要的参数数组。
		//也就是，调用subscriber这个对象的subscription.subscriberMethod.method方法，其中event是这个方法的参数...JAVA反射好厉害-
		subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
	} catch (InvocationTargetException e) {
		handleSubscriberException(subscription, event, e.getCause());
	} catch (IllegalAccessException e) {
		throw new IllegalStateException("Unexpected exception", e);
	}
}
```

**1K. 差点以为是1E...**

``` java
void invokeSubscriber(PendingPost pendingPost) {
	Object event = pendingPost.event; //取出这个post对应的event obj
	//再取出对应的订阅信息
	Subscription subscription = pendingPost.subscription;
	PendingPost.releasePendingPost(pendingPost);---->1L
	if (subscription.active) {
	//这个变量注释就说是这里会用到，为了避免race conditions（资源竞争）
		invokeSubscriber(subscription, event);--->1E
	}
}
```

**SubscriberMethodFinder.JAVA**

**C.**
``` java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass){
	//先直接从缓存中取
	List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
	if (subscriberMethods != null) {
		return subscriberMethods;
	}
	//如果没有的话，判断是否要忽略注解器：
	if (ignoreGeneratedIndex) {
		//忽略的话，用反射来获取类的方法信息
		subscriberMethods = findUsingReflection(subscriberClass);--->D
		/*D~N，都是利用反射找方法的过程，主要是通过findState，把subscriberClass放进去，找到对应方法，再找它的父类，依次向上。总体来说findUsingReflection就是一个while循环遍历所有相关类，根据类找方法时需要用到反射和注解...*/
	} else {
		subscriberMethods = findUsingInfo(subscriberClass); --->O
	}
	if (subscriberMethods.isEmpty()) {
		throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the \@Subscribe annotation");
	} else {
		METHOD_CACHE.put(subscriberClass, subscriberMethods);
		return subscriberMethods;
	}
}
```

**P.**

``` java
private SubscriberInfo getSubscriberInfo(FindState findState) {
	// 看描述subscriberInfo本来就一直为null啊... 是一个接口的变量
	// getSuperSubscriberInfo()是接口的一个函数，AbstractSubscriberInfo是接口实现
	if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) { ---->Q
		//如果订阅者信息不为空且订阅者信息子类不为空
		SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
		//如果findstate当前的clazz和订阅者子类一样，返回这个子类信息。
		if (findState.clazz == superclassInfo.getSubscriberClass()) {
			return superclassInfo;
		}
	}
	//如果订阅者信息下标不为空（是一个List),那么遍历List，通过index取出clazz的信息，并返回。
	if (subscriberInfoIndexes != null) {
		for (SubscriberInfoIndex index : subscriberInfoIndexes) {
			SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
			if (info != null) {
				return info;
			}
		}
	}
	return null;
}
```

**O.看看和DList<SubscriberMethod> findUsingReflection(Class<?>
subscriberClass)有什么不同**
``` java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
	FindState findState = prepareFindState(); // same
	findState.initForSubscriber(subscriberClass); // same
	while (findState.clazz != null) { // same
		findState.subscriberInfo =
getSubscriberInfo(findState);--->P,经过PQ，取到订阅者的信息
		if (findState.subscriberInfo != null) { //订阅者信息不为空
			SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();--->R//调用接口函数，根据方法信息构建方法，并返回
		for (SubscriberMethod subscriberMethod : array) {
		// 遍历方法，但是否能将订阅者方法和要监听的事件类型添加到findState中
			if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
			// 将这个方法加入到findState的list中去
				findState.subscriberMethods.add(subscriberMethod);
			}
		}
	} else {
	//订阅者信息为空，那就通过这个方法，用反射得到findState.clazz的相关信息，并构造订阅方法（new SubscriberMethod（....)），add进findState中
		findUsingReflectionInSingleClass(findState);
	}
	findState.moveToSuperclass(); //继续往上找
}
	return getMethodsAndRelease(findState); //释放这个findState
}

D.
private List<SubscriberMethod> findUsingReflection(Class<?>
subscriberClass) {
	FindState findState = prepareFindState();---->E,得到一个FindState
	findState.initForSubscriber(subscriberClass);--->F,为这个findstate赋值它记在的订阅类，也就是subscriberClass，这是起点，此时findState中的clazz就是它
	while (findState.clazz != null) {
		findUsingReflectionInSingleClass(findState);---->G
	/*G到K，终于把subscriberClass里面的订阅方法加入到相应的findState了
	findState.moveToSuperclass();--->L,把findState的clazz换成clazz的父类（超类不能为系统类）
	注意这是一个while循环，所以是从子类开始一直向上，把相应clazz的方法加入到findState*/
}
	return getMethodsAndRelease(findState);----->M，释放findstate,while循环结束它就功成身退了，把它还给statePool
}
```

E.从findStatePool中取出第一个不为空的state，取出后把池中这个位置的state置为空。
有可能池为空，那么直接new一个FindState();

``` java
private FindState prepareFindState() {
	synchronized (FIND_STATE_POOL) {
		for (int i = 0; i < POOL_SIZE; i++) {	
			FindState state = FIND_STATE_POOL[i];
			if (state != null) {
				FIND_STATE_POOL[i] = null;
				return state;
			}
		}
	}
	return new FindState();
}
```

**G.利用反射原理取出当前findState中的订阅类的方法**

``` java
private void findUsingReflectionInSingleClass(FindState findState) {
	Method[] methods;
	try {
	// This is faster than getMethods, especially when subscribers are fat classes
	like Activities
	/*getMethods()返回某个类的所有公用（public）方法包括其继承类的公用方法，当然也包括它所实现接口的方法。getDeclaredMethods()对象表示的类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。当然也包括它所实现接口的方法。*/
	methods = findState.clazz.getDeclaredMethods();
	} catch (Throwable th) {
	/* Workaround for java.lang.NoClassDefFoundError, see
	https://github.com/greenrobot/EventBus/issues/149*
	*/
		methods = findState.clazz.getMethods();
		findState.skipSuperClasses = true;
	}
	for (Method method : methods) {
	//返回类或接口的以整数编码的JAVA语言修饰符，比如得到整数i=1，那么Modefier.toString(i)可以得到String的public
	int modifiers = method.getModifiers();
	//如果是public，不是abstract,不是static，不是BRIDGE,不是SYNTHETIC？
	if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE)
== 0){
	//获得方法的各参数的类型数组，不管类型是否一样，都算1，比如 int string
	int，那么共3个
	Class<?>[] parameterTypes = method.getParameterTypes();
	// 3.0开始，不是onEvent..开头的方法，也可以作为响应函数了
	// 但是必须只有一个参数（也就是监听的事件event这个参数
	if (parameterTypes.length == 1) {
		//获得Subscribe.class的注解，这个注解标签（定义的名称）是Subscribe。
		// 有@Subscribe 注解的，都是响应函数，也就是不仅能用onEvent..这四个函数了
		Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
		if (subscribeAnnotation != null) { // 如果\@Subscribe不为空
			Class<?> eventType = parameterTypes[0]; //取出参数对应的类
			if (findState.checkAdd(method, eventType)) {----->H，一系列检查看是否能为这个事件类型添加对应的方法
				ThreadMode threadMode = subscribeAnnotation.threadMode();--->
			findState.subscriberMethods.add(new SubscriberMethod(method, eventType,
threadMode,subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
			}
		//把这个订阅方法以及相关的事件类型、响应模式等封装后，加入到findState保存的订阅者方法的list中去，这是newSubscriberMethod的起点，顺便去看看这个类
		}
	} else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
			// 如果不止一个参数类型并且.....
			// 获取这个方法的所在的类名，以及方法名
			String methodName = method.getDeclaringClass().getName() + "." +
		method.getName();
			throw new EventBusException("\@Subscribe method " + methodName +
			"must have exactly 1 parameter but has " + parameterTypes.length);
		}
	} else if (strictMethodVerification &&
		method.isAnnotationPresent(Subscribe.class)) { 
			//这个elseif对应的是最外层那个public && 非abstract ....
			String methodName = method.getDeclaringClass().getName() + "." + method.getName();
			throw new EventBusException(methodName + " is a illegal \@Subscribe method: must be public, non-static, and non-abstract");
		}
	}
}
```
M.

``` java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState){
	//把findState里面的subscriberMethods列表复制过来
	List<SubscriberMethod> subscriberMethods = new
	ArrayList<>(findState.subscriberMethods);
	findState.recycle(); --->N ,就是把findState的各个成员变量清空。
	synchronized (FIND_STATE_POOL) {
		// 从前开始遍历，把这个findstate还给这个pool
		// 之前取的时候，把[i]位置置为了null，所以这里还回去时候找值为null的=
		// 这思路真是简单易懂- -
		for (int i = 0; i < POOL_SIZE; i++) {
			if (FIND_STATE_POOL[i] == null) {
				FIND_STATE_POOL[i] = findState;
				break;
			}
		}
	}
	return subscriberMethods;
}
```

SubscriberMethodFinder.java中的一个静态内部类：
static class FindState {}
成员变量：
``` java
//subscriber的订阅方法
final List<SubscriberMethod> subscriberMethods = new
ArrayList<>();
//订阅方法的事件类型
final Map<Class, Object> anyMethodByEventType = new HashMap<>();
final Map<String, Class> subscriberClassByMethodKey = new
HashMap<>();
final StringBuilder methodKeyBuilder = new StringBuilder(128);
Class<?> subscriberClass;
Class<?> clazz;
boolean skipSuperClasses;
SubscriberInfo subscriberInfo;
//主要保存了订阅类的所有订阅方法信息.
```
**N.** 
***集合类用clear()，builder用setLength(0),对象实例用null。***
``` java
void recycle() {
	subscriberMethods.clear();
	anyMethodByEventType.clear();
	subscriberClassByMethodKey.clear();
	methodKeyBuilder.setLength(0);
	subscriberClass = null;
	clazz = null;
	skipSuperClasses = false;
	subscriberInfo = null;
}
```

**F**.
``` java
void initForSubscriber(Class<?> subscriberClass) {
	this.subscriberClass = clazz = subscriberClass;
	skipSuperClasses = false;
	subscriberInfo = null;
}
```
**H**.
**检查是否添加这个方法，**checkAddWithMethodSignature这个方法有点迷惑...记得回来看

两层检查，第一层检查事件类型，第二层检查签名（方法名称+参数类型）

``` java
boolean checkAdd(Method method, Class<?> eventType) {
	/* 2 level check: 1st level with event type only (fast), 2nd level with
	complete signature when required.
	Usually a subscriber doesn't have methods listening to the same event type.*/
	// map的put方法，如果之前Key没有对应的value，那么返回null，否则返回之前对应的值
	Object existing = anyMethodByEventType.put(eventType, method);
	if (existing == null) {
	//说明之前这个key没有对应值，那么这个类型可以对应这个方法
	//也就是这个事件只有当前这种方法订阅了
	return true;
	} else {
	// 如果之前订阅这个事件A的method和当前想订阅事件A的method类型一样
		if (existing instanceof Method) {
			if (!checkAddWithMethodSignature((Method) existing, eventType))----->I{ // 如果之前的classOld是现在methodClass的子类，那么抛出异常
				// Paranoia check
				// 也就是之前存在的classOld是当前方法的子类时，不用重新订阅，需要遵循子类覆写。
				throw new IllegalStateException();
			}
			// Put any non-Method object to "consume" the existing Method
			//没有出现父类想和子类抢同一个事件的异常，那么这个事件类型的订阅者就是this
			anyMethodByEventType.put(eventType, this);
		}
		return checkAddWithMethodSignature(method, eventType);
	}
}
```
**I.方法签名的检查**

这是一个优化，如果现在保存的是旧的子类，那么保存新的，返回true
否则没必要保存，直接保存旧的那个父类就行了。

``` java
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
	methodKeyBuilder.setLength(0);
	methodKeyBuilder.append(method.getName()); //方法名
	methodKeyBuilder.append('>').append(eventType.getName());
	// 类名即事件类型名称
	// 用这两个值当做key
	String methodKey = methodKeyBuilder.toString();
	// 得到这个方法所对应的类对象
	Class<?> methodClass = method.getDeclaringClass();
	Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
	//methodClassOld为空，或者是父类
	//methodClassOld为空时：说明这个方法签名（methodKey=方法名+事件类型)之前没有彼此对应。那么允许为这个方法添加订阅事件，返回true
	if (methodClassOld == null ||
	methodClassOld.isAssignableFrom(methodClass)) {
		//这个函数，Class.isAssignableFrom()是用来判断一个类Class1是否Class2的superClass或者superInterface，或者是相同类或者接口。
		格式为： Class1.isAssignableFrom(Class2)
		这是判断类之间的关系，instanceof是判断实例之间的关系
		// Only add if not already found in a sub class
		return true;
	} else {
		// Revert the put, old class is further down the class hierarchy
		/*methodClassOld不为空，并且它是新的methodClass的子类，把methodClassOld放回去并且返回false，**代表子类classOld覆写了父类的订阅方法，让子类method的订阅方法订阅这个事件而不是父类。*/
		subscriberClassByMethodKey.put(methodKey,methodClassOld);
		return false;
	}
}
```

**L.**
``` java
void moveToSuperclass() {
	if (skipSuperClasses) {
		clazz = null;
	} else {
		clazz = clazz.getSuperclass();
		String clazzName = clazz.getName();
		/* Skip system classes, this just degrades performance. */
		//跳过系统类，也就是以java, javax,android开头的类
		if (clazzName.startsWith("java.") ||clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
			clazz = null;
		}
	}
}
```

**③注解类\@Subscribe**

**J.**

终于遇到注解了！！看别人的分析，注解中应该还有 
String[] actions() default {};  

这个版本好像没有了！！

``` java
@Documented
@Retention(RetentionPolicy.*RUNTIME*)
@Target({ElementType.*METHOD*})
public @interface Subscribe {
	ThreadMode threadMode() default
	ThreadMode.POSTING;--->K
	/*如果是真，如果有订阅者在这个sticky事件发布后才订阅这个事件，那么依然会把最近的这个事件类型的sticky事件发送给这个subscriber*/
	/* If true, delivers the most recent sticky event (posted with {\@link EventBus #postSticky(Object)}) to this subscriber (if event available).*/
	boolean sticky() default false;
	/* 对于同一个传送线程，优先级高的订阅者更快地接收到事件
	Subscriber priority to influence the order of event delivery.*/
	/* Within the same delivery thread ({\@link ThreadMode}), higher priority subscribers will receive events before*/ others with a lower priority. The default priority is 0. Note: the priority
	does \*NOT\* affect the order of delivery among subscribers with different {\@link ThreadMode}s! */
	int priority() default 0;
}
```
**K.**
public enum ThreadMode{}
枚举类型，共四种：
*POSTING*：订阅者的响应函数直接由发起post事件的线程执行，如果发起post的是主线程，那么响应函数不应该是耗时操作。如果要执行的task耗时很短，建议使用这种模式。
*MAIN*: 订阅者的响应函数由主线程执行。如果posting
thread是主线程，事件响应函数将会直接被这个线程call。
*BACKGROUND*：订阅者的响应函数由后台线程执行，如果posting
thread不是主线程，事件响应函数将会直接被这个线程call。如果posting
thread是主线程，那么eventBus将会使用一个单独的single后台线程，这个线程会按序执行这些响应函数。但还是不建议有很耗时的操作，以避免阻塞这个后台线程。
*ASYNC：*
订阅者的响应函数由一个独立的线程去执行，这个线程不是主线程，也不是posting
thread。如果有比较耗时的操作，建议用这种模式，比如网络访问。EventBus使用了线程池来重用线程进行异步操作。

**SubscriberMethod.JAVA**

``` java
public class SubscriberMethod {
	final Method method;
	final ThreadMode threadMode;
	final Class<?> eventType;
	final int priority;
	final boolean sticky;
	/* Used for efficient comparison */
	String methodString;
....
}
```

构造函数就是对前五个变量进行赋值，很好理解。

**AbstractSubscriberInfo.JAVA，实现SubscriberInfo接口**
public abstract class AbstractSubscriberInfo implements SubscriberInfo
{}
private final Class subscriberClass;
private final Class<? extends SubscriberInfo>
superSubscriberInfoClass;
private final boolean shouldCheckSuperclass;
主要是三个成员变量：
订阅者对应的类；订阅者信息的子类（这是什么鬼...); 检查超类的布尔变量


**P.**

``` java
@Override
public SubscriberInfo getSuperSubscriberInfo() {
	// 如果没有信息子类，直接返回Null
	if(superSubscriberInfoClass == null) {
		return null;
	}
	try {
		return superSubscriberInfoClass.newInstance();
	} catch (InstantiationException e) {
		throw new RuntimeException(e);
	} catch (IllegalAccessException e) {
		throw new RuntimeException(e);
	}
}
```

**S.**
``` java
simpleSubscriberInfo中的getSubscriberMethods()会调用该函数：
protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType, ThreadMode threadMode,int priority, boolean sticky) {
	try {
		Method method = subscriberClass.getDeclaredMethod(methodName, eventType);
		//通过方法名称，和方法参数列表（其实就是方法签名）从class中找到类
		return new SubscriberMethod(method, eventType, threadMode, priority, sticky); //然后构造subscriber方法，似乎比第一种通过反射的方式简单
	} catch (NoSuchMethodException e) {
		throw new EventBusException(**"Could not find subscriber method in " + subscriberClass + ". Maybe a missing ProGuard rule?", e);
	}
}
```

**⑥SimpleSubscriberInfo.JAVA
又继承了AbstractSubscriberInfo（就不能一次写完吗...)
一个成员变量：
``` java
private final SubscriberMethodInfo[] methodInfos;
// 存储了订阅者所有的方法信息，用数组保存。
```
**R.**
``` java
@Override
public synchronized SubscriberMethod[] getSubscriberMethods() {
	int length = methodInfos.length;
	//建立一个方法数组（不是方法信息）
	SubscriberMethod[] methods = new SubscriberMethod[length];
	for (int i = 0; i < length; i++) {
		//取出方法信息	
		SubscriberMethodInfo info = methodInfos[i];
		//根据信息建立方法（调用父类的createSubscriberMethod）
		methods[i] = createSubscriberMethod(info.methodName, info.eventType, info.threadMode, info.priority, info.sticky);---->S
	}
	return methods; //根据信息构造了方法数组，返回去。
}
```

**⑦Subscription.JAVA**
``` java
final class Subscription {}
//三个成员变量：
final Object subscriber; //注册的订阅者
final SubscriberMethod subscriberMethod; //订阅者响应事件的函数
/*Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.*/

volatile boolean active; // unrigester之后为false
```

**1B.**

构造函数就是为这三个变量赋值：
``` java
Subscription(Object subscriber, SubscriberMethod subscriberMethod){
	this.subscriber = subscriber;
	this.subscriberMethod = subscriberMethod;
	active = true;
}
```

**⑧HandlerPoster.JAVA**

``` java
final class HandlerPoster extends Handler {}
	//四个成员变量：
	private final PendingPostQueue queue;
	private final int maxMillisInsideHandleMessage;
	private final EventBus eventBus;
	private boolean handlerActive;
```

**1F.**
``` java
void enqueue(Subscription subscription, Object event) {
	PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);---> 1G,根据订阅信息和event取出pendingPost，
		synchronized (this) {
			queue.enqueue(pendingPost); //插入等待队列
			if (!handlerActive) {
			//如果handler不活跃，就打开它（可能之前没有msg给handler处理）直接发送消息进行处理
				handlerActive = true;
			if (!sendMessage(obtainMessage())) { 
			//handler发送msg出去---->1H
				throw new EventBusException(**"Could not send handler message");
			}
		}
	}
}
```

**1H.消息处理函数**，但是handleMessage有三个级别,handler的是最不优先的- -

``` java
@Override
public void handleMessage(Message msg) {
	boolean rescheduled = false;
	try {
		long started = SystemClock.uptimeMillis();
		while (true) {
			PendingPost pendingPost = queue.poll();
			//移出并返回队头，如果queue是空的，就返回null，不会抛异常
			if (pendingPost == null) {
			//如果队列为空
				synchronized (this) { //锁住当前handler
				// Check again, this time in synchronized*
					pendingPost = queue.poll(); //再取？
					if (pendingPost == null) {
					//还是空，那么这个handler没事干了
						handlerActive = false;
						return;
					}
				}
			}
		//又调用eventBus的invokeSubscriber函数，1E已经分析了。利用反射调用订阅方法。
		eventBus.invokeSubscriber(pendingPost);---->不是1E！参数不同！1K,不过最终还是可能调用1E
		long timeInMethod = SystemClock.uptimeMillis() - started;
		//如果执行method的时间大于了handleMsg规定的时间
		if (timeInMethod >= maxMillisInsideHandleMessage) {
			if (!sendMessage(obtainMessage())) { //又发送出去执行？？
				throw new EventBusException("Could not send handler message");
			}
			rescheduled = true;// 重新调度了
			return;
		}
	}
} finally {
	handlerActive = rescheduled; //如果发生了重新调度，handlerActive为真
	}
}
```

**⑨PendingPost.JAVA**

四个成员变量：
``` java
private final static List<PendingPost> pendingPostPool = new
ArrayList<PendingPost>();
Object event;
Subscription subscription;
PendingPost next;
```

一个构造函数：
``` java
private PendingPost(Object event, Subscription subscription) {
	this.event = event;
	this.subscription = subscription;
}
```

**1G.**
``` java
static PendingPost obtainPendingPost(Subscription subscription, Object event) {
	synchronized (pendingPostPool) {
		int size = pendingPostPool.size();
		if (size > 0) {
		//取出postPool中最后一个pendingPost
			PendingPost pendingPost = pendingPostPool.remove(size - 1);
			pendingPost.event = event;
			pendingPost.subscription = subscription;
			pendingPost.next = null;
			return pendingPost;
		}
	}
	//如果pengdingPostPool为空，直接用事件和订阅信息new一个新的pendingPost
	return new PendingPost(event, subscription);
}
```

**1L.**

``` java
static void releasePendingPost(PendingPost pendingPost) {
	//将pendingPost的各变量置为null
	pendingPost.event = null;
	pendingPost.subscription = null;
	pendingPost.next = null;
	synchronized (pendingPostPool) {
		/*锁住postPool，把这个pendingPost放进去= =话说，为什么不直接pendingPost=null...恩，如果里面的这三个变量不为null，那么应该无法回收，但是也可以先把变量置为null,再把pendingPost=null
		但是这里直接把变量置为null然后重用？ 所以这是传说中的对象池...恩
		// Don't let the pool grow indefinitely
		if (pendingPostPool.size() < 10000) {
			pendingPostPool.add(pendingPost);
		}
	}
}
```

**⑩BackgroundPoster.java**

``` java
final class BackgroundPoster implements Runnable {}
//是一个线程,三个成员变量：
private final PendingPostQueue queue; //等到这个线程处理的事件队列
private final EventBus eventBus;
private volatile boolean executorRunning;
```

构造函数：
``` java
BackgroundPoster(EventBus eventBus) {
	this.eventBus = eventBus;
	queue = new PendingPostQueue();
}
```
持有了eventBus的引用，也许有内存泄漏？

**1I.**
``` java
public void enqueue(Subscription subscription, Object event) {
PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
//根据subscription和要发布的事件构造pendingPost
	synchronized (this) {
		queue.enqueue(pendingPost); //入队
		if (!executorRunning) { //这个runnable此时没有在执行
			executorRunning = true;//那么可以利用起来了
			//eventBus的线程池执行当前runnable
			eventBus.getExecutorService().execute(**this**);
		}
	}
}
```

**⑪AsyncPoster.JAVA**

三个成员变量：
``` java
private final PendingPostQueue queue;
private final EventBus eventBus;
```

构造函数：持有了eventBus的引用
``` java
AsyncPoster(EventBus eventBus) {
	this.eventBus = eventBus;
	queue = new PendingPostQueue();
}
```

**1J.**

``` java
//构造了pendingPost后直接进入队列，执行让eventBus的线程池执行它= =
public void enqueue(Subscription subscription, Object event) {
	PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
	queue.enqueue(pendingPost);
	eventBus.getExecutorService().execute(this);
}
```

``` java
@Override
public void run() {
	//从队列里面取出pendingPost
	PendingPost pendingPost = queue.poll();
	if(pendingPost == null) {
		throw new IllegalStateException("No pending post available");
	}
	/*1E分析过，反射机制执行subscription里面的method（打脸，这不是1E
	eventBus.invokeSubscriber(pendingPost);
}
```