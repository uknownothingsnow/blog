入口MainActivity onCreate
mReactInstanceManager = ReactInstanceManager.builder()
        .setApplication(getApplication())
        .setBundleAssetName("MoviesApp.android.bundle")
        .setJSMainModuleName("Examples/Movies/MoviesApp.android")
        .addPackage(new MainReactPackage())
        .setUseDeveloperSupport(true)
        .setInitialLifecycleState(LifecycleState.RESUMED)
        .build();

((ReactRootView) findViewById(R.id.react_root_view))
        .startReactApplication(mReactInstanceManager, "MoviesApp", null);


ReactRootView startReactApplication
  mReactInstanceManager.attachMeasuredRootView(this);

ReactInstanceManager.attachMeasuredRootView
  if (mCurrentReactContext == null) {
    initializeReactContext();
  } else {
    attachMeasuredRootViewToInstance(rootView, mCurrentReactContext.getCatalystInstance());
  }

initializeReactContext
if (mUseDeveloperSupport) {
  if (mDevSupportManager.hasUpToDateJSBundleInCache()) {
    // If there is a up-to-date bundle downloaded from server, always use that
    onJSBundleLoadedFromServer();
    return;
  } else if (mBundleAssetName == null ||
      !mDevSupportManager.hasBundleInAssets(mBundleAssetName)) {
    // Bundle not available in assets, fetch from the server
    mDevSupportManager.handleReloadJS();
    return;
  }
}
因为在MainActivity onCreate里面设置了.setUseDeveloperSupport(true)
mDevSupportManager.handleReloadJS();其实就是从网络请求js文件，并且显示一个progress dialog，下载成功之后
还是会调用onJSBundleLoadedFromServer()；

private void onJSBundleLoadedFromServer() {
  recreateReactContext(
      new JSCJavaScriptExecutor(),
      JSBundleLoader.createCachedBundleFromNetworkLoader(
          mDevSupportManager.getSourceUrl(),
          mDevSupportManager.getDownloadedJSBundleFile()));
}

JSCJavaScriptExecutor构造函数中直接调用了jni层的initialize方法
public JSCJavaScriptExecutor() {
  initialize();
}

private native void initialize();

JSBundleLoader是一个抽象类
public abstract void loadScript(ReactBridge bridge);
默认的几个匿名实现都是通过ReactBridge来加载资源文件
bridge.loadScriptFromNetworkCached(sourceURL, null)

recreateReactContext方法
mCurrentReactContext = createReactContext(jsExecutor, jsBundleLoader);
for (ReactRootView rootView : mAttachedRootViews) {
  attachMeasuredRootViewToInstance(
      rootView,
      mCurrentReactContext.getCatalystInstance());
}

createReactContext方法
CoreModulesPackage coreModulesPackage =
        new CoreModulesPackage(this, mBackBtnHandler);
processPackage(coreModulesPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);

private void processPackage(
      ReactPackage reactPackage,
      ReactApplicationContext reactContext,
      NativeModuleRegistry.Builder nativeRegistryBuilder,
      JavaScriptModulesConfig.Builder jsModulesBuilder) {
  for (NativeModule nativeModule : reactPackage.createNativeModules(reactContext)) {
    nativeRegistryBuilder.add(nativeModule);
  }
  for (Class<? extends JavaScriptModule> jsModuleClass : reactPackage.createJSModules()) {
    jsModulesBuilder.add(jsModuleClass);
  }
}
processPackage做的事情很简单，就是把CoreModulesPackage中声明的NativeModule和JSModule添加到对应的modulesBuilder中去。

