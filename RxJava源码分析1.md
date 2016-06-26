
Observable.just
Observable.create
最终都会调用到
```Java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```
Observable.subscribe方法最终都会到
```Java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}
```
```Java
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    ...
    // new Subscriber so onStart it
    subscriber.onStart();

    if (!(subscriber instanceof SafeSubscriber)) {
        subscriber = new SafeSubscriber<T>(subscriber);
    }

    // allow the hook to intercept and/or decorate
    hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
    return hook.onSubscribeReturn(subscriber);
}
```
调用了onSubscribe.call(subscriber)
```Java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("hello");
        subscriber.onCompleted();
    }
});
```
call方法调用之后，就会开始取数据，并通过subscriber.onNext()来通知subscriber
subscribe对象一般都会是一个匿名对象，比如下面的实现
```Java
.subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {
    }

    @Override
    public void onError(Throwable e) {
    }

    @Override
    public void onNext(String s) {
    }
})
```
