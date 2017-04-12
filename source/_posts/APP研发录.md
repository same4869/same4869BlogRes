---
title: 《APP研发录》读书笔记
tags: [个人随笔]
---
>前言：这篇是很早以前看《APP研发录》的读书笔记，也是很长一段时间唯一能看完一本并还能留下点东西的技术书籍。最近在整理越来越多的印象笔记，感觉只有不断地输出和写作才能营造出消化了部分的假象，为了让万事开头难更加简单，还是得坚持啊科科。

------
#### 第一部分 高效框架设计与重构
-----

<!-- more -->

##### 重构相关
- 包结构相关
![Alt text](/img/004/20170412-1.png)
- 每个文件只有一个单独的类，不要有嵌套类，比如在Activity中嵌套Adapter、Entity。
![Alt text](/img/004/20170412-2.png)
- 以上为一级分包样例参考，如涉及到具体模块，可以再继续分包
- 单一职责的定义是：一个类或方法，只做一件事情。 
- 设置BaseAcivity，重新定义生命周期，规范化整体风格

```
public abstract class BaseActivity extends Activity {
  @Override
    public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    initVariables();
    initViews(savedInstanceState);
    loadData();
  }
  protected abstract void initVariables();
  protected abstract void initViews(Bundle savedInstanceState);
  protected abstract void loadData();
} 
```
- 不建议在onClick里面写过多逻辑，建议抽成方法，让onClick简洁。
- 与服务器的数据交流不建议使用自带的JSONObject，会出现序列化和代码不简洁的问题，建议使用fastJson,Gson等等，注意混淆时该Keep的地方。
- 不要使用类间的全局变量，因为随时可能因为内存不足被系统回收，非要使用的话需要本地化这些值作保证。
- 类型转换建议在公共util类里面设置相应的安全方法，例如

```
public final static int convertToInt(Object value, int defaultValue)
{
  if (value == null || "".equals(value.toString().trim())) {
    return defaultValue;
  }
  try {
    return Integer.valueOf(value.toString());
  } catch (Exception e) {
    try {
      return Double.valueOf(value.toString()).intValue();
    } catch (Exception e1) {
      return defaultValue;
    }
  }
} 
```
- 时时刻刻注意空值，越界等异常的捕获与处理，永远不要相信服务器或者外部传过来的数据的合法性。   

------
##### 底层网络框架设计
- 如果自己写网络框架，使用AsyncTask居多，一般写法如下

``` 
Override
protected Response doInBackground(String… url) {
  return getResponseFromURL(url[0]);
}
private Response getResponseFromURL(String url) {
  Response response = new Response();
  HttpGet get = new HttpGet(url);
  String strResponse = null;
  try {
    HttpParams httpParameters = new BasicHttpParams();
    HttpConnectionParams.setConnectionTimeout(httpParameters, 8000);
    HttpClient httpClient = new DefaultHttpClient(httpParameters);
    HttpResponse httpResponse = httpClient.execute(get);
    if (httpResponse.getStatusLine().getStatusCode() 
        == HttpStatus.SC_OK) {
      strResponse = EntityUtils.toString(httpResponse.getEntity());
    }
  } catch (Exception e) {
    response.setErrorType(-1);
    response.setError(true);
    response.setErrorMessage(e.getMessage());
  }
  if (strResponse == null) {
    response.setErrorType(-1);
    response.setError(true);
    response.setErrorMessage("网络异常, 返回空值");
  } else {
    strResponse = "{'isError':false,'errorType':0,'errorMessage':'',
    'result':{'city':'北京','cityid':'101010100','temp':'17', 'WD':'西南风','WS':'2级','SD':'54%','WSE':'2','time':'23:15',
        'isRadar':'1','Radar':'JC_RADAR_AZ9010_JB',
          'njd':'暂无实况','qy':'1016'}}";
          response = JSON.parseObject(strResponse, Response.class);
}
return response;
} 

public abstract class RequestAsyncTask 
 extends AsyncTask<String, Void, Response> {
   public abstract void onSuccess(String content);
   public abstract void onFail(String errorMessage);
   @Override
     protected void onPreExecute() {
   }
   @Override
     protected void onPostExecute(Response response) {
     if(response.hasError()) {
       onFail(response.getErrorMessage());
     } else {
       onSuccess(response.getResult());
     }
   } 
``` 
- 至少两个地方可以优化：
1. 使用线程池
2. 支持实时取消
3. 一些回调可以抽出公共默认动作
- 书中介绍了一种ThreadPoolExecutor+Runnable+Handler的写法
>源码参见http://www.cnblogs.com/Jax/p/4656789.html   
>其他东西也可以在这里找到源码就是了

