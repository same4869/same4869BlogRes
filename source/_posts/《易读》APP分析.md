---
title: 《易读》APP分析
date: 2017-7-19
tags: [Android]
---
>开篇：万事开头难，真的一点也不假。从意识到只有通过不断写作输出来整理笔记里的内容才有可能达到想达到的状态开始，也有好一段时间了。从基础开始吧，又觉得太散了无从下手，写架构吧，又怕能力不够变了纯吹牛逼。写作的目的本来也还是学习，在笔记本里收藏了N个开源优秀的项目了，也从来没有从头到尾好好看过，希望借这个机会能够从一而终一次吧，当然这也只是此时此刻单纯美好的愿望而已哈哈。

### 1.项目介绍
#### 1.1
git上的大牛也是越来越多，分享出来的优秀项目也不占少数，这个项目也是其中之一。想要研究这个项目的原因一是现在Rxjava+Retrofit+dagger2+MVP的开发框架都主流得烂大街了有点，而自己目前接触到的公司项目却并没有用到这些，虽然零零星星做过不少demo，毕竟跟一个具有相对完整功能的APP的触感是不一样的。再就是这个项目开发的比较新，功能相对完善，代码结构也还算清晰简单。
这里贴出项目的git地址，大家都可以去clone下来玩玩看，也可以去给作者点个star，哈哈
https://github.com/laotan7237/EasyReader

<!-- more -->

#### 1.2
大家可以通过git项目主页的介绍加上自己下下来跑一下就能了解APP实现的主要功能了，下面是从功能的角度出发画了个脑图，请品尝：
![Alt text](/img/006/20170418-01.png)
还算一目了然吧，所以这个APP的功能也还算简单的，就是一个新闻类的（好像开源的80%都是吧），也没有特别另类的功能，功能大概了解了，接下来就可以进入正题看代码了。

### 2.代码结构分析（1）
#### 2.1
整个项目只有一个moudle，包结构如下
![Alt text](/img/006/20170418-02.png)
传统分包模式，简单看一下
- adpater：适配器，基本都是继承BaseQuickAdapter，所以看列表实现基本上都是采用的RecyclerView了，后面慢慢分析了。
- app：Application和Constants配置
- bean：不解释了
- http：Retrofit相关的一些东西
- injector：Dagger2相关的一些东西
- presenter：MVP的P层，各种接口，和实现
- rx：RxBus相关的东西
- ui：activity和fragment
- utils：工具类
- view：自定义view
- webview：里面也是一些工具类，和公共的webview页面和相关自定义配置
到目前为止一级包结构也就这样了，还挺清晰吧。

#### 2.2
接下来就真的开始要愚公移山了，根据平时看代码的习惯开始好了，入口，一定是先从入口开始。
**App**
入口就是它了，里面代码很少，onCreate里初始化了一个全局变量外，有个`Utils.init(this)`，一个非常全的常用工具类，具体参考
https://github.com/Blankj/AndroidUtilCode/blob/master/README-CN.md
此外还有个`getAppComponent`方法，这个就要从漫长的Dagger2说起了。。。
Dagger2要从头说起估计这篇文章就写不完了，然后刚刚又去看了下这篇博文
https://dreamerhome.github.io/2016/07/07/dagger/
看一遍可能还是会一头雾水，那就多在网上搜一下相关的东西吧，然后后面就是默认对Dagger2有个一知半解才能继续愉快地玩耍了。

### 2.3
injector包的结构如下
![Alt text](/img/006/20170418-03.png)
看过Dagger2应该就比较清晰一点了，scope里面定义的注解一般都是`ActivityScope`和`FragmentScope`，相当于是提供Activity和Fragment单例实例化的支持。再一个就是用qualifier定义的注解，他这里是定义了一些具体的网络请求，可以通过这个注解来区分具体返回哪个请求的Retrofit对象。
`module`里面属于具体的提供者，提供什么就看需要什么了，这里面有各种adapter和各种http请求。然后我们知道component才是中间的粘合层，所以基本上module也会对应相应的component。
如果还是一头雾水的，可以继续去看Dagger2的相关，或者记住一个信条，这样做了之后，需要的对象只要在这里弄好然后在使用的时候添加下注解就行了，不需要再new了。

