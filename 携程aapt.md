前几天，携程无线部门开源了他们的[插件框架](https://github.com/CtripMobile/DynamicAPK)，使用该框架可以方便的实现app的插件化开发和热更新。
在陈博士发表的关于该框架的[blog](http://www.infoq.com/cn/articles/ctrip-android-dynamic-loading)中，有这么一段

>为aapt增加--apk-module参数。
如前所述，资源ID其实有一个PackageID的内部字段。我们为每个插件工程指定独特的PackageID字段，这样根据资源ID就很容易判明，此资源需要从哪个插件apk中去查找并加载了。在后文的资源加载部分会有进一步阐述。

很多同学都很关心这里应该怎么修改aapt来实现为不同的插件工程指定不同的PackageID，这里我来分析一下aapt的源码，提供一个大概的思路吧。

个人才疏学浅，如有不对，还请携程的同学指正一下。

appt相关的源码都在framework/base/tools/aapt目录下。

首先查看`ResourceTable.cpp`中的构造函数

```java
ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)
ssize_t packageId = -1;
switch (mPackageType) {
    case App:
    case AppFeature:
        packageId = 0x7f;
        break;

    case System:
        packageId = 0x01;
        break;

    case SharedLibrary:
        packageId = 0x00;
        break;

    default:
        assert(0);
        break;
}
```
资源的packageId就是在这里根据packageType来确定的，其中0x01是给系统资源使用的，在R.java中可以看到系统资源的id都是以0x01开头的，0x7f是给我们的应用程序资源使用的，
同样在R.java中，你可以看到，自己的app的资源id都是以0x7f开头的。这也就是说0x01到0x7f之间的的值我们都可以作为packageId来用。

接下来我们看看是在哪里创建了ResourceTable对象。

打开`Resource.cpp`，buildResources方法，就是在这个方法中构造了resourceTable对象。
```
status_t buildResources(Bundle* bundle, const sp<AaptAssets>& assets, sp<ApkBuilder>& builder)

ResourceTable::PackageType packageType = ResourceTable::App;
if (bundle->getBuildSharedLibrary()) {
    packageType = ResourceTable::SharedLibrary;
} else if (bundle->getExtending()) {
    packageType = ResourceTable::System;
} else if (!bundle->getFeatureOfPackage().isEmpty()) {
    packageType = ResourceTable::AppFeature;
}

ResourceTable table(bundle, String16(assets->getPackage()), packageType);

```
可以看到这里是根据bundle的几个方法的返回值，来确定生成的资源的packageId的。

那么，我们就可以在这里做一些手脚，来生成我们自己想要的packageId。我的做法就是给bundle对象加上一个getPackageId方法，
该方法会返回我们在命令行传入的packageId。代码类似下面
```
else if (!bundle->getFeatureOfPackage().isEmpty()) {
    packageType = ResourceTable::AppFeature;
} else if (bundle->getPackageId() != 0) {
  packageType = bundle.getPackageId();
}
```
bundle类上还会定义一个setPackageId方法，用来保存packageId信息。
bundle对象是在Main.cpp的main方法，也就是appt程序的入口中构造出来的，下面列出一个代码片段
```
switch (*cp) {
case 'v':
    bundle.setVerbose(true);
    break;
case 'a':
    bundle.setAndroidList(true);
    break;
```
bundle根据命令行传入的各种参数，来设置自己的一些状态，这里我们要加入自己的逻辑，来出来--apk-module参数，同时调用setPackageId方法就好了。

另外Dr.Chen的blog中还提到了
> 为aapt增加--public-R-path参数。

这里无非就是在Main.cpp中检测到这个参数的时候，也保存到bundle中，然后在`Resource.cpp`的writeResourceSymbols方法中，
在生成 R.java的时候，把传入的参数所指定的R.java中的变量插入到这个要生成的 R.java文件中就可以了。

以上就是我个人的一些猜测，如有不对，请大家帮忙指正。
