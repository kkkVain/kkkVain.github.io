---
title: okhttp3源码解析(1)
date: 2017-08-18 23:13:25
tags: [okhttp3,网络,框架,Android]
categories: [Android]
---


okhttp是**高性能的http库**，支持同步、异步，而且**实现了spdy、http2、websocket协议**，api很简洁易用，和volley一样实现了http协议的缓存。picasso就是利用okhttp的缓存机制实现其文件缓存，实现的很优雅，很**正确，**反例就是UIL（universal image loader），自己做的文件缓存，而且不遵守http缓存机制。

#### 协议介绍
##### SPDY
用以最小化网络延迟，提升网络速度，优化用户的网络使用体验。是对HTTP协议的增强。
a.复用连接，可在一个TCP连接上传送多个资源。应对了TCP慢启动的特性。降低了延迟的同时提高了宽带利用率。
b.请求分优先级，为了避免关键请求被阻塞，可以设置优先级，重要的资源优先传送。
c.HTTP头部数据也被压缩，省流量。
d.服务器端可主动连接客户端来推送资源（Server Push）。
e.基于HTTPS的加密协议传输

**HTTP2.0 支持明文HTTP传输，SPDY强制使用HTTPS协议。**
**HTTP2.0消息头的压缩算法采用HPACK，而非SPDY采用的DEFLATE**

##### HTTP2.0
a.新的二进制格式，只使用0和1，增加了代码的健壮性。文本协议的表现形式有很多，所以要考虑很多使用场景。
b.复用连接
c.header压缩
d.服务端推送

##### Websocket
Websocket借用了HTTP的协议来完成一部分握手，
a.通讯过程建立在一次连接上，只需要一次HTTP握手，避免了HTTP的非状态性。连接建立后，双方都可以发送/响应消息，不需要像HTTP那样等待响应。
b.服务器有消息时就主动发送信息给客户端


#### 源码解读

第一步
client.newCall(request).execute() ，链式调用，第一个调用返回了一个realCall对象，然后realCall实例调用属于它的方法execute()---->BEGIN

A.而 dispatcher的executed(RealCall call)其实是：
``` java
synchronized void executed(RealCall call) {
	runningSyncCalls.add(call);
}
```

runningSyncCalls是一个deque,它把这个call加入了队列。
``` java
/* Running synchronous calls. Includes canceled calls that haven't finished yet. */
// 执行同步调用的队列，包括取消的但还没有执行的call
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

##### **①RealCall.java**
实现了Call的接口。
1）call是一次准备执行的请求，realCall是一次请求和一次响应的paris，不能执行两次。
2）Response#body可以获得响应体，为了避免内存泄漏需要close这个reseponse
3）传输层成功（收到响应码，响应头，响应体），不代表应用层成功。依然可能有404,或者500这些坑爹的响应码。
4）cancellation, a connectivity problem or timeout会造成IOE的抛出，
5）请求已经执行时，抛出IllegalStateException

BEGIN.
``` java
public Response execute()
	//做的最重要的事情是：
	try {
		client.dispatcher().executed(this); -->A
		Response result = getResponseWithInterceptorChain(); -->B
		if (result == null) throw new IOException("Canceled");
			return result;
		} finally {
			client.dispatcher().finished(**this**);
		}
```

B.getResponseWithInterceptorChain();在Recall的这个函数中，主要是添加了一系列拦截器到一个链表interceptors中，添加过后，初始化RealInterceptorChain，执行RealInterceptorChain的proceed方法：

这个构造函数，传入的streamAllocation,httpCode,conn都是null
``` java
Interceptor.Chain chain = new RealInterceptorChain(
	interceptors, null, null, null, 0, originalRequest);
		return chain.proceed(originalRequest); --->C
```

##### ②Interceptor接口中还有Chain接口
Chain接口由http/**RealInterceptorChain.java**来实现（找半天...）

**C.两个proceed函数，可以只传request。最终归于以下这个函数**
``` java
public Response proceed(Request request, StreamAllocation streamAllocation,HttpCodec httpCodec,RealConnection connection)

