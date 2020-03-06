# 输入法适配

## windowSoftInputMode
属性值|意义
---|---
adjustPan|当前窗口的内容将自动移动以便当前焦点从不被键盘覆盖和用户能总是看到输入内容的部分
adjustResize|调整window的大小以便留出软键盘的空间
adjustNothing|window布局不做任何调整


## 获取键盘高度
首先要获得键盘未弹出时根布局(这里指我们xml布局的根，非decorView)的可视高度，假定软键盘弹出时，根布局高度会进行调整，这时只要我们再次测量出根布局的可视高度，这前后的差值就是软件盘的高度。

获取布局的可视高度，使用`getWindowVisibleDisplayFrame`方法

1. 获取键盘未弹出时根布局的可视高度
时机有很多，比如第一次onSizeChanged、第一次onGlobalLayout等等

2. 监控根布局的onLayout回调
为了让软键盘弹出时，根布局会进行调整，需要调整activity的`windowSoftInputMode`为`adjustResize`模式，当键盘弹出/落下/高度变化时，根布局会重新layout，这时可以再次测量根布局可视高度

3. 根布局的size是否会变
当`windowSoftInputMode`为`adjustResize`模式时，键盘弹出/落下/高度变化时，根布局也将回调onSizeChanged，但是如果给decorView设置了全屏标志，比如沉浸式状态栏场景，则不会回调onSizeChanged，意味着View只调整了layout，而不改变size。
```java
getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN)
```

4. 全面屏手势的影响
全面屏手势，即可以快捷的开启或关闭手机底部的导航栏，当全面屏手势发生改变时，根布局的size将发生改变，这时我们需要重新计算`步骤1`中的可视高度，但是当键盘已经处于弹出状态时，就没法直接测量了，我们可以根据导航栏的状态和导航栏高度重新校准该值，但是导航栏的状态在不同机型上的获取方法不尽相同，当然也可能存在某些特殊方法能够检测，比如下面的方法通过检测导航栏view的当前高度来判断导航栏是否显示或隐藏：
```java
private static final String NAVIGATION = "navigationBarBackground";
    
    public static boolean isNavigationBarExist(Activity activity) {
        ViewGroup vp = (ViewGroup) activity.getWindow().getDecorView();
        if (vp != null) {
            for (int i = 0; i < vp.getChildCount(); i++) {
                vp.getChildAt(i).getContext().getPackageName();
                if (vp.getChildAt(i).getId() != NO_ID 
                    && NAVIGATION.equals(activity.getResources().getResourceEntryName(vp.getChildAt(i).getId()))) {
                    return vp.getChildAt(i).getHeight() > 0;
                }
            }
        }
        return false;
    }
```
导航栏的高度是一个确定值，这个可以比较容易的拿到。
```java
public static int getNavigationHeight(Context activity) {
        Resources resources = activity.getResources();
        int resourceId = resources.getIdentifier("navigation_bar_height", "dimen", "android");
        int height = 0;
        if (resourceId > 0) {
            height = resources.getDimensionPixelSize(resourceId);
        }
        return height;
}
```
因为根布局的size将发生改变，我们还可以根据size的新旧值的差值来校准`步骤1`中的可视高度，这个差值一般就是导航栏的高度，不过这个前提是导航栏隐藏和显示除此之外时布局是稳定的，所以我们一般使用根布局。