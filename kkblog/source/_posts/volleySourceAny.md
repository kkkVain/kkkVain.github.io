---
title: volly源码解析
date: 2017-09-07 14:09:30
tags: Android,volley,源码,网络
categories: Android
---


### ①Request.java

**实现了** Comparable<Request<T>> 接口

抽象类：

``` java
public abstract class Request<T> implements
Comparable<Request<T>> {}
```

两个构造函数，有一个已经废弃了(所以后面的所有具体实现中，有一个构造函数也是被弃用了

``` java
public Request(int method, String url, Response.ErrorListener listener);
```

具体分析一下几种request：

a.StringRequest:

1）方法、URL、监听器
``` java
public StringRequest(int method, String url, Listener<String> listener,ErrorListener errorListener)
```

2）结束时，调用父类的方法（其实父类中也是置空监听器）

``` java
protected void onFinish() {
	super.onFinish();
	mListener = null;
}
```

3）回调接口函数
``` java
@Override
protected void deliverResponse(String response) {
	if(mListener != null) {
		mListener.onResponse(response);
	}
}
```

4）把接收到的response信息解析，主要是cache分发器和network分发器会用到，response不是像我们一样简单的弄成String，而是专门根据类型存放一个在Response<>对象中
``` java
protected Response\<String\> parseNetworkResponse(NetworkResponse response)
{
	String parsed;
	try {
		parsed = new String(response.data,
		HttpHeaderParser.parseCharset(response.headers));
	} catch (UnsupportedEncodingException e) {
		parsed = new String(response.data);
	}
	return Response.success(parsed,
	HttpHeaderParser.parseCacheHeaders(response)); --> 这个函数其实是response构造
}
```

b.JsonRequest

1)构造函数
参数：方法、url、requestBody、监听器
``` java
public JsonRequest(int method, String url, String requestBody,
	Listener<T> listener,ErrorListener errorListener) {
	super(method, url, errorListener);
	mListener = listener;
	mRequestBody = requestBody;
}
```

2）onFinish（）和diliverResponse（）与上面一样。

3）交给JsonArrayRequest和JsonObjectRequest去实现
``` java
abstract protected Response<T> parseNetworkResponse(NetworkResponse response);
```

4）
``` java
@Override
public String getBodyContentType() {
	return PROTOCOL_CONTENT_TYPE; // utf-8
}
```

5）把请求体的内容转为bytes[]类型（String的方法 getBytes）

``` java
@Override
public byte[] getBody() {
	try {
		return mRequestBody == null ? null : mRequestBody.getBytes(PROTOCOL_CHARSET);
	} catch (UnsupportedEncodingException uee) {
		VolleyLog.wtf("Unsupported Encoding while trying to get the bytes of %s
		using %s",mRequestBody, PROTOCOL_CHARSET);
		return null;
	}
}
```

c.ImageRequest（这种方法过时了）

1)构造函数
在加载图片时如果图片超过期望的最大宽度和高度则会进行压缩。注意两个类型：

``` java
//@param scaleType The ImageViews ScaleType used to calculate the needed image size.
//@param decodeConfig Format to decode the bitmap to
public ImageRequest(String url, Response.Listener<Bitmap> listener,
int maxWidth, int maxHeight, ScaleType scaleType, Config decodeConfig, Response.ErrorListener errorListener)
{
	super(Method.GET, url, errorListener);
	setRetryPolicy(new DefaultRetryPolicy(IMAGE_TIMEOUT_MS, IMAGE_MAX_RETRIES,IMAGE_BACKOFF_MULT));
	mListener = listener;
	mDecodeConfig = decodeConfig;
	mMaxWidth = maxWidth;
	mMaxHeight = maxHeight;
	mScaleType = scaleType;
}
```

2)同样的，要对response进行解析：

``` java
@Override
protected Response<Bitmap> parseNetworkResponse(NetworkResponse response)
{
	// Serialize all decode on a global lock to reduce concurrent heap usage.*
	synchronized (sDecodeLock) {
		try {
			return doParse(response);
		} catch (OutOfMemoryError e) {
			VolleyLog.e("Caught OOM for %d byte image, url=%s",
			response.data.length, getUrl());
			return Response.error(new ParseError(e));
		}
	}
}
```

3）解析函数

和最近写图片压缩方法一样，但是，desiredWidth 和desiredHeight ， 到底是什么鬼？ =
= 没看懂，仔细看传值，maxPrimary 和 secondPrimary 主维和二维..就是图片的宽高...