### 2.4
饶了一圈，又回来看`Application`了，刚刚其实就是这里还很多疑惑
![Alt text](/img/006/20170418-04.png)
现在应该只有些许疑惑了吧，这里是返回的component，这个component拥有`appModule`和`doubanHttpModule`这两个module所提供的能力，具体是这个样子的
![Alt text](/img/006/20170418-05.png)
所以就是`App`这个类提供了返回`AppComponent`的对象，这个对象可以提供获得App的`context`和或者一个`mRetrofitDouBanUtils`对象。

### 2.5
`mRetrofitDouBanUtils`是什么呢，然后又会吧Rxjava和Retrofit带出来，是不是有点拔出萝卜带出泥的感觉，哦不，萝卜还早着呢。所以决定跳到第一个activity里面去。
第一个是`SplashActivity`，简单的闪屏页，黄油刀butterknife应该都很熟悉了，配合AS的自动生成插件已经也算挺方便的了，解决了findViewById的重复劳动，虽然有不乏有人更加推崇Databinding+MVVM的方式，不过凡事总得慢慢来，好的东西可是可以去了解学习的，好了不扯远了，butterknife需要注意的地方是一般持有一个Unbinder，在`onDestroy`的时候要反绑定，也算是一个性能上的注意点了。
其他的没啥好说的，两秒后跳到`MainActivity`。

### 2.6
写到现在终于能看到主页面了，然后还是轮不到`MainActivity`，还有他的基类`BaseActivity`呢。`BaseActivity`里面里大致做了三件事情
- 管理所有的acticity页面，声明一个static的List，activity在onCreate时把自己放到list里面去，在onDestroy的时候从list里面移除，并且提供一个`killAll`的方法来关闭所有acticity。
- 实现类似IOS的滑动关闭页面，这个属于锦上添花的东西了，有兴趣的可以自己了解下。
- 实现`LifeSubscription`，添加rx的监听，然后在页面销毁时释放防止泄露。Rx的每个操作都可以作为一个Subscription添加到CompositeSubscription里面去，所以合理的就是在基类统一管理，在销毁页面的时候切断操作，防止内存泄漏。
除了以上三点外当然还应该做一些生命周期的常规封装，这个都是仁者见仁又大同小异，就不单独拧出来说了。

### 2.7
终于可以看看`MainActivity`，多少不容易啊，先看UI，UI能让人快乐。
看过APP效果的都知道用的`DrawerLayout`，用法当然不属于这里的讨论范围，不过还是贴个连接好了
http://blog.csdn.net/wangyangyang_n/article/details/50586921
然后布局就不用再细说啦，虽然侧滑里面的跟layout是用的`NestedScrollView`，不过在那个界面没啥卵用，所有关于`Material Design`的一些后面还有机会讲的。
`MainActivity`里面就是初始化了UI，结构就是`DrawerLayout`+`ViewPager`+`Fragment`这样子，然后值得一提的还有个`MaterialSearchView`开源控件，就是搜索框，也是属于特定需求的开源控件了，有兴趣了解下就行。
`ToastUtils`和`SPUtils`属于基本工具类了，怎么实现的姿势都行，还记得在application里面初始化的那个工具类吗，拿过来用就行。
以上都是架子，内容还没有，之前从产品功能上看不是分为三个大类吗，其实就是三个fragment啦。
![Alt text](/img/006/20170418-06.png)

### 2.8
从那张脑图上可以看出`HomeFragment`是最复杂的，所以先分析另外两个吧。
首先看`AndroidFragment`，不过说了这么久怎么连点MVP的影子都还没看到，恩，这才是刚刚开始。
先看父类`BaseFragment`，首先是持有泛型`BasePresenter`，`LifeSubscription`接口和之前分析的`BaseActivity`一样的，不多说了，另外一个接口`Stateful`又要扯到`LoadingPage`了。
`LoadingPage`是个不错的东西，在每个APP里面也都是必须的，封装了加载时，加载成功失败，失败后刷新等的页面状态，只需要传一个state，其他逻辑跟着回调来就行了，虽然也封装过类似的东西，不过这种思路还是挺值得学习的。
里面还有个点就是Fragment的懒加载，基本上也是出行必备的东西了，网站随便找一篇博客可以看看
http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1021/1813.html
此外还有几个虚函数共子类重写，状态保持和`mLoadingPage`一致。
这个类最开始用`@Inject`标签标注的也是给子类用的，相当于在这里new了一个该对象，至于为什么还是可以去看看Dagger2的东西哈。

