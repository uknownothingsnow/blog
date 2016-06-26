[原文链接](http://trickyandroid.com/gradle-tip-3-tasks-ordering/)

我注意到我在使用Gradle的时候遇到的大多数问题都是和task的执行顺序有关的。很明显如果我的构建会工作的更好如果我的task都是在正确的时候执行。下面我们就深入了解一下如何更改task的执行顺序。

## dependsOn
我认为最直接的方式来说明的你task的执行时依赖别的task的方法就是使用dependsOn方法。
比如下面的场景，已经存在task A，我们要添加一个task B，它的执行必须要在A执行完之后：
![B->A](http://trickyandroid.com/content/images/2015/07/1-3.png)
这是一个很简单的场景，假定A和B的定义如下：
```groovy
task A << {println 'Hello from A'}
task B << {println 'Hello from B'}
```
只需要简单的调用B.dependsOn A，就可以了。
这意味着，只要我执行task B，task A都会先执行。
```shell
paveldudka$ gradle B
:A
Hello from A
:B
Hello from B
```
另外，你也可以在task的配置区中来声明它的依赖：
```groovy
task A << {println 'Hello from A'}
task B {
    dependsOn A
    doLast {
        println 'Hello from B'  
    }
}
```

如果我们想要在已经存在的task依赖中插入我们的task该怎么做呢？
![insert task](http://trickyandroid.com/content/images/2015/07/2.png)
过程和刚才类似。假定已经存在如下的task依赖：
```groovy
task A << {println 'Hello from A'}
task B << {println 'Hello from B'}
task C << {println 'Hello from C'}

B.dependsOn A
C.dependsOn B
```
加入我们的新的task
```groovy
task B1 << {println 'Hello from B1'}
B1.dependsOn B
C.dependsOn B1
```
输出：
```shell
paveldudka$ gradle C
:A
Hello from A
:B
Hello from B
:B1
Hello from B1
:C
Hello from C
```
注意dependsOn把task添加到依赖的集合中，所以依赖多个task是没有问题的。
![multi depends](http://trickyandroid.com/content/images/2015/07/3.png)
```groovy
task B1 << {println 'Hello from B1'}
B1.dependsOn B
B1.dependsOn Q
```
输出：
```shell
paveldudka$ gradle B1
:A
Hello from A
:B
Hello from B
:Q
Hello from Q
:B1
Hello from B1
```

## mustRunAfter
现在假定我又一个task，它依赖于其他两个task。这里我使用一个真实的场景，我有两个task，一个单元测试的task，一个是UI测试的task。另外还有一个task是跑所有的测试的，它依赖于前面的两个task。
![all tests](http://trickyandroid.com/content/images/2015/07/4.png)
```groovy
task unit << {println 'Hello from unit tests'}
task ui << {println 'Hello from UI tests'}
task tests << {println 'Hello from all tests!'}

tests.dependsOn unit
tests.dependsOn ui
```
输出：
```shell
paveldudka$ gradle tests
:ui
Hello from UI tests
:unit
Hello from unit tests
:tests
Hello from all tests!
```
尽管unitest和UI test会子啊test task之前执行，但是unit和ui这两个task的执行顺序是不能保证的。虽然现在来看是按照字母表的顺序执行，但这是依赖于Gradle的实现的，你的代码中绝对不能依赖这种顺序。
由于UI测试时间远比unit test时间长，因此我希望unit test先执行。一个解决办法就是让ui task依赖于unit task。
![ui->unit](http://trickyandroid.com/content/images/2015/07/5.png)
```groovy
task unit << {println 'Hello from unit tests'}
task ui << {println 'Hello from UI tests'}
task tests << {println 'Hello from all tests!'}

tests.dependsOn unit
tests.dependsOn ui
ui.dependsOn unit // <-- I added this dependency
```
输出:

```shell
paveldudka$ gradle tests
:unit
Hello from unit tests
:ui
Hello from UI tests
:tests
Hello from all tests!
```
现在unit test会在ui test之前执行了。
但是这里有个很恶心的问题，我的ui测试其实并不依赖于unit test。我希望能够单独的执行ui test，但是这里每次我执行ui test，都会先执行unit test。
这里就要用到[mustRunAfter](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:mustRunAfter(java.lang.Object[]))了。mustRunAfter并不会添加依赖，它只是告诉Gradle执行的优先级如果两个task同时存在。比如我们这里就可以指定ui.mustRunAfter unit，这样如果ui task和unit task同时存在，Gradle会先执行unit test，而如果只执行gradle ui，并不会去执行unit task。
```groovy
task unit << {println 'Hello from unit tests'}
task ui << {println 'Hello from UI tests'}
task tests << {println 'Hello from all tests!'}

tests.dependsOn unit
tests.dependsOn ui
ui.mustRunAfter unit
```
输出：
```groovy
paveldudka$ gradle tests
:unit
Hello from unit tests
:ui
Hello from UI tests
:tests
Hello from all tests!
```
依赖关系如下图:
![mustRunAfter](http://trickyandroid.com/content/images/2015/07/6.png)

>mustRunAfter在Gradle2.4中目前还是实验性的功能。

## finalizedBy
现在我们已经有两个task，unit和ui，假定这两个task都会输出测试报告，现在我想把这两个测试报告合并成一个:
![merge](http://trickyandroid.com/content/images/2015/07/7.png)
```groovy
task unit << {println 'Hello from unit tests'}
task ui << {println 'Hello from UI tests'}
task tests << {println 'Hello from all tests!'}
task mergeReports << {println 'Merging test reports'}

tests.dependsOn unit
tests.dependsOn ui
ui.mustRunAfter unit
mergeReports.dependsOn tests
```
现在如果我想获得ui和unit的测试报告，执行task mergeReports就可以了。
```shell
paveldudka$ gradle mergeReports
:unit
Hello from unit tests
:ui
Hello from UI tests
:tests
Hello from all tests!
:mergeReports
Merging test reports
```
这个task是能工作，但是看起来好笨啊。mergeReports从用户的角度来看感觉不是特别好。我希望执行tests task就可以获得测试报告，而不必知道mergeReports的存在。当然我可以把merge的逻辑挪到tests task中，但我不想把tests task搞的太臃肿，我还是继续把merge的逻辑放在mergeReports task中。
[finalizeBy](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:finalizedBy(java.lang.Object[]))来救场了。顾名思义，finalizeBy就是在task执行完之后要执行的task。修改我们的脚本如下：
```groovy
task unit << {println 'Hello from unit tests'}
task ui << {println 'Hello from UI tests'}
task tests << {println 'Hello from all tests!'}
task mergeReports << {println 'Merging test reports'}

tests.dependsOn unit
tests.dependsOn ui
ui.mustRunAfter unit
mergeReports.dependsOn tests

tests.finalizedBy mergeReports
```
现在执行tests task就可以拿到测试报告了：
```shell
paveldudka$ gradle tests
:unit
Hello from unit tests
:ui
Hello from UI tests
:tests
Hello from all tests!
:mergeReports
Merging test reports
```

 >注意，finalizedBy也是Gradle2.4的实验性功能