``` java
private Response<Bitmap> doParse(NetworkResponse response) {
	byte[] data = response.data;
	BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
	Bitmap bitmap = null;
	if (mMaxWidth == 0 && mMaxHeight == 0) {
		decodeOptions.inPreferredConfig = mDecodeConfig;
		bitmap = BitmapFactory.decodeByteArray(data, 0, data.length,
		decodeOptions);
	} else {
		// If we have to resize this image, first get the natural bounds.
		decodeOptions.inJustDecodeBounds = true;
		BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
		int actualWidth = decodeOptions.outWidth;
		int actualHeight = decodeOptions.outHeight;
		// Then compute the dimensions we would ideally like to decode to.
		int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,
		actualWidth, actualHeight, mScaleType);
		int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,
		actualHeight, actualWidth, mScaleType);
		// Decode to the nearest power of two scaling factor.
		decodeOptions.inJustDecodeBounds = false;
		// TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
		// decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
		decodeOptions.inSampleSize =
		findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
		Bitmap tempBitmap =
		BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
		// If necessary, scale down to the maximal acceptable size.
		if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth ||
		tempBitmap.getHeight() > desiredHeight)) {
		// 可以根据原来的位图创建一个新的位图。= =有点无语，那之前为什么要有inSamplesize?
		// 因为上面是图片内存的压缩= =
		由于要考虑samplesize，所以可能达不到desired，这里就 重新来一遍？
		bitmap = Bitmap.createScaledBitmap(tempBitmap,
		desiredWidth, desiredHeight, true);
		tempBitmap.recycle();
		} else {
		bitmap = tempBitmap;
		}
	}
	if (bitmap == null) {
		return Response.error(new ParseError(response));
	} else {
		return Response.success(bitmap,
		HttpHeaderParser.parseCacheHeaders*(response));
	}
}
```

5）
这个函数用来计算desiredWidth和desiredHeight，计算出来这两个值，再用他们来计算samplesize，这其实就是自己写 图片压缩 时候的reqWidth,和 reqHeight， 感觉这里处理得比较巧妙。
自己实现的时候，就在想reqWidth和reqHeight要赋值什么- - ，
这里让用户设置可以接收的maxWidth和maxHeight（也可以不设置），然后根据规则再计算req的宽高。
计算desiredHeight的时候，主维就是mMaxHeight。
没懂这个规则= =暂时跳过了...

``` java
private static int getResizedDimension(int maxPrimary, int
maxSecondary, int actualPrimary,int actualSecondary, ScaleType scaleType) {
	// If no dominant value at all, just return the actual.
	if ((maxPrimary == 0) && (maxSecondary == 0)) {
		return actualPrimary;
	}
	// If ScaleType.FIT_XY fill the whole rectangle, ignore ratio.*
	// 填充整个矩形
	if (scaleType == ScaleType.FIT_XY) {
		if (maxPrimary == 0) {
			return actualPrimary;
		}
		return maxPrimary;
	}
	// If primary is unspecified, scale primary to match secondary's scaling
	ratio.
	if (maxPrimary == 0) {
		double ratio = (double) maxSecondary / (double) actualSecondary;
		return (int) (actualPrimary \ ratio);
	}
	if (maxSecondary == 0) {
		return maxPrimary;
	}
	double ratio = (double) actualSecondary / (double) actualPrimary;
	int resized = maxPrimary;
	// If ScaleType.CENTER_CROP fill the whole rectangle, preserve aspect ratio.
	if (scaleType == ScaleType.CENTER_CROP) {
		if ((resized * ratio) < maxSecondary) {
			resized = (int) (maxSecondary / ratio);
		}
		return resized;
	}
	if ((resized * ratio) > maxSecondary) {
		resized = (int) (maxSecondary / ratio);
	}
	return resized;
}
```

d.ImageLoader（太长了...在代码里写注释了 = = 不在这里写了...

1）构造函数
有七个成员变量，两个涉及构造函数：
``` java
public ImageLoader(RequestQueue queue, ImageCache imageCache) {
	mRequestQueue = queue;
	mCache = imageCache;
}
```

2）内部类，请求的holder，记录该种请求，对应的key,对应的bitmap,以及监听器。
``` java
public class ImageContainer {}
```

3）内部类
``` java
private class BatchedImageRequest {}
```

②RequestQueue.java
涉及到的数据结构：

0）
用于请求队列的递增序号 ， 也即记录了请求的数量
``` java
private AtomicInteger mSequenceGenerator = new AtomicInteger();
```