### 2.9
接下来可以看`AndroidFragment`里面的东东了，这里决定先来张图以至于不要那么晕
![Alt text](/img/006/20170418-07.png)
是不是有点MVP的感觉了，Fragment作为View层持有了Presenter层的实例并继承View层的接口，那么Presenter拿到了数据就会回调到接口中，实现业务与界面的分离。
`SwipeRefreshLayout`是一个谷歌自己的下拉刷新组件，没什么特别的要求用这个应该是挺合适的，里面是标准的`RecyclerView`，相信现在出去说我们项目列表还是ListView实现估计要遭人鄙视了吧。
在`BaseFragment`的`onCreateView`方法里面会调用`loadBaseData`方法，然后会调`loadData`方法，在子类里面实现这个方法，在`AndroidFragment`中为
```
mPresenter.fetchGankIoData(page,PRE_PAGE);
```
就是在网络加载数据了，还是进去看看吧。

### 2.10
`BasePresenter`是持有一个`BaseView`的接口的，然后里面的`invoke`方法传入Observable和Subscriber的callback。
![Alt text](/img/006/20170418-08.png)
然后自然就关系HttpUtils的invoke方法了，里面更新了状态，判断了网络情况，然后就是RxJava的观察者绑定了
![Alt text](/img/006/20170418-09.png)
还会把subscription放到一个统一的池子里管理。
然后跳回来看看`GankIoAndroidPresenterImpl`的`fetchGankIoData`方法
```
    @Override
    public void fetchGankIoData(int page, int pre_page) {
        invoke(mRetrofitGankIoUtils.fetchGankIoData("Android",page,pre_page),new Callback<GankIoDataBean>(){
            @Override
            public void onResponse(GankIoDataBean data) {
                List<GankIoDataBean.ResultBean> results = data.getResults();
                checkState(results);
                mView.refreshView(results);
            }
        });
    }
```
所以第一个参数就是Retrofit返回的Observable了
```
    /**
     * 分类数据: http://gank.io/api/data/数据类型/请求个数/第几页
     * 数据类型： 福利 | Android | iOS | 休息视频 | 拓展资源 | 前端 | all
     * 请求个数： 数字，大于0
     * 第几页：数字，大于0
     * eg: http://gank.io/api/data/Android/10/1
     */
    @GET("data/{type}/{pre_page}/{page}")
    Observable<GankIoDataBean> getGankIoData(@Path("type") String id, @Path("page") int page, @Path("pre_page") int pre_page);
```
不得不说作者的注释真是写得够详细，点个赞。
中间的`RetrofitGankIoUtils`持有一个`GankIoService`作为`HttpUtils`的子类，这样就可以成为很好的分类封装了。
回调的`Callback`拿到数据之后检查状态看看应该显示怎样的页面，然后把数据交回给view完成显示。
到这里一个基于MVP的网络请求流程就走通了，感觉很爽有木有。
还有个比较值得说的是`AndroidFragment`里面的adapter，我们已经知道这个页面使用的是RecycleView，然后有一个比较牛逼的组件`BaseRecyclerViewAdapterHelper`，据说非常灵活还精简代码还完善了普通RecycleView的一些短板，比较点击事件，header和footer之类的，可以参考下面这个博文
http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0417/4154.html
可以结合本项目参考下使用方法，希望能在下个自己的项目中用进去。
最后需要注意的一个地方是`initInject`这个重载方法，我们在每个具体子页面里面通过这样注入了网络框架依赖和适配器依赖，就可以在这个页面里随便用啦。

### 2.11
看界面也能看出来，`WeChatFragment`和`AndroidFragment`里面的东西也是半斤八两啦。然后就是根据具体业务使用的是`WeChatPresenterImpl`和`WeChatPresenter.View`，这个逻辑就不重复说了，有点区别的就是还记得`MainActivity`里面的`MaterialSearchView`吗，里面有这么一段代码
```
        searchView.setOnQueryTextListener(new MaterialSearchView.OnQueryTextListener() {
            @Override
            public boolean onQueryTextSubmit(String query) {
                vpContent.setCurrentItem(2);
                RxBus.getDefault().post(AppConstants.WECHA_SEARCH, query);
                return false;
            }

            @Override
            public boolean onQueryTextChange(String newText) {
                return true;
            }
        });
```
所以这里就是RxBus的封装和使用了，看这篇好了
http://www.loongwind.com/archives/264.html
看完其他感觉也不用怎么说了，跟eventBus差不多的姿势就行了，所有我们可以发现在这里发出了要查询的关键字，然后通过`AppConstants.WECHA_SEARCH`这个常量在`WeChatFragment`里面接收到这个关键字然后
```
   mPresenter.fetchWXHotSearch(20, 1, s);
```
当然还不要忘了`CompositeSubscription`来管理，保持优雅的姿势。
至于`Presenter`都是差不多的流程，直接看代码会清楚很多的。

