---
title: 基于Fragment的安卓插件化Demo
---
**说明:**本篇文章主要以代码的形式还原结构，并非从原理的角度来阐述插件化，相关原理请自行谷歌。

####1.工程总体结构
![Alt text](./0AB12936-5E1E-4DBC-907E-82EA671BED5E.png)
- app是主module，也就是唯一一个配置成`com.android.application`的module，里面是不需要插件化的主页面等。
- bbcomm是公共依赖库，里面放一些app或者其他插件都会用到的公共资源与控件。
- login是没有被插件化的模块，作为app的依赖库，本质上和app没什么区别。
- p_scanword是插件。
- pluginbase是插件框架。

####2.依赖关系
- app依赖bbcomm（如果需要使用公共控件）和login
- bbcomm依赖pluginbase
- login依赖bbcomm（如果需要使用公共控件）
- p_scanword依赖bbcomm（如果需要使用公共控件）
- pluginbase不依赖任何其他

####3.从零开始

1. 新建一个app的module，类型是application，正常新建一个activity，打造万能测试页面，如下图所示：
![Alt text](./4C318F8A-D351-4122-8EDD-674C18326103.png)
稍微解释下这个页面的元素：
- 第一个button是跳转到login里面的另一个activity的。
- 第二个button加载插件，把插件的页面展示到下面那个蓝色的layout里面去。
- 第三个button来自于bbcomm，同时插件里也使用了bbcomm里面的这个控件。

一个button没什么好说的，代码如下
```java
Intent loginIntent = new Intent(MainActivity.this,LoginMainActivity.class);
startActivity(loginIntent);
```

2. 在pluginbase里面先定义一个`PluginInfo`的实体类，里面包含了插件的相关信息，如下：
```java
public class PluginInfo {
    public String name;
    public String libDir;
    public ClassLoader loader;
    public Resources resources;
    public Resources.Theme theme;
    public PluginBase pluginBase;
    public Context context;
}
```
其他基本都是基本类型或者安卓基本类型，那么`PluginBase`是什么东西呢？先看看PluginBase里面的代码
```java
public abstract class PluginBase {
    public static final String PLUGIN_SCANWORD = "p_scanword";

    private static Context mContext;
    private FragmentManager mFragmentManager;
    private Context pluginContext;

    public void attach(Context context, FragmentManager manager) {
        mContext = context.getApplicationContext();
        mFragmentManager = manager;

        init();
    }

    public Context getHostContext() {
        return mContext;
    }

    public Context getPluginContext(){
        if(pluginContext != null){
            return pluginContext;
        }
        pluginContext = PluginManager.getInstance(mContext).makePluginContext(getPluginName(), mContext);
        return pluginContext;
    }
    
    protected FragmentManager getFragmentManager() {
        return mFragmentManager;
    }
    
    protected FragmentTransaction getFragmentTransaction(){
        FragmentManager fragmentManager = getFragmentManager();
        return fragmentManager.beginTransaction();
    }
    public abstract String getPluginName();
    public abstract void init();
    public abstract void show(int containerId);
    public abstract void hide();
    public abstract void detach();
}
```
可以看出来，`PluginBase`是一个虚类，里面声明了若干的虚函数，有一个attch的公共方法来接收宿主的`context`和`FragmentManager`，并且提供公共方法让外部使用这两个属性，此外还实现了一个`getPluginContext`的方法，这个方法可以获得插件的`context`，对于插件化具有重要的意义。
说到`getPluginContext`方法，可以看看里面的实现，其实主要就是` PluginManager.getInstance(mContext).makePluginContext(getPluginName(), mContext);`这句话没错吧。
然后就可以看看`PluginManager`是什么鬼了。

