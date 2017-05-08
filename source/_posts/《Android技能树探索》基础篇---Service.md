---
title: 《Android技能树探索》基础篇---Service
tags: Android
---

>开始填坑了，刚开始肯定也是从最基本的东西开始撸，其实东西最后要归根究底都没有什么最基础的，总之是一次自我复习的过程，能够发散多少也要看了。为了不显得特别零散，尽量保持一篇一个主题吧，内容保持问答的方式，也供人产生更多发散。
#### 1.Service有几种启动方式？
Service是一个专门在后台处理长时间任务的Android组件，它没有UI。它有两种启动方式，startService和bindService。

startService只是启动Service，启动它的组件（如Activity）和Service并没有关联，只有当Service调用stopSelf或者其他组件调用stopService服务才会终止。
bindService方法启动Service，其他组件可以通过回调获取Service的代理对象和Service交互，而这两方也进行了绑定，当启动方销毁时，Service也会自动进行unBind操作，当发现所有绑定都进行了unBind时才会销毁Service。

<!-- more -->

在这里再附上一张service的生命周期
![Alt text](/img/009/20170508-01.png)

#### 2.bindService是怎么进行绑定的？
和startService一样，bindService需要这样传参数：`bindService(bindIntent, connection, BIND_AUTO_CREATE);  `，一个参数和startService传入的Intent一样，第二个参数是一个ServiceConnection的匿名类对象，一般可以如下声明
```
private ServiceConnection connection = new ServiceConnection() {  
  
        @Override  
        public void onServiceDisconnected(ComponentName name) {  
        }  
  
        @Override  
        public void onServiceConnected(ComponentName name, IBinder service) {  
            myBinder = (MyService.MyBinder) service;  
            myBinder.startDownload();  
        }  
    };  
```
所以在需要启动的Service里面的`onBind`方法就不能返回null了
```
private MyBinder mBinder = new MyBinder();  
  @Override  
    public IBinder onBind(Intent intent) {  
        return mBinder;  
    }  
  
    class MyBinder extends Binder {  
  
        public void startDownload() {  
            Log.d("TAG", "startDownload() executed");  
            // 执行具体的下载任务  
        }  
  
    }  
```
在`onServiceConnected`里面返回的这个`IBinder`对象就是service中定义的那个，所以相当于是activity拿到了service暴露的方法接口，然后想怎么浪都随意了。
第三个参数参考一下，一般用第一个flag
>BIND_AUTO_CREATE表示当收到绑定请求时，如果服务尚未创建，则即刻创建，在系统内存不足，需要先销毁优先级组件来释放内存，且只有驻留该服务的进程成为被销毁对象时，服务才可被销毁；
>BIND_DEBUG_UNBIND通常用于调试场景中判断绑定的服务是否正确，但其会引起内存泄漏，因此非调试目的不建议使用；BIND_NOT_FOREGROUND表示系统将阻止驻留该服务的进程具有前台优先级，仅在后台运行，该标志位在Froyo中引入。

#### 3.bindService和startService的使用场景和注意事项？
- 使用startService方式来启动Service时，生命周期是`onCreate`-->`onStartCommand`，当在stopService之前多次startService会执行多次`onStartCommand`，但之后执行一次`onCreate`，使用`stopService`之后会调`onDestroy`。
- 使用`bindService`生命周期是`onCreate`，不会走`onStartCommand`，绑定成功会回调传入`ServiceConnection`的`onServiceConnected`的方法。
- `bindService`一旦绑定成功就相当于activity和service进行了一次连接，要么主动`unBindService`，要么actvity销毁的时候也会自动解绑。一个service可以和多个activity进行绑定，相当于多个activity可以功能操作一个service，这个也是bindService的精髓了，当所有activity和service解绑了，这个service也会销毁。
- 如果我们既点击了Start Service按钮，又点击了Bind Service按钮，不管你是单独点击Stop Service按钮还是Unbind Service按钮，Service都不会被销毁，必要将两个按钮都点击一下，Service才会被销毁。也就是说，点击Stop Service按钮只会让Service停止，点击Unbind Service按钮只会让Service和Activity解除关联，一个Service必须要在既没有和任何Activity关联又处理停止状态的时候才会被销毁。

#### 4.Service的onCreate回调函数可以做耗时的操作吗？
Service和Thread之间没有任何关系，Service中执行的代码也都是在主线程中执行的。
所以答案是不能。
使用Service主要是满足多个activity关注一个后台事件（比如下载，心跳，横竖屏切换状态恢复等）和一些其他需求（多进程，多APP间通信等），一定要和多线程区分开。
Service处理耗时操作同样是需要自己新开子线程来处理的，或者直接使用IntentService，可以直接在`onHandleIntent`方法里面处理耗时操作，里面东西设计到Handler的一些机制，后面有机会可以分析下，现在就不跑偏了。


#### 5.Service的保活策略
APP保活永远都是一场博弈论，Service保活作为一个小小分支主要也是了解了解。
- 将 Service 设置为 START_STICKY，利用系统机制在 Service 挂掉后自动拉活。
- 在service 的onDestory里面重启服务。
- 创建前台Service。
![Alt text](/img/009/20170508-02.png)
 可以通过以下方式来创建一个前台Service
```
public class MyService extends Service {  
  
    public static final String TAG = "MyService";  
  
    private MyBinder mBinder = new MyBinder();  
  
    @Override  
    public void onCreate() {  
        super.onCreate();  
        Notification notification = new Notification(R.drawable.ic_launcher,  
                "有通知到来", System.currentTimeMillis());  
        Intent notificationIntent = new Intent(this, MainActivity.class);  
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0,  
                notificationIntent, 0);  
        notification.setLatestEventInfo(this, "这是通知的标题", "这是通知的内容",  
                pendingIntent);  
        startForeground(1, notification);  
        Log.d(TAG, "onCreate() executed");  
    }  
  
    .........  
  
}  
```

#### 6.Service进程间通信
前面说到Service中执行耗时操作会阻塞UI线程导致ANR的，但是在注册Service的时候声明一下`android:process=":remote" `就不会ANR了。
因为这样Service就会运行到其他进程中，虽然还是阻塞了线程，不过倒不会影响到界面中去。
但是这种情况想要通过bindService来启动就万万不行了，华丽丽地会挂吧。
针对于Service进程间通信安卓给出了很好的解决方案---AIDL。
这种进程间（也可以是APP间）的通信方案在很多地方都有用武之地的，一般都是Messenger&AIDL。
大概流程就是自定义AIDL接口，系统会自动生成对应的JAVA类，这个类作为binder的子类可以通过bindService的方式传给activity，需要注意以下几点：
- 在两个APP中都是用同一份AIDL文件，然后通过bindService的方式就可以进行通信，主要包名路径名也需要是一样的。
- 再不同APP中，bindService的第一个参数那个intent需要使用隐式的action来启动，5.0后系统不允许隐式启动service，不过并没什么关系，网上搜索是有解决方案的。
- AIDL一般只能传递Java的基本数据类型、字符串、List或Map，传递自定义类型可以实现Parcelable接口。

>参考博客
>http://blog.csdn.net/guolin_blog/article/details/11952435
>http://www.jianshu.com/p/7a7db9f8692d?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=qq
>http://blog.csdn.net/lmj623565791/article/details/47143563