### 2.12
到此为止以为这两页面功能都已经分析完了，结果发现尼玛详情页完全没有说啊，幸好是个webview哈哈。这一小节来扫扫尾。
不得不说使用了`BaseQuickAdapter`代码真心精简了好多，viewholder也不需要自己写了的样子。`WeChatAdapter`使用的是一个cardview，然后详情页就是把title和url塞到`WebViewActivity`里面去。
`GankIoAndroidAdapter`里面的逻辑基本也都是一样的，值得一提的是把Glide根据不同的业务需求封装成了`GlideUtils`，可以参考一下，Glide对GIF也比较友好哟呵呵。
然后说说`WebViewActivity`，也算是居家旅行必备的一个累了，作者的介绍说满足拨打电话、发送短信、发送邮件、上传图片、播放视频等需求了，应该还不错吧，虽然也可以自己写，贴个地址
http://www.jianshu.com/p/163d39e562f0
能拿来主义的时候千万不要手软。
`WebViewActivity`在`toolbar`里面可以分享，复制和在浏览器中打开网页也比较好用，分享就是使用了安卓原生的`Intent.ACTION_SEND`就行分享，看上去效果也还不错，是时候干掉sharkSDK了呵呵，可以参考下面博文
http://blog.csdn.net/wuzhiguo1314/article/details/52950764

### 3.代码结构分析（2）
#### 3.1
终于只剩最后也是最大的这一块了，进度比我想象的要快很多啊，稳住我们能赢。
先来看看`HomeFragment`吧，这也页面使用了`TabLayout`+`ViewPager`的布局，就注定它也只是个容器，也没什么MVP，所以就直接继承了`Fragment`。`TabLayout`只需要两行代码就能实现一个指示器，还是非常推荐用的。
```
        mFragments.add(new ZhiHuHomeFragment());
        mFragments.add(new TopNewsFragment());
        mFragments.add(new DouBanMovieTopFragment());
        mFragments.add(new DouBanMovieLatestFragment());
```
然后里面又是四个Fragment，想想也知道这些大多结构逻辑都会是一样的，所以先看第一个吧。
额···其实分析了之前两个列表Fragment后感觉也没啥好说的了，找找亮点，找找亮点。
轮播图是个好东西啊，最开始来自web页面，现在在android上也是直接拿来
https://github.com/youth5201314/banner/
里面说的应该很详细了，虽然也没用过，不过需要的时候照着画瓢不至于画不出来吧。
这个轮播图和下面那个分类（项目中没啥卵用）是统一作为header添加到`RecyclerView`里面去的。
等等，为什么`RecyclerView`可以添加header??
还记得`BaseQuickAdapter`这个牛逼玩意儿吗，让能够像`listview`一样添加header和footer，真是鱼和熊掌都要啊。
这个页面的数据请求与加载也有点意思，一切要从
```
    @Override
    protected void loadData() {
        mPresenter.fetchData();
    }
```
说起。我们点进去看发现是这样的，首先执行了`fetchDailyData`方法请求了日报内容，成功后先是设置了title和title的type为0，再把请求到的数据填入了一个大的bean里面就是`homeListBean`，设置type为1，然后再执行`fetchHotList`，设置title和title的type为0，再把请求到的数据填入`homeListBean`，设置type为2，如此循环到最后的`fetchSectionList`，最后一个接口回来了再去回调view层。
然后这个type就是adpter的type了，`BaseQuickAdapter`有个子类`BaseMultiItemQuickAdapter`，可以使用`addItemType`来定义不同的列表项type，刚开始看到这个页面还以为是多个列表，没想到一个recycleView就能搞定，`ZhiHuAdapter`里面也已经做了很好的例子，然后把点击事件回调到外面，大概就是这样子的，其实这个adapter的排版方式有点臃肿，不过一想全部又嵌套一层list的话肯定有是更多坑了，不吐槽。

#### 3.2
从回调回来的结构看，日报和热门是走的同一个回调的，然后主题和专栏是分别不同的，所以是三个回调，但是前两者都是跳转到`ZhiHuDetailActivity`，后两者都是跳转到`ZhihuThemeActivity`。
值得一说的是跳转中使用了`ActivityOptionsCompat`来控制跳转动画，这个在Material Design是非常有用的，看上去就酷炫叼有木有，参考下面这篇
http://blog.csdn.net/qibin0506/article/details/48129139/