3. 其实分123并没有啥卵用，只是为了稍微有点层次感不至于太累，好了继续吧。
`PluginManager`既然是manager一般是单例的，所以首先是这样：
```java
    private PluginManager(Context context) {
        this.appContext = context.getApplicationContext();
    }

    public static PluginManager getInstance(Context context) {
        if (sInstance == null) {
            synchronized (PluginManager.class) {
                if (sInstance == null) {
                    sInstance = new PluginManager(context);
                }
            }
        }
        return sInstance;
    }
```
这个没啥好说的，接下来看看刚刚那个`makePluginContext`方法:
```java
 public Context makePluginContext(final String name, final Context outerContext) {
        if (isPluginTestMode) {
            return outerContext;
        }

        Context context = mPluginMap.get(name).context;
        if (context != null) {
            return context;
        }

        context = new ContextWrapper(outerContext) {
            @Override
            public Resources getResources() {
                return mPluginMap.get(name).resources;
            }

            @Override
            public AssetManager getAssets() {
                return mPluginMap.get(name).resources.getAssets();
            }

            @Override
            public Resources.Theme getTheme() {
                return mPluginMap.get(name).theme;
            }

            @Override
            public Object getSystemService(String name) {
                if (!Context.LAYOUT_INFLATER_SERVICE.equals(name)) {
                    return super.getSystemService(name);
                }

                LayoutInflater inflater = (LayoutInflater) super.getApplicationContext()
                        .getSystemService(Context.LAYOUT_INFLATER_SERVICE);

                LayoutInflater proxyInflater = inflater.cloneInContext(this);

                return proxyInflater;

            }
        };

        return context;
    }
```
`isPluginTestMode`这里可以先不管，我们可以看到这个`context`其实是根据name在一个名为`mPluginMap`的容器里面取出来的，如果取出为空就new一个`ContextWrapper`重新创建，并在各个重写方法里面都使用`mPluginMap`的值，这里需要注意的是`getSystemService`方法的重写，这样就能保证在使用`LayoutInflater`的时候能够使用正确的`context`而不会出错。
接下来需要看看`mPluginMap`是什么了吧，里面的数据是怎么放进去的。首先看看声明：
```java
private final Map<String, PluginInfo> mPluginMap = new HashMap<String, PluginInfo>();
```
所以`mPluginMap`就是一个hashmap，key就是插件对应的name，而value就是插件对应的PluginInfo。来看看是怎么把数据塞进去的：
```java
    public PluginInfo getPluginInfo(String pluginName) {
        if (mPluginMap.containsKey(pluginName)) {
            return mPluginMap.get(pluginName);
        }

        String pluginPath = getPluginPath(pluginName);

        PluginInfo pluginInfo = null;
        try {
            pluginInfo = loadPluginInfo(pluginName, pluginPath);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if (pluginInfo != null) {
            mPluginMap.put(pluginName, pluginInfo);
        }
        return pluginInfo;
    }
```
这种东西其实就像玩RPG游戏一样，游戏刚开始可能主角只想完成一个单纯的任务A，而在完成A的途中发现需要完成B，完成B的途中又要完成C才行，以此类推，所以一个游戏的任务链是A->B->C->.....->C->B-A，吐槽而已，同理，从代码中看到了其实核心方法是`loadPluginInfo`，而它的参数又由`getPluginPath`提供，所以先看看`getPluginPath`的内部代码吧：
```java
    private String getPluginPath(String pluginName) {
        File dir = new File(appContext.getFilesDir(), PLUGIN_DIR);
        if (!dir.exists()) {
            dir.mkdirs();
        }

        File saveFile = new File(dir, pluginName + ".apk"); // pluginA.apk
        if (saveFile != null && saveFile.exists() && saveFile.isFile()) {
            saveFile.setExecutable(true);
            return saveFile.getAbsolutePath();
        }
        return null;
    }
```
里面的appContext是在初始化单例的时候传入的宿主context，代码上面已经有了，来看下上面这个方法，我们从特定的目录中去取插件的apk，如果有则返回插件的路径，没有则返回空。好了，终于可以看`loadPluginInfo`方法了。
```java
private PluginInfo loadPluginInfo(String pluginName, String pluginPath)
            throws ClassNotFoundException, InstantiationException, IllegalAccessException {
        String optDir = getPluginOptDir(pluginName);
        String libDir = getPluginLibDir(pluginName);

        DexClassLoader dexClassLoader = createDexClassLoader(pluginPath, optDir, libDir);
        AssetManager assetManager = createAssetManager(pluginPath);
        Resources resources = createResources(assetManager);

        PluginInfo pluginInfo = new PluginInfo();

        pluginInfo.libDir = libDir;
        pluginInfo.loader = dexClassLoader;
        pluginInfo.resources = resources;

        Resources.Theme theme = resources.newTheme();
        theme.setTo(appContext.getTheme());
        pluginInfo.theme = theme;

        Class cls = Class.forName(getPluginLauncherName(pluginName), true, dexClassLoader);
        PluginBase pluginBase = (PluginBase) cls.newInstance();
        pluginInfo.pluginBase = pluginBase;

        return pluginInfo;
    }
```
核心来了！！首先通过`getPluginOptDir`和`getPluginLibDir`这两个方法新建和指定下opt和dir的目录，代码如下：
```java
 private String getPluginOptDir(String name) {
        File dir = new File(appContext.getFilesDir(), "/plugin/" + name + "/opt");
        dir.mkdirs();
        return dir.getAbsolutePath();
    }

    private String getPluginLibDir(String name) {
        File dir = new File(appContext.getFilesDir(), "/plugin/" + name + "/libs");
        dir.mkdirs();
        return dir.getAbsolutePath();
    }
```
然后通过
```java
    private DexClassLoader createDexClassLoader(String pluginPath, String optDir, String libDir) {
        return new DexClassLoader(pluginPath, optDir, libDir, appContext.getClassLoader());
    }
```
这个方法把`pluginPath`，`optDir`，`libDir`注入到`dexClassLoader`中生成新的`classLoader`，这个`classLoader`就会在我们指定的目录下面去找dex文件加载，这便是插件化中java代码插件化的原理了。
除了Java代码，当然还有资源文件，看到这两行代码木有：
```java
        AssetManager assetManager = createAssetManager(pluginPath);
        Resources resources = createResources(assetManager);
```
我们会构造一个resources出来，这个就是插件的resources的啦，那是怎么构造出来的呢，接着看：
```java
 private AssetManager createAssetManager(String dexPath) {
        try {
            AssetManager assetManager = AssetManager.class.newInstance();
            Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
            addAssetPath.invoke(assetManager, dexPath);
            return assetManager;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    private Resources createResources(AssetManager assetManager) {
        Resources superRes = appContext.getResources();
        Resources resources = new Resources(assetManager, superRes.getDisplayMetrics(), superRes.getConfiguration());
        return resources;
    }
```
第一个方法是通过反射把插件路径注入到`assetManager`中生成能够支持插件`assetManager`，然后用这个`assetManager`重新构造出一个
`resources`。
好了，万事俱备了，可以构建`pluginInfo`了，翻上去看代码就懂了，然后会反射出一个`PluginBase`给`pluginInfo`的`PluginBase`赋值，看看是怎么反射出来的:
```java
    private String getPluginLauncherName(String pluginName) {
        return new StringBuffer("com.example.xunwang.").append(pluginName).append(".LauncherPlugin").toString();
    }
```
是不是代码不要太简单，其实拼的这个字符串是我们每一个插件的入口，所以在每一个插件的这个路径下必须要有这样一个入口，不然当然就反射不到啦，再就是`LauncherPlugin`必须是继承`PluginBase`的，这个不用多说吧。
到现在为止`PluginInfo`已经全部初始化完毕了，是不是都忘了最开始要干啥了。

