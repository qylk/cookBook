## webview历史
* 从Android4.4系统开始，Chromium内核取代了Webkit内核。
* 从Android5.0系统开始，WebView移植成了一个独立的apk，可以不依赖系统而独立存在和更新。
* 从Android7.0 系统开始，如果用户手机里安装了 Chrome ， 系统优先选择 Chrome 为应用提供 WebView 渲染。
* 从Android8.0系统开始，默认开启WebView多进程模式，即WebView运行在独立的沙盒进程中。

## 优化 - 内核提前初始化
提前初始化webview内核，可提升第一次webview打开速度
```java
public class App extends Application {

    private WebView mWebView ;
    @Override
    public void onCreate() {
        super.onCreate();
        mWebView = new WebView(new MutableContextWrapper(this));
    }
}
```

## 优化 - 复用webview
在使用webview比较频繁的场景下, 复用webview可提高性能和节俭内存。
> <https://www.jianshu.com/p/fc7909e24178>
```java
public class WebPools {
    private final Queue<WebView> mWebViews;

    private Object lock = new Object();
    private static WebPools mWebPools = null;

    private static final AtomicReference<WebPools> mAtomicReference = new AtomicReference<>();
    private static final String TAG=WebPools.class.getSimpleName();

    private WebPools() {
        mWebViews = new LinkedBlockingQueue<>();
    }

    public static WebPools getInstance() {
        if (mWebPools != null)
            return mWebPools;
        if (mAtomicReference.compareAndSet(null, new WebPools()))
            return mWebPools=mAtomicReference.get();
    }

    public void recycle(WebView webView) {
        recycleInternal(webView);
    }

    public WebView acquireWebView(Activity activity) {
        return acquireWebViewInternal(activity);
    }

    private WebView acquireWebViewInternal(Activity activity) {

        WebView mWebView = mWebViews.poll();

        LogUtils.i(TAG,"acquireWebViewInternal  webview:"+mWebView);
        if (mWebView == null) {
            synchronized (lock) {
                return new WebView(new MutableContextWrapper(activity));
            }
        } else {
            MutableContextWrapper mMutableContextWrapper = (MutableContextWrapper) mWebView.getContext();
            mMutableContextWrapper.setBaseContext(activity);
            return mWebView;
        }
    }

    private void recycleInternal(WebView webView) {
        try {

            if (webView.getContext() instanceof MutableContextWrapper) {

                MutableContextWrapper mContext = (MutableContextWrapper) webView.getContext();
                mContext.setBaseContext(mContext.getApplicationContext());
                LogUtils.i(TAG,"enqueue  webview:"+webView);
                mWebViews.offer(webView);
            }
            if(webView.getContext() instanceof  Activity){
//            throw new RuntimeException("leaked");
                LogUtils.i(TAG,"Abandon this webview  ， It will cause leak if enqueue !");
            }

        }catch (Exception e){
            e.printStackTrace();
        }
    }

}
```
注意在 WebView 进入 WebPools 之前 ， 需要重置 WebView ，包括清空注入 WebView 的注入对象 ， 否则非常容易泄露。

## 优化 - webview独立进程
系统需要时间 Fork 出新进程， 那么加载变得更慢了，因为进程的创建也是一件耗时的事情，所谓的预加载进程， 就是提前把进程创建出来， 提升加载速度, 可以使用广播Broadcast提前启动 webview 进程
```xml
<activity
    android:name=".WebActivity"
    android:process=":web"
    />
```

## 优化 - 显示加载进度条
```java
 webView.setWebChromeClient(new WebChromeClient() {
            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                if (newProgress == 100) {
                    progressBar.setVisibility(View.GONE);//加载完网页进度条消失
                } else {
                    progressBar.setVisibility(View.VISIBLE);//开始加载网页时显示进度条
                    progressBar.setProgress(newProgress);//设置进度值
                }
                super.onProgressChanged(view, newProgress);
            }
        });
```

## 优化 - 开启硬件加速
性能提升还是很明显的，但是会耗费更大的内存
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        webView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        webView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
 } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
        webView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
}
```

## 优化 - 开启硬件加速
拦截请求，使用本地资源代替远程静态资源。
```java
@Override
public WebResourceResponse shouldInterceptRequest(WebView view, String url) {
    // 替换资源
    WebResourceResponse response = new WebResourceResponse("image/png",
                            "utf-8", resInputStream);
    // 参数1：http请求里该图片的Content-Type,此处图片为image/png
    // 参数2：编码类型
    // 参数3：存放着替换资源的输入流（上面创建的那个）
    return response;
}
```

## 优化 - webview使用注意事项
```java
//激活WebView为活跃状态，能正常执行网页的响应
webView.onResume() ；

//当页面被失去焦点被切换到后台不可见状态，需要执行onPause
//通过onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。
webView.onPause()；

//当应用程序(存在webview)被切换到后台时，这个方法不仅仅针对当前的webview而是全局的全应用程序的webview
//它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
webView.pauseTimers()
//恢复pauseTimers状态
webView.resumeTimers()；

//销毁Webview
//在关闭了Activity时，如果Webview的音乐或视频，还在播放，就必须销毁Webview
rootLayout.removeView(webView); 
webView.destroy();
```

```java
//声明WebSettings子类
WebSettings webSettings = webView.getSettings();

//如果访问的页面中要与Javascript交互，则webview必须设置支持Javascript
webSettings.setJavaScriptEnabled(true);  

//支持插件
webSettings.setPluginsEnabled(true); 

//设置自适应屏幕，两者合用
webSettings.setUseWideViewPort(true); //将图片调整到适合webview的大小 
webSettings.setLoadWithOverviewMode(true); // 缩放至屏幕的大小

//缩放操作
webSettings.setSupportZoom(true); //支持缩放，默认为true。是下面那个的前提。
webSettings.setBuiltInZoomControls(true); //设置内置的缩放控件。若为false，则该WebView不可缩放
webSettings.setDisplayZoomControls(false); //隐藏原生的缩放控件

//其他细节操作
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); //关闭webview中缓存 
webSettings.setAllowFileAccess(true); //设置可以访问文件 
webSettings.setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口 
webSettings.setLoadsImagesAutomatically(true); //支持自动加载图片
webSettings.setDefaultTextEncodingName("utf-8");//设置编码格式
```

```java
//SSL证书错误，不可忽略，需要由用户确认
@Override
public void onReceivedSslError(final WebView view, final SslErrorHandler handler, final SslError error) {
    new AlertDialog.Builder(context)
            .setTitle("安全警告")
            .setMessage("该网站的安全证书有问题")
            .setPositiveButton("继续", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    handler.proceed();
                }
            })
            .setNegativeButton("取消", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    handler.cancel();
                }
            }).create().show();
}
```

----
一个第三方webview封装库：<https://github.com/Justson/AgentWeb>