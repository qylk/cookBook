## Intent
1. 尽量使用显式intent，如果可能有多个应用来接收一个Intent，请使用Intent Chooser。
2. 在多个自己可管理的应用间共享数据，请加自定义权限。
    ```xml
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <permission android:name="my_custom_permission_name"
                android:protectionLevel="signature" />
    <!--具有相同签名的app，才可以申请使用该权限-->            
    ```
3. 如果不必要，关闭组件导出功能
    ```xml
    <provider
        ...
        android:exported="false">
        <!-- Place child elements of <provider> here. -->
    </provider>
    ```
4. 尽量使用https协议和服务端通信
5. 网络安全配置，具体参考：<https://developer.android.com/training/articles/security-config.html?hl=zh-cn>
```xml
<manifest ... >
    <application
        android:networkSecurityConfig="@xml/network_security_config"
        ... >
        <!-- Place child elements of <application> element here. -->
    </application>
</manifest>

<!--xml/network_security_config.xml-->
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">secure.example.com</domain>
        ...
    </domain-config>
</network-security-config>

<!--开发阶段，可信任用户安装的证书-->
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

6. 小心处理webview与native通信，参考：[webview](./webview_native.md)
7. 使用FileProvider生成资源链接，在多个app间传递数据，并控制好读写权限
```java
Intent viewPdfIntent = new Intent(Intent.ACTION_VIEW);
viewPdfIntent.setData(Uri.parse("content://com.example/personal-info.pdf"));

// This flag gives the started app read access to the file.
viewPdfIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);

// Make sure that the user has a PDF viewer app installed on their device.
if (viewPdfIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(viewPdfIntent);
}
```
8. 敏感数据要放在沙盒存储内，必要时采取加密措施，秘钥等敏感数据也可以存储到系统`KeyStore`中
9. SQL参数化查询，避免SQL注入风险
10. 不向控制台输出敏感数据日志