- 优化接口访问速度，可以为每个接口定义一个缓存时间，在一定时间内还是使用上次的结果，而不是重新请求后台。尤其针对一些get方式的查询类请求。
- 缓存存在SD卡中，URL及参数作为KEY，数据作为值。
- 请求时，先查看缓存时间时候过期，未过期则找本地缓存，如果没有再请求，并把请求结果存本地，更新缓存时间，如果有直接返回。
- 记得设置一个强制立即更新的方法，以备不时之需。
- 如果后台接口还没好，可以考虑先定义字段，然后使用MockService，其实就是造假数据啦。
- 书上那个略复杂，其实在BaseBean里面设置`public abstract String getJsonData();`方法，然后再具体的Bean里面生成假数据返回回去就行啦。
- 登录逻辑一般分为3种：
1. 点击登录按钮，进入登录页面LoginActivity，登录成功后，直接进入个人中心PersonCenterActivity。这种情况最直截了当，一路执行startActivity(intent)就能达到目的。
2. 在页面A，想要跳转到页面B，并携带一些参数，却发现没有登录，于是先跳转到登录页，登录成功后，再跳转到B页面，同时仍然带着那些参数。
3. 在页面A，执行某个操作，却发现没有登录，于是跳转到登录页，登录成功后，再回到页面A，继续执行该操作。 
   
```
@Override
  public void onSuccess(String content) {
    UserInfo userInfo = JSON.parseObject(content,
                                         UserInfo.class);
    if (userInfo != null) {
      User.getInstance().reset();
      User.getInstance().setLoginName(userInfo.getLoginName());
      User.getInstance().setScore(userInfo.getScore());
      User.getInstance().setUserName(userInfo.getUserName());
      User.getInstance().setLoginStatus(true);
      User.getInstance().save();
    }
    if(needCallback) {
      setResult(Activity.RESULT_OK);
      finish();
    } else {
      Intent intent = new Intent(LoginActivity.this, 
                                 PersonCenterActivity.class);
      startActivity(intent);
    }
  }
  }; 
```
- 自动登录一般依赖于cookie(token)，一般存放在Http-Response的header中，APP端不关心里面内容，每次请求当参数传给服务器，每次获得应答则刷新cookie。
- APP判断用户是否登录则检查这个值就行。服务器来判断cookie是否过期。
- 出于安全考虑，登录API如果被某一IP频繁访问后端需要作出反应，APP端应该给予配合，比如需要输入验证码啥的，防止恶意注册。密码什么的一定不要用明文。
- 每次HTTP Response头的Date属性里面返回服务器真实时间，减去APP本地时间获得差值并保存，同时暴露接口获得这个值，在请求的时候可能会用到。
- 在HTTPRequest头中key是Accept-Encoding，value是gzip，可以开启Gzip压缩。
- MobileAPI的逻辑是，检查HTTP请求头中的Accept-Encoding是否有gzip值，如果有，就会执行gzip压缩。
如果执行了gzip压缩，那么在返回值也就是HTTPResponse的头中，有一个content-encoding字段，会带有gzip的值；否则，就没有这个值。
- App检查HTTPResponse头中的content-encoding字段是否包含gzip值，这个值的有无，导致了App解析HTTPResponse的姿势不同，如下所示（以下代码参见HTTPRequest这个类）：

  
```
String strResponse = "";
if ((response.getEntity().getContentEncoding() != null)
    && (response.getEntity().getContentEncoding()
        .getValue() != null)) {
  if (response.getEntity().getContentEncoding()
      .getValue().contains("gzip")) {
    final InputStream in = response.getEntity()
      .getContent();
    final InputStream is = new GZIPInputStream(in);
    uest.inputStreamToString(is);
    is.close();
  } else {
    response.getEntity().writeTo(content);
    strResponse = new String(content.toByteArray()).trim();
  }
} else {
  response.getEntity().writeTo(content);
  strResponse = new String(content.toByteArray()).trim();
}      
```
-----