接着就是添加用户自定义的module进来
for (ReactPackage reactPackage : mPackages) {
  processPackage(reactPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
}

CatalystInstance.Builder catalystInstanceBuilder = new CatalystInstance.Builder()
        .setCatalystQueueConfigurationSpec(CatalystQueueConfigurationSpec.createDefault())
        .setJSExecutor(jsExecutor)
        .setRegistry(nativeRegistryBuilder.build())
        .setJSModulesConfig(jsModulesBuilder.build())
        .setJSBundleLoader(jsBundleLoader)
        .setNativeModuleCallExceptionHandler(mDevSupportManager);

nativeRegistryBuilder.build()
public NativeModuleRegistry build() {
  JsonFactory jsonFactory = new JsonFactory();
  StringWriter writer = new StringWriter();
  try {
    JsonGenerator jg = jsonFactory.createGenerator(writer);
    jg.writeStartObject();
    for (ModuleDefinition module : mModuleDefinitions) {
      jg.writeObjectFieldStart(module.name);
      jg.writeNumberField("moduleID", module.id);
      jg.writeObjectFieldStart("methods");
      for (int i = 0; i < module.methods.size(); i++) {
        MethodRegistration method = module.methods.get(i);
        jg.writeObjectFieldStart(method.name);
        jg.writeNumberField("methodID", i);
        jg.writeEndObject();
      }
      jg.writeEndObject();
      module.target.writeConstantsField(jg, "constants");
      jg.writeEndObject();
    }
    jg.writeEndObject();
    jg.close();
  } catch (IOException ioe) {
    throw new RuntimeException("Unable to serialize Java module configuration", ioe);
  }
  String moduleDefinitionJson = writer.getBuffer().toString();
  return new NativeModuleRegistry(mModuleDefinitions, mModuleInstances, moduleDefinitionJson);
}
就是构造一个js对象，并且将标注了ReactMethod的方法添加到这个对象上。格式如下
{
  'moduleA': {
    moduleId: 0,
    methods: {
      'method1': {
        methodID: 1
      }
    },
    constants: {
      'key1': 'value1',
      'key2': 'value2'
    }
  },
  'moduleB': {
    moduleId: 0,
    methods: {
      'method1': {
        methodID: 1
      }
    },
    'key1': 'value1',
    'key2': 'value2'
  }
}

CatalystInstance catalystInstance = catalystInstanceBuilder.build();

catalystInstance.initialize();
看一下CatalystInstance的注释，其实就是对JSC bridge的一个封装，用来实现Js和Java的互相调用
/**
 * A higher level API on top of the asynchronous JSC bridge. This provides an
 * environment allowing the invocation of JavaScript methods and lets a set of
 * Java APIs be invokable from JavaScript as well.
 */
@DoNotStrip
public class CatalystInstance

看看他的实例变量
private final NativeModuleRegistry mJavaRegistry;
// Access from JS thread
private @Nullable ReactBridge mBridge;
private @Nullable JavaScriptModuleRegistry mJSModuleRegistry;
其中mJavaRegistry中保存了了所有的native modules，也就是js中可以调用的Java代码。
mJSModuleRegistry中保存了所有的Js模块，也就是Java中可以调用的Js代码。
mBridge其实就是调用jni的接口。

在catalystInstance的构造函数中
mCatalystQueueConfiguration.getJSQueueThread().runOnQueue(
  new Runnable() {
    @Override
    public void run() {
      initializeBridge(jsExecutor, registry, jsModulesConfig, jsBundleLoader);
      mJSModuleRegistry =
          new JavaScriptModuleRegistry(CatalystInstance.this, jsModulesConfig);

      initLatch.countDown();
    }
  });

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

initializeBridge方法主要就是用来new一个ReactBridge对象出来。
ReactBridge加载了so reactnativejni，并在构造函数中调用了jni层的initialize代码。看看ReactBridge的注释
Interface to the JS execution environment and means of transport for messages Java<->JS.
JS执行环境的接口，在Java和Js之间转换消息。
其实就是一个壳，所以的方法都是在jni中声明的，这里是为了方便Java调用Jni中的方法。

mBridge.setGlobalVariable(
        "__fbBatchedBridgeConfig",
        buildModulesConfigJSONProperty(registry, jsModulesConfig));

buildModulesConfigJSONProperty
JsonGenerator jg = jsonFactory.createGenerator(writer);
      jg.writeStartObject();
      jg.writeFieldName("remoteModuleConfig");
      jg.writeRawValue(nativeModuleRegistry.moduleDescriptions());
      jg.writeFieldName("localModulesConfig");
      jg.writeRawValue(jsModulesConfig.moduleDescriptions());
      jg.writeEndObject();
      jg.close();
返回一个json，包含了native和js的module描述信息，这个描述就是上面说到的这个json对象，包含了module的名字，id以及方法和常量信息。

至于setGlobalVariable方法，根据名字就可以猜测到，是在js里注入了一个全局变量__fbBatchedBridgeConfig，这样在任何js中都可以获取到模块信息。

jsBundleLoader.loadScript(mBridge);
这里的jsBundleLoader就是前面创建出来的
JSBundleLoader.createCachedBundleFromNetworkLoader(
    mDevSupportManager.getSourceUrl(),
    mDevSupportManager.getDownloadedJSBundleFile()));
会调用bridge.loadScriptFromNetworkCached(sourceURL, null);来加载网络上的资源文件。
reactBridge其实是调用的jni中定义的loadScriptFromNetworkCached方法。

CatalystInstance构造函数中，随后构造了一个JavaScriptModuleRegistry对象
mJSModuleRegistry =
                new JavaScriptModuleRegistry(CatalystInstance.this, jsModulesConfig);