**1）用于查找是否有相同cachekey的重复请求（get(cachekey）为null说明没有**

**key是string,value是一个请求队列！等待中的请求集合，也就是没有被分给cache也没被分给network**

``` java
private final Map<String, Queue<Request<?>>> mWaitingRequests =
new HashMap<String, Queue<Request<?>>>();
```

2）当前正在处理的请求

如果请求正在任意一个queue中wating，或者正在由任意一个分发函数处理，就会在这个set中
``` java
private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();
```

3）cache类的请求

``` java
private final PriorityBlockingQueue<Request<?>> mCacheQueue = new PriorityBlockingQueue<Request<?>>();
```

4）network类的请求

``` java
private final PriorityBlockingQueue<Request<?>> mNetworkQueue =
new PriorityBlockingQueue<Request<?>>();
```

5）网络请求最大线程数目：4
``` java
private static final int DEFAULT_NETWORK_THREAD_POOL_SIZE = 4;
```

6）完成请求后的回调接口：
``` java
\\ Callback interface for completed requests. 
// 完成请求后的回调接口*
public static interface** RequestFinishedListener<T> {
\\ Called when a request has finished processing. 
// 请求事务结束后会回调
public void onRequestFinished(Request<T> request);
}
```

7）任务完成后的监听器队列
``` java
private List<RequestFinishedListener> mFinishedListeners = new ArrayList<RequestFinishedListener>();
```

8）三个构造函数：cache,network参数是必须。其余两个分别是线程池默认大小4以及
``` java
new ExecutorDelivery(new Handler(Looper.getMainLooper())。
```

最终归于这个：
``` java
public RequestQueue(Cache cache, Network network, int threadPoolSize, ResponseDelivery delivery) {
	mCache = cache;
	mNetwork = network;
	mDispatchers = new NetworkDispatcher[threadPoolSize];
	mDelivery = delivery;
}
```

delivery其实是是来自这里，= =
读完了response回到这里，发现这个局早在requestQueue里面就布好了...厉害...
一开始就把UI线程的handler传给了ExecutorDelivery，这样在cache分发线程的mDelivery.postResponse（）才能很好的调用= =。
``` java
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
	this(cache, network, threadPoolSize,
	new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}
```

9）过滤器接口
``` java
public interface RequestFilter {
	public boolean apply(Request\<?\> request);
}
```

10）两个cancelAll的方法，一个是通过过滤器，一个是通过Object tag

其实tag的方法中，也是靠定义一个 request.getTag()==tag的过滤器实现的。

**11） 把请求添加到分发队列中 , add**

``` java
public <T> Request<T> add(Request<T> request) ；
首先request.setRequestQueue(this); 将这个请求标记为属于这个队列
然后synchronized (mCurrentRequests) {
	mCurrentRequests.add(request);
}在当前请求队列中加上这个请求
再request.setSequence(getSequenceNumber()); 设置这个请求的序列号
并加上标记。
接着
if (!request.shouldCache()) {
	mNetworkQueue.add(request);
	return request;
}
//如果该请求不要求缓存， 将请求加到newtwork队列中，返回请求。
request.shouldCache()
//是一个返回boolean的函数，可知有一个boolean为每个请求设置了是否需要缓存的标志。
//如果要求缓存，继续下一步：
//判断mWaitingRequests中是否存在相同key的请求队列，也就是是否有相同key的请求在等待，在该结构中，一个string对应了一个队列，其实是一个链表吧...linkedlist
Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
//如果有的话，取出这个请求队列（Queue<Request<?>> ） 为空的话:
if (stagedRequests == null) {
	stagedRequests = new LinkedList<Request<?>>();
}
//然后把请求加到这个链表中。再把<Key,链表>存入到 mWaitingRequests 中。
//如果不存在的话，将<key,null>存入到这 mWaitingRequests
//中，然后给cacheQueue加上请求。也就是：
mWaitingRequests.put(cacheKey, null);
mCacheQueue.add(request);
```


思考：为什么含有key时，stagedRequests可能为空，并且如果为空也要新建一个实例后才把<key,链表>插入
而没有key时，直接<key,null> 插入？？
JAVA基础啊 = = hashmap.remove(key)之后，再用key去get，得到的是null。
因为，含有key时，说明有相同请求正在执行（有可能调用了remove(key），所以导致这个key对应的队列为null)（甚至可能还有在等待的相同请求，所以需要先把当前请求用一个
LinkedList封装起来，再放到等待队列中，当然了，如果相同key中除了正在执行的，还有在等待的，那么stageRequest不会为空，直接把request放进去就可以了。
而没有key的话，这个请求会mCacheQueue.add(request);
也就是丢给cache处理，所以它此时正在被处理，那么自然是把<key,null>存到mWaitingRequests中了。

最后finish:

这个finish可能在request的finish中被调用：
``` java
if (mRequestQueue != null) { // 这是这个reuquest对应的queue
	// 如果请求队列不为空，结束当前请求
	// 让当前请求处于onFinish（）
	mRequestQueue.finish(this);
	onFinish();
}
```

注意这个finish是requestQueue的finish，不是request的finish！两者是有区别的。


思考：为什么还有相同key的请求时，把这些所有请求都一下加入到cacheQueue中呢？
别人解释：
第二步就是判断这个请求有没有要求缓存，如果有（waitingRequests不为null）：
``` java
Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
```
注意这个remove(key)，虽然说把key对应的value从map中清理了，但是返回了这个key对应的value（删除前的value）。
于是要将mWaitingQueue中相同CacheKey的所有requests放入cacheQueue中，
因为前面我们不知道相同CacheKey的那个请求到底在缓存中有没有，如果没有，它需要去网络中获取，那就等到它从网络中获取之后，**放到缓存中后，它结束了，并且已经缓存了（意思就是到这步时，这个key的response已经放到cache中了）**，这个时候，我们就可以保证后面那堆相同CacheKey的请求可以在缓存中去取到数据了，而不需要再去网络中获取了。

### ②DiskCacheBased.java

1)inputSteam的三种read:

