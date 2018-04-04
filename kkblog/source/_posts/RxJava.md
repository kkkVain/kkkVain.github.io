---
title: RxJAVA初步理解(1)
date: 2017-09-11 23:29:17
tags: [RxJava,Android,源码]
categories: Android
---

**RxJAVA**

① subscribe函数：

调用了observable.onSubscribe.call(subscriber)
observable:最后一个变换后的observable
onSubscribe创建最后一个observable时传入的参数
``` java
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
	ObjectHelper.requireNonNull(observer, "observer is null");
	try {
	// onXXXX相关的，和刚才的onAssembly是一个套路，同样的使用了Hook
		observer = RxJavaPlugins.onSubscribe(this, observer);
		ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
		//开始订阅，这个subscribeActual函数，作为onAssembly()的参数时的类，都有重写，
		subscribeActual(observer);-----> 2A
	} catch (NullPointerException e) { // NOPMD
		throw e;
	} catch (Throwable e) {Exceptions.throwIfFatal(e);
		// can't call onError because no way to know if a Disposable has been set or not can't call onSubscribe because the call might have set a Subscription
		already RxJavaPlugins.onError(e);
		NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
			npe.initCause(e);
			throw npe;
	}
}
```

②create函数
onJust：
A.
``` java
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> just(T item) {
	ObjectHelper.requireNonNull(item, "The item is null");
	return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));---->B
}
```

**B.**
所有onXXXX的函数，都是hook function.
onAssembly这个函数：把传进来的各种new
ObservableXXX<T>转换为Observable<T>

``` java
Calls the associated hook function.
/* @param <T> the value type
   @param source the hook's input value
   @return the value returned by the hook
*/

@SuppressWarnings({ "rawtypes", "unchecked" })
public static <T> Observable<T> onAssembly(Observable<T> source) {
	Function<Observable, Observable> f = onObservableAssembly; // 初始值是null
	if (f != null) {
		return apply(f,source); ---\>C,函数f是源码自带的onObservableAssembly
	}
	return source;
}
```

**C.**
``` java
/* Wraps the call to the function in try-catch and propagates thrown checked exceptions as RuntimeException.
@param <T> the input type
@param <R> the output type
@param f the function to call, not null (not verified)
@param t the parameter value to the function
@return the result of the function call

static <T, R> R apply(Function<T, R> f, T t) {
	try {
		return f.apply(t); --->D，自带的函数f进行apply
	} catch (Throwable ex) {
		throw ExceptionHelper.wrapOrThrow(ex);
	}
}
```

**D.**
``` java
/* A functional interface that takes a value and returns another value, possibly with a different type and allows throwing a checked exception.
@param <T> the input value type
@param <R> the output value type

public interface Function<T, R> {
/* Apply some calculation to the input value and return some other value.*/
	@param t the input value
	@return the output value
	@throws Exception on error
	
	R apply(T t) throws Exception;
}
```

**E.subscribeOn:**

``` java
@param scheduler the {@link Scheduler} to perform subscription actions on
@return the source ObservableSource modified so that its subscriptions happen on the specified {@link Scheduler}
@see <a href="http://reactivex.io/documentation/operators/subscribeon.html">ReactiveX operators documentation: SubscribeOn</a>
@see <a href="http://www.grahamlea.com/2014/07/rxjava-threading-examples/">RxJava Threading Examples</a>
@see #observeOn
@SchedulerSupport(SchedulerSupport.CUSTOM)

public final Observable<T> subscribeOn(Scheduler scheduler) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
		return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this,scheduler));
}
```

onAssembly就是刚才分析的那个，把当前的observable和scheduler整合，并且以observable<t>的形式返回。

**F.**
``` java
@param <R> the value type of the inner ObservableSources and the output type
@param mapper a function that, when applied to an item emitted by the source
ObservableSource, returns an ObservableSource
@return an Observable that emits the result of applying the transformation function to each item emitted by the source ObservableSource and merging the results of the
ObservableSources obtained from this transformation
@see <a href="http://reactivex.io/documentation/operators/flatmap.html"\>ReactiveX operators documentation: FlatMap</a>
@SchedulerSupport(SchedulerSupport.NONE)

public final <R> Observable<R> flatMap(Function<? super
T, ? extends ObservableSource<? extends R>> mapper) {
	return flatMap(mapper, false); ----> G
}
```

**G.**
``` java
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors, int maxConcurrency, int bufferSize) {
	ObjectHelper.requireNonNull(mapper, "mapper is null");
	ObjectHelper.verifyPositive(maxConcurrency, "maxConcurrency");
	ObjectHelper.verifyPositive(bufferSize, "bufferSize");
	if (this instanceof ScalarCallable) {
		@SuppressWarnings("unchecked")
		T v = ((ScalarCallable<T>)this).call(); ---->this可能会重写call函数（这不就是flatmap实际使用时重写的call函数吗....(lambda传入的那个函数）
		if (v == null) {
			return empty(); ---> H
		}
		return ObservableScalarXMap.scalarXMap(v,mapper); ---> I
	}
	return RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
}
```

**H.**
``` java
@param <T> the type of the items (ostensibly) emitted by the ObservableSource
@return an Observable that emits no items to the {\@link Observer} but immediately invokes the {@link Observer}'s {@link Observer#onComplete() onComplete} method
@see <a href="http://reactivex.io/documentation/operators/empty-never-throw.html">ReactiveX operators documentation: Empty</a>


@SchedulerSupport(SchedulerSupport.NONE)
@SuppressWarnings("unchecked")
public static <T> Observable<T> empty() {
	return RxJavaPlugins.onAssembly((Observable<T> ObservableEmpty.INSTANCE);
}
```

