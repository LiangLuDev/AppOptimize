## App启动优化


### App启动方式

- **冷启动**
> App没有启动过或App进程被killed, 系统中不存在该App进程,
App启动需要创建App进程, 加载相关资源, 启动Main Thread, 初始化首屏Activity等此时启动App即为冷启动.
>
>在这个过程中, 屏幕会显示一个**空白的窗口(白屏)**,直至首屏Activity完全启动.
- **热启动**
> 热启动意味着你的App进程只是处于后台,系统只是将其从后台带到前台, 展示给用户.
>
>在这个过程中, 屏幕会显示一个**空白的窗口(白屏)**,直至activity渲染完毕.
- **温启动**
> 介于冷启动和热启动之间, 一般来说在以下两种情况下发生:
>
> **1.** 用户back退出了App, 然后又启动. App进程可能还在运行, 但是activity需要重建.
>
> **2.** 用户退出App后,系统可能由于内存原因将App杀死,进程和activity都需要重启, 但是可以在onCreate中将被动杀死锁保存的状态(saved instance state)恢复.

我们要做的启动优化其实就是针对冷启动. 热启动和温启动都相对较快.

### Application启动优化

> 一般Application的onCreate会用于第三方服务、依赖库初始化等操作,针对这些耗时操作我们放到IntentService来做初始化工作。
>
>**IntentService：** 不同于Service,可以看做是Service和HandlerThread的结合体，在完成了使命之后会自动停止，适合需要在工作线程处理UI无关任务的场景。

#### 具体代码

- IntentService代码如下：

```
public class InitializeService extends IntentService{

 private static final String ACTION = "InitializeService";

     public InitializeService() {
        super("InitializeService");
    }

     /**
     * 启动服务
     *
     * @param context
     */
    public static void start(Context context) {
        Intent intent = new Intent(context, InitializeService.class);
        intent.setAction(ACTION);
        context.startService(intent);
    }

      @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        if (intent != null) {
            final String action = intent.getAction();
            if (ACTION.equals(action)) {
                initialize();
            }
        }
    }

     /**
     * 初始化工作
     */
    private void initialize() {
        initLeakCanary();
        initEaseUi();
        initRefresh();
        initBugly();
        initJpush();
    }

}

```

- Applicetion代码如下：

```
public class DevApplication extends MultiDexApplication {

     @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }

     @Override
    public void onCreate() {
        super.onCreate();
        InitializeService.start(this);
    }
}

```
- AndroidManifest.xml中声明service：

```
<service android:name=".service.InitializeService"/>

```

### 首屏Activity的渲染优化

> Android最新的Material Design有这么个建议的. 建议我们使用一个placeholder UI来展示给用户直至App加载完毕。
当App没有完全起来时,屏幕会一直显示一块空白的窗口(一般来说是黑屏或者白屏, 根据App主题)。这里我们加个**没有布局，只有主题**的Activity作为启动页。



- **主题设置**

> 图片+背景设置

```
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 底层白色 -->
    <item android:drawable="@color/white" />

    <!-- 顶层Logo居中 -->
    <item>
        <bitmap
            android:gravity="center"
            android:src="@mipmap/Splash" />
    </item>
</layer-list>
```
然后根据**图片直接设置**设置主题


> 图片直接设置

```
<style name="SplashTheme" parent="AppTheme">
    <item name="android:windowBackground">@mipmap/Splash</item>
</style>

```

- **创建没有布局的Activity**


```
public class SplashActivity extends AppCompatActivity {
      @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        startActivity(new Intent(SplashActivity.this, MainActivity.class));
        finish();
    }
}
```
- **AndroidManifest.xml中设置其为启动屏, 并加上主题**

```
<activity
    android:name=".mvp.view.activity.SplashActivity"
    android:theme="@style/SplashTheme">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
</activity>

```

### 总结

> 其实优化可以从以下几个角度考虑：
> - Application的onCreate中不要做太多事情
> - 首屏Activity尽量简化
> - 善用工具分析.
> - 多阅读官方文档, 很多地方貌似无关, 实际有关联, 例如这次就用了Material Design文档中的解决方案.