**a.read() : 对流一个字节一个字节读，返回的int就是这个byte的int表示方式。**

``` java
InputStream in = Test.class.getResourceAsStream("/tt.txt");
byte[]tt=new byte[15];//测试用的事前知道有15个字节码
while(in.available()!=0){
	for(int i=0;i\<15;i++){ tt[i]=(byte)in.read();}
}
String ttttt=new String(tt,"utf-8");
System.out.println(ttttt);
in.close();
```

**b.read(byte[] b):规定一个数组长度，把流中的字节缓冲到数组b中去，返回真实的字节个数**
``` java
in = Test.class.getResourceAsStream("/tt.txt");
byte [] tt=new byte[1024];
int b;
while((b=in.read(tt))!=-1){ System.out.println(b); }
String tzt=new String(tt,"utf-8");
System.out.println(tzt);
```
读取的字节先存到b[0] 再存到b[1].....


**c.read(byte[] b, int off, int len) ：从输入流中读取len个字节到数组b中，off是在数组b中写入数据的偏移。**

2.)cache有，cacheHader没有的：
byte[] data ; isExpired() ; refreshNeeded()

cache没有，cacheHeader有的：
string key

3)
``` java
public synchronized void clear() ；
```
把根目录的所有file取到数组中，不为空，则将数组里的file清空。

4）
``` java
public synchronized Entry get(String key) {}
```

先通过key取出CacheHeader entry,如果entry不为空，根据key取出根目录中对应的文件。把这个文件转为CountingIS,接着
CacheHeader.readHeader（cis);
静态类，直接用类名调用函数！！根据cis获取头部数据，并把流cis转为byte[]类型，把byte[]数据通过entry.toCacheEntry(data)存到entry中。

5）
``` java
public synchronized void initialize() ；
```

根据根目录下的文件，初始化cache。把文件全部用BIS保存为流，再通过
``` java
CacheHeader entry = CacheHeader.*readHeader*(fis);
```
转化为cacheheader的实例再通过putEntry(entry.key, entry);把他放入mEntries这个cache的MAP集合里面。

6）
``` java
private void putEntry(String key, CacheHeader entry)
```
把<key,cacheHeader>放到map集合里面


### ③CacheDispatcher.java

1)主要成员变量，构造函数需要的四个：
``` java
private final BlockingQueue\<Request\<?\>\> **mCacheQueue**;
private final BlockingQueue\<Request\<?\>\> **mNetworkQueue**;
private final Cache **mCache**;
private final ResponseDelivery **mDelivery**;
```

2）
``` java
public void quit() {
	mQuit = true;
	interrupt();
}
//停止cache的分发处理，调用thread的interrupt()
```

3)
``` java
public void run() {}
```

用request = mCacheQueue.take();取得队列中的请求。

然后有几种情况：
a.请求取消了，request.isCanceled()为真，调用r.finish
然后此次循环结束，去处理下一个请求。
``` java
Cache.Entry entry = **mCache**.get(request.getCacheKey());
```

b.通过r.getKey取得的key判断mCache中是否有这个entry，没有，交给Network

c.有这个key，但过期了,设置这个request的cacheEntry为entry ，并且交给net，结束循环

d.有key，且cache没过期，把entry里的数据解析了赋值给response

e.如果entry不需要更新，把response用deliver发送回去

f.要更新(Soft-expired cache hit)，response.intermediate = true;
意味着这个未更新的响应发回后，还有第二个更新过的响应要发回去。
由于还需要更新后的响应，所以这个请求还是要交给net。

### ④HttpStack
``` java
public interface HttpStack { public HttpResponse performRequest ....}
```
这个接口被HurlStack.java实现了。


**HurlStack.java**

1)三个构造函数。

``` java
**public** HurlStack(UrlRewriter urlRewriter, SSLSocketFactory sslSocketFactory)
```

参数列表可为空。

**2）重写接口中唯一的一个方法（重要，用Httpurlconnection进行请求，并处理响应的函数）**
``` java
public HttpResponse performRequest(Request<?> request, Map<String,
String> additionalHeaders)
先使用
HashMap<String, String> map = new HashMap<String, String>();
map.putAll(request.getHeaders());
map.putAll(additionalHeaders);
把请求头部的<headerName,对应内容>存到一个map中。
用
String url = request.getUrl();取得url，然后mUrlRewriter.rewriteUrl(url);取得一个新的url（没找到这个函数的实现...
然后创建一个新的httpurlconnection, 遍历map，把所有的header加入conn的porperty中。

然后：
//给conn设置请求的方法
setConnectionParametersForRequest(connection, request);
接着从这步开始都是处理响应了：
// 调用了getResponseCode()就会自动connect，不用明文调用.connect
int responseCode = connection.getResponseCode();
```