##### Android经典场景设计
- 1）简介ImageLoader。
地址：http://blog.csdn.net/yueqinglkong/article/details/27660107
2）Android-Universal-Image-Loader图片异步加载类库的使用（超详细配置）。
地址：http://blog.csdn.net/vipzjyno1/article/details/23206387
3）Android开源框架Universal-Image-Loader完全解析。
地址：http://blog.csdn.net/xiaanming/article/details/39057201
- 关于Fresco的更多介绍请参见：
·Fresco在GitHub上的源码：https://github.com/mkottman/AndroLua
·Fresco官方文档：http://fresco-cn.org/docs/index.html
- 加载网络图片的时候，一定要看显示控件的大小，然后把要显示的图片缩放到相同等级的大小内再显示，尤其是列表的item那种，不然轻松OOM。
- 针对以上策略，需要一个工具类，传入需要的宽高，然后给出缩放后的图片（bitmap）。
- 以上缩放也可以服务器操作，服务器和APP还可以约定一个低流量模式(降低图片质量)与极速模式（基本无图），通过APP端的不同网络监听进行合理的切换。
- 城市列表设计，建议本地保存一份，带版本号，服务器根据APP端的版本号返回增删改的数据，APP端更新并保存。
- App操作HTML5页面的方法
1. HTML5

```
 <a onclick="baobao.callAndroidMethod(100,100,'ccc',true)">
  CallAndroidMethod</a>
```
2. Android

```
wvAds.getSettings().setJavaScriptEnabled(true);
wvAds.loadUrl("file:// /android_asset/104.html");
btnShowAlert.setOnClickListener(new View.OnClickListener() {   
@Override    
public void onClick(View v) {      
String color = "#00ee00";      
wvAds.loadUrl("javascript: changeColor ('" + color + "');");    }}); 
```
- HTML5页面操作App页面的方法
1. HTML5

```
<a onclick="baobao.callAndroidMethod(100,100,'ccc',true)">
  CallAndroidMethod</a> 
```
2. Android

```
class JSInteface1 {
  public void callAndroidMethod(int a, float b, 
                                String c, boolean d) {
                                    if (d) {
      String strMessage = "-" + (a + 1) + "-" + (b + 1) 
        + "-" + c + "-" + d;
      new AlertDialog.Builder(MainActivity.this)
        .setTitle("title")
        .setMessage(strMessage).show();
    }
  }
} 
```
同时，需要注册baobao和JSInterface1的对应关系：

```
wvAds.addJavascriptInterface(new JSInteface1(), "baobao"); 
```
要在方法前增加@JavascriptInterface，否则，就不能触发JavaScript方法。
- 可以在APP的方法内加入参数，来让H5页面开发者能控制页面在APP内的跳转，之前需先约定好跳转的规则对应列表。
- 有时也可以使用H5来帮助做一些复杂的动画，比如在本地放一个HTML：

```
<html>
  <head>
  </head>
  <body>
    <table>
      <data1DefinedByBaobao>
        </table>    
      </body>
    </html> 
```
然后使用真实数据来替换 `<data1DefinedByBaobao>`
- Native和H5由后台可配到底显示哪个页面，可能一些特定的需求会用到这个策略，先暂存。

----
##### Android命名规范和编码规范
- Java类文件命名规范
1. Activity命名规范：以Activity作为后缀。比如说PersonActivity。
2. Adapter命名规范：以Adapter作为后缀。比如说PersonAdapater。
3. Entity命名规范：大多以Entity作为后缀。比如说PersonEntity。（Bean）
- 资源文件命名规范
1. 页面布局文件。以act_为前缀，以Activity所在的Package作为中缀，以Activity的名称（去掉Activity后缀）作为后缀。注意都是小写。
·例如，对于Person这个模块下的AddCustomerActivity，它的layout文件就应该是：act_person_addcustomer.xml。
2. ListView中的item布局文件。以item_作为固定前缀，列表项的名称为后缀。注意都是小写。例如，某个页面下有一个用户列表，控件名为lvUserList，那么item的layout就应该是：item_lvUserList.xml。
3. Dialog布局文件。
以dlg_作为固定前缀，Dialog的功能名称为后缀。注意都是小写，例如：dlg_hint.xml。
- JAVA中的对象使用驼峰式，常量大写，控件id和string等用下划线式，多多强迫症，多多处女座。
- codeFormat最好所有人共用一份xml，以免格式化后git上一片红。


------
#### 第二部分 App开发中的高级技巧-----
##### Crash异常收集与统计
- Crash分析三部曲
1. 收集：把Crash收集到本地数据库。
2. 统计：对每天线上大量的Crash进行去重、分类。
3. 分析：逐个分析各类Crash，重现异常发生的例子，给出解决方案。

- 使用UncaughtExceptionHandler收集异常，网上代码很多就不贴代码了，一般在handleException里面做三件事：
1. 错误日志到服务器。
2. 给用户崩溃前的友好提示。
3. 把错误日志记录到SD卡。

- 对服务器发送以下数据字段（参考）
![Alt text](/img/004/20170412-3.png)
- 在具体的Activity中，我们会将CrashType设置为0，而在CrashHandler中才会将CrashType设置为1。
- 服务器要及时统计这些数据并分类去重后由高到低排序。


