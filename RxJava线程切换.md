
`return hook.onSubscribeReturn(subscriber);`返回的就是subscriber对象，因为Subscriber实现了Subscription接口。
我们上面new出来的Subscriber对象，调用了Subscriber(null, false)构造函数
```Java
protected Subscriber(Subscriber<?> subscriber, boolean shareSubscriptions) {
    this.subscriber = subscriber;
    this.subscriptions = shareSubscriptions && subscriber != null ? subscriber.subscriptions : new SubscriptionList();
}
```
subscriptions是SubscriptionList类的实例,SubscriptionList的文档描述
`Subscription that represents a group of Subscriptions that are unsubscribed together.`


## subscribeOn
```Java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    return nest().lift(new OperatorSubscribeOn<T>(scheduler));
}
```

## nest

注意返回值，是一个释放出Observable类型的Observable
```Java
public final Observable<Observable<T>> nest() {
    return just(this);
}
```

## just

```Java
public final static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);
}
```

## ScalarSynchronousObservable
```Java
protected ScalarSynchronousObservable(final T t) {
    super(new OnSubscribe<T>() {

        @Override
        public void call(Subscriber<? super T> s) {
            s.onNext(t);
            s.onCompleted();
        }

    });
    this.t = t;
}
```

也就是说nest()方法返回一个生产Observable的ScalarSynchronousObservable，
而这个ScalarSynchronousObservable所生产的唯一的一个元素，就是我们最开始创建的那个Observable对象。

## lift
```Java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber<? super R> o) {
            Subscriber<? super T> st = hook.onLift(operator).call(o);
            // new Subscriber created and being subscribed with so 'onStart' it
            st.onStart();
            onSubscribe.call(st);//注意，这里的onSubscribe属性，其实就是刚才创建的ScalarSynchronousObservable对象中的onSubscribe属性。
    });
}
```
lift就是创建了一个新的Observable对象，然后将这个新创建的Observable返回。因为我们在lift之后调用了subscribe方法，所以就会调用到这个新创建的
Observable对象的onSubscribe.call方法，call方法中做了三件事情：
1.调用operator.call(subscriber)创建一个新的subscriber方法，通常我们会在这个新的subscriber对象的onNext方法中，先对数据进行一些处理，
  然后调用传递过来的subscriber对象的onNext方法。
2.调用新创建的subscriber对象的onStart方法。
3.调用Observable对象的onSubscribe.call(st)方法，其中st是operator新创建的Subscriber对象。

然后再新的OnSubscribe对象的call方法中调用了原来的onSubscribe对象的call(st)方法
st是operator.call的返回值，这里我以OperatorSubscribeOn这个operator来举例分析
```Java
@Override
public Subscriber<? super Observable<T>> call(final Subscriber<? super T> subscriber) {
    final Worker inner = scheduler.createWorker();
    subscriber.add(inner);
    return new Subscriber<Observable<T>>(subscriber) {
        ...
        @Override
        public void onNext(final Observable<T> o) {
            inner.schedule(new Action0() {

                @Override
                public void call() {
                  o.unsafeSubscribe(new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }
                  }
                }
              });
            }）；
    };
}
```
第一行就调用了`final Worker inner = scheduler.createWorker();`

demo中是这样使用subscribeOn的`.subscribeOn(Schedulers.newThread())`,也就是我们的这里的scheduler对象是NewThreadScheduler类型。
```Java
@Override
public Worker createWorker() {
    return new NewThreadWorker(THREAD_FACTORY);
}
```
NewThreadWorker也是实现了Subscription接口。

接着往下看`subscriber.add(inner);`。这里的subscriber其实就是我们在.subscribe()方法中new出来的Subscriber对象。
然后使用这个subscriber对象创建了一个新的subscriber对象。`return new Subscriber<Observable<T>>(subscriber) {`，并作为call方法的返回值。
我们接着看这个新建的subscriber对象的onNext方法做了什么
```Java
@Override
public void onNext(final Observable<T> o) {
    inner.schedule(new Action0() {
      @Override
      public void call() {
          final Thread t = Thread.currentThread();
          o.unsafeSubscribe(new Subscriber<T>(subscriber) {
              ...
              @Override
              public void onNext(T t) {
                  subscriber.onNext(t);
              }

              @Override
              public void setProducer(final Producer producer) {
                  subscriber.setProducer(new Producer() {
                      @Override
                      public void request(final long n) {
                          if (Thread.currentThread() == t) {
                              // don't schedule if we're already on the thread (primarily for first setProducer call)
                              // see unit test 'testSetProducerSynchronousRequest' for more context on this
                              producer.request(n);
                          } else {
                              inner.schedule(new Action0() {
                                  @Override
                                  public void call() {
                                      producer.request(n);
                                  }
                              });
                          }
                      }

                  });
              }

          });
      }
    });
}
```
首先调用了NewThreadWorker.schedule方法，schedule方法最终会调用scheduleActual方法
```Java
public ScheduledAction scheduleActual(final Action0 action, long delayTime, TimeUnit unit) {
    Action0 decoratedAction = schedulersHook.onSchedule(action);
    ScheduledAction run = new ScheduledAction(decoratedAction);
    Future<?> f;
    if (delayTime <= 0) {
        f = executor.submit(run);
    } else {
        f = executor.schedule(run, delayTime, unit);
    }
    run.add(f);

    return run;
}
```
这里首先构造了一个ScheduledAction对象，然后将这个对象提交给executor来执行。
ScheduledAction实现了Runnable和Subscription接口，类描述是这样的`A {@code Runnable} that executes an {@code Action0} and can be cancelled.`
我们来看看这里的action做了哪些事情:
```Java
@Override
public void call() {
    final Thread t = Thread.currentThread();
    o.unsafeSubscribe(new Subscriber<T>(subscriber) {
        ...
        @Override
        public void onNext(T t) {
            subscriber.onNext(t);
        }

        @Override
        public void setProducer(final Producer producer) {
            subscriber.setProducer(new Producer() {

                @Override
                public void request(final long n) {
                    if (Thread.currentThread() == t) {
                        // don't schedule if we're already on the thread (primarily for first setProducer call)
                        // see unit test 'testSetProducerSynchronousRequest' for more context on this
                        producer.request(n);
                    } else {
                        inner.schedule(new Action0() {

                            @Override
                            public void call() {
                                producer.request(n);
                            }
                        });
                    }
                }

            });
        }

    });
}
```