**I.**
``` java
/*Maps a scalar value into an Observable and emits its values.*/
@param <T> the scalar value type
@param <U> the output value type
@param value the scalar value to map
@param mapper the function that gets the scalar value and should return an ObservableSource that gets streamed
@return the new Observable instance*/

public static <T, U> Observable<U> scalarXMap(T value, Function<? super T, ? extends ObservableSource<? extends U>>
mapper) {
	return RxJavaPlugins.onAssembly(new ScalarXMapObservable<T,
U>(value, mapper));--->J 同样的，new一个observable
}
```

**J.**
``` java
/* 把标准值T映射为observableSource，并且订阅他 Maps a scalar value to an ObservableSource and subscribes to it.*/
@param <T> the scalar value type
@param \<R\> the mapped ObservableSource's element type.
static final class ScalarXMapObservable <T, R> extends
Observable<R> {
	final T value;
	final Function<? super T, ? extends ObservableSource<? extends R>> mapper;
	ScalarXMapObservable(T value, Function <? super T, ? extends ObservableSource<? extends R>> mapper) {
		this.value = value;
		this.mapper = mapper;
	}
}
```

2A.
``` j
public final class ObservableCreate<T> extends Observable<T> {}
@Override
protected void subscribeActual(Observer<? super T> observer) {
	//创建发射器，这个observer是我们在应用层自己定义的observer（重写了onNext,onSuccess等函数)
	CreateEmitter<T> parent = new CreateEmitter<T>(observer);
	observer.onSubscribe(parent); // 每个observer都要重写的四个函数之一
	try {
		source.subscribe(parent); // 这个source是每个onXXXX(new ObservableXXX(ffff))中new这个类时传入的参数（其实就是我们在应用层调用create（ffff）时的参数ffff,那么，每个类，比如create的参数类ObservableOnSubscribe<T>，都会重写属于自己的subscribe函数。这样就创建了一个emmiter
	} catch (Throwable ex) {
		Exceptions.throwIfFatal(ex);
		parent.onError(ex);
	}
}
```

**K.**

``` java
public static final class ScalarDisposable<T> extends AtomicInteger
implements QueueDisposable <T>, Runnable {}
public ScalarDisposable(Observer <? super T> observer, T
value) {
	this.observer = observer;
	this.value = value;
}
```

**L.**
``` java
@Override
public void run() {
// 继承了原子类，又用了CAS = =
	if (get() == START && compareAndSet(START, ON_NEXT)) {
		observer.onNext(value);
		if (get() == ON_NEXT) {
			lazySet(ON_COMPLETE); 
			bserver.onComplete();
		}	
	}
}
```

* 复习一下原子类
``` java
	public final void lazySet(int newValue) {
		U.putOrderedInt(this, VALUE, newValue);
	}
```
原子类设置值的时候像volatile一样，可以保证其他线程读取到最新的值，这个putOrderedInt其实是volatile的延迟版，也就是其他线程可能在值改变后仍然读到旧值。这有什么好处呢？性能：在多核处理器下，内存以及cpu缓存的读和写常常是顺序执行的，所以在多个cpu缓存之间同步一个内存值的代价是很昂贵的。


**2B.**

``` java
public ScalarDisposable(Observer<? super T> observer, T
value) {
	this.observer = observer;
	this.value = value;
}
```



static final class ScalarXMapObservable<T, R> extends
Observable<R> {}

``` java
@SuppressWarnings("unchecked")
@Override
public void subscribeActual(Observer<? super R> s){
ObservableSource<? extends R> other;
	try {
		other = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper returned
		a null ObservableSource"); //这个函数mapper就不再是源码自带的，而是我自己传的参数function
	} catch (Throwable e) {
		EmptyDisposable.error(e, s);
		return;
	}
	if (other instanceof Callable) {
		R u;
		try {
			u = ((Callable<R>)other).call(); //调用这个Function的call函数（我们重写的)
		} catch (Throwable ex) {
			Exceptions.throwIfFatal(ex);
			EmptyDisposable.error(ex, s);
			return;
		}
	
		if (u == null) {
			EmptyDisposable.complete(s);
			return;
		}
		ScalarDisposable<R> sd = new ScalarDisposable<R>(s,u);--->K
		s.onSubscribe(sd); //observer都会重写onSubscribe函数（好像也没有...迷）
		sd.run(); ----->L
	} else {
		other.subscribe(s);
	}
}
```

**总结一下几个常用的操作符：**
总结几个点：

* 操作符的参数，就是我们应用层使用时候传入的参数

* 这个参数，又会被传进各种ObservableXXX的类进行改造（new
....），这些类，通常都会进行source （以及value等的存储），这个source，就是
subscribeActual中最后 source.subscribe(parent);
时的source(parent就是我们subscribe时写的四个函数的包装）


**1）**

``` java
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T>
source) {
	ObjectHelper.requireNonNull(source, "source is null");
		return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

**2）**

``` java
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> just(T item) {
	ObjectHelper.requireNonNull(item, "The item is null");
		return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));
}
```

**3）**
``` java
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler){
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
		return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this,scheduler));
}
```

**4）**
``` java
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean
delayError, int bufferSize) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	ObjectHelper.verifyPositive(bufferSize, "bufferSize");
	return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this,scheduler, delayError, bufferSize));
}
```

**思考一、为什么一定要调用subscribe？**

**看源码可以看出，**
observer = RxJavaPlugins.onSubscribe(this,observer);
**和**
subscribeActual(observer);
**这两句**
