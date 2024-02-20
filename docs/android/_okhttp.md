# OKHttp
[GitHub](https://github.com/square/okhttp)

### ExecutorService
在OkHttp中，`ExecutorService`是来自 Java 核心库`java.util.concurrent`包的一个接口，它用于管理和控制线程池。在处理异步网络请求时，OkHttp 可以与 `ExecutorService` 配合使用，允许开发者自定义执行网络回调的线程。
当您需要在特定的线程池上执行 OkHttp 的异步任务（例如通过`enqueue()`方法发起一个异步请求）时，可以将自定义的 ExecutorService 传递给 OkHttpClient.Builder 的`dispatcher()`方法进行配置：
    
    import java.util.concurrent.Executors
    import okhttp3.OkHttpClient

    // 创建一个单线程的 ExecutorService
    val executorService = Executors.newSingleThreadExecutor()

    // 构建 OkHttpClient 实例，并设置自定义的 ExecutorService
    val client = OkHttpClient.Builder()
        .dispatcher(Dispatcher(executorService))
        // 其他配置...
        .build()

    // 使用这个 OkHttpClient 进行异步请求，回调将在指定的 ExecutorService 中执行
    val request = Request.Builder().url("https://example.com").build()
    client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            // 处理失败回调
        }

        override fun onResponse(call: Call, response: Response) {
            // 处理成功回调
        }
    })

    // 不要忘记在应用退出或不再需要时关闭 ExecutorService
    executorService.shutdown()

在这个例子中，所有通过此 OkHttpClient 发起的异步网络请求的回调将会在我们提供的`ExecutorService`线程池中执行，而不是使用 OkHttp 内置的默认调度器。这样可以更灵活地控制异步任务的执行环境和并发策略

## 异步请求的调用顺序：
1. 使用者调用`Call.enqueue(Callback)`；
2. `Call.enqueue`中调用了`client.dispatcher().enqueue(new AsyncCall(responseCallback))`；
3. `dispatcher().enqueue`调用`promoteAndExecute`；
4. `promoteAndExecute`中会遍历`readyAsyncCalls`，放到`executableCalls`和`runningAsyncCalls`中，并调用`runningCallsCount`重新计算待执行的同步异步请求数量，然后遍历`executableCalls`，调用`asyncCall.executeOn(executorService())`；
5. `asyncCall.executeOn`中调用`executorService.execute(this)`，其中this为runnable类型的`asyncCall`，最后会调用其`run方法`；
6. Runnable的`run方法`中调用了`Response response = getResponseWithInterceptorChain()`，并调用`callback`,最终调用`dispatcher().finished`；
8. `dispatcher().finished`中又调用了`promoteAndExecute`方法，直到队列中的请求都执行完毕；

## 拦截器链
+ 拦截器是okhttp中一个强大的机制，可以实现网络监听，请求及响应重写，请求失败重试等功能；
+ 上面的同步请求异步请求源码中都有调用`getResponseWithInterceptorChain`方法，其代码如下:


    Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        //创建一系列拦截器，并放入list中
        List<Interceptor> interceptors = new ArrayList<>();
        interceptors.addAll(client.interceptors());
        //1. 重试和失败重定向拦截器
        interceptors.add(new RetryAndFollowUpInterceptor(client));
        //2. 桥接适配拦截器（如补充请求头，编码方式，压缩方式）
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        //3. 缓存拦截器
        interceptors.add(new CacheInterceptor(client.internalCache()));
        //4. 连接拦截器
        interceptors.add(new ConnectInterceptor(client));
        if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
        }
        //5. 网络io流拦截器
        interceptors.add(new CallServerInterceptor(forWebSocket));

        //创建拦截器链chain，并执行chain.proceed方法
        Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
            originalRequest, this, client.connectTimeoutMillis(),
            client.readTimeoutMillis(), client.writeTimeoutMillis());

        boolean calledNoMoreExchanges = false;
        try {
        Response response = chain.proceed(originalRequest);
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        return response;
        } catch (IOException e) {
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
        } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
        }
    }

### RealInterceptorChain
在OkHttp中，`RealInterceptorChain`是一个内部类，它是拦截器（Interceptor）链的具体实现。在OkHttp的请求处理过程中，拦截器机制是一个核心设计，通过定义一系列拦截器并串行或并行执行它们来实现对HTTP请求和响应生命周期中的不同阶段进行定制化处理。
`RealInterceptorChain`类通常包含以下关键信息和功能：
1. **Interceptors**：它存储了一个有序的拦截器列表，这些拦截器按照添加到客户端时的顺序排列。
2. **Index**：当前正在执行的拦截器索引，用于遍历整个拦截器链。
3. **Call**：封装了HTTP请求和其相关上下文信息的对象。
4. **Request**：当前正在进行处理的HTTP请求。
5. **Connection**：与服务器建立的连接对象，可能为空，因为在拦截链中某些拦截器可能会负责建立连接。
6. **ExchangeFinder**：用于查找或创建与请求对应的网络交换（exchange）的对象。
7. **EventListener**：监听HTTP请求和响应事件的接口实现，用于收集统计数据或调试信息。
8. **CallStackTrace**：用于记录调用栈信息，帮助开发者定位问题。

