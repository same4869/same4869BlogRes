---
title: 《Android技能树探索》总览
date: 2017-8-8
tags: Android
---

>这个系列也是属于构思了很久但是迟迟没法下手的，一方面也是因为自己心里面的那张技能图也不够清晰。深知在茫茫知识海洋中漫游拥有一张导航图是何等重要，前段时间发现了下面这张脑图，深以为然。于是想开始这个系列，在能力范围内把知识点都串一遍，看看又能收获多少了。

#### 1.Android App 开发技术图谱
>此图来自于https://github.com/mingjunli/AndroidDevResources/wiki/Android-App%E5%BC%80%E5%8F%91%E6%8A%80%E6%9C%AF%E5%9B%BE%E8%B0%B1
侵权联系删除

https://github.com/mingjunli/AndroidDevResources/wiki/media/14695437899506/Android_App_Skill_Map.png
![Alt text](/img/008/20170507-01.png)

感谢作者能够画出这样比较详细的结构脑图，接下来可以稍微大概看一下。

<!-- more -->


#### 2.庖丁初解牛

#### 2.1. 操作系统
Windows/MacOSX/Linux

#### 2.2. 编程语言
Java
HTML/JS (Hybrid/Web App)
C/C++ (NDK)
SQL (DB)
Kotlin

#### 2.3. 开发工具
##### 2.3.1 IDE
Android Studio
Eclipse

##### 2.3.2 调试工具
##### 2.3.2.1 网络调试
Charles
Wireshark
Fiddler
tcpdump
Paw/Postman
##### 2.3.2.2 内存分析
monitor
MAT
##### 2.3.2.3 android tools
adb
draw9patch
hierarchyviewer
uiautomatorviewer

##### 2.3.3 版本管理
##### 2.3.3.1 Git
Git命令
Github/GitLab
##### 2.3.3.2 SVN

##### 2.3.4 CodeReview
Gerrit
Github pull request

##### 2.3.5 Bug/任务管理
Redmine
JIRA
Bugzilla
Teambition
Tower

##### 2.3.6 编译工具
Gradle

##### 2.3.7 持续集成
Jenkins
Travis CI

##### 2.3.8 应用分发
蒲公英
fir.im

#### 2.4. App基础
##### 2.4.1 基本组件
Activity
Service
Content Provider
Broadcast Receiver
Intent/ Intent Filter
App Manifest File

##### 2.4.2 UI
Layouts
Widgets
Resources
Animations
设备适配

##### 2.4.3 Connectivity
Wifi
Mobile网络
网络状态监听

##### 2.4.4 MultiMedia
Audio/Video
Camera/Gallery

##### 2.4.5 GPS&Location&Map
###### 2.4.5.1 系统定位
GPS定位
Network定位
###### 2.4.5.2第三方Map定位
百度Map
高德Map

#### 2.5. App进阶
##### 2.5.1 Process&Thread
##### 2.5.1.1 Process
Linux进程
App进程原理
##### 2.5.1.2 AIDL
实现方式
原理
##### 2.5.1.3 Handler/Looper/MQ/Thread
##### 2.5.1.4 Loader
##### 2.5.1.5 AsyncTask

##### 2.5.2 性能优化
ANR
布局层级性能优化

##### 2.5.3 内存优化
内存检测工具
内存分析工具
Bitmap优化
内存泄漏查找及分析

##### 2.5.4 网络优化
##### 2.5.4.1 API优化
##### 2.5.4.2 低网速下优化
##### 2.5.4.3 流量使用优化
判断当前网络类型
使用缓存

##### 2.5.5 单元测试

#### 2.6. App高级
##### 2.6.1 相关原理熟悉
##### 2.6.1.1 Activity
启动流程
生命周期回调原理
与View/Window的关系
与Fragment的关系
##### 2.6.1.2 View/Window
View/Window关系
View渲染
View事情分发处理流程
##### 2.6.1.3 编译打包
编译打包原理
逆向工程分析
热修复

##### 2.6.2 Hybrid App
##### 2.6.2.1 与Native App的异同
##### 2.6.2.2 主流框架
PhoneGap
ionic
React Native

##### 2.6.3 架构能力
##### 2.6.3.1 架构
MVC
MVP
MVVM
Flux
Clean Architecture
##### 2.6.3.2 APP框架
分包
分层
##### 2.6.3.3 设计模式
OOD原则
常用设计模式运用

##### 2.6.4 ART&Dalvik
AOT compilation
GC
Bytecode&Dex

##### 2.6.5 自动化测试
monkey/monkey runner
UIAutomator
Espresso
Robotium

#### 2.7. 扩展学习
##### 2.7.1 响应式编程
##### 2.7.1.1 RX
RxJava
RxAndroid
RxBinding
##### 2.7.1.2 Agera

##### 2.7.2 主流开源库
##### 2.7.2.1 快速开发
Android Annotation
ButterKnife
##### 2.7.2.2 Views
太多
##### 2.7.2.3 Http模型
Retrofit
OkHttp
Volley
##### 2.7.2.4 图片处理
Glide
Fresco
Picasso
UIL
##### 2.7.2.5 依赖注入
Dagger2
##### 2.7.2.6 数据库
ORMLite
GreenDAO
Realm
Sugar
##### 2.7.2.7 辅助
Logger
LeakCanary
Dbinspector

#### 3.系列初步
以上基本是根据某位大神画的脑图来的，后面想根据这个依据来一个系列，然后对号入座吧，每个point都可能是很长很长的技术之路，可能能力有限不过争取尽可能地详细，总是也是一个边分析边学习的过程了。这边文章既是一个提纲也是一个依据，以后可能还会陆续扩充和完善。