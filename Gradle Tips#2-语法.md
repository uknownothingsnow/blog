Gradle Tips#2-语法

在[第一篇](http://blog.csdn.net/lzyzsd/article/details/46934187)博客中，我讲解了关于tasks和构建过程中task的不同阶段。在写完这篇之后，我意识到我应该更详尽的讲述一下Gradle。弄懂语法很重要，免得我们碰到复杂的构建脚本的时候直接晕菜。这篇文章我就会讲解一些语法上的东西。

## 语法
Gradle脚本是使用Groovy语言来写的。Groovy的语法有点像Java，希望你能接受它。
如果你对Groovy已经很熟悉了，可以跳过这部分了。
Groovy中有一个很重要的概念你必要要弄懂--Closure（闭包）

### Closures
Closure是我们弄懂Gradle的关键。Closure是一段单独的代码块，它可以接收参数，返回值，也可以被赋值给变量。和Java中的Callable接口，Future类似，也像函数指针，你自己怎么方便理解都好。。。

关键是这块代码会在你调用的时候执行，而不是在创建的时候。看一个Closure的例子：
```groovy
def myClosure = { println 'Hello world!' }

//execute our closure
myClosure()

#output: Hello world!
```

下面是一个接收参数的Closure：
```groovy
def myClosure = {String str -> println str }

//execute our closure
myClosure('Hello world!')

#output: Hello world!
```

如果Closure只接收一个参数，可以使用it来引用这个参数：
```groovy
def myClosure = {println it }

//execute our closure
myClosure('Hello world!')

#output: Hello world!
```

接收多个参数的Closure：
```groovy
def myClosure = {String str, int num -> println "$str : $num" }

//execute our closure
myClosure('my string', 21)

#output: my string : 21
```

另外，参数的类型是可选的，上面的例子可以简写成这样：
```groovy
def myClosure = {str, num -> println "$str : $num" }

//execute our closure
myClosure('my string', 21)

#output: my string : 21
```

很酷的是Closure中可以使用当前上下文中的变量。默认情况下，当前的上下文就是closure被创建时所在的类：
```groovy
def myVar = 'Hello World!'
def myClosure = {println myVar}
myClosure()

#output: Hello world!
```

另外一个很酷的点是closure的上下文是可以改变的，通过Closure#setDelegate()。这个特性非常有用：
```groovy
def myClosure = {println myVar} //I'm referencing myVar from MyClass class
MyClass m = new MyClass()
myClosure.setDelegate(m)
myClosure()

class MyClass {
    def myVar = 'Hello from MyClass!'
}

#output: Hello from MyClass!
```

正如你锁看见的，在创建closure的时候，myVar并不存在。这并没有什么问题，因为当我们执行closure的时候，在closure的上下文中，myVar是存在的。这个例子中。因为我在执行closure之前改变了它的上下文为m，因此myVar是存在的。
	
### 把closure当做参数传递
closure的好处就是可以传递给不同的方法，这样可以帮助我们解耦执行逻辑。前面的例子中我已经展示了如何把closure传递给一个类的实例。下面我们将看一下各种接收closure作为参数的方法：

1.只接收一个参数，且参数是closure的方法： myMethod(myClosure)
2.如果方法只接收一个参数，括号可以省略： myMethod myClosure
3.可以使用内联的closure： myMethod {println 'Hello World'}
4.接收两个参数的方法： myMethod(arg1, myClosure)
5.和4类似，单数closure是内联的： myMethod(arg1, { println 'Hello World' })
6.如果最后一个参数是closure，它可以从小括号从拿出来： myMethod(arg1) { println 'Hello World' }

这里我只想提醒你一下，3和6的写法是不是看起来很眼熟？

## Gradle
现在我们已经了解了基本的语法了，那么如何在Gradle脚本中使用呢？先看下面的例子吧：
```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.2.3'
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```

知道了Groovy的语法，是不是上面的例子就很好理解了？
首先就是一个buildscript方法，它接收一个closure：
```groovy
	def buildscript(Closure closure)
```

接着是allprojects方法，它也接收一个closure参数：
```groovy
	def allprojects(Closure closure)
```
其他的都类似。。。

现在看起来容易多了，但是还有一点不明白，那就是这些方法是在哪里定义的？答案就是Project

## Project
这是理解Gradle脚本的一个关键。
>构建脚本顶层的语句块都会被委托给Project的实例

这就说明[Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)正是我要找得地方。
在[Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)的文档页面搜索buildscript方法,会找到[buildscript{}](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:buildscript(groovy.lang.Closure)) script block(脚本块).等等，script block是什么鬼？根据[文档](https://docs.gradle.org/current/dsl/)：

>script block就是只接收closure作为参数的方法

继续阅读buildscript的文档，文档上说Delegates to: ScriptHandler from buildscript。也就是说，我们传递给buildscript方法的closure，最终执行的上下文是[ScriptHandler](https://docs.gradle.org/current/javadoc/org/gradle/api/initialization/dsl/ScriptHandler.html)。在上面的例子中，我们的传递给buildscript的closure调用了repositories(closure)和dependencies(closure)方法。既然closure被委托给了ScriptHandler，那么我们就去ScriptHandler中寻找dependencies方法。

找到了[void dependencies(Closure configureClosure)](https://docs.gradle.org/current/javadoc/org/gradle/api/initialization/dsl/ScriptHandler.html#dependencies(groovy.lang.Closure))，根据文档，dependencies是用来配置脚本的依赖的。而dependencies最终又是委托到了[DependencyHandler](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/dsl/DependencyHandler.html)。

看到了Gradles是多么广泛的使用委托了吧。理解委托是很重要滴。

### Script blocks
默认情况下，Project中预先定义了很多script block，但是Gradle插件允许我们自己定义新的script blocks！
这就意味着，如果你在build脚本顶层发了一些{...}，但是你在Gradle的文档中却找不到这个script blocks或者方法，绝大多情况下，这是一些来自插件中定义的script block。

### android Script block
我们来看看默认的Android app/build.gradle文件:
```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 22
    buildToolsVersion "22.0.1"

    defaultConfig {
        applicationId "com.trickyandroid.testapp"
        minSdkVersion 16
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
As we can see, it seems like there should be android method which accepts Closure as a parameter. But if we try to search for such method in Project documentation - we won't find any. And the reason for that is simple - there is no such method :)

If you look closely to the build script - you can see that before we execute android method - we apply com.android.application plugin! And that's the answer! Android application plugin extends Project object with android script block (which is simply a method which accepts Closure and delegates it to AppExtension class1).

可以看到，这里有一个android方法，它接收一个closure参数。但是如果我们在Project的文档中搜索，是找不到这个方法的。原因很简单，这不是在Project中定义的方法。
仔细查看build脚本，可以看到在抵用android方法之前，我们使用了com.android.application插件。Android application插件扩展了Project对象，添加了android Script block。
哪里可以找到Android插件的文档呢？[Android Tools website](https://developer.android.com/tools/building/plugin-for-gradle.html)或者[这里](https://developer.android.com/shareables/sdk-tools/android-gradle-plugin-dsl.zip)

如果我们打开AppExtension的文档，我们就可以找打build script中使用的方法了属性：
1.compileSdkVersion 22. 如果我们搜索compileSdkVersion，将会找到这个属性. 这里我们给这个属性赋值 "22"。
2.buildToolsVersion和1类似
3.defaultConfig - 是一个script block将会委托给ProductFlavor类来执行。
4.其他
现在我们已经能够理解Gradle脚本的语法了，也知道如何查找文档了。

## 练习
来配置点东西练练手吧。
在AppExtension中，我找了一个script block testOptions，它会把closure参数委托给TestOptions类。根据文档，TestOptions类有两个属性：reportDir和resultsDir。reportDir是用来保存test报告的。我们来改改这个属性试试。
```groovy
android {
......
    testOptions {
        reportDir "$rootDir/test_reports"
    }
}
```
这里我使用了Project中定义的[rootDir](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:rootDir)属性，该属性值为项目的根目录。
现在如果我执行./gradlew connectedCheck，test报告将会被保存到[rootProject]/test_reports目录中。
不要在真实的项目尝试这个修改，所有的构建输出都应该在build目录中，这样才不至于污染你的项目目录结构。