/*
在这里面，各种判断流是否只调用了一次proceed，等等（其实没看懂..)
多次判断了this.httpCodec != null，只有ConnectInterceptor执行完后就不为null了,所以这是对connectInterceptor之后的拦截器进行了判断。在ConnectInterceptor之后的拦截器必须满足：request的url要一致，interceptor必须执行一次proceed()。这样子做是为了保证递推的正常运作。该函数里面的calls++,由于proceed（）是interceptor的intercept里面调用的，所以对于每个interceptor其实calls不能大于1。*/
// 最重要的是：
// Call the next interceptor in the chain.
// 创建一个新的拦截链

RealInterceptorChain next = new RealInterceptorChain(
interceptors, streamAllocation, httpCodec, connection, index + 1, request);

// 从list中取出第index个拦截器
// 执行这个连接器的intercept方法。
Interceptor interceptor = interceptors.get(index);
Response response = interceptor.intercept(next); --->D
```
由于拦截器有多种，所以根据不同的拦截器，会调用不同的intercept方法。

##### **③RealConnection.JAVA**

继承了Http2Connection.Listener，实现了Connection接口.
客户端和服务端之间的连接抽象为了realConnection，为了管理这些连接的复用设计了connectionPool,共享address的请求可以复用连接。
这个类管理一次连接，在里面用socket进行连接处理，以及handshake处理握手。
其实是真正的建立socket连接的地方。

**M.**
``` java
public boolean isEligible(Address address, Route route) {...}
```

该函数主要是判断这个链接是否能匹配这个address以及这个route。要经过三次判断：
* 一是这个链接是否接受新的流或者它接受的流的size是不是满了；
* 二是调用Internal.instance.equalsNonHost比较路线的address和传入的address除了host部分是否相同；
* 三是只调用address.url().host().equals(this.route().address().url().host()看主机名是否匹配...

**有点奇怪= =为什么二和三不一起比较呢？？迷...**

**N**.
public void connect(...）{}**中主要是根据路由是否需要隧道而判断调用connectTunnel(）还是connectSocket()；**

``` java
if (route.requiresTunnel()) {
	connectTunnel(connectTimeout, readTimeout, writeTimeout);
} else {
*//否则直接使用socket连接*
	connectSocket(connectTimeout, readTimeout);
}
establishProtocol(connectionSpecSelector);
```

//通过代理服务器，来做https请求的连接(http1.1的https以及http2)需要建立隧道连接，而其它的连接则不需要建立隧道连接。假设不需要隧道链接，直接使用socket链接：
对于private void connectSocket(int connectTimeout, int readTimeout) throws IOException {}

* 首先根据路由取得代理和的address，然后根据代理类型使用工厂创建无参数的socket或者直接用new创建有代理的socket,再用这个socket进行connect：
``` java
Platform.get().connectSocket(rawSocket, route.socketAddress(),
connectTimeout);
```
其中Platform.get()是用于判断平台是在Android 还是 OpenJDK 9+.还是 OpenJDK 7 or OpenJDK 8 等等。这里我们去Android的平台调用 --->O

服务端的信息存储在路由的socketAdress中。然后用okio封装rawsocket的输入和输出流。
socket连接结束后，调用establishProtocol(connectionSpecSelector); --->P

P.
``` java
private void establishProtocol(ConnectionSpecSelector
connectionSpecSelector) {}
```
这个函数主要是判断使用哪种协议，主要有三种：
* 一是如果route.address().sslSocketFactory() == null
那么就是不需要HTTPS，使用的是HTTP协议，直接用HTTP1.1，socket直接用rassocket，返回。
* 一里面不返回，自然是要使用HTTPS协议，那么调用connectTls(connectionSpecSelector);--->Q
* 三是判断if (protocol == Protocol.HTTP_2) ，因为二的调用导致了protocol的赋值.

总之如果if满足的话，就创建了一个http2Connection，并且执行了http2Connection.start()，去到http2Connection这个类里面，涉及了一些handler,runnable的操作，开启了connection。

Q.
``` java
private void connectTls(ConnectionSpecSelector connectionSpecSelector){}
```
这个函数比较长就不详细看了...大概就是用SSLSocketFactory创建了sslSocket然后选择了密码版本，TLS版本，配置了address的主机和协议，然后进行了握手，认证，最后用okio封装了sslSocket的输入和输出流。

在封装前还做了这样一件事：

``` java
// Success! Save the handshake and the ALPN protocol.

