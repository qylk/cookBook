## RetryAndFollowUpInterceptor

它是一个负责失败重试和重定向的拦截器，OkHttpClient默认支持重试，相应的配置方式如下
```java
  public void setRetryOnConnectionFailure(boolean retryOnConnectionFailure) {
    this.retryOnConnectionFailure = retryOnConnectionFailure;
  }
```

### 异常的重连机制
但是并不是所有的网络请求失败了都可以进行重试的，因此RetryAndFollowUpInterceptor内部会进行检测网络请求异常和响应码的情况，根据这些情况判断是否需要重新进行网络请求。

RetryAndFollowUpInterceptor的功能主要包括如下几点

* 创建StreamAllocation对象  
* 调用realChain.proceed(request, streamAllocation, null, null)进行网络请求  
* 根据异常结果或者响应结果判断是否进行重新请求   
* 调用下一个拦截器，对responese进行处理，返回给上一个拦截器

其中第二和第三点是在while(true)内部执行的，也就是系统通过死循环来实现重连机制。
```java
    while (true) {
      ...
      Response response = null;
      boolean releaseConnection = true;
      try {
        //进行网络请求
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        //路由异常，socket连接失败
        //判断是否可以恢复重试，不行就跑异常退出，否则continue，再次进行网络请求
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        //网络连上了出现的IO异常
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        //其他异常不管了，releaseConnection为true时表示需要释放连接了。
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      ...
    }
```
重点方法：recover
```java
private boolean recover(IOException e, boolean requestSendStarted, Request userRequest) {
    //首先是来记录该次异常，进而在RouteDatabase中记录错误的route以降低优先级，
    //避免下次相同address的请求依然使用这个失败过的route。
    streamAllocation.streamFailed(e);

    // 1.判断 OkHttpClient 是否关闭了失败重连的机制
    if (!client.retryOnConnectionFailure()) return false;

    //body不能重复，就不能重试
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    //是否是严重异常
    if (!isRecoverable(e, requestSendStarted)) return false;

    //没有其他可用路由，比如断网
    if (!streamAllocation.hasMoreRoutes()) return false;

    //其他异常，可以重试
    return true;
  }

//判断是否是严重异常
private boolean isRecoverable(IOException e, boolean requestSendStarted) {
    //协议异常不重试
    if (e instanceof ProtocolException) {
      return false;
    }

    if (e instanceof InterruptedIOException) {
    //对于SocketTimeoutException的异常，表示连接超时异常，这个异常是可以进行重连的
      return e instanceof SocketTimeoutException && !requestSendStarted;
    }

    //ssl握手异常
    if (e instanceof SSLHandshakeException) {
        //证书验证异常不重试
      if (e.getCause() instanceof CertificateException) {
        return false;
      }
    }
    
    //域名检验失败异常不重试
    if (e instanceof SSLPeerUnverifiedException) {
      // e.g. a certificate pinning error.
      return false;
    }

    return true;
  }  
```
可见通过while循环，异常下的重试机制次数是不限制的，直到产生了不可重试的异常，比如无可用路由。

### followUpRequest 响应码检测
上面的while循环后面还有处理重定向的代码
```java
    Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      //内部有一个 MAX_FOLLOW_UPS常量，它表示该请求可以重定向多少次，如果重定向次数超过20次，将不再重定向
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
```
当代码可以执行到 followUpRequest 方法就表示这个请求是成功的，但是服务器返回的状态码可能不是 200 的情况，这时还需要对该请求进行检测，其主要就是通过返回码进行判断的。这里我们主要关注重定向相关的逻辑。

```java
case HTTP_MULT_CHOICE:
case HTTP_MOVED_PERM:
case HTTP_MOVED_TEMP:
case HTTP_SEE_OTHER:
    // 300,301,302,303几个重定向相关的状态码

    // Does the client allow redirects?
    if (!client.followRedirects()) {
        return null;
    }
    // 获取响应头 Location 值，这就是要重定向的地址；
    String location = userResponse.header("Location");
        if (location == null) {
             return null;
    }
    HttpUrl url = userResponse.request().url().resolve(location);
    // Don't follow redirects to unsupported protocols.
    if (url == null) return null;
    // 进行重定向的相关操作
```

一些常用的状态码：
```
100~199：指示信息，表示请求已接收，继续处理
200~299：请求成功，表示请求已被成功接收、理解
300~399：重定向，要完成请求必须进行更进一步的操作
400~499：客户端错误，请求有语法错误或请求无法实现
500~599：服务器端错误，服务器未能实现合法的请求
```