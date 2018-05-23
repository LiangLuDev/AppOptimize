## ANR系列
---
### ANR是什么？
ANR全名Application Not Responding, 也就是"应用无响应". 当操作在一段时间内系统无法处理时, 系统层面会弹出**系统无响应，是否重新启动**类似的ANR对话框.

### 产生ANR的原因
在Android里, App的响应能力是由Activity Manager和Window Manager系统服务来监控的.造成ANR的首要原因是**在主线程(UI线程)里面做了太多的阻塞耗时操**
例如：文件读写、数据库读写、网络查询

### 解决ANR
> **不要在主线程(UI线程)里面做繁重的操作.**

### ANR分析
#### 获取ANR产生的trace文件
ANR产生时, 系统会生成一个traces.txt的文件放在/data/anr/下. 可以通过adb命令将其导出到本地:
```
$adb pull data/anr/traces.txt
```
#### 分析traces文件
> 这里针对ANR分析常见的三种情况

##### 普通阻塞导致的ANR
强行sleep thread产生的一个ANR.
```
----- pid 2976 at 2018-05-08 23:02:47 -----
Cmd line: com.luliangdev.dev  // 最新的ANR发生的进程(包名)

...

DALVIK THREADS (41):
"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x73467fa8 self=0x7fbf66c95000
  | sysTid=2976 nice=0 cgrp=default sched=0/0 handle=0x7fbf6a8953e0
  | state=S schedstat=( 0 0 0 ) utm=60 stm=37 core=1 HZ=100
  | stack=0x7ffff4ffd000-0x7ffff4fff000 stackSize=8MB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x35fc9e33> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x35fc9e33> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985) // 主线程中sleep过长时间, 阻塞导致无响应.
  at com.tencent.bugly.crashreport.crash.c.l(BUGLY:258)
  - locked <@addr=0x12dadc70> (a com.tencent.bugly.crashreport.crash.c)
  at com.tencent.bugly.crashreport.CrashReport.testANRCrash(BUGLY:166)  // 产生ANR的那个函数调用
  - locked <@addr=0x12d1e840> (a java.lang.Class<com.tencent.bugly.crashreport.CrashReport>)
  at com.luliangdev.dev.common.wrapper.CrashHelper.testAnr(CrashHelper.java:23)
  at com.luliangdev.dev.ui.module.main.MineFragment.onClick(MineFragment.java:80) // ANR的起点
  at com.luliangdev.dev.ui.module.main.MineFragment_ViewBinding$2.doClick(MineFragment_ViewBinding.java:47)
  at butterknife.internal.DebouncingOnClickListener.onClick(DebouncingOnClickListener.java:22)
  at android.view.View.performClick(View.java:4780)
  at android.view.View$PerformClick.run(View.java:19866)
  at android.os.Handler.handleCallback(Handler.java:739)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:135)
  at android.app.ActivityThread.main(ActivityThread.java:5254)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:903)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:698)
```
如上trace信息中的**添加的中文注释**已基本说明了trace文件该怎么分析。

#####  CPU满负荷
这个时候你看到的trace信息可能会包含这样的信息:
```
Process:com.luliangdev.dev
...
CPU usage from 3330ms to 814ms ago:
6% 178/system_server: 3.5% user + 1.4% kernel / faults: 86 minor 20 major
4.6% 2976/com.anly.githubapp: 0.7% user + 3.7% kernel /faults: 52 minor 19 major
0.9% 252/com.android.systemui: 0.9% user + 0% kernel
...

100%TOTAL: 5.9% user + 4.1% kernel + 89% iowait
```
最后一句表明了**CPU占用100%**, 满负荷了。其中绝大数是被I/O操作占用了。一般来说会发现是方法中有**频繁的文件读写**或是**数据库读写操作放在主线程**来做。


##### 内存不足
其实内存原因有可能会导致ANR, 例如如果由于内存泄露, App可使用内存所剩无几, 我们点击按钮启动一个大图片作为背景的activity, 就可能会产生ANR, 这时trace信息可能是这样的:
```
// 以下trace信息来自网络, 用来做个示例
Cmdline: android.process.acore

DALVIK THREADS:
"main"prio=5 tid=3 VMWAIT
|group="main" sCount=1 dsCount=0 s=N obj=0x40026240self=0xbda8
| sysTid=1815 nice=0 sched=0/0 cgrp=unknownhandle=-1344001376
atdalvik.system.VMRuntime.trackExternalAllocation(NativeMethod)
atandroid.graphics.Bitmap.nativeCreate(Native Method)
atandroid.graphics.Bitmap.createBitmap(Bitmap.java:468)
atandroid.view.View.buildDrawingCache(View.java:6324)
atandroid.view.View.getDrawingCache(View.java:6178)

...

MEMINFO in pid 1360 [android.process.acore] **
native dalvik other total
size: 17036 23111 N/A 40147
allocated: 16484 20675 N/A 37159
free: 296 2436 N/A 2732
```
可以看到free的内存已所剩无几.这种情况可能更多的是会产生**OOM**的异常...


