## Activity四种启动模式：
1. standard：标准模式，默认的  
重复创建多个实例
谁启动了这种模式的 Activity，新 Activity 就会运行在启动者所在的栈中
ApplicationContext 启动 standard 的 Activity，会报错，需要加NEW_TASK标记

2. singleTop：栈顶复用模式  
如果位于栈顶则不会重复创建，不调用 onCreate 和 onStart，直接调用 onNewIntent() 方法

3. singleTask：栈内复用模式  
    * 只要 Activity 在一个栈中有实例，多次启动此 Activity 都不会创建实例，也是直接调用 onNewIntent()
    * 启动 singleTask 的 Activity 时，系统会先找有没有想要的任务栈，没有就新建个任务栈；有就看栈里有没有实例
    * 栈内有实例，clearTop（之前在它前面的都被清除），就会把该 Activity 调到栈顶
    * 一般用于 MainActivity，因为回到首页后需要清除之前的页面

4. singleInstance：栈内唯一  
就是霸道一点的 singleTask
启动后新建一个任务栈，这个栈里只会有它一个

## 标志位
1. FLAG_ACTIVITY_NEW_TASK  
如果 Activity 对应的 Task 已经存在就不会创建新的 Task，而是把旧的 Task 带到前台，同时其中的 Activity 也会保持之前的状态
一般用于一个类似“桌面”的 Activity，它的作用就是启动许多不同于当前 Task 的 Activity
2. FLAG_ACTIVITY_CLEAR_TOP  
    * 如果当前 Task 已经有要启动的 Activity，就不会直接创建新的，但是还要分下面两种情况
    * 如果这个 Activity 的启动模式是 standard 并且也没有使用 FLAG_ACTIVITY_SINGLE_TOP，会销毁已有的，新建 Activity
    * 如果是其他启动模式或者使用了 FLAG_ACTIVITY_SINGLE_TOP，就会直接调用已有的的 onNewIntent
    * 一般结合 FLAG_ACTIVITY_NEW_TASK 使用，达到的效果就和 singleTask 差不多了，比如用于通知栏中启动 Activity ，以达到将 Activity 所在 Task 调到前台，同时 clearTop 的效果
3. FLAG_ACTIVITY_SINGLE_TOP  
和 singleTop 效果一致