#### 3.3
这次没忘，接着看详情页吧。
先应该是`ZhiHuDetailActivity`，然后就可以聊聊Material Design了。
还是乐观了，还以为对activity的封装只有BaseActivity那么简单，其实并不是，都还没有MVP呢。还没有封装LoadingPage，Material Design也得封装一下吧。
所以结构应该是这个样子的
![Alt text](/img/006/20170418-10.png)
当然其实`LoadingBaseActivity`有6个子类，总是都是这种尿性了。
其实`LoadingBaseActivity`和之前分析的`BaseFragment`有些神似，个人感觉这样的封装还是很不错的，的确让子类省去了一些重复劳动，用接口重载的方式也比较灵活。
`ZhihuDetailBaseActivity`其实代码很少，Material Design可以看看这篇
http://blog.csdn.net/qq_31340657/article/details/51918773
感觉这种也没什么好分析，自己写过玩过踩过坑才算真正了解了。
再回到`ZhiHuDetailActivity`这个类，中间有个webview，需要在initview的时候各种初始化配置，下面那个点赞和评论的layout不是behavior，是一个简单的属性动画，然后在`loadData`的地方使用`mPresenter`请求了页面内容（也就是webview的填充内容）和评论点赞内容。
从`ZhiHuDetailPresenterImpl`里面跟进过去可以知道通过`showExtraInfo`和`refreshView`回调。
点击评论可以到评论详情里面去，当然是携带了这个页面的一堆参数过去的。

#### 3.4
阿西吧，`ZhiHuCommentActivity`又是一个`Toolbar`+`TabLayout`+`ViewPager`的布局，刚刚传过来参数只是简单的显示条数，肯定还是有具体的`Fragment`的。
对，就是`ZhiHuCommentFragment`，两个都是。其他的MVP结构就不再描述了，所持有的Presenter有`fetchShortCommentInfo`和`fetchLongCommentInfo`两个方法去请求具体的评论，再注入到`ZhiHuCommentAdapter`里面去。`CircleImageView`用的可以自取，参考
https://github.com/hdodenhof/CircleImageView
剩下里面的逻辑结合APP对应的页面看看就好，那个收起和展开点了似乎没啥反应啊，不过那都不重要。这个就先到这里了。

#### 3.5
接下来是不是应该看`ZhihuThemeActivity`，前面分析之后感觉这个也没啥好说，唯一有点不一样的是这个是个公共页面。`initView`和`loadData`都是走分支的，然后回调也不走一样的，填充到不同的adpter，刷新数据。然后每个item点击带上id继续走到`startZhiHuDetailActivity`里面，这样就通了。顺便说一句Glide+recycleview真是好组合啊，现在面试去问别人怎么样优化listview滑动卡顿感觉自己都不好意了。

#### 3.6
接下来是一个有意思的东西，`ZhiHuHomeFragment`里面的几个模块，日报，热门，专栏，主题，有个按钮跳到一个页面，可以动态调整这几个栏目的顺序，看上去还挺好玩儿的吧。这个页面是`HomeAdjustmentListActivity`，其实在recycleview时代实现这种效果已经不难了，看看这个吧
https://github.com/CymChad/BaseRecyclerViewAdapterHelper/wiki/%E6%B7%BB%E5%8A%A0%E6%8B%96%E6%8B%BD%E3%80%81%E6%BB%91%E5%8A%A8%E5%88%A0%E9%99%A4
然后配合sp就可以很轻松愉快地实现这个功能了。

#### 3.7
头条新闻这个`TopNewsFragment`刷不出来数据，就先不管了，反正都差不多的啦。`DouBanMovieTopFragment`也是RecyclerView没跑了，adapter太强大了，连LoadMoreView都可以传个自定义xml就行，然后跳转到`MovieTopDetailActivity`，跳转的动画其实挺好的，不过之前也已经分析过了，特定的UI结构也没啥好说的。`DouBanMovieLatestFragment`也是一样的，他们之间布局不同是因为一个使用了`StaggeredGridLayoutManager`一个使用了`LinearLayoutManager`。

### 4.总结
这个APP到目前为止就基本上告一段落，没想到原本计划1个月左右结果就搞了三天，总体来说这个APP还是有不少亮点和值得学习的地方，再次感谢作者，这也是第一个完整分析得差不多的APP，还是感觉收获不少，还是那句话，输出是消化的重要途径，所以，后面还要加油。