### ANR处理方式

> - **主线程阻塞：** 开辟单独的子线程来处理耗时阻塞事务.
> - **CPU满负荷, I/O阻塞：** I/O阻塞一般来说就是文件读写或数据库操作执行在主线程了, 也可以通过开辟子线程的方式异步执行.
> - **内存不足：** 增大VM内存, 使用largeHeap属性, 排查内存泄露等.

#### 常用执行在主线程的操作
- Activity的所有生命周期回调都是执行在主线程的.
- Service默认是执行在主线程的
- BroadcastReceiver的onReceive回调是执行在主线程的.
- 没有使用子线程的looper的Handler的handleMessage, post(Runnable)是执行在主线程的.
- View的post(Runnable)是执行在主线程的.

#### 子线程使用方式

- **RxJava（强烈推荐）**[使用文档](https://github.com/LiangLuDev/RxPractice)
- 继承Theard
```
class PrimeThread extends Thread {
    long minPrime;
    PrimeThread(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
         . . .
    }
}

PrimeThread p = new PrimeThread(143);
p.start();
```
- 实现Runnable接口
```
class PrimeRun implements Runnable {
    long minPrime;
    PrimeRun(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
         . . .
    }
}

PrimeRun p = new PrimeRun(143);
new Thread(p).start();
```


- HandlerThread

Android中结合Handler和Thread的一种方式. 前面有云, 默认情况下Handler的handleMessage是执行在主线程的, 但是如果我给这个Handler传入了子线程的looper, handleMessage就会执行在这个子线程中的. HandlerThread正是这样的一个结合体:
```
// 启动一个名为new_thread的子线程
HandlerThread thread = new HandlerThread("new_thread");
thread.start();

// 取new_thread赋值给ServiceHandler
private ServiceHandler mServiceHandler;
mServiceLooper = thread.getLooper();
mServiceHandler = new ServiceHandler(mServiceLooper);

private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
      super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
      // 此时handleMessage是运行在new_thread这个子线程中了.
    }
}

```



- IntentService

前面有关于IntentService的使用方式。
Service是运行在主线程的, 然而IntentService是运行在子线程的.
实际上IntentService就是实现了一个HandlerThread + ServiceHandler的模式.


- **特别注意**

使用Thread和HandlerThread时, 为了使效果更好, 建议设置Thread的优先级偏低一点:
```
Process.setThreadPriority(THREAD_PRIORITY_BACKGROUND);
```
因为如果没有做任何优先级设置的话, 你创建的Thread默认和UI Thread是具有同样的优先级的, 你懂的. 同样的优先级的Thread, CPU调度上还是可能会阻塞掉你的UI Thread, 导致ANR的.


### ANR结语
对于ANR问题, 个人认为还是预防为主, 认清代码中的阻塞点, 善用线程.（学学RxJava还是很必要的）

### 卡顿
---
### 卡顿来源
用户对卡顿的感知, 主要来源于界面的刷新. 而界面的性能主要是依赖于设备的UI渲染性能. 如果我们的UI设计过于复杂, 或是实现不够好, 设备又不给力, 界面就会像卡住了一样, 给用户卡顿的感觉.

### 16ms原则
> Android系统每隔16ms会发出VSYNC信号重绘我们的界面(Activity).
为什么是16ms, 因为Android设定的刷新率是60FPS(Frame Per Second), 也就是每秒60帧的刷新率, 约合16ms刷新一次.

![16ms-1](https://upload-images.jianshu.io/upload_images/851999-feaaebea717ba97b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/529)

这就意味着, 我们需要在16ms内完成下一次要刷新的界面的相关运算, 以便界面刷新更新. 然而, 如果我们无法在16ms内完成此次运算会怎样呢?

例如, 假设我们更新屏幕的背景图片, 需要24ms来做这次运算. 当系统在第一个16ms时刷新界面, 然而我们的运算还没有结束, 无法绘出图片. 当系统隔16ms再发一次VSYNC信息重绘界面时, 用户才会看到更新后的图片. 也就是说用户是32ms后看到了这次刷新(注意, 并不是24ms). 这就是传说中的丢帧(dropped frame):

![16ms-2](https://upload-images.jianshu.io/upload_images/851999-1904e950165ab33d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/529)

丢帧给用户的感觉就是卡顿, 而且如果运算过于复杂, 丢帧会更多, 导致界面常常处于停滞状态, 卡到爆.