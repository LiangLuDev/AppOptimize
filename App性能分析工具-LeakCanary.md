### 前言
> LeakCanary是Android 和 Java 内存泄露检测的工具。
>
> A small leak will sink a great ship. -- Benjamin Franklin
>
> 千里之堤， 毁于蚁穴。 -- 《韩非子·喻老》
### 为什么使用LeakCanary
> **内存泄漏：** 一些对象有着有限的生命周期。当这些对象所要做的事情完成了，我们希望他们会被回收掉。但是如果有一系列对这个对象的引用，那么在我们期待这个对象生命周期结束的时候被收回的时候，它是不会被回收的。它还会占用内存，这就造成了内存泄露。持续累加，内存很快被耗尽。
>
> **比如**，当 ```Activity.onDestroy``` 被调用之后，activity 以及它涉及到的 view 和相关的 bitmap 都应该被回收。但是，如果有一个后台线程持有这个 activity 的引用，那么 activity 对应的内存就不能被回收。这最终将会导致内存耗尽，然后因为 OOM 而 crash。


**如果你也想消灭 OOM crash，那还犹豫什么，赶快使用 [LeakCanary](https://github.com/square/leakcanary)**

### LeakCanary核心类分析
> - **HeapAnalyzerService :** 内存堆分析服务， 为了保证App进程不会因此受影响变慢&内存溢出，运行于独立的进程
> - **HeapAnalyzer：** 分析由RefWatcher生成的堆转储信息， 验证内存泄漏是否真实存在
> - **HeapDump：** 堆转储信息类，存储堆转储的相关信息
> - **ServiceHeapDumpListener：** 一个监听，包含了开启分析的方法
> - **RefWatcher：** 核心类，检测不可达引用（可能地），当发现不可达引用时，它会触发HeapDumper(堆信息转储)
> - **ActivityRefWatcher：** Activity引用检测，包含了Activity生命周期的监听执行与停止


通过以上列表，让大家对LeakCanary框架的主要类有个大体的了解，并基于以上列表，对这个框架的大体功能有一个模糊的猜测。

### 开始使用
在```build.gradle```中加入引用，仅适用于测试环境
```
dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5.4'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.4'
 }
```
在```Applicetion```中
```
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```
这样，就万事俱备了！ 如果检测到某个 activity 有内存泄露，LeakCanary 就是自动地显示一个通知。



