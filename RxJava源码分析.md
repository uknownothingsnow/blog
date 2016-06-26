## 基本结构
我们先来看一段最基本的代码，分析这段代码在RxJava中是如何实现的。
```Java
Observable.OnSubscribe<String> onSubscriber1 = new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("1");
        subscriber.onCompleted();
    }
};
Subscriber<String> subscriber1 = new Subscriber<String>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {

    }
};

Observable.create(onSubscriber1)
        .subscribe(subscriber1);
```

首先我们来看一下Observable.create的代码
```Java
public final static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(hook.onCreate(f));
}

protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```
直接就是调用了Observable的构造函数来创建一个新的Observable对象，这个对象我们暂时标记为observable1，以便后面追溯。
同时，会将我们传入的OnSubscribe对象onSubscribe1保存在observable1的onSubscribe属性中，这个属性在后面的上下文中很重要，大家留心一下。

接下来我们来看看subscribe方法。
```Java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}

private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    ...
    subscriber.onStart();
    hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
    return hook.onSubscribeReturn(subscriber);
}
```
可以看到，subscribe之后，就直接调用了observable1.onSubscribe.call方法，也就是我们代码中的onSubscribe1对象的call方法
，传入的参数就是我们代码中定义的subscriber1对象。call方法中所做的事情就是调用传入的subscriber1对象的onNext和onComplete方法。
这样就实现了观察者和被观察者之间的通讯，是不是很简单？
```Java
public void call(Subscriber<? super String> subscriber) {
    subscriber.onNext("1");
    subscriber.onCompleted();
}
```

## lift

lift方法是RxJava中实现自定义operator的关键，这里我们以最简单的map为例，来分析一下lift方法的工作原理，我们对上面的demo代码稍作修改
```Java
Observable.OnSubscribe<String> onSubscriber1 = new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("1");
        subscriber.onCompleted();
    }
};
Subscriber<Integer> subscriber1 = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Integer i) {

    }
};
Func1<String, Integer> transformer1 = new Func1<String, Integer>() {
    @Override
    public Integer call(String s) {
        return Integer.parseInt(s);
    }
};

Observable.create(onSubscriber1)
        .map(transformer1)
        .subscribe(subscriber1);
```

和刚才不同的是我们在create之后调用了map方法，然后才调用subscribe方法。
map方法的代码如下:
```
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return lift(new OperatorMap<T, R>(func));
}
```
一堆泛型参数是不是略晕啊，别急，我们慢慢来看。

首先来介绍一下Func这个接口。RxJava中有一系列Action+数字，Func+数字的接口，这些接口中都只有一个call方法，其中Action接口的call方法都没有返回值，
Func接口的call方法都有返回值，后面的那个数字表示call方法接受几个泛型类型的参数。

其实主要是因为Java中函数不是一等公民，所以只能用接口这么啰嗦的格式，还好我们可以使用lambda简化我们的代码。（羡慕函数式语言）

这里map方法接收的参数类型为`Func1<? super T, ? extends R> func`，表示func的call方法接收一个T类型的参数，返回一个R类型的返回值。

OperatorMap又是什么鬼呢？
```Java
public final class OperatorMap<T, R> implements Operator<R, T>

public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>>
```
这里可以看到OperatorMap继承自Operator, 而Operator又继承自Func1接口，也就是说Operator接口的call方法会接收一个Subscriber类型的参数，
并且返回另外一个Subscriber类型的对象。Operator.call方法返回一个Subscriber对象，其实我们可以这么理解，每一个operator也是一个订阅者，
它返回的Subscriber对象正好用来订阅Observable发出来的消息。

有一点需要注意的是OperatorMap和Operator的泛型参数顺序刚好是相反的，为什么要这么做呢？其实很简单，因为Operator本身是对Observable发出的数据
进行转换的，所以经常会出现operator转换之后返回的数据类型变了，而OperatorMap这里刚好颠倒了一下顺序，就可以保证call方法返回的Subscriber<T>类型
可以订阅Observable发出的数据。

OperatorMap的代码我们先不看，先来看一下lift方法中都做了些啥吧。
```Java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber<? super R> o) {
            Subscriber<? super T> st = hook.onLift(operator).call(o);
            st.onStart();
            onSubscribe.call(st);
        }
    });
}
```
lift方法会返回一个新创建的Observable对象，这里我们给这个Observable一个标识observable2。observable2的onSubscribe属性就是lift中new出来的这个
OnSubscribe对象。

对照demo中的代码，我们调用map之后，就调用了subscribe方法，也就是调用了这里的observable2的subscribe方法。
根据上面的介绍，调用subscribe之后，就会调用observable2.onSubscribe.call方法，call中首先做的事情就是调用OperatorMap的call方法
```Java
@Override
public Subscriber<? super T> call(final Subscriber<? super R> o) {
    return new Subscriber<T>(o) {
        @Override
        public void onNext(T t) {
            o.onNext(transformer.call(t));
        }
    };
}
```
OperatorMap的call方法返回了一个Subscriber对象，这里我们标记为subscriber$map，改对象有一个transformer属性，就是我们在demo中定义的transformer1对象，
他是一个Func1类型，用来对每一个数据项进行变换。这里call方法中接收的Subscriber其实就是demo中的subscriber1对象。

我们回到lift中创建的observable2的call方法中继续，拿到OperatorMap返回的Subscriber对象之后，接着调用了onSubscribe.call方法，并且将返回的Subscriber
对象传递进去。这里需要注意的一点就是onSubscribe对象就是我们在demo中定义的onSubscribe1变量，所以就是调用了onSubscribe1.call(subscriber$map)方法。

现在我们就可以从onSubscribe1.call的调用来分析一下数据的转换过程，首先调用了subscriber$map.onNext("1"), subscriber$map.onNext中会首先调用
transformer1.call("1"),然后使用返回值1，来调用onSubscribe1.onNext(1)方法，最终demo中的onSubscribe1就收到了1这个值。
