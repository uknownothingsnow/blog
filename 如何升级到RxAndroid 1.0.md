[原文链接](http://blog.danlew.net/2015/09/01/how-to-upgrade-to-rxandroid-10/)

最近很多人问我：[RxAndroid](https://github.com/ReactiveX/RxAndroid)在搞什么鬼？

事实上市，RxAndroid之前的版本确实是有点换乱，因此最近进行了一次大得重构。[这里](https://github.com/ReactiveX/RxAndroid/issues/172)有详细的说明，概括来说就是：

> 从头开始对RxAndroid进行模化的改造，让这个库变成一个可服用的，可组合的模块。

这个目标已经达成，但是如果你升级到1.0，你可能会很奇怪：东西都跑到哪里去了，如何才能让我的代码通过编译？

## RxAndroid

**AndroidSchedulers** 是RxAndroid中唯一保留下来的，但是一些方法签名已经变了。

## 迁移部分

**WidgetObservable** 和 **ViewObservable** 被打包进了[RxBinding](https://github.com/JakeWharton/RxBinding)项目中，并且做了一些改进。

**LifecycleObservable** 迁移到了[RxLifecycle](https://github.com/trello/RxLifecycle)项目中。另外需要注意的是，这里进行了一些相对比较大幅度的重构，所以使用的时候请参考一下修改日志。

**ContentObservable.fromSharedPreferencesChanges()** 迁移到了[rx-preferences](https://github.com/f2prateek/rx-preferences)项目。

## 删除部分

**AppObservable** 连同它的bind方法已经被完全删除掉了。AppObservable本身有很多问题：

- AppObservable尝试来做自动unsubscribe，但是仅仅是在Activity或者Fragment已经paused之后Observable再发出一个事件，才会触发自动unsubscribe。也就是说，如果Activity或者Fragment如果没有paused，一个不会complete的Observable将永远不会被unsubscribe。

- AppObservable被设计用来在pause之后避免继续受到消息，但是因为HandlerScheduler的[一个bug](https://github.com/ReactiveX/RxAndroid/issues/214)，导致某些场景存在缺陷。

- AppObservable自动调用了observeOn(AndroidSchedulers.mainThread())，不管你是不是想在主线程这么做。

换句话来说，AppObservable并没有做到它所描述的功能，它的可定制性也比较差，并且还会有一些非期望的副作用。

删除AppObservable的时候，可以这样做：

手动的处理Subscription（或者使用RxLifecycle），来在适当的时机做unsubscribe。检查一下你是否需要使用observeOn(AndroidSchedulers.mainThread())。