-----
##### ProGuard相关    
- 以下是混淆最基本的配置信息，任何App都要使用，可以作为模板使用，作者为每行代码都增加了注释：  

```
# 代码混淆压缩比, 在0~7之间, 默认为5, 一般不需要改
  -optimizationpasses 5
  # 混淆时不使用大小写混合, 混淆后的类名为小写
    -dontusemixedcaseclassnames
    # 指定不去忽略非公共的库的类
      -dontskipnonpubliclibraryclasses
      # 指定不去忽略非公共的库的类的成员
        -dontskipnonpubliclibraryclassmembers
        # 不做预校验, preverify是proguard的4个步骤之一
        # Android不需要preverify, 去掉这一步可加快混淆速度
          -dontpreverify
          # 有了verbose这句话, 混淆后就会生成映射文件
          # 包含有类名->混淆后类名的映射关系
          # 然后使用printmapping指定映射文件的名称
            -verbose
              -printmapping proguardMapping.txt
              # 指定混淆时采用的算法, 后面的参数是一个过滤器
              # 这个过滤器是谷歌推荐的算法, 一般不改变
                -optimizations !code/simplification/arithmetic,!field/*,!class/merging/*
# 保护代码中的Annotation不被混淆
# 这在JSON实体映射时非常重要, 比如fastJson
-keepattributes *Annotation*
# 避免混淆泛型, 
# 这在JSON实体映射时非常重要, 比如fastJson
-keepattributes Signature
// 抛出异常时保留代码行号, 在第6章异常分析中我们提到过
-keepattributes SourceFile,LineNumberTable 
```
- 然后加入需要保留不混淆的东西，示例如下：

```
# 保留所有的本地native方法不被混淆
-keepclasseswithmembernames class * {
  native <methods>;
  }
# 保留了继承自Activity、Application这些类的子类
# 因为这些子类都有可能被外部调用
# 比如说, 第一行就保证了所有Activity的子类不要被混淆
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class * extends android.view.View
-keep public class com.android.vending.licensing.ILicensingService
# 如果有引用android-support-v4.jar包, 可以添加下面这行
-keep public class com.tuniu.app.ui.fragment.** {*;}
# 保留在Activity中的方法参数是view的方法, 
# 从而我们在layout里面编写onClick就不会被影响
-keepclassmembers class * extends android.app.Activity {
  public void *(android.view.View);
}
# 枚举类不能被混淆
-keepclassmembers enum * {
  public static **[] values();
  public static ** valueOf(java.lang.String);
}
# 保留自定义控件(继承自View)不被混淆
-keep public class * extends android.view.View {
  *** get*();
  void set*(***);
  public <init>(android.content.Context);
  public <init>(android.content.Context, android.util.AttributeSet);
  public <init>(android.content.Context, android.util.AttributeSet, int);
}
# 保留Parcelable序列化的类不被混淆
-keep class * implements android.os.Parcelable {
  public static final android.os.Parcelable$Creator *;
}
# 保留所有的本地native方法不被混淆
-keepclasseswithmembernames class * {
  native <methods>;
}
# 保留了继承自Activity、Application这些类的子类
# 因为这些子类都有可能被外部调用
# 比如说, 第一行就保证了所有Activity的子类不要被混淆
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class * extends android.view.View
-keep public class com.android.vending.licensing.ILicensingService
# 如果有引用android-support-v4.jar包, 可以添加下面这行
-keep public class com.tuniu.app.ui.fragment.** {*;}
# 保留在Activity中的方法参数是view的方法, 
# 从而我们在layout里面编写onClick就不会被影响
-keepclassmembers class * extends android.app.Activity {
  public void *(android.view.View);
}
# 枚举类不能被混淆
-keepclassmembers enum * {
  public static **[] values();
  public static ** valueOf(java.lang.String);
}
# 保留自定义控件(继承自View)不被混淆
-keep public class * extends android.view.View {
  *** get*();
  void set*(***);
  public <init>(android.content.Context);
  public <init>(android.content.Context, android.util.AttributeSet);
  public <init>(android.content.Context, android.util.AttributeSet, int);
}
# 保留Parcelable序列化的类不被混淆
-keep class * implements android.os.Parcelable {
  public static final android.os.Parcelable$Creator *;
}
# 保留Serializable序列化的类不被混淆
-keepclassmembers class * implements java.io.Serializable {
  static final long serialVersionUID;
  private static final java.io.ObjectStreamField[] serialPersistentFields;
  private void writeObject(java.io.ObjectOutputStream);
  private void readObject(java.io.ObjectInputStream);
  java.lang.Object writeReplace();
  java.lang.Object readResolve();
}
# 对于R(资源)下的所有类及其方法, 都不能被混淆
-keep class **.R$* {
  *;
}
# 对于带有回调函数onXXEvent的, 不能被混淆
-keepclassmembers class * {
  void *(**On*Event);
} 
```