首先判断响应码是不是符合要求，如果符合要求：
a.用函数C将connection得到的响应转化为实体

b.通过connection.getHeaderFields()，得到所有的响应头列表，foreach遍历这个结合，把每一条通过key单独取出来，做成header，加入到response的headers集合中。

2）
**C**
``` java
private static HttpEntity entityFromConnection(HttpURLConnection
connection) {}
```

根据connection返回的信息，建立实体。

实体主要包括四个信息：content,contentLength,contentEncoding,contentType

使用volley时，新建一个queue（请求队列）,再将多个request放入queue中，之后发送queue

这时候处理queue的线程是异步，queue中也有调度，保证若干个request异步处理

**④BasickNetwork.java（实现了netWork的接口**
**缓存与重试策略的重点！！**
**原来上次看到这儿就停止了= = 难怪没有看到重试策略**
**而这个重要的函数，就是接口函数中的重写！**
**先复习一下有关缓存的这些变量：**

**1.request:**
-   Cache-Control: max-age=0 以秒为单位
-   If-Modified-Since: Mon, 19 Nov 2012 08:38:01 GMT 缓存文件的最后修改时间。
-   If-None-Match: "0693f67a67cc1:0" 缓存文件的Etag值
-   Cache-Control: no-cache 不使用缓存
-   Pragma: no-cache 不使用缓存

**2.response:**
-   Cache-Control: public 响应被缓存，并且在多用户间共享
-   Cache-Control: private 响应只能作为私有缓存，不能在用户之间共享
-   Cache-Control:no-cache 提醒浏览器要从服务器提取文档进行验证
-   Cache-Control:no-store 绝对禁止缓存（用于机密，敏感文件）
-   Cache-Control: max-age=60 60秒之后缓存过期（相对时间）
-   Date: Mon, 19 Nov 2012 08:39:00 GMT 当前response发送的时间
-   Expires: Mon, 19 Nov 2012 08:40:01 GMT 缓存过期的时间（绝对时间）
-   Last-Modified: Mon, 19 Nov 2012 08:38:01 GMT 服务器端文件的最后修改时间
-   ETag: "20b1add7ec1cd1:0"
    服务器端文件的Etag值，客户端收到后再次请求时用if-none-match返回，如
    果etag状态没变，则返回状态304然后不返回

x-cache-lookup项指专门查看代理服务器中**是否有**某个网页缓存。有就返回HIT,没有返回MISS。而x-cache项指浏览器从何处、是在哪个代理缓存载入的网页文件。服务器名后的3128指服务器端口。
X-Cache :HIT from proxy.domain.tld, MISS from proxy.local
X-Cache: 表示你的 http request 是由 proxy server 回的 .

MISS 表 proxy无资料,代理动作, HIT 表 proxy 直接回应

**1）**

``` java
@Override
public NetworkResponse performRequest(Request\<?\> request) **throws**
VolleyError {}
```

方法是执行网络请求的方法，那么理所当然的要进行请求重试也是在这里进行。如何进行请求重试，注意在**方法的内部是用while(true)括起来的**，也就是说如果该方法正常执行完毕或者抛出异常时，必然就跳出循环了，但是如果**请求失败没有return并且在catch内也没有超过重试策略限定条件时**，必然会while(true)下重新请求一次，这样就达到了重试的目的。

**这个函数比较长，分为几部分来说：**

**a.**
``` java
// Gather headers.

Map<String, String> headers = new HashMap<String, String>();

// 向headers这个集合中添加cache entry
addCacheHeaders(headers, request.getCacheEntry());--->B

// mHttpStack (httpStack接口，有hurlStack类实现）
// 真正进行httpurlconnection 连接和处理 响应的地方，发送了连接请求，并且进行了响应的处理（主要是响应头的处理）

httpResponse = mHttpStack.performRequest(request, headers);
StatusLine statusLine = httpResponse.getStatusLine();
int statusCode = statusLine.getStatusCode(); // 得到响应码
responseHeaders = convertHeaders(httpResponse.getAllHeaders());
```

然后通过statusCode判断是否缓存：
a.如果是304，那么就是not_modified：
``` java
// 尝试获得request的entry
Entry entry = request.getCacheEntry();
```
如果entry为空，就用networkResponse根据之前的响应头构建响应并返回。
否则，就把刚才得到的响应头存到entry的响应头中，然后用新响应头构建响应并返回。

b.如果是301（永久搬迁）或者302（暂时搬迁）
那么需要重定向,request根据返回的重定向地址进行设置重定向url
String newUrl = responseHeaders.get("Location");
request.setRedirectUrl(newUrl);

c.可能没有响应体：
``` java
responseContents = new byte[0];
//有的话转为bytes:
//这里面就用到了bytespool了，似乎加快了内存的利用？但是也导致了无法存储太多东西？？
responseContents = entityToBytes(httpResponse.getEntity());--->D
```

