* 任意代码执行漏洞
* 密码明文存储漏洞
* 域控制不严格漏洞

出现漏洞的原因有三个：

* WebView 中 addJavascriptInterface接口
* WebView 内置导出的 searchBoxJavaBridge_对象
* WebView 内置导出的 accessibility 和 accessibilityTraversalObject 对象

### 1、addJavascriptInterface接口导致任意代码执行漏洞

```javascript
webView.addJavascriptInterface(new JSObject(), "myObj");
//通过对象映射将Android中的本地对象和JS中的对象进行关联，从而实现JS调用Android的对象和方法
```
漏洞产生原因是：当JS拿到Android这个对象后，就可以调用这个Java对象中所有的方法，包括系统类（java.lang.Runtime），从而进行任意代码执行。
具体的：Java对象有公共方法getClass()，可以获取到当前类的Class。Class还有Class.forName方法，可以加载任意类，包括java.lang.Runtime，从而可以执行任意本地linux命令。

解决：
Google 在Android 4.2 版本中规定对被调用的函数以 @JavascriptInterface进行注解从而避免漏洞攻击。

### 2、WebView默认开启密码保存功能

```JavaScript
mWebView.setSavePassword(true)
```
开启后，在用户输入密码时，会弹出提示框：询问用户是否保存密码；

如果选择”是”，密码会被明文保到 /data/data/packageName/databases/webview.db 中，这样就有被盗取密码的危险。

解决：关闭密码保存功能

### 3、加载file协议的url
```JavaScript
// 设置是否允许 WebView 使用 File 协议
webView.getSettings().setAllowFileAccess(true);     
// 默认为true，即允许在 File 域下执行任意 JavaScript 代码
```
使用 file 域加载的 js 代码能够使用进行同源策略跨域访问（指的是对私有目录文件进行访问），从而导致隐私信息泄露。

解决：


```JavaScript
//关闭file协议支持
setAllowFileAccess(true); 
​
//对于确实需要file协议的页面，禁止加载 JavaScript
if (url.startsWith("file://") {
    setJavaScriptEnabled(false);
} else {
    setJavaScriptEnabled(true);
}
```
只禁用 javascript并不能完全杜绝文件泄露：因为如果应用的webview实现了下载功能，对于无法加载的页面，会自动下载到 sd 卡中；攻击者通过构造一个 file URL 指向被攻击应用的沙盒内文件，然后用此 URL 启动被攻击应用的 WebView，这样由于该 WebView 无法加载该文件，就会将该文件下载到 sd 卡下面，造成沙盒文件流出。

