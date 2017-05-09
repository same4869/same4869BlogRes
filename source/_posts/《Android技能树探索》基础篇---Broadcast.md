---
title: 《Android技能树探索》基础篇---Broadcast
tags: Android
---

>广播一直以来都是使用频率仅次于Activity的四大组件，使用也非常简单。虽然现在很多项目都慢慢使用EventBus来替代广播，但必要的了解了对各种坑的预防依然还是必不可少的，好了废话就到这里。
#### 1.广播的类型？
- Normal broadcasts无序广播（sendBroadcast），会异步的发送给所有的Receiver，接收到广播的顺序是不确定的，有可能是同时。
- Ordered broadcasts有序广播（sendOrderBroadcast），广播会先发送给优先级高(android:priority -1000到1000)的Receiver，而且这个Receiver有权决定是继续发送到下一个Receiver或者是直接终止广播。
其他衍生广播：
- sendStickyBroadcast：Sticky简单说就是，在发送广播时Reciever还没有被注册，但它注册后还是可以收到在它之前发送的那条广播。
- LocalBroadcastManager.getInstance(this). sendBroadcast：应用内广播，没有特殊需求推荐使用，方便高效安全。

<!-- more -->

#### 2.广播的注册方式和注意事项？
动态注册和静态注册。
- 静态订阅广播又叫：常驻型广播，当你的应用程序关闭了，如果有广播信息来，你写的广播接收器同样的能接受到，他的注册方式就是在你的应用程序中的AndroidManifast.xml进行订阅的。
- 动态订阅广播又叫：非常驻型广播，当应用程序结束了，广播自然就没有了，比如你在activity中的onCreate或者onResume中订阅广播，同时你必须在onDestory或者onPause中取消广播订阅。不然会报异常，这样你的广播接收器就一个非常驻型的了。
>这里面还有一个细节那就是这两种订阅方式，在发送广播的时候需要注意的是：动态注册的时候使用的是隐式intent方式的，所以在发送广播的时候需要使用隐式Intent去发送，不然是广播接收者是接收不到广播的，这一点要注意。但是静态订阅的时候，因为在AndroidMainfest.xml中订阅的，所以在发送广播的时候使用显示Intent和隐式Intent都可以(当然这个只针对于我们自己定义的广播接收者)，所以以防万一，我们一般都采用隐式Intent去发送广播。
注：内部广播接收器一定要是`static`不然无效。

#### 3.BroadcastReceiver的生命周期？
Receiver也是有生命周期的，而且很短，当它的onReceive方法执行完成后，它的生命周期就结束了。这时BroadcastReceiver已经不处于active状态，被系统杀掉的概率极高，也就是说如果你在onReceive去开线程进行异步操作或者打开Dialog都有可能在没达到你要的结果时进程就被系统杀掉。因为这个进程可能只有这个Receiver这个组件在运行，当Receiver也执行完后就是一个empty进程，是最容易被系统杀掉的。替代的方案是用Notificaiton或者Service（这种情况当然不能用bindService）。
换句话说，当BroadcastReceiver在10秒内没有执行完毕，Android会认为该程序无响应。如果需要完成一项比较耗时的工作，应该通过发送Intent给Service，由Service来完成。而不是使用子线程的方法来解决，因为BroadcastReceiver的生命周期很短（在onReceive()执行后BroadcastReceiver 的实例就会被销毁），子线程可能还没有结束BroadcastReceiver就先结束了。如果BroadcastReceiver结束了，它的宿主进程还在运行，那么子线程还会继续执行。但宿主进程此时很容易在系统需要内存时被优先杀死，因为它属于空进程（没有任何活动组件的进程）。

#### 4.使用广播来更新界面是否合适？
综上，低频不耗时的更新没啥问题，如果是高频或者耗时的话就应该考虑其他方法了。发广播虽然方便但一定要统一管理，滥发广播会让项目变得相当不可维护，进程间通信最好使用aidl等更加安全可靠。

#### 5.安卓中跨进程通信的方式？
- 隐式Intent访问其他应用程序的Activity。
- Content Provider 
- 广播（Broadcast） 
- AIDL

#### 6.参考博文
>http://blog.csdn.net/jiangwei0910410003/article/details/19150705
>http://blog.chinaunix.net/uid-25314474-id-2846624.html
>http://www.jianshu.com/p/df7af437e766