先来看一个简单的demo，如何在RN中调用Android原生的的Toast模块。
`index.android.js`

```
var React = require('react-native');
var {
  ToastAndroid,
} = React;
...
ToastAndroid.show('This is a toast with short duration', ToastAndroid.SHORT)
```
index.android.js是ReactNative的入口文件，后缀Android表示是在Android平台使用的代码。ReactNative内置了babel，所以可以使用最新的JavaScript语法来开发（ECMAScript6简称es6），不熟悉es6的同学可以看看阮一峰写的这本[e6入门教程](http://es6.ruanyifeng.com/)。这里我简单介绍一下require，Android程序员可以把require对应到Java的import，使用来导入一个JavaScript模块的。`var {ToastAndroid} = React`这种写法叫结构赋值，就是从React这个对象中，提取出ToastAndroid这个属性所对应的值，并赋值给ToastAndroid这个变量。可以看出toast模块就是从react-native这个模块中的ToastAndroid属性，js中的Toast模块API和Android中的JavaAPI基本是保持一致的。

我们来看看`react-native.js`中，是如何定义ToastAndroid这个属性的
```
var ReactNative = Object.assign(Object.create(require('React')), {
  ...
  ToastAndroid: require('ToastAndroid'),
});
module.exports = ReactNative;
```
react-native.js其实就是声明了ReactNative提供的可以在js中使用的各种模块。

这里我们拿ToastAndroid这个模块来分析RN中js和native的通讯。ToastAndroid模块的源代码在ToastAndroid.android.js中。
```
var RCTToastAndroid = require('NativeModules').ToastAndroid;

var ToastAndroid = {

  SHORT: RCTToastAndroid.SHORT,
  LONG: RCTToastAndroid.LONG,

  show: function (
    message: string,
    duration: number
  ): void {
    RCTToastAndroid.show(message, duration);
  },

};

module.exports = ToastAndroid;

```
可以看到show方法其实就是调用了RCTToastAndroid.show(message, duration)，而RCTToastAndroid又是定义在NativeModules中的。
`NativeModules.js`
```
var NativeModules = require('BatchedBridge').RemoteModules;

var nativeModulePrefixNormalizer = require('nativeModulePrefixNormalizer');

nativeModulePrefixNormalizer(NativeModules);

module.exports = NativeModules;
```
最终，所有的模块都是来自BatchedBridge
`BatchedBridge.js`
```
let MessageQueue = require('MessageQueue');

let BatchedBridge = new MessageQueue(
  __fbBatchedBridgeConfig.remoteModuleConfig,
  __fbBatchedBridgeConfig.localModulesConfig,
);

module.exports = BatchedBridge;
```
BatchedBridge.js中做的事情就是构造一个MessageQueue对象。
*注意*，我们最开始的ToastAndroid.show方法其实最终调用的是MessageQueue.ToastAndroid.show方法。

检查MessageQueue源码，其实你是找不到ToastAndroid属性的声明的，因为这个属性是动态生成的，下面我们就看看这个方法是怎么生成，又是怎么调用的吧。

在分析MessageQueue源码之前，先大概说一下构造函数中传入的两个参数

```
__fbBatchedBridgeConfig.remoteModuleConfig,
__fbBatchedBridgeConfig.localModulesConfig,
```

__fbBatchedBridgeConfig是一个全局js变量，它是在CatalystInstance.java中声明赋值的，通过调用ReactBridge.setGlobalVariable方法。setGlobalVariable是在Jni中声明的方法，最终会调用JavaScriptCore，把Java中定义的JSON字符串，赋值给js的全局对象__fbBatchedBridgeConfig，这个对象会有两个属性remoteModuleConfig和localModulesConfig。
```
private void initializeBridge(
    JavaScriptExecutor jsExecutor,
    NativeModuleRegistry registry,
    JavaScriptModulesConfig jsModulesConfig,
    JSBundleLoader jsBundleLoader) {
  mCatalystQueueConfiguration.getJSQueueThread().assertIsOnThread();
  Assertions.assertCondition(mBridge == null, "initializeBridge should be called once");
  mBridge = new ReactBridge(
      jsExecutor,
      new NativeModulesReactCallback(),
      mCatalystQueueConfiguration.getNativeModulesQueueThread());
  mBridge.setGlobalVariable(
      "__fbBatchedBridgeConfig",
      buildModulesConfigJSONProperty(registry, jsModulesConfig));
  jsBundleLoader.loadScript(mBridge);
}
```
__fbBatchedBridgeConfig.remoteModuleConfig代表的是Java中定义的一些模块，这些模块可以在js中被调用。
__fbBatchedBridgeConfig.remoteModuleConfig的格式大概如下：

```js
{
  "remoteModuleConfig": {
    "Logger": {
      "constants": { /* If we had exported constants... */ },
      "moduleID": 1,
      "methods": {
        "requestPermissions": {
          "type": "remote",
          "methodID": 1
        }
      }
    }
  }
}
{
  'ToastAndroid': {
    moduleId: 0,
    methods: {
      'show': {
        methodID: 0
      }
    },
    constants: {
      'SHORT': '0',
      'LONG': '1'
    }
  },
  'moduleB': {
    moduleId: 0,
    methods: {
      'method1': {
        methodID: 0
      }
    },
    'key1': 'value1',
    'key2': 'value2'
  }
}
```
至于这里的数据是如何生成的，我后面会再写一篇文章分析，这里我们先知道它的格式，以方便我们梳理js代码。

好了，我们回过头来继续分析MessageQueue。MessageQueue构造函数中首先定义了一些实例变量，下面的代码中，我加入了一些注释，来解释这些变量的作用，注释里面的js module指的是只在js中定义的模块，native module指的是在native(这里就是Java)层定义的模块，这些模块都可以在js中使用。

```js
this.RemoteModules = {};//存储最终生成的各个模块信息，包含模块名，模块中的方法，常量等信息

this._require = customRequire || require;//用于加载模块的函数
this._queue = [[],[],[]];//队列，用于存放调用的模块，方法和参数信息，分别存储在第一二三个数组中
this._moduleTable = {};//moduleId查找moduleName的map，用于js module
this._methodTable = {};//methodId查找methodName的map，用于js module
this._callbacks = [];//回调函数数组，和queue一一对应，每个queue中调用的方法，如果有回调函数，那么就在这个数组的对应坐标上
this._callbackID = 0;//回调函数的id，自增

this._genModules(remoteModules);
localModules && this._genLookupTables(
  localModules, this._moduleTable, this._methodTable);

this._debugInfo = {};//放置一些debug相关的信息，主要是调用模块，函数，参数的信息
this._remoteModuleTable = {};//moduleId查找moduleName的map，用于native module
this._remoteMethodTable = {};//methodId查找methodName的map，用于native module

this._genLookupTables(
  remoteModules, this._remoteModuleTable, this._remoteMethodTable);
```

我们来看一下_genModules方法
```js
_genModules(remoteModules) {
  let moduleNames = Object.keys(remoteModules);
  for (var i = 0, l = moduleNames.length; i < l; i++) {
    let moduleName = moduleNames[i];
    let moduleConfig = remoteModules[moduleName];
    this.RemoteModules[moduleName] = this._genModule({}, moduleConfig);
  }
}
```
遍历传过来的remoteModules所有的key，得到moduleName，然后针对每个module调用_genModule方法
```js
_genModule(module, moduleConfig) {
  let methodNames = Object.keys(moduleConfig.methods);
  for (var i = 0, l = methodNames.length; i < l; i++) {
    let methodName = methodNames[i];
    let methodConfig = moduleConfig.methods[methodName];
    module[methodName] = this._genMethod(
      moduleConfig.moduleID, methodConfig.methodID, methodConfig.type);
  }
  Object.assign(module, moduleConfig.constants);
  return module;
}
```
_genModule方法和_genModules方法类似，遍历module下面的所有的方法，对每个方法，调用_genMethod方法。
```js
_genMethod(module, method, type) {
  ...
  fn = function(...args) {
    let lastArg = args.length > 0 ? args[args.length - 1] : null;
    let secondLastArg = args.length > 1 ? args[args.length - 2] : null;
    let hasSuccCB = typeof lastArg === 'function';
    let hasErrorCB = typeof secondLastArg === 'function';
    hasErrorCB && invariant(
      hasSuccCB,
      'Cannot have a non-function arg after a function arg.'
    );
    let numCBs = hasSuccCB + hasErrorCB;
    let onSucc = hasSuccCB ? lastArg : null;
    let onFail = hasErrorCB ? secondLastArg : null;
    args = args.slice(0, args.length - numCBs);
    return self.__nativeCall(module, method, args, onFail, onSucc);
  };
}
```
_genModule分两种情况，根据type来，如果是remoteAsync类型，其实就是异步方法，就用promise来包装一下，我们先不看这种情况，来看看普通的同步方法是如何处理的。
处理部分也比较简单，就是处理一下参数，把onFail和onSucc回调函数从参数中取出来，再调用__nativeCall方法。
```js
__nativeCall(module, method, params, onFail, onSucc) {
  if (onFail || onSucc) {
    // eventually delete old debug info
    (this._callbackID > (1 << 5)) &&
      (this._debugInfo[this._callbackID >> 5] = null);

    this._debugInfo[this._callbackID >> 1] = [module, method];
    onFail && params.push(this._callbackID);
    this._callbacks[this._callbackID++] = onFail;
    onSucc && params.push(this._callbackID);
    this._callbacks[this._callbackID++] = onSucc;
  }
  this._queue[MODULE_IDS].push(module);
  this._queue[METHOD_IDS].push(method);
  this._queue[PARAMS].push(params);

  var now = new Date().getTime();
  if (global.nativeFlushQueueImmediate &&
      now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS) {
    global.nativeFlushQueueImmediate(this._queue);
    this._queue = [[],[],[]];
    this._lastFlush = now;
  }

  if (__DEV__ && SPY_MODE && isFinite(module)) {
    console.log('JS->N : ' + this._remoteModuleTable[module] + '.' +
      this._remoteMethodTable[module][method] + '(' + JSON.stringify(params) + ')');
  }
}
```
__nativeCall方法首先检查是否有onFail和onSucc，如果有的话就压入_callbacks栈中，同时把_callbackID存入参数中
接着可以看到，_queue其实被当做了三个栈来使用，分别压入模块名，方法名和参数信息。

到这里MessageQueue的构造函数就分析的差不多了，那么我们最开始的ToastAndroid.show(message, duration);方法调用，
其实就是往_queue栈中压入了一些信息而已，那么最终是怎么在Java层调用到Android原生的Toast模块的呢？这里的关键就是__nativeCall中调用的nativeFlushQueueImmediate方法，这个方法其实C++代码中注入到Js的一个全局变量，具体怎么注入的，就是在`JSCExecutor.cpp`中调用installGlobalFunction，installGlobalFunction的是通过JavaScriptCore的API来实现让Js可以调用C++代码的。
```
installGlobalFunction(m_context, "nativeFlushQueueImmediate", nativeFlushQueueImmediate);
installGlobalFunction(m_context, "nativeLoggingHook", nativeLoggingHook);
installGlobalFunction(m_context, "nativePerformanceNow", nativePerformanceNow);
```
nativeFlushQueueImmediate其实又调用了`JSCExecutor.cpp`中的flushQueueImmediate方法
`JSCExecutor.cpp`
```
void JSCExecutor::flushQueueImmediate(std::string queueJSON) {
  m_flushImmediateCallback(queueJSON);
}
```
其中，m_flushImmediateCallback是在JSCExecutor的构造函数中初始化
```
JSCExecutor::JSCExecutor(FlushImmediateCallback cb) :
    m_flushImmediateCallback(cb) {
```
那么JSCExecutor对象又是在哪了被创建出来的呢？RN中是通过JSCExecutorFactory这个工厂的createJSExecutor方法来创建JSCExecutor对象的，而这个方法的实现刚好就在`JSCExecutor.cpp`中
```
std::unique_ptr<JSExecutor> JSCExecutorFactory::createJSExecutor(FlushImmediateCallback cb) {
  return std::unique_ptr<JSExecutor>(new JSCExecutor(cb));
}
```
接下来问题又来了，又是谁调用了JSCExecutorFactory.createJSExecutor呢？答案就是`Bridge.cpp`,`Bridge.cpp`是RN的Jni层的入口，Java层大部分调用的Jni函数都是在这个文件中定义的。

`Bridge.cpp`
```
Bridge::Bridge(const RefPtr<JSExecutorFactory>& jsExecutorFactory, Callback callback) :
  m_threadState.reset(new JSThreadState(jsExecutorFactory, std::move(proxyCallback)));

JSThreadState(const RefPtr<JSExecutorFactory>& jsExecutorFactory, Bridge::Callback&& callback) :
    m_callback(callback)
  {
    m_jsExecutor = jsExecutorFactory->createJSExecutor([this, callback] (std::string queueJSON) {
      m_callback(parseMethodCalls(queueJSON), false /* = isEndOfBatch */);
    });
  }adb
```

`ReactBridge.java`回去调用jni中注册的initialize方法。RN所有jni中注册的方法，都在`OnLoad.cpp`
```
registerNatives("com/facebook/react/bridge/ReactBridge", {
    makeNativeMethod("initialize", "(Lcom/facebook/react/bridge/JavaScriptExecutor;Lcom/facebook/react/bridge/ReactCallback;Lcom/facebook/react/bridge/queue/MessageQueueThread;)V", bridge::create),
    makeNativeMethod(
      "loadScriptFromAssets", "(Landroid/content/res/AssetManager;Ljava/lang/String;)V",
      bridge::loadScriptFromAssets),
    makeNativeMethod("loadScriptFromFile", bridge::loadScriptFromFile),
    makeNativeMethod("callFunction", bridge::callFunction),
    makeNativeMethod("invokeCallback", bridge::invokeCallback),
    makeNativeMethod("setGlobalVariable", bridge::setGlobalVariable),
    makeNativeMethod("supportsProfiling", bridge::supportsProfiling),
    makeNativeMethod("startProfiler", bridge::startProfiler),
    makeNativeMethod("stopProfiler", bridge::stopProfiler),
    makeNativeMethod("handleMemoryPressureModerate", bridge::handleMemoryPressureModerate),
    makeNativeMethod("handleMemoryPressureCritical", bridge::handleMemoryPressureCritical),

});
```
可以看到initialize方法其实就是OnLoad.cpp中的bridge这个namespace下得create方法
```
static void create(JNIEnv* env, jobject obj, jobject executor, jobject callback,
                   jobject callbackQueueThread) {
  auto weakCallback = createNew<WeakReference>(callback);
  auto weakCallbackQueueThread = createNew<WeakReference>(callbackQueueThread);
  auto bridgeCallback = [weakCallback, weakCallbackQueueThread] (std::vector<MethodCall> calls, bool isEndOfBatch) {
    dispatchCallbacksToJava(weakCallback, weakCallbackQueueThread, std::move(calls), isEndOfBatch);
  };
  auto nativeExecutorFactory = extractRefPtr<JSExecutorFactory>(env, executor);
  auto bridge = createNew<Bridge>(nativeExecutorFactory, bridgeCallback);
  setCountableForJava(env, obj, std::move(bridge));
}
```
这里调用了Bridge类的构造函数，而Bridge的构造函数中，回去构造一个JSThreadState对象，Bridge所有的API调用都会委托给这个创建的JSThreadState对象。
```
Bridge::Bridge(const RefPtr<JSExecutorFactory>& jsExecutorFactory, Callback callback) :
  m_callback(callback),
  m_destroyed(std::shared_ptr<bool>(new bool(false)))
{
  auto destroyed = m_destroyed;
  auto proxyCallback = [this, destroyed] (std::vector<MethodCall> calls, bool isEndOfBatch) {
    if (*destroyed) {
      return;
    }
    m_callback(std::move(calls), isEndOfBatch);
  };
  m_threadState.reset(new JSThreadState(jsExecutorFactory, std::move(proxyCallback)));
}
```
JSThreadState的构造函数中，会去创建一个JSCExecutor类的对象
```
JSThreadState(const RefPtr<JSExecutorFactory>& jsExecutorFactory, Bridge::Callback&& callback) :
    m_callback(callback)
{
  m_jsExecutor = jsExecutorFactory->createJSExecutor([this, callback] (std::string queueJSON) {
    m_callback(parseMethodCalls(queueJSON), false /* = isEndOfBatch */);
  });
}
```
这里createJSExecutor方法中，传递的是一个C++的函数（猜测是CPP新规范中的lambda的写法，我不熟悉，欢迎指导），这个函数会被赋值给m_flushImmediateCallback成员变量。

找到了创建JSCExecutor对象的地方了，回到刚才，我们说，所有的Js调用Native Module的API的时候，都会调用flushQueueImmediate方法，而flushQueueImmediate方法中会去调用m_flushImmediateCallback函数。

```
JSCExecutor::JSCExecutor(FlushImmediateCallback cb) :
    m_flushImmediateCallback(cb) {

std::unique_ptr<JSExecutor> JSCExecutorFactory::createJSExecutor(FlushImmediateCallback cb) {
  return std::unique_ptr<JSExecutor>(new JSCExecutor(cb));
}
```
也就是说，Js层所有的API调用，都会走到这个m_flushImmediateCallback函数的调用中，这个函数是实现Js和Native通讯的核心。而这里的m_flushImmediateCallback在CPP层，最终是在`OnLoad.cpp`的bridge::create方法中传入的，这个create方法又是由Java层来调用的，`Bridge.java`的构造函数中会调用这个create方法，所以说，Js层调用Native API的时候，最终就是调用了`Bridge.java`中传递过来的ReactCallback对象
```
public ReactBridge(
      JavaScriptExecutor jsExecutor,
      ReactCallback callback,
      MessageQueueThread nativeModulesQueueThread) {
        .....
      }
```
ReactBridge对象的创建，是在`CatalystInstanceImpl.java`中
```
private ReactBridge initializeBridge
  bridge = new ReactBridge(
            jsExecutor,
            new NativeModulesReactCallback(),
            mCatalystQueueConfiguration.getNativeModulesQueueThread());

  private class NativeModulesReactCallback implements ReactCallback
      public void call(int moduleId, int methodId, ReadableNativeArray parameters) {
        mJavaRegistry.call(CatalystInstanceImpl.this, moduleId, methodId, parameters);
    }
```
这里传递的是ReactCallback类的子类`NativeModulesReactCallback`,最终调用的是call方法，而call方法又调用了mJavaRegistry.call方法。看到mJavaRegistry，开发过RN应用的同学应该感到很熟悉了，对的，这里就是对应我们在应用启动的时候注册的NativeModule对象。
`NativeModuleRegistry.Java`
```
/* package */ void call(
    CatalystInstance catalystInstance,
    int moduleId,
    int methodId,
    ReadableNativeArray parameters) {
  ModuleDefinition definition = mModuleTable.get(moduleId);
  if (definition == null) {
    throw new RuntimeException("Call to unknown module: " + moduleId);
  }
  definition.call(catalystInstance, methodId, parameters);
}
```
我们在RN中注册NativeModule是通过add一个ReactPackage对象来实现，家下来我们看一下，RN是如何把我们注册的各个module添加到NativeModuleRegistry中的。
```
mReactInstanceManager = ReactInstanceManager.builder()
                .addPackage(new MainReactPackage())
                .build();
```
ReactInstanceManager.Builder的build方法，会new一个ReactInstanceManagerImpl对象，把我们的ReactPackage对象传递过去。ReactInstanceManagerImpl类的核心就是createReactContext方法，createReactContext会首先遍历所有注册的ReactPackage，对所有的NativeModule，构造一个ModuleDefinition对象，保存到nativeModuleRegistry对象的mModuleTable中。最后，createReactContext会通过CatalystInstanceImpl.Builder构造一个CatalystInstance对象，并把包含各个NativeModule信息的nativeModuleRegistry对象传递过去。具体的模块的注册过程，我后面再写一篇单独的博客介绍，此处就不啰嗦了。
```
private ReactApplicationContext createReactContext(JavaScriptExecutor jsExecutor, JSBundleLoader jsBundleLoader) {
    ...
    for (ReactPackage reactPackage : mPackages) {
      processPackage(reactPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
    }
    ...
    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
      .setCatalystQueueConfigurationSpec(CatalystQueueConfigurationSpec.createDefault())
      .setJSExecutor(jsExecutor)
      .setRegistry(nativeModuleRegistry)
      .setJSModulesConfig(javaScriptModulesConfig)
      .setJSBundleLoader(jsBundleLoader)
      .setNativeModuleCallExceptionHandler(exceptionHandler);
}
```

回到我们最开始的例子中，我们要在Js中使用Toast，那么我们的MainReactPackage中，就需要构造一个ToastModule，来注册给RN，这样才可以给Js调用。
```
public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    return Arrays.<NativeModule>asList(
      new ToastModule(reactContext));
  }
```
需要注意的是，所有要给Js中使用的模块，都需要在Native这边注册一个对应的Module，具体的规范，请参考RN的官方文档，我就不啰嗦了。

最终我们在Js中调用ToastAndroid这个模块的流程就是，Js调用会调用到C++中m_flushImmediateCallback函数，参数就是Js函数的参数加上调用的模块名构成的一个JSON字符串。C++中，有一个parseMethodCalls方法，会从Js传递的JSON中，解析出moduleName,function name,参数等一些列信息，然后C++层会调用Java层的ReactCallback类，Java代码中，会根据传递来的ModuleName functionName找到对应的模块中的方法，然后通过反射执行这些方法，并把参数传递过去。这样就完成了Js对Native代码的调用。