----
##### 持续集成
- 关于git分支的流程，书中介绍了几种不同的方法与场景，比较值得认可的是主干开发，分支上线的一种策略。
- 主干开发，分支上线是在主干上开发新功能，测试阶段或者上个迭代没完成下个开发周期又开始的情况下，merge到分支上，留一部分人改bug或者开发遗留功能，另一部分在主干上开发新功能，分支上功能完成打包发版后合并到主干上。
- 分享一个以前在问问的时候代码提交流程：每个developer在专属自己的本地分支开发，非master成员（master成员才有资格把代码merge到主干分支）每次对master发送一个merge request而不是push，必须要master进行code review后由master合并代码。
- ant打包自行查阅
![Alt text](/img/004/20170412-4.png)
- 如果有Monkey包，可以在全局常量中设置isMonKey，还可以在N级页面中留暗门，来控制MonKey可达的页面。

----
##### App竞品技术分析
- APP打开速度优化关注点
1. 闪屏的下载策略
2. 引导动画的复杂度，引导页面的数量
3. 城市列表加载，城市定位
4. 首页的复杂度
5. 打点统计，推送，异常捕获的处理机制
6. 使用WireShark或者fiddler抓竞品包分析

- H5页面打开速度优化关注点
1. APP内放一个包含H5的ZIP包，首次启动解压，根据版本号判断，满足则加载本地。
2. 这个ZIP包可以是APP在适当位置下载的。
3. 可以使用增量包的形式，但要保证包不宜过大。
4. 开启webview缓存
5. 比较极端的做法，可以在上一个页面创建一个WebView，让它预先加载这个URL，这样就能提前把HTML5页面缓存到本地，一定要记住，要把这个WebView设置为不可见，否则就露馅了。

- APP大小优化关注点
1. 图片质量往往都是不必要的过大，一定要合理控制压缩，尤其超过1M的。
2. 音效，音频也应该选择合理比率和长度。
3. splash图片可以用jpg,其他的可以用png。
4. google的WebP格式图片尤其是安卓可以尝试。
5. 大图尽量使元素拆分。
6. 某些场景图标字体绝对是最佳解决方案。
7. 定期使用lint清理未使用的图片，当然还有代码中的各种不规范。

- APP网络请求性能优化关注点
1. 对于有多个线上服务器的，本地APP可以维护一个服务器列表，并自动选择最佳服务器。
2. 刚开的时候，遍历所有服务器接口，一般多次取平均值，获得最优的服务器使用，一般一段时间后比如一个小时重新遍历。
3. TCP+ProtoBuf策略代替HTTP+JSON，没试过，保留。

- 热修补方案，即H5和native页面通过后台来控制到底显示那个，需要注意页面的出入口，传递的参数，打点等等。完善还是需要一定积累的。
- AndroLua有机会可以研究下。

-根据Android的META-INFzhu'ru快速打渠道包，直接上脚本了

```
import zipfile
import shutil
import os
base_dir = '/Users/Shared/'
apk_name = 'ChannelDemo'
apk_path = base_dir + apk_name + '.apk'
empty_file = base_dir + 'baojianqiang'
f = open(empty_file, 'w')
f.close()
channel_file = base_dir + 'channel.txt'
f = open(channel_file)
lines = f.readlines()
f.close()
output_dir = base_dir + 'output'
if not os.path.exists(output_dir)
os.mkdir(output_dir)
for line in lines
target_channel = line.strip()
target_apk = output_dir + '/ChannelDemo_' + target_channel + '.apk'
shutil.copy(apk_path, target_apk)
zipped = zipfile.ZipFile(target_apk, 'a', zipfile.ZIP_DEFLATED)
empty_channel_file = 'META-INF/channel_' + target_channel
zipped.write(empty_file, empty_channel_file)
zipped.close()
```
- 还有一个获得渠道号的方法

```
private String getChannel(Context context) {
  try {
    PackageManager pm = context.getPackageManager();
    ApplicationInfo appInfo = pm.getApplicationInfo(
      context.getPackageName(), PackageManager.GET_META_DATA);
    return appInfo.metaData.getString("channel");
  } catch (PackageManager.NameNotFoundException ignored) {
  }
  return "";
  }
```