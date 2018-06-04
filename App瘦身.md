## App瘦身

### 前言
> 虽然现在无线网广泛，流量不要钱，但是：
> - 类型功能一样的产品，用户肯定会选择APK小的下载。
> - 对于我们来说，也是技术优化提升的体验。

### 安装包的组成
这里我们使用Android Studio自带工具(Android Studio 2.2.3之后)查看：```Android Studio -> Build -> Analyze APK```选择要Apk文件：

![Analyze APK](https://diycode.b0.upaiyun.com/photo/2017/76c364d5ad294e871cdffb25c68e549e.png)

这里就很清晰的查看Apk的组成，这边整理一下组成部分作用：

![image](https://diycode.b0.upaiyun.com/photo/2017/95233032fec3913389fbfee1a5728549.png)

这样Apk组成就一目了然了，下面那我们开始针对性优化瘦身。


### 代码瘦身 
- **移除无用的代码：** 移除无用代码以及无用功能，有助于减少代码量，直接体现就是Dex的体积会变小。不用的代码在项目中存在属于一个普遍现象，相当于僵尸代码，而且这类代码过多也会导致Dex文件过大。
- **及时移除无用的库、避免功能雷同的库**
- **项目中基础功能的库要统一实现，避免出现多套网络请求、图片加载器等实现。**
- **一些功能可以曲线救国的话就不要引入SDK，例如定位功能，可以不引入定位SDK，而通过拿到经纬度然后调用相关接口来实现；同样实现了功能而没有引入SDK**

### Proguard工具
> Proguard经常被看做Android平台的代码混淆工具，其实混淆只是其中一个功能。
> Proguard是一个免费的Java类文件压缩、优化、混淆、预先验证的工具，可以检测和移除未使用的类、字段、方法、属性，优化字节码并移除未使用的指令，并将代码中的类、字段、方法的名字改为简短、无意义的名字。
>
> 推荐一个微信大佬开源的混淆的工具 [AndResGuard](https://github.com/shwenzhang/AndResGuard)

在```build.gradle```里面配置```Proguard```：

```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile(‘proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

其中，```proguard-android.txt```是获取默认```ProGuard```设置，```proguard-rules.pro```文件用于添加自定义```ProGuard```规则。


**备注：** 对于Proguard，虽然效果很明显，但仍然需要谨慎；


### 缩减方法数
> Android的64K方法数问题都遇到过，虽然Google官方给出了解决方案，但是我们这里针对瘦身处理。

- 避免在内部类中访问外部类的私有方法、变量。挡在Java内部类（包含匿名内部类）中访问外部类的私有方法、变量的时候，编译器会生成额外的方法，会增加方法数；
- 避免调用派生类中的未被覆写的方法，避免在派生类中调用未覆写的基类的方法；避免用派生类的对象调用派生类中未覆盖的基类的方法。调用派生类中的未被覆盖的方法时，会多产生一个方法数；
- 去掉部分类的get、set方法；当然这样会牺牲一些面向对象的观念。

### 资源瘦身
- 移除无用的资源文件

 直接使用Android Studio功能即可：
 
 ![移除无用的资源文件](https://upload-images.jianshu.io/upload_images/4056837-5f12e2ba7cff7210.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
 
 不打包没有使用的资源，在项目的 ```build.gradle``` 中配置 ```shrinkResources true``` 即可:
 ![shrinkResources](https://diycode.b0.upaiyun.com/photo/2017/21535a04995ce10c95ada81855dd6b6b.png)
 
 - Drawable目录只保留一个(**推荐drawable-xxhdpi**)
 - 图片压缩：要求很高的就自己写算法脚本压缩，否则可以使用压缩工具[TinyPng](https://tinypng.com/)这个工具使用者挺多的，压缩的图片基本无差别。
 - SVG：可缩放矢量图形（英语：Scalable Vector Graphics，SVG），我自己的开源项目图标基本使用的都是SVG，不损伤图片质量，一套图适配所有，比使用位图小十几倍，有利有弊，系统渲染SVG会多消耗一些时间，色调单一。
 - WebP：在同画质下体积更小，WebP支持透明度（Android从4.0才开始WebP的原生支持）
 - 网络加载资源：将部分使用频率不高的资源例如图片，放在网上，在恰当的时机提前下载，这样也能节约部分空间

### so文件瘦身
> so（shared object，共享库）是机器可以直接运行的二进制代码，是Android上的动态链接库，类似于Windows上的dll。每一个Android应用所支持的ABI是由其APK提供的.so文件决定的，这些so文件被打包在apk文件的lib/目录下。

armeabi目录下的So可以兼容别的平台的So，但是性能会有所损耗，失去对特定平台的优化。因此需要根据自己使用到的So功能来做具体的区分：对于性能敏感模块使用的So可以都放在armeabi目录，然后通过代码判断设备的CPU类型，再加载其对应架构的SO文件，例如微信就是这么做的。既缩减了Apk的体积，也不影响性能敏感模块的执行。



移除特定平台so的方式，这样打包就只保存armeabi里的so:
```
ndk {
    //设置支持的SO库架构
    abiFilters  'armeabi'
}
```

### 总结

操作so以及资源文件是最明显的瘦身方式。其他的代码优化等等，效果不是很明显，但是养成一个好的代码习惯是有必要的。