4. 经过上面的分析可能有点小晕了，还记得`getPluginPath`这个方法吗，不记得了翻回去看看，注意`File dir = new File(appContext.getFilesDir(), PLUGIN_DIR);`和`File saveFile = new File(dir, pluginName + ".apk"); `这两句话，其实这个路径是在getFilesDir()的PLUGIN_DIR文件夹下找pluginName.apk这个文件。
但是其实打包脚本是把插件的apk打包到app这个module的asset目录下的，那么就需要作一个拷贝操作了，也就是`PluginManager`的`loadPlugins`方法了：
```java
public void loadPlugins(String[] pluginNames) {
        PluginLoader pluginLoader = new PluginLoader(appContext);
        pluginLoader.setPluginLoaderListener(new PluginLoader.PluginLoaderListener() {

            @Override
            public void loadResult(String[][] results) {
                Log.e(TAG "loadPlugins finished");
                if (results == null) {
                    return;
                }
                for (int i = 0; i < results.length; i++) {
                    Log.e(TAG, results[i][0] + ":" + results[i][1]);
                }
            }
        });
        pluginLoader.execute(pluginNames);
    }
```
执行了一个名叫`PluginLoader`的异步操作，里面有一个加载完成的回调，所谓的加载也就是把asset目录下插件的apk拷贝到getFilesDir()的PLUGIN_DIR目录下而已，不说了上代码：
```java
public class PluginLoader extends AsyncTask<String, Void, String[][]> {
	public static final String PLUGIN_DIR = "plugin";
	private Context mContext;
	private PluginLoaderListener pluginLoaderListener;

	public interface PluginLoaderListener {
		public void loadResult(String[][] result);
	}

	public void setPluginLoaderListener(PluginLoaderListener pluginLoaderListener) {
		this.pluginLoaderListener = pluginLoaderListener;
	}

	public PluginLoader(Context context) {
		this.mContext = context;
	}

	@Override
	protected String[][] doInBackground(String... params) {
		if (params == null || params.length == 0) {
			return null;
		}
		String[][] results = new String[params.length][2];
		for (int i = 0; i < params.length; i++) {
			String apkPath = null;
			try {
				apkPath = loadPlugin(params[i]);
			} catch (Exception e) {
				e.printStackTrace();
			}
			results[i][0] = params[i];
			results[i][1] = apkPath;
		}
		return results;
	}

	private String loadPlugin(String plugName) throws Exception {
		File dir = new File(mContext.getFilesDir(), PLUGIN_DIR);
		dir.mkdirs();
		File saveFile = new File(dir, plugName + ".apk");
		saveFile.setExecutable(true);
		FileOutputStream outputStream = null;
		InputStream inputStream = null;
		try {
			int count = 0;
			byte[] buf = new byte[1024];
			inputStream = mContext.getAssets().open(plugName + ".apk");
			outputStream = new FileOutputStream(saveFile);
			while ((count = inputStream.read(buf)) > 0) {
				outputStream.write(buf, 0, count);
			}
		} finally {
			inputStream.close();
			outputStream.close();
		}

		Log.e(TAG, "load pulgin write: " + saveFile.length());
		return saveFile.getAbsolutePath();
	}

	@Override
	protected void onPostExecute(String[][] result) {
		if (pluginLoaderListener != null) {
			pluginLoaderListener.loadResult(result);
		}
	}
}
```
`loadPlugins`通常在Application的时候就做这个操作，把所有的插件名放到一个数组中就行了：
```java
private String[] plugins = { PluginBase.PLUGIN_SCANWORD };
PluginManager.getInstance(getApplicationContext()).loadPlugins(plugins);
```

