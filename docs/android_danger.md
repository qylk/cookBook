风险名称|风险|解决方案
---|---|---
1.App防止反编译|被反编译的暴露客户端逻辑，加密算法，密钥，等等|加固
2.java层代码源代码反编译风险|被反编译的暴露客户端逻辑，加密算法，密钥，等等|加固 ，混淆
3.so文件破解风险|导致核心代码泄漏。|so文件加固
4.篡改和二次打包风险|修改文件资源等，二次打包的添加病毒，广告，或者窃取支付密码，拦截短信等|资源文件混淆和校验签名的hash值
5.资源文件泄露风险|获取图片，js文件等文件，通过植入病毒，钓鱼页面获取用户敏感信息|资源混淆，加固等等
6.应用签名未交验风险|反编译或者二次打包，添加病毒代码，恶意代码，上传盗版App|对App进行签名证书校验
7.代码为混淆风险|业务逻辑暴露，加密算法，账号信息等等。|混淆（中文混淆）
8.webview明文存储密码风险|用户使用webview默认存储密码到databases/webview.db root的手机可以产看webview数据库，获取用户敏感信息|关闭wenview存储密码功能
9.明文数字证书风险|APK使用的数字证书用来校验服务器的合法性，保证数据的保密性和完成性 明文存储的证书被篡改造成数据被获取等|客户端校验服务器域名和数字证书等
10.调试日志函数调用风险|日志信息里面含有用户敏感信息等|关闭调试日志函数，删除打印的日志信息
11.AES/DES加密方法不安全使用风险|在使用AES/DES加密使用了ECB或者OFB工作模式，加密数据被选择明文攻击破解等|使用CBC和CFB工作模式等
12.RSA加密算法不安全风险|密数据被选择明文攻击破解和中间人攻击等导致用户敏感信息泄露|密码不要太短，使用正确的工作模式
13.密钥硬编码风险|用户使用加密算法的密钥设置成一个固定值导致密钥泄漏|动态生成加密密钥或者将密钥进程分段存储等
14.动态调试攻击风险|攻击者使用GDB，IDA调试追踪目标程序，获取用户敏感信息等|在so文件里面实现对调试进程的监听
15.应用数据任意备份风险|AndroidMainfest中allowBackup=true 攻击者可以使用adb命令对APP应用数据进行备份造成用户数据泄露|allowBackup=false
16.全局可读写内部文件风险。|实现不同软件之间数据共享，设置内部文件全局可读写造成其他应用也可以读取或者修改文件等|（1）.使用MODE_PRIVATE模式创建内部存储文件（2）.加密存储敏感数据3.避免在文件中存储明文和敏感信息
17.SharedPrefs全局可读写内部文件风险。|被其他应用读取或者修改文件等|使用正确的权限
18.Internal Storage数据全局可读写风险|当设置MODE_WORLD_READBLE或者设置android:sharedUserId导致敏感信息被其他应用程序读取等|设置正确的模式等
19.getDir数据全局可读写风险|当设置MODE_WORLD_READBLE或者设置android:sharedUserId导致敏感信息被其他应用程序读取等|设置正确的模式等
20.java层动态调试风险|AndroidManifest中调试的标记可以使用jdb进行调试，窃取用户敏感信息。|android：debuggable=“false”
21.内网测试信息残留风险|通过测试的Url，测试账号等对正式服务器进行攻击等|讲测试内网的日志清除，或者测试服务器和生产服务器不要使用同一个
22.随机数不安全使用风险|在使用SecureRandom类来生成随机数，其实并不是随机，导致使用的随机数和加密算法被破解。|（1）不使用setSeed方法（2）使用/dev/urandom或者/dev/random来初始化伪随机数生成器
23.Http传输数据风险|未加密的数据被第三方获取，造成数据泄露|使用Hpps
24.Htpps未校验服务器证书风险，Https未校验主机名风险，Https允许任意主机名风险|客户端没有对服务器进行身份完整性校验，造成中间人攻击|（1）.在X509TrustManager中的checkServerTrusted方法对服务器进行校验（2）.判断证书是否过期（3）.使用HostnameVerifier类检查证书中的主机名与使用证书的主机名是否一致
25.webview绕过证书校验风险|webview使用https协议加密的url没有校验服务器导致中间人攻击|校验服务器证书时候正确
26.界面劫持风险|用户输入密码的时候被一个假冒的页面遮挡获取用户信息等|（1）.使用第三方专业防界面劫持SDK（2）.校验当前是否是自己的页面
27.输入监听风险|用户输入的信息被监听或者按键位置被监听造成用户信息泄露等|自定义键盘
28.截屏攻击风险|对APP运行中的界面进行截图或者录制来获取用户信息|添加属性getWindow().setFlags(FLAG_SECURE)不让用户截图和录屏
29.动态注册Receiver风险|当动态注册Receiver默认生命周期是可以导出的可以被任意应用访问|使用带权限检验的registerReceiver API进行动态广播的注册
30.Content Provider数据泄露风险|权限设置不当导致用户信息|正确的使用权限
31.Service ，Activity，Broadcast,content provider组件导出风险|Activity被第三方应用访问导致被任意应用恶意调用|自定义权限
32.PendingIntent错误使用Intent风险|使用PendingIntent的时候，如果使用了一个空Intent，会导致恶意用户劫持修改Intent的内容|禁止使用一个空Intent去构造PendingIntent
33.Intent组件隐式调用风险|使用隐式Intent没有对接收端进行限制导致敏感信息被劫持|1.对接收端进行限制 2.建议使用显示调用方式发送Intent
34.Intent Scheme URL攻击风险|webview恶意调用App|对Intent做安全限制
35.Fragment注入攻击风险|出的PreferenceActivity的子类中，没有加入isValidFragment方法，进行fragment名的合法性校验，攻击者可能会绕过限制，访问未授权的界面|（1）.如果应用的Activity组件不必要导出，或者组件配置了intent filter标签，建议显示设置组件的“android:exported”属性为false（2）.重写isValidFragment方法，验证fragment来源的正确性
36.webview远程代码执行风险|风险：WebView.addJavascriptInterface方法注册可供JavaScript调用的Java对象，通过反射调用其他java类等|建议不使用addJavascriptInterface接口，对于Android API Level为17或者以上的Android系统，Google规定允许被调用的函数，必须在Java的远程方法上面声明一个@JavascriptInterface注解
37.zip文件解压目录遍历风险|Java代码在解压ZIP文件时，会使用到ZipEntry类的getName()方法，如果ZIP文件中包含“../”的字符串，该方法返回值里面原样返回，如果没有过滤掉getName()返回值中的“../”字符串，继续解压缩操作，就会在其他目录中创建解压的文件|（1）. 对重要的ZIP压缩包文件进行数字签名校验，校验通过才进行解压。 （2）. 检查Zip压缩包中使用ZipEntry.getName()获取的文件名中是否包含”../”或者”..”，检查”../”的时候不必进行URI Decode（以防通过URI编码”..%2F”来进行绕过），测试发现ZipEntry.getName()对于Zip包中有“..%2F”的文件路径不会进行处理。
38.Root设备运行风险|已经root的手机通过获取应用的敏感信息等|检测是否是root的手机禁止应用启动
39.模拟器运行风险|刷单，模拟虚拟位置等|禁止在虚拟器上运行
40.从sdcard加载Dex和so风险|未对Dex和So文件进行安全，完整性及校验，导致被替换，造成用户敏感信息泄露|（1）.放在APP的私有目录 （2）.对文件进行