d.还可以打印请求花费的时间：
``` java
// requestLifetime 代表这个请求已经消耗的时间，可以用Log打印出来。*
long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
logSlowRequests(requestLifetime, request, responseContents, statusLine);
```

e.如果状态码小于200超过299，直接抛出异常。

最后就是到return了，如果没有continue,异常等的抛出，就可以返回通过network得到的响应了。

3)
``` java
D.private byte[] entityToBytes(HttpEntity entity) throws IOException,
ServerError {}
```

终于用到了bytespool。
首先构造实体长度的一个bytearrayoutputStream: **mPool就是这个basicNetwork的全局变量，byte池。**
``` java
PoolingByteArrayOutputStream bytes = new PoolingByteArrayOutputStream(mPool, (int) entity.getContentLength());
byte[] buffer = null;
try {
	InputStream in = entity.getContent();
	if (in == null) {
		throw new ServerError();
	}
//以下，从pool里面遍历mBuffersBySize，
//找到还有多余1024容量的bytes[],从这个数组中取走1024长度的内存，在pool的这个bytes[]中记得把取走的容量减去
buffer = mPool.getBuf(1024); ----> byteArrayPool的函数
int count;
// 把inputstream的内容读到buffer中，又把buffer的内容写到函数开始创建的 流bytes中
while ((count = in.read(buffer)) != -1) {
	bytes.write(buffer, 0, count);
}
return bytes.toByteArray();
```

2）
``` java
B.
private void addCacheHeaders(Map<String, String> headers, Cache.Entry entry) {
// If there's no cache entry, we're done.
if (entry == null) {
	return;
}
if (entry.etag != null) {
	headers.put("If-None-Match", entry.etag);
}
if (entry.lastModified > 0) {
// 原来这个地方是entry.serverDate，现在已经改过来了...
	Date refTime = new Date(entry.lastModified);
		headers.put("If-Modified-Since", DateUtils.formatDate(refTime));
	}
}
```

大神解释：
Date代表了响应产生的时间，正常情况下Date时间在Last-Modified时间之后,即Date比较晚，也就是Date>=Last-Modified。
通过以上原理，既然Date>=Last-Modified。那么我利用Date构建，也是完全正确的。
可能的问题出在服务端的 Http 实现上，如果服务端完全遵守 Http语义，采用时间比较的方式来验证If-Modified-Since，判断服务器资源文件修改时间是不是在If-Modified-Since之后。那么使用Date完全正确。
我：比如服务端LastModified在8点，Date时间在8点5秒，而客户端传来的If-Modified-Since是在7点，很明显无论如何I-M-S肯定小于L-M和Date,使用服务端的Date来和我们自己的request Entry的L-M-S比较也没关系。
可是有的服务端实现不是比较时间，而是直接的判断服务器资源文件修改时间，是否和If-Modified-Since所传时间相等。这样使用Date就不能实现正确的再验证，因为Date的时间总不会和服务器资源文件修改时间相等。
我：可以比较相等是因为客户端接收到这次资源的I-M-S等于服务端的L-M。所以如果服务端资源不变，那么L-M不变，还是等于客户端那边的I-M-S


### ⑤ ExecutorDelivery.java 实现了 ResponseDelivery ，用于把response传送出去

有一个Excutor成员。
``` java
private final Executor mResponsePoster;
```

1）
``` java
public ExecutorDelivery(final Handler handler) {
	// Make an Executor that just wraps the handler.
	mResponsePoster = new Executor() {
		@Override
		public void execute(Runnable command) {
			handler.post(command);
		}
	};
}
```

2）对于接口的重要实现（重要方法，在cache和net调度线程里用来传送响应的
最终其实是用excutor执行runnble:
``` java
@Override
public void postResponse(Request<?> request, Response<?> response) { 
	postResponse(request, response, null);
}

@Override
public void postResponse(Request<?> request, Response<?> response,Runnable runnable) {
	request.markDelivered();
	request.addMarker("post-response");
	mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
}
```

``` java
new ResponseDeliveryRunnable(request, response, runnable)
```
很明显是一个runnble，Excutor接口的execute函数在这个类的构造函数
``` java
public ExecutorDelivery(final Handler handler) {}中实现。
```

### ⑥PoolingByteArrayOutputStream
继承自ByteArrayOutputStream 。不表。

### ⑦btyeArrayPool
byte[]的缓存池，主要就是为了提高用户频繁进行数据请求时的性能，如果用户频繁进行数据请求，对象在很快的时间内创建又被丢弃，明显降低了性能，这里提供了一个一直存在的缓存池，提高了获取堆内存的性能。

两个集合：
``` java
private List<byte[]> mBuffersByLastUse = new LinkedList <byte[]>();
private List<byte[]> mBuffersBySize = new ArrayList<byte[]>(64);
```