5. 到现在为止，又完成一大步了，我们来看看pluginbase这个module的文件
![Alt text](./68BF56AB-C752-44EB-A52A-3410B2FA9D73.png)
后面四个文件上面基本上都了解完了，最后来看看PlugFragment吧。
```java
public abstract class PlugFragment extends Fragment {
    private Context context;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        context = PluginManager.getInstance(getActivity()).makePluginContext(getPluginName(), getActivity());
    }

    public Context getContext(){
        return context;
    }

    protected abstract String getPluginName();

}
```
它是所有插件里面fragment的基类，封装了context为插件的context，里面的实现前面也详细了解过了，所以PluginBase这个module就是这些啦，接下来看看插件模块应该怎么写。

6. 我们的插件名为`p_scanword`，这个名字必须和PluginBase类里面定义的插件名常量的名字是一样的，为什么呢，因为反射啊。而包名是`com.example.xunwang.scanword`,这个不用纠结了，如果后面有机会了解这个打包脚本再说吧。
还记得前面说的每个插件的入口类吗，没错就是它`LauncherPlugin`。
```java
public class LauncherPlugin extends PluginBase {
    public static int CONTAINER_ID;
    private Fragment mFragment;

    @Override
    public String getPluginName() {
        return PLUGIN_SCANWORD;
    }

    @Override
    public void init() {
        mFragment = new DefaultFragment();
    }

    @Override
    public void show(int containerId) {
        CONTAINER_ID = containerId;
        FragmentTransaction ft = getFragmentTransaction();
        ft.replace(containerId, mFragment);
        ft.commitAllowingStateLoss();
    }

    @Override
    public void hide() {
        if(mFragment.getView() != null){
            mFragment.getView().setVisibility(View.INVISIBLE);
        }
    }

    @Override
    public void detach() {
        getFragmentTransaction().detach(mFragment).commit();
    }
}
```
可能看到这个还不是特别清楚，那再看看我在点击第二个button的时候在onclick里面做了啥事（不记得第二个button是干什么的翻到最前吧）:
```java
PluginInfo pluginInfo = PluginManager.getInstance(getApplicationContext()).getPluginInfo(
					PluginBase.PLUGIN_SCANWORD);
			Log.d("kkkkkkkk", "pluginInfo --> " + pluginInfo);
			PluginBase pluginBase = null;
			if (pluginInfo != null) {
				pluginBase = pluginInfo.pluginBase;
			}
			if (pluginBase == null) {
				Toast.makeText(getApplicationContext(), "plugin not loaded", Toast.LENGTH_SHORT).show();
				return;
			}
			pluginBase.attach(MiniXbjMainActivity.this, getSupportFragmentManager());
			pluginBase.show(R.id.plugin_layout);
```
看到没有，第4段写了那么那么多，几乎整个`PluginManager`都是为了在构建`PluginInfo`这个东东，这个地方终于用了。
然后取出`PluginInfo`里面的`pluginBase`，是怎么生成的倒回去看吧，简而言之，就是通过`PluginBase.PLUGIN_SCANWORD`绕了好大一圈反射出了插件中的`LauncherPlugin`这个类，然后调了`LauncherPlugin`类的attach和show两个方法，attach之前我们说了其实是`PluginBase`的，相当于把宿主的`context`和`FragmentManager`传给插件，而show就是通过传进来的`FragmentManager`获得`FragmentTransaction`,然后使用replace实现显示插件页面的效果。
所以`LauncherPlugin`作为入口类一般是一个壳，生成的真正插件主页的fragment在它的init方法里获得，这里也就是`DefaultFragment`啦。
最后来看看`DefaultFragment`吧。
```java
public class DefaultFragment extends PlugFragment {
	@Override
	protected String getPluginName() {
		return PluginBase.PLUGIN_SCANWORD;
	}

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle
            savedInstanceState) {
        LayoutInflater factory = LayoutInflater.from(getContext());
        ViewGroup viewGroup = (ViewGroup) factory.inflate(R.layout.pla_linear, container, false);
        return viewGroup;
    }
}
```
`LayoutInflater`的时候是不是传入的context不一样啦，其实也看不出来，其实它是plugFragment包装的那个插件context，因为pla_linear在插件的module里，不用插件的context绝逼是找不到的。
**再次强调插件开发的规则**，宿主的资源使用宿主的context，插件的资源使用插件的context，公共组件的资源也使用宿主的context(因为在打包过程中，公共组件不打到插件中而是打到宿主中)，这个很重要，谁用谁知道。
到目前为止一个插件框架的基本雏形已经出来了，最后来看看bbcomm吧，我在项目里面添加了一个自定义view
```java
public class CommButtonView extends Button {
    private Drawable bg;
    private Paint paint;

	public CommButtonView(Context context) {
		super(context);
        init(context);
	}

	public CommButtonView(Context context, AttributeSet attrs) {
		super(context, attrs);
        init(context);
	}

	public CommButtonView(Context context, AttributeSet attrs, int defStyleAttr) {
		super(context, attrs, defStyleAttr);
        init(context);
	}

    private void init(Context context) {
        bg = context.getApplicationContext().getResources().getDrawable(R.drawable.audio_play_bg);
        paint = new Paint();
        paint.setColor(Color.RED);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.drawBitmap(((BitmapDrawable)bg).getBitmap(), 0, 0 , paint);
    }
}
```
`audio_play_bg`这张图片放到bbcomm的module里的，注意到context没有，一言不合就会crash哟。
宿主和插件由于都是依赖bbcomm的，所以都正常使用这个自定义控件就行了，最后上一张图吧。
![Alt text](./47B4DF1FCA73EBA75CB9A0A7AA7F2D43.jpg)


**总结:**以上写了一大堆，也只不过是插件化的最基本雏形而已，但同时也是基础，虽然基于fragment的插件化目前已不再主流，但再某些地方总是会有用武之地，后面还有很长的路要走，多多站在巨人和别人的肩膀上会走得更快。

PS：本文仅作为学习之用，不提供代码，禁止转载，敬请谅解。