String maybeProtocol = connectionSpec.supportsTlsExtensions()
? Platform.get().getSelectedProtocol(sslSocket) : null;
// 封装了后,
protocol = maybeProtocol != null ? Protocol.get(maybeProtocol)
: Protocol.HTTP_1_1;

// 总之就是maybeProtocol似乎影响了protocol的选择= =唉不管了,比较涉及socket方面的内容.
```

##### **④StreamAllocation.JAVA**

**HTTP连接前需要进行socket握手，根据域名或代理确定socket的ip和端口。主要由这个类完成。**
使用了类似于**引用计数的方式跟踪Socket流的调用**，这里的计数对象是StreamAllocation，它被反复执行aquire与release操作，这两个函数其实是在改变RealConnection中的List>
的大小。

**E.**
``` java
public HttpCodec newStream(OkHttpClient client, **boolean**
doExtensiveHealthChecks)
该函数根据client设置的连接超时、读写超时、失败是否重试，通过调用findHealthyConnection函数真正的建立一个RealConnection连接，又通过这个连接创建一个HttpCodec：

RealConnection resultConnection = findHealthyConnection(connectTimeout,readTimeout,writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);--->F
，F主要是查找可用连接，一系列的步骤都有分析(F~Q都是），找到可用的连接后执行下一句，用这个链接创建HttpCodeC

HttpCodec resultCodec = resultConnection.newCodec(client, this); --->R

得到httpCodec之后，终于可以把结果返回给conn拦截器的入口了，即它的intercept函数里面了...
```

R.
``` java
public HttpCodec newCodec(）{}
```
这个函数比较简单，就是http2Con != NULL,就建立http2codec，否则就设置好各种超时限制建立http1codec,就是调用构造函数。不再详解。


F.
``` java
/**
该函数中查找健康连接，如果是全新连接可以直接返回，否则还要对候选的连接进行健康检查。如果检查结果不健康就调用noNewStream和continue结束这次循环操作，进入下个循环。
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
	  //找到候选连接
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);--->G

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
	  // 不是全新连接，继续检查：
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

G.
```java
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,boolean connectionRetryEnabled)
```

返回一个用于流执行底层IO的连接，优先复用已经创建的连接，没有可复用的就创建一个。

首先是锁住连接池，防止别的线程加入和移出连接，然后尝试上次分配的连接是否可用，行的话就返回给调用者。不行的话又通过Internal.instance.get()从连接池中取得连接，行的话就返回。如果还是不行的话，选择一个路由。连接池块结束。

如果路由为空，那么selectedRoute = routeSelector.next(); --->H
创建新的路由(HI的内容）.
I结束后出创建了新的路由，回到这里。再次锁住连接池，并且再次：
Internal.instance.get(connectionPool, address, this, selectedRoute);---> J
这个函数其实调用了pool的get，然后pool根据address选择链接，如果找到合适的连接，就使用acquire获取这个conn对象。

然后结束这个同步块，进行TCP+TLS的握手，真正开始建立连接:
``` java
result.connect(connectTimeout, readTimeout,
writeTimeout,connectionRetryEnabled);--->N
routeDatabase().connected(result.route());
```

L.
``` java
public void acquire(RealConnection connection) {}
```
在这个函数中，
connection.allocations.add(new StreamAllocationReference(this,
callStackTrace));

**把当前SA对象的引用保存到了RealConn的allocation里面，主要是为了追踪RealConn;**

这样看来，同一个连接RealConnection似乎可以同时为多个HTTP请求服务，但实际上，多个HTTP/1.1请求是不能在一个连接上交叉处理的。

这就是要看connection.allocationLimit的更新设置，当使用HTTP1.1的请求时，这个值就一直为1，当然了，如果是2.0协议，自然可以将这个值设置大于1，实现HTTP2.0的多路复用。

##### **⑤各种拦截器**

**T.CacheInterceptor**

在构造okHTTPclient时，可以设置cache对象，设置缓存的目录和大小。
如果不想用它的缓存策略，可以实现internalCache接口实现自定义缓存。

``` java
@Override
public Response intercept(Chain chain) throws IOException {
```

1）首先判断用户是否设置了cache，如果有的话，就调用cache.get(chain.request())尝试获取这个request对应的response。

**下面这段话是哪儿来的？？**
**就从cache里面获取cacheDandiate:**
**cacheCandiate就是上次与服务器交互缓存的Response，可能为null，它目前是一个可以读取缓存header的response：**

Response cacheCandidate = cache != null
? cache.get(chain.request()) ---->1A (1B,1C都是1A中引发的）: null;

2）然后缓存策略对当前信息进行一点加工：
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get(); ----> U// 参数为当前时间、当前请求、当前缓存响应。

3）如果cache不为空，那么调用cache.trackResponse(strategy);

4）如果有cache的候选，但是stragey返回的响应cacheResponse为空，说明这个cache候选用不了，关闭它，closeQuietly(cacheCandidate.body());

5）策略加工后输出的newtworkRequest（进行网络请求）和cacheResponse（可以用的缓存）有以下几种情况：

5.1）如果不能使用newtworkRequest并且stragey返回的响应cacheResponse为空，那么返**回504的response。表明不进行网络请求，且缓存不存在或者过期。**

5.2）如果只是不需要使用networkRequest，那么把strategy返回的响应cacheResponse给return就行了。**缓存可以用，而且不用进行网络请求。**

再下来当然就是可以用network了，

networkResponse = chain.proceed(networkRequest); 处理当前请求。

5.3）如果cacheResponse不为空，那么看network返回的响应，如果响应码是HTTP_NOT_MODIFIED（没有更新），那么直接返回cache的响应。**这是因为requst的头部中含有ETag/Last-Modified标签，需要在条件请求下使用，虽然有了缓存，但还是需要访问网络。**

4）没有返回cache的响应，自然要返回network的响应了。**这是因为缓存不存在或者过期了。**

然后或许要把这个请求的network的响应保存到cache中：

``` java
if (HttpHeaders.hasBody(response)) {
	CacheRequest cacheRequest = maybeCache(response, networkResponse.request(),
	cache);---->V
	
	//一圈下来终于得到了cacheRequest，终于可以把这个response写到这个cacherequest中了。
	response = cacheWritingResponse(cacheRequest, response);
}
```

最后返回response，这个拦截器的intercept就执行结束了。

人家关于缓存策略的总结（记得找参考的地址贴上来...)：
**Okhttp的缓存是自动完成的，完全由服务器Header决定的，自己没有必要进行控制。**网上热传的文章在Interceptor中手工添加缓存代码控制（比如服务器不支持的时候，重写cacheInterceptor，把它加到networkInterceptor中），它固然有用，但是属于Hack式的利用，违反了RFC文档标准，不建议使用，OkHttp的官方缓存控制在注释中。如果读者的需求是对象持久化，建议用文件储存或者数据库即可（比如realm）。


**1A:**
``` java
Response get(Request request) {}
```
cache中的函数，首先根据url得到key，然后看是否能从diskcache中取到snapshot，如果取到：
``` java
entry = new Entry(snapshot.getSource(ENTRY_METADATA));
--->括号中1B,得到[0]处的非缓冲流，并且构造一个Entry。Entry是cache中的静态常量类！！new Entry-->1C
```

cache类中一个内部类Entry。构造出了entry，也就是确定了响应信息的各种变量，再调用如下函数,也就是得到entry后：
```
Response response = entry.response(snapshot);
--->1D
//这个函数也是cache的内部类Entry的.所以是上一句，把各种信息构造成了Entry（entry的各变量被赋值），而这一步就吧entry的信息取出来构造成了response。。看起来很多余啊。。。。。。但感觉是为了把snapshot作为变量也传进去..
```
**V.**
发现这个函数现在版本没有了，待更新。
``` java
private CacheRequest maybeCache(Response userResponse, Request
networkRequest,InternalCache responseCache)
```

``` java
//该函数中，如果这个响应不能保存在这个request的cache中（isCacheable返回假）:
if (!CacheStrategy.isCacheable(userResponse, networkRequest)) {
那么判断这个request是否禁止cache（put中有详细禁止的判断，有五种方法禁止），是的话，就从cache中把这个请求remove掉：
	if (HttpMethod.invalidatesCache(networkRequest.method())) {
		try {
			responseCache.remove(networkRequest);
只进入第一个if就返回null。
否则就return responseCache.put(userResponse);---> W
```

**D.connection拦截器：**

httpCodec 和 realconnection的建立都是在这个拦截器中。通过sA的newStream中的findHealthyConn找到了符合目标服务器的realConn，然后利用这个realConn创建了httpcodec。
``` java
@Override 
public Response intercept(Chain chain) throws IOException {
	RealInterceptorChain realChain = (RealInterceptorChain) chain;
	// 获得这个链的request和stream
	Request request = realChain.request();
	StreamAllocation streamAllocation = realChain.streamAllocation();
	// We need the network to satisfy this request. Possibly for validating a conditional GET.
	boolean doExtensiveHealthChecks = !request.method().equals("GET");
	// 用stream建立一个httpcode,完成建立工作
	HttpCodec httpCodec = streamAllocation.newStream(client,
	doExtensiveHealthChecks); --->E 。 R执行完了之后终于回来了。
	RealConnection connection = streamAllocation.connection();//由于E-R的过程中，streamAllocation保存了connection，所以这个函数其实是return了这个sA对象保存的realConn。
	return realChain.proceed(request, streamAllocation, httpCodec, connection);

	//然后继续调用proceed，proceed中当然是应该执行callandserver这个拦截器了。--->S
}
```
**目前ABCD的执行过程：在realcall的**getResponseWithInterceptorChain()中，创建一个只含有拦截器链表和origirequest的chain,该chain执行proceed（requsest），在其中创建第index+1个拦截链next（其实就是去执行第index+1个拦截器），interact(next）执行当前index的拦截器的该方法，要把next这个链作为proceed的参数传下去，因为intercept中要继续调用proceed。

E到R整个过程都是这句话带来的：
HttpCodec httpCodec = streamAllocation.newStream(client,
doExtensiveHealthChecks);

主要就是创建了一个连接，并且记录了httpcodec。
**HttpCodec 则是利用 Okio 实现具体的数据 IO 操作。**

按照顺序，connect拦截器做完工作之后就是CallServerInterceptor。



**S.CallServerInterceptor**


**照样只有一个函数，居然那么长：**
``` java
@Override public Response intercept(Chain chain) throws IOException {
	HttpCodec httpCodec = ((RealInterceptorChain) chain).httpStream();
	StreamAllocation streamAllocation = ((RealInterceptorChain)
	chain).streamAllocation();
	Request request = chain.request();
	long sentRequestMillis = System.currentTimeMillis();
	
	//发送http头部信息
	httpCodec.writeRequestHeaders(request);
	Response.Builder responseBuilder = null;
	
	//如果http方法允许有请求body并且请求body不为null
	if (HttpMethod.permitsRequestBody(request.method()) && request.body() !=
	null) {
		// If there's a "Expect: 100-continue" header on the request, wait for a
		"HTTP/1.1 100
		// Continue" response before transmitting the request body. If we don't get
		that, return what
		// we did get (such as a 4xx response) without ever transmitting the request
		body.
		//等待截断body前得到的服务器同意100 continue的response，如果没有得到这个回应，返回我们实际得到的响应
		if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
			httpCodec.flushRequest();
			// 由于传入了true，如果statusLine.code == HTTP_CONTINUE，那么就会返回null
			responseBuilder = httpCodec.readResponseHeaders(true);
		}
		// Write the request body, unless an "Expect: 100-continue" expectation
failed.
		//是否运行当前方法带body，通过okio把请求body写入到
		if (responseBuilder == null) {
			Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
			BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
			request.body().writeTo(bufferedRequestBody); //这个函数，在rb这个类的writeTo函数中，把body的内容写到了这个缓冲池中
			bufferedRequestBody.close();
		}
	}

	//刷新。
	httpCodec.finishRequest();
	//再读取一遍响应头
	if (responseBuilder == null) {
		responseBuilder = httpCodec.readResponseHeaders(**false**);
	}
	
	// 读取响应头设置response
	Response response = responseBuilder
	.request(request)
	.handshake(streamAllocation.connection().handshake())
	.sentRequestAtMillis(sentRequestMillis)
	.receivedResponseAtMillis(System.currentTimeMillis())
	.build();
	int code = response.code();
	if (forWebSocket && code == 101) {
	// Connection is upgrading, but we need to ensure interceptors see a non-null response body.
		response = response.newBuilder()
			.body(Util.EMPTY_RESPONSE)
			.build();
	} else {
	//在有响应头的情况下读取响应body
	response = response.newBuilder()
			.body(httpCodec.openResponseBody(response))// 提供具体 HTTP 协议版本的响应 body
			.build();
	}
	if("close".equalsIgnoreCase(response.request().header("Connection")) || "close".equalsIgnoreCase(response.header("Connection"))) {
		streamAllocation.noNewStreams();
	}

	if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
		throw new ProtocolException(
			"HTTP " + code + " had non-zero Content-Length: " +
			response.body().contentLength());
	}
	return response;
}
```

关于except:100-continue:
在使用curl做POST的时候, 当要POST的数据大于1024字节的时候,
curl并不会直接就发起POST请求, 而是会分为俩步,；
　　流程如下：
　　1. 发送一个请求, 包含一个Expect:100-continue, 询问Server使用愿意接受数据
　　2. 接收到Server返回的100-continue应答以后, 才把数据POST给Server

使用100（不中断，继续）状态码的目的是为了在客户端发出请求体之前，让服务器根据客户端发出的请求信息（根据请求的头信息）来决定是否愿意接受来自客户端的包含了请求内容的请求；

##### **⑥SelectorRoute.JAVA**

**H**
这个函数好像也改过了...待更新
``` java
public Route next();
该函数中，首先三个if嵌套。顺序为：
判断是否有socket地址--》 判断是否有代理---》判断是否有route--》 抛出异常
一直是not状态。如果是yes状态跳到如下：
确定下一个socket地址 
lastProxy = nextProxy(); 
return nextPostponed();

// IP地址，代理，socket地址都找到后，建立新的路由
Route route = new Route(address, lastProxy, lastInetSocketAddress);-->I
```

##### **⑦Route.JAVA**

I.构造函数主要就是把参数address,proxy,socketaddress赋值给这个类的成员变量。

**⑧okHttpClient.JAVA**

``` java
**J.**
Internal.instance = new Internal() {....} 
是这个类中的一个static代码块。它的get函数其实是调用了连接池的get：

public RealConnection get(ConnectionPool pool, Address address,StreamAllocation streamAllocation, Route route) {
	return pool.get(address, streamAllocation, route); --> K
}
```

##### **⑨ConnectionPool.JAVA**

**主要用来管理realConnection。**

**K.**
``` java
RealConnection get(Address address, StreamAllocation streamAllocation,
Route route)；
	//这个函数遍历了所有连接，查找是否有address可复用的连接：
	for (RealConnection connection : connections) {
	//如果找到了，就把这个链接交给sA复用
	if (connection.isEligible(address, route)) { ---->M
		streamAllocation.acquire(connection); ---> L
		return connection;
	}
}
```
##### **⑩AndroidPlatform.JAVA**

**O**.
``` java
public void connectSocket(Socket socket, InetSocketAddress address, int connectTimeout)
```
该函数中调用了socket.connect(address,connectTimeout);成功与服务器建立连接，并且设置了链接的超时时间。address就是InetSocketAddress，包含了服务器的主机地址和端口号。

##### **⑪CacheStrategy.java**

1)private Date expires;
缓存的响应的到期时间，如果到期了，就需要网络连接.如果maxAge也设置了，以maxAge优先。

2)private String etag;
对资源文件的一种摘要，客户端不需要了解实现的细节，第一次请求时，服务器返回：

ETag: "5694c7ef-24dc"
客户端再次请求时，发送 If-None-Match:"5694c7ef-24dc"
交给服务器判断是否使用缓存。

**U**.
``` java
public Factory(long nowMillis, Request request, Response
cacheResponse)
//设置这个策略的各个成员变量。
//然后构造之后是一个链式调用，调用了get():

public CacheStrategy get() {
	CacheStrategy candidate = getCandidate();
	if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
		// We're forbidden from using the network and the cache is insufficient.
		return new CacheStrategy(null, null);
	}
	return candidate;
}
```
getCandidate();也是这个类里面的，太长了。大概就是根据构造函数里面的成员变量，选定了conditionalRequest和cacheResponse，然后new了新的cacheStragy给返回了。
选的这个过程大概就是策略了= =

#####**⑫Cache.JAVA**

(实现了internalCache接口， okhttp里面自带的cache）

**W.**
``` java
CacheRequest put(Response response) {}
	/*
	一是httpmethod的invalidatesCache判断该请求是否不能缓存（五个），不能的话当然从cache中把这个request删掉，然后返回null。五个不支持缓存的：
	post;put;patch;delete;move
	
	二是判断是不是GET方法，不是的话也返回NULL。
	虽然结果一样，但写清楚情况是非常必要的！！
	官方给的解释是缓存get方法得到的Response效率高，其它方法的Response缓存效率低。通常通过get方法获取到的数据都是固定不变的的，因此缓存效率自然就高了。其它方法会根据请求报文参数的不同得到不同的Response，因此缓存效率自然而然就低了。
	
	三是if(HttpHeaders.hasVaryAll(response))判断是否有通配符“*”，有的话也直接返回null。
	接着用response构造一个entry。
	然后构造一个diskLruCache的editor对象：
	*/

	Entry entry = new Entry(response);
	DiskLruCache.Editor editor = null;
	try {
		editor = cache.edit(key(response.request().url()));
		if (editor == null) {
			return null;
		}
	entry.writeTo(editor);
	return new CacheRequestImpl(editor);
}
```

edit方法得到了一个sink对象
接着用entry.writeTo(editor);把entry中的数据写到这个缓冲池editor里面。
最后：
return new CacheRequestImpl(editor); ---> X

CacheRequestImpl.JAVA
CacheRequestImpl是这个类中的一个实现了CacheRequest接口的常量类，主要是四个成员变量：
private final DiskLruCache.Editor editor;
private Sink cacheOut;
private Sink body;
boolean done;

X.
``` java
//在这个构造函数中，this.cacheOut = editor.newSink(ENTRY_BODY); 这一句 ，
把editor中保存的sink对象取出赋值给了crI的成员变量。构造函数最后一定要调用editor.commit();

public CacheRequestImpl(final DiskLruCache.Editor editor) {
	this.editor = editor;
	this.cacheOut = editor.newSink(ENTRY_BODY);--->Y //ENTRY_BODY的值是1
	this.body = new ForwardingSink(cacheOut) {
	@Override public void close() throws IOException {
		synchronized (Cache.this) {
			if (done) {
				return;
			}
			done = true;
			writeSuccessCount++;
		}
		super.close();
		editor.commit();
	}};
}
```

**Class Entry:**

```java
private static final class Entry {}
```

两种构造方式：
一是从Source in这个流得到信息进行处理，一是从response得到数据，直接赋值变量。

**1C.**
``` java
/*Reads an entry from an input stream.*/
public Entry(Source in) throws IOException {}
其实就是处理响应头，一般entry有两种形式：HTTP或者HTTPS
首先根据in得到BufferedSource source:
BufferedSource source = Okio.buffer(in);
然后根据source,得到响应头需要的信息：

public Entry(Source in) throws IOException {
	try {
		BufferedSource source = Okio.buffer(in);
		url = source.readUtf8LineStrict();
		requestMethod = source.readUtf8LineStrict();
		Headers.Builder varyHeadersBuilder = new Headers.Builder();
		int varyRequestHeaderLineCount = readInt(source);
		for (int i = 0; i < varyRequestHeaderLineCount; i++) {
			varyHeadersBuilder.addLenient(source.readUtf8LineStrict());
		}
		varyHeaders = varyHeadersBuilder.build();
		StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());
		protocol = statusLine.protocol;
		code = statusLine.code;
		message = statusLine.message;
		
		// Builder模式哦~~
		Headers.Builder responseHeadersBuilder = new Headers.Builder();
		int responseHeaderLineCount = readInt(source);
		for (int i = 0; i < responseHeaderLineCount; i++) {
			responseHeadersBuilder.addLenient(source.readUtf8LineStrict());
		}
		String sendRequestMillisString = responseHeadersBuilder.get(SENT_MILLIS);
		String receivedResponseMillisString = responseHeadersBuilder.get(RECEIVED_MILLIS);
		responseHeadersBuilder.removeAll(SENT_MILLIS);
		responseHeadersBuilder.removeAll(RECEIVED_MILLIS);
		sentRequestMillis = sendRequestMillisString != null ? Long.parseLong(sendRequestMillisString) : 0L;
		receivedResponseMillis = receivedResponseMillisString != null ?Long.parseLong(receivedResponseMillisString) : 0L;
		responseHeaders = responseHeadersBuilder.build();
		if (isHttps()) {
			String blank = source.readUtf8LineStrict();
			if (blank.length() > 0) {
				throw new IOException("expected \"\" but was \"" + blank +"\"");
			}
			String cipherSuiteString = source.readUtf8LineStrict();
			CipherSuite cipherSuite = CipherSuite.forJavaNam(cipherSuiteString);
			List<Certificate> peerCertificates = readCertificateList(source);
			List<Certificate> localCertificates = readCertificateList(source);
			TlsVersion tlsVersion = !source.exhausted() ? TlsVersion.forJavaName(source.readUtf8LineStrict()) : null;
			handshake = Handshake.get(tlsVersion, cipherSuite, peerCertificates,localCertificates);
		} else {
			handshake = null;
		}
	}finally { // 对应开始的try
		in.close();
	}
}
```
二是这种构造方法，response这种很好理解：
``` java
public Entry(Response response) {
	this.url = response.request().url().toString();
	this.varyHeaders = HttpHeaders.varyHeaders(response);
	this.requestMethod = response.request().method();
	this.protocol = response.protocol();
	this.code = response.code();
	this.message = response.message();
	this.responseHeaders = response.headers();
	this.handshake = response.handshake();
	this.sentRequestMillis = response.sentRequestAtMillis();
	this.receivedResponseMillis = response.receivedResponseAtMillis();
}
```
服气了= =先调用entry的构造方法构造了entry，又用entry反过来调用这个builder方法来构造response。response除了包括响应信息(Response的内容）外，还有对应的request。
``` java
public Response response(DiskLruCache.Snapshot snapshot) {
	String contentType = responseHeaders.get("Content-Type");
	String contentLength = responseHeaders.get("Content-Length");

	Request cacheRequest = new Request.Builder()
	.url(url)
	.method(requestMethod, null)
	.headers(varyHeaders)
	.build();

	return new Response.Builder()
	.request(cacheRequest)
	.protocol(protocol)
	.code(code)	
	.message(message)
	.headers(responseHeaders)
	.body(new CacheResponseBody(snapshot, contentType, contentLength))
	.handshake(handshake)	
	.sentRequestAtMillis(sentRequestMillis)
	.receivedResponseAtMillis(receivedResponseMillis)
	.build();
}
```
##### **⑬DiskLRUcache**

**Y**.
``` java
public Sink newSink(int index) {}
```

把File dirtyFile = entry.dirtyFiles[index];这个文件，写入到 a new unbuffered
output stream

**1B.**
``` java
/* Returns the unbuffered stream with the value for {\@code index}. */
public Source getSource(int index) {
	return sources[index];
}
```