第一个是按照使用顺序存储的byte[]集合，第二个是按照大小存储的byte[]集合。
需要用到byte[]时，从这个pool里面取，用完了又归还，提高了性能。


关于优缺点：
问题一、为什么不适合下载大文件，只适合数据量小的频繁请求？
1）存储时，从btyeArrayPool中取出一块已经分配的内存，这样不用每次都进行内存分配，而是先查找缓冲池中有没有合适的内存区域，有的话可以直接使用，减少了内存分配的次数。
如果数据量过大，byteArrayPool这个存储空间就会溢出。

2）线程池大小默认为4，如果上传数据大或者下载数据大，浪费比较长的时间，占用了线程，其余请求就会阻塞。


问题二、缓存机制（BasicNetwork.java)
只使用了Date来进行缓存验证（原来是这样，现在改了）
在BasicNetwork.java中：

###  ⑧ HttpHeaderParser.java
**(你亲自读过的呀... 注意重点是复习...复习，理解！！ ）**
**通过解析服务器返回的响应头部，构建保存的缓存请求的头部（ Cache.Entry）**，比如
解析http头部的方法，有三个。

1）
``` java
public static Cache.Entry parseCacheHeaders(NetworkResponse response) {}
```

**首先，“DATE”，**用parseDateAsEpoch（）把时间解析成EPOCH格式，响应生成时间。于是得到serverDate。

**第二，"Cache-Control"：**有这个值，意味着hasCacheControl = true;，那么判断完头部，还要去设置
softExpire和finalExpire这两个值。

a.如果是no-cache或者no-store，就不缓存结果。前者是要从服务器提取文档进行验证。后者是绝对禁止缓存，用于机密文件2333

b.max-age开头：得到max-age。相对时间，多少秒之后过期。
比如max-age = 60，那么60秒之后缓存过期。

c.stale-while-revalidate 开头：得到stale-while-revalidate.RFC文档解释：
When present in an HTTP response, the stale-while-revalidate Cache- Control
extension indicates that caches MAY serve the response in which it appears after
it becomes stale, up to the indicated number of seconds.
也就是说，到达给定的时间后，服务器可能会返回新的内容。（也就是这个时间过后这个响应就陈旧了，所以会发新的过来）

d.如果是must-revalidate, 作用与no-cache相同，但更严格，强制意味更明显。但这只是理论上的描述。或者是proxy-revalidate，都是强制刷新。
于是mustRevalidate = true;

**第三，“Expires”，**用parseDateAsEpoch（）把的时间解析成EPOCH格式。于是得到serverExpires。**缓存过期的绝对时间，**所以要取缓存时用到的isExpired()函数是这样写的：

``` java
// 如果缓存过期，返回true
public boolean isExpired() {
    return this.ttl < System.currentTimeMillis();
}
```

**第四，“Last-Modified“，**用parseDateAsEpoch（）把的时间解析成EPOCH格式。于是得到lastModified。（**客户端用if-not-modified返回**，如果服务端文件在这个时间之后都没修改，那么返回304）

**第五，"ETag"，**得到serverEtag（如果客户端用if-none-match加上etag的内容发回，服务端etag没变的话，返回状态304，内容不更新）


然后要看有没有cachecontrol的控制：

``` java
/* Cache-Control takes precedence over an Expires header, even if both exist and Expires is more restrictive.过期时间， Cache-Control比Expires优先级高。*/
if (hasCacheControl) {
	softExpire = now + maxAge * 1000; //新鲜度时间，maxAge是秒为单位，但是开发时是毫秒为单位
	finalExpire = mustRevalidate ? softExpire : softExpire + staleWhileRevalidate * 1000; // 过期时间
} else if (serverDate > 0 && serverExpires >= serverDate) {
	//没有cacheControl
	//Default semantic for Expire header in HTTP specification is softExpire.
	softExpire = now + (serverExpires - serverDate); 
	//现在时间+过期的绝对时间-生成文件的时间，因为serverExpires - serverDate代表从发送时的时间到过期时间还有多少时间T，这段时间过后就是过期了。
	//这是软过期时间，意味着从接受到响应开始（now),一定还可以保存T那么长的时间。
	finalExpire = softExpire;
}
// 各个值终于算好了，全部赋值给cache.entry，这样就用entry保存了这个request对应的cache
```

印象最深的地方：

1）接口的使用很厉害(读到最后才明白这个妙处）
比如新建一个requsertQueue的时候，cache和network在volley.java的newRequestQueue里面是一定要有的，通常来说 ResponseDelivery delivery这个参数是可以不自己定义的，那么它就是这个值：

A.
``` java
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
    this(cache, network, threadPoolSize,
            new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}
```
将ExcutorDelivery和UI线程的handler绑定起来，

所以在ExDel的构造函数中，实现了Excutror接口中的函数excute，这个函数就是调用主线程的handler去处理Runn.
``` java
public ExecutorDelivery(final Handler handler) {
    // Make an Executor that just wraps the handler.
    // 新建实例的时候重写接口方法，但是也是要.excute才会运行的...不是说实例建了就开始运行...
    mResponsePoster = new Executor() {
        @Override
        public void execute(Runnable command) {
            handler.post(command);
        }
    };
}
```


当然了，这只是构造函数中，那么这个excute何时被调用呢....？？我们读到response的时候就会解开谜底。去到B！

这个ExcutorDel本身也是 接口的实现！！重载了这个函数：

可以看到，runnble经过了一点改造，在

private class ResponseDeliveryRunnable implements Runnable {}
中，除了本身的runnalbe要被run外，还有增加了：

C.
对runnable进行了一点改造，用来**在request的类的delResponse函数中调用我们的回调接口！**
``` java
if (mResponse.isSuccess()) {
	mRequest.deliverResponse(mResponse.result);
```
有了这一句，才会在**各个xxxRequest类中，重载request类的deliverResponse函数**，onResponse是什么呢。。。就是调用了我们自己设置的onResponse函数，这样就可以干我们想做的事了....所以其实最后的response的回调显示也是在各个request中完成的。

在每个xxxRequest中都重写了这个函数：
``` java
@Override
protected void deliverResponse(String response) {
    if (mListener != null) {
        mListener.onResponse(response);
    }
}
```

B.在A中（Request的时候）构造好了，最终postResponse的时候才会用。比如在networdDispatch这个线程中，终于从网络取到了线程，
``` java
mDelivery.postResponse(request, response);
```

这个mDelivery就是每个request的response的传送者~
mDelivery 是一个ResponseDelivery，而我们刚才提到的ExcutorDelivery是这个抽象类的实现。
所以mDelivery.postResponse其实是调用了ExcutorDelivery里面的postResponse.

``` java
@Override
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
    request.markDelivered();
    request.addMarker("post-response");
    mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
}
```

而这个postResponse做了什么呢？它终于，调用了ExcutorDelivery。构造函数里面的Excutor接口实现的excute函数，刚才我们分析得知，这个excute会把传入的runnable用handler去执行里面的run（），而这个runnable其实被暗自改造了\~\~我们刚才刚才分析过了。

在cacheDispatcher这个线程的run中，得到response后，结果就是通过postResponse返回去的！这个设计真的很精妙= =！！

``` java
mDelivery.postResponse(request, response, new Runnable() {
    @Override
    public void run() {
        try {
            mNetworkQueue.put(finalRequest);
        } catch (InterruptedException e) {
            // Not much we can do about this.
        }
    }
});
```

还有就是各个xxxRequest，除了传入url外，实现了response类中的两个回调接口
Listener<T> 和ErrorListener，request和response的结合，一开始就想好了最后 - -很巧妙啊。（其实刚才哪一篇叙述已经讲到了）

重试机制：
volley的重试策略是在request构造函数里面确定的：
``` java
public Request(int method, String url, Response.ErrorListener listener) {
    mMethod = method;
    mUrl = url;
    mIdentifier = createIdentifier(method, url);
    mErrorListener = listener;
    setRetryPolicy(new DefaultRetryPolicy());
    mDefaultTrafficStatsTag = findDefaultTrafficStatsTag(url);
}
```

首先new出来的东西是一个DefaultRetryPolicy：
``` java
public class DefaultRetryPolicy implements RetryPolicy {}
```
这个类有什么呢，它其实不是执行重试操作，只是更新有关重试的两个变量。

成员变量：
``` java
	  // The current timeout in milliseconds. 当前超时时间 
      private int mCurrentTimeoutMs;  

      // The current retry count. 当前重试次数  
      private int mCurrentRetryCount;  

      // The maximum number of attempts. 最大重试次数 
      private final int mMaxNumRetries;  

      // The backoff multiplier for for the policy. 表示每次重试之前的 timeout 该乘以的因子，每重试一次，超时时间就变化一次
      private final float mBackoffMultiplier;  

      // The default socket timeout in milliseconds 默认超时时间
      public static final int DEFAULT_TIMEOUT_MS = 5000;  

      // The default number of retries  默认最大重试次数
      public static final int DEFAULT_MAX_RETRIES = 1;  
```

除此之外只有两个函数：

``` java
更新当前请求重试次数，并且更新当前超时时间
@Override
public void retry(VolleyError error) throws VolleyError {
	mCurrentRetryCount++;
	mCurrentTimeoutMs += (**mCurrentTimeoutMs * mBackoffMultiplier);
	if (!hasAttemptRemaining()) {
		throw error;
	}
}
```

而以下函数是判断当前重试次数是否超过最大请求数：
``` java
protected boolean hasAttemptRemaining() {
	return mCurrentRetryCount <= mMaxNumRetries;
}
```
set函数里，只是设置了重试策略：mRetryPolicy = 构造的这个retryPolicy对象。