当发起一个网络请求时，OkHttp会创建一个`RealInterceptorChain`实例，并从第一个拦截器开始执行`intercept(Chain chain)`方法，每个拦截器在其方法内有机会修改请求、获取响应、重试请求或者执行其他操作。拦截器链会在所有拦截器都执行完毕后返回最终的HTTP响应给调用者

    // 这是一个简化的示例，实际源码结构会更复杂
    class RealInterceptorChain(
        private val interceptors: List<Interceptor>,
        private var index: Int,
        private val call: Call,
        private val request: Request,
        // 其他成员变量...
    ) : Chain {

        fun proceed(request: Request): Response {
            if (index < interceptors.size) {
                val next = copy(index = index + 1, request = request)
                val interceptor = interceptors[index]
                return try {
                    interceptor.intercept(next)
                } catch (e: IOException) {
                    throw e
                } catch (t: Throwable) {
                    throw IOException(t)
                }
            } else {
                // 如果已到达链尾，则执行实际的网络请求
                // 此处逻辑省略，涉及复杂的网络交互过程
            }
        }

        // 其他实现方法...
    }

上述代码是根据OkHttp的设计思路编写的简化示例，真实情况下的实现会更加详细和复杂，包括错误处理、连接复用以及与网络层的交互等细节。

**举个例子**
> 拦截器链例如 A->B->C  
> 执行顺序应该是 RealInterceptorChain(index = 0).proceed -> A.intercept -> RealInterceptorChain(index = 1).proceed -> B.intercept -> RealInterceptorChain(index = 2).proceed -> C.intercept -> 由于链尾的拦截器一定是`CallServerInterceptor`，其`intercept`方法会结束拦截器流程并执行网络请求返回结果

### CallServerInterceptor
在OkHttp库中，`CallServerInterceptor`是一个内置的拦截器，它是整个请求响应拦截链中的最后一个执行阶段。当请求到达这个拦截器时，客户端已经与服务器建立了连接，并且所有前置的拦截器（如`ConnectInterceptor`等）都已经完成了它们的工作。
`CallServerInterceptor`的主要职责是：
1. 发送HTTP请求到服务器：读取并写出请求头和请求体到已建立的Socket连接上。
2. 接收服务器响应：从Socket连接中读取HTTP响应头和响应体，并将它们封装成 Response 对象。
3. 处理可能存在的重定向、HTTP状态码和其他网络层面的交互。


    // 以下是一个简化的CallServerInterceptor实现示例，实际源码会更加复杂
    import okhttp3.*
    import java.io.IOException

    class CallServerInterceptor : Interceptor {

        @Throws(IOException::class)
        override fun intercept(chain: Chain): Response {
            val realChain = chain as RealInterceptorChain // 强制类型转换为RealInterceptorChain以便获取更多信息
            val request = realChain.request()

            // 创建流用于写入请求数据到服务器
            val sink = realChain.httpStream().writeRequestHeaders(request)

            // 如果请求有body，则写入body
            val requestBody = request.body()
            if (requestBody != null) {
                requestBody.writeTo(sink)
                sink.close()
            }

            // 从服务器读取响应
            val responseBuilder = Response.Builder()
                .request(request)
            val code = readResponseCode(realChain, sink)
            responseBuilder.code(code)

            // 读取响应头
            var line: String?
            while ({ line = realChain.httpStream().readUtf8LineStrict(); line }() != null) {
                val header = line!!.split(": ", limit = 2)
                responseBuilder.addHeader(header[0], header[1])
            }

            // 如果需要，读取响应体
            val responseBody = realChain.httpStream().newResponseBody(responseBuilder.build())
            return responseBuilder
                .body(responseBody)
                .build()
        }

        // 其他辅助方法，例如读取响应码等...

        private fun readResponseCode(realChain: RealInterceptorChain, sink: BufferedSink): Int {
            // 实现代码省略，这里是读取HTTP响应码的方法
        }
    }


请注意，上述代码仅作为概念性的简化实现，真实的 CallServerInterceptor 在处理HTTP请求和响应时会有更复杂的逻辑来确保正确性和高效性，同时还要处理各种异常情况和协议细节。
