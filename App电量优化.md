### 前言

> 现在手机软件装的越来越多，很多手机电量基本一天都坚持不了。所以在应用开发的时候对应用多做做优化，不仅提升应用性能，还可以省电。岂不是美滋滋。

### 电量分析工具

> **Battery Historian：** 通过Battery Historian可以方便的看到各耗电模块随着时间的耗电情况：包含操作类型、执行时间、对应App等；还可以进行筛选特定的App，给出一个总结性的说明，包括：**Network Information、 Syncs、WakeLock、Services、Process info、Scheduled Job、Sensor Use**等，查看每一个模块的总结，可以看出来每一项的耗时以及执行次数。当发现异常的时候可以针对性的进行排查。总之：Battery Historian真的很强大。

[Battery Historian安装教程](https://github.com/google/battery-historian)


### 电量优化

> Android系统上App的电量消耗主要由**cpu、wakelock、数据传输（流量和wifi）、wifi运行、gps、other senior**组成，而耗电异常也是由于这几个模块的使用不当。


- #### CPU时间片优化
当检测到CPU时间片消耗异常时，需要使用**TraceView**，获取进程执行信息，定位CPU占用率异常的问题

- #### 网络传输

使用移动网络传输数据，电量的消耗有三种状态：

1. **Full power:** 能量最高的状态，移动网络连接被激活，允许设备以最大的传输速率进行操作。
2. **Low power:** 一种中间状态，对电量的消耗差不多是Full power状态下的50%。
3. **Standby:** 最低的状态，没有数据连接需要传输，电量消耗最少。

- #### 数据压缩
通过数据压缩等方式缩减传输时间，降低电量消耗。可以参考[App网络优化](https://github.com/LiangLuDev/AppOptimize/blob/master/App%E7%BD%91%E7%BB%9C%E4%BC%98%E5%8C%96.md)


- #### 请求集中发送
分析和统计之类的非重要操作，可以在合适状态（电量充足或Wifi状态）下发送

- #### 无网状态避免网络请求
**之前在网络优化的文章里写过，网络请求失败之后的重试机制，但是要注意这个重试是在有网状态下的重试。否则无网状态下重试不会请求成功，只会消耗电量。尤其是与AlarmManager或者WakeLock连用的场景下，耗电量会更多。**


- #### GPS-根据需求选择合适的位置提供者
LocationManager的位置提供者有以下四种：

> **1.GPS定位（GPS_PROVIDER）:** 利用GPS芯片通过卫星获得自己的位置信息。定位精准度高，一般在10米左右，耗电量大；但是在室内，GPS定位基本没用。
>
> **2.网络定位（NETWORK_PROVIDER）:** 利用手机基站和WIFI节点的地址来大致定位位置，这种定位方式取决于服务器，即取决于将基站或WIF节点信息翻译成位置信息的服务器的能力。
>
> **3.被动定位（PASSIVE_PROVIDER）:** 就是用现成的，当其他应用使用定位更新了定位信息，系统会保存下来，该应用接收到消息后直接读取就可以了。比如如果系统中已经安装了百度地图，高德地图(室内可以实现精确定位)，你只要使用它们定位过后，再使用这种方法在你的程序肯定是可以拿到比较精确的定位信息。


示例代码：
```
String provider = LocationManager.GPS_PROVIDER;
Location location = locationManager.getLastKnownLocation(provider);
double latitude = location.getLatitude();//维度
double longitude = location.getLongitude();//经度
```
有很多官方API可以翻翻官方文档（不过国内基本使用高德百度提供的定位API）


- #### 注销定位监听
在不需要定位，或者应用进入后台了，注销定位监听。

示例代码：
```
public void onPause() {
        super.onPause();
        locationManager.removeListener(locationListener);
    }
    
public void onResume(){
        super.onResume();
        locationManager.requestLocationUpdates(locationManager.getBestProvider(criteria, true),6000,100,locationListener);
    }
```

- #### 多模块使用定位尽量复用
多个模块使用定位，尽量复用上一次的结果，而不是都重新走定位的过程，节省电量损耗；例如：在应用启动的时候获取一次定位，保存结果，之后再用到定位的地方都直接去取。


- #### 谨慎使用WakeLock
Android为了节省电量，会在用户无操作一段时间之后进入休眠状态。**Wake Lock**是一种锁的机制，只要有人拿着这个锁，系统就**无法进入休眠**。一些App为了能在后台持续做事情，就会持有一个WakeLock，那么手机就不会进入休眠状态，App要做的事情能做了，但是也更加耗电。

> **1.App在前台不要申请WakeLock，此时无需申请，申请的话会计算到应用电量消耗；**
>
>**2.App在后台由于业务需要必须要申请WakeLock时使用带有超时参数的方法，防止由于忘记或者异常情况下没有释放；**
>
>**3.App申请使用WakeLock，任务结束之后及时释放，让系统再次进入休眠状态。**
>
>**4.如果只是需要屏幕常亮的话，可以使用FLAG_KEEP_SCREEN_ON，无需考虑释放WakeLock的问题。**

- #### 传感器使用

使用传感器，选择合适的采样率（更新速度），越高的采样率类型则越费电；

SENSOR_DELAY_NOMAL (200000微秒)

SENSOR_DELAY_UI (60000微秒)

SENSOR_DELAY_GAME (20000微秒)

SENSOR_DELAY_FASTEST (0微秒)

**注意：** 如果应用进入后台不需要更新传感器数据，及时注销传感器监听


- #### JobScheduler-后台数据处理模式

使用JobScheduler，一些任务通过JobScheduler来触发，例如可推迟的网络请求、下载、GPS等，可以在特定场景：连接Wifi、连接电源等场景触发。既完成了任务，也无需考虑由于一些任务导致的电量消耗。


### 后记
其实Android系统是不怎么耗电。只是国内的软件质量不高。不注意优化导致电量哗哗掉。






