---
title: commlib使用指南
tags: Android
---

>一般来说重新开发一个项目都会有很多重复劳动，一些底层非业务的东西也总是大同小异，这几年看到的项目一般都是有个comm之类的lib，特点就是业务无关，通用性强，使用方便等等。网上也不乏有很多快速开发的框架，但是建议还是把日常积累的东西整理成一个自己的专属lib，不仅可以达到学习的目的，而且扩展性和熟悉程度都是不言而喻的，这个项目希望能够持续扩展，本篇文章也权当一个索引和使用指南了。

<!-- more -->

### 1.概况
本项目属于开源项目，git地址：https://github.com/same4869/XwangCommLib
然后这个项目已经上传上jcenter上面，所以任何项目想要使用里面的内容只需要在gradle里面添加`compile 'com.xwang:commlib:1.0.0'`就行了，最后一个是版本号，最新的版本号请查看最新的git log。
此外如果自己想把自己封装的lib上传到jcenter请查看 https://gold.xitu.io/entry/58992a452f301e00697a17b9 这篇博文。

### 2.索引
#### 2.1 CommCountDownTimer
和5.0以上的`CountDownTimer`源码完全一致，大概对比了一下，应该是解决低版本倒计时cancel可能出现的问题。
倒计时控件，使用方式大概如下
```
 new CommCountDownTimer(30000, 1000) {
 
      public void onTick(long millisUntilFinished) {
          mTextField.setText("seconds remaining: " + millisUntilFinished / 1000);
      }
 
      public void onFinish() {
          mTextField.setText("done!");
      }
   }.start();
```

#### 2.2 CommThreadPool
里面包含一个基于`newCachedThreadPool`的线程池，还包含一个`HandlerThread`和一个主线程的`Handler`。
暴露了3个方法出来
`poolExecute(Runnable runnable)`，使用线程池，一般耗时方法采用这个方法使用线程池管理。
`serialExecute(Runnable runnable)`，使用`HandlerThread`创建的固定线程。
`runOnUiThread(Runnable runnable)`，主线程中运行。
使用大概如下
```
     WenbaThreadPool.poolExecute(new Runnable() {

            @Override
            public void run() {
              ...
            }
        });
```

#### 2.3 SharedSetting
这个组件主要解决多应用间存储共享的问题，使用`ContentProvider`来实现，封装后可以像使用SP一样存储和读取多应用间的共享数据。
使用方式如下
##### 2.3.1
多应用中必须`有且仅`有一个应用（一般是主应用，其他应用使用的时候保证这个应用必须已经安装）在`AndroidManifest.xml`中声明
```
      <provider
            android:name="commlib.xun.com.commlib.sharedsetting.CommSharedSettingContentProvider"
            android:authorities="commlib.xun.com.commlib.sharedsetting.CommSharedSettingContentProvider"
            android:exported="true"/>
```
##### 2.3.2
在需要使用此功能的APP中建立`SharedSetting`类，里面跟封装的SP规则差不多
```
public class SharedSetting {
    private static final String TEST_OH = "test_oh";

    public static void setSavedPenAddress(String address) {
        CommSharedSettingOperator.save(SameMvpApplicationContext.application, TEST_OH, address, null);
    }

    public static String getSavedPenAddress() {
        return CommSharedSettingOperator.queryValue(SameMvpApplicationContext.application, TEST_OH);
    }
}
```
##### 2.3.3
存储使用如下
```
SharedSetting.setSavedPenAddress("我来自SameMvpDemo");
```
读取使用如下
```
SharedSetting.getSavedPenAddress()
```

#### 2.4 CommMultiThreadAsyncTask
继承自`AsyncTask`，用法也一致，其中暴露了`executeMultiThread`方法，会在API 11以上使用之前`CommThreadPool`里面所创建的线程池，然后也暴露出了`poolExecute`方法，和`CommThreadPool`的方法一致。
使用如下
```
 new CommMultiThreadAsyncTask<Void, Void, HomeWork>() {

            protected void onPreExecute() {
            }

            @Override
            protected HomeWork doInBackground(Void... params) {
           
            }

            @Override
            protected void onPostExecute(HomeWork result) {
          
            }

        }.executeMultiThread();
```
当然也可以继承`CommMultiThreadAsyncTask`来进行进一步的封装。

#### 2.5 CommWeakHandler
`CommWeakHandler`主要是用来解决日常中使用`handler`带来的内存泄露问题，用法跟普通的`handler`一样用就行了，主要要声明成全局和activity一个生命周期
```
private WeakHandler mHandler = new WeakHandler();
```
了解原理请移步
git地址https://github.com/badoo/android-weak-handler
在谷歌上搜索WeakHandler会用不少分析文章。