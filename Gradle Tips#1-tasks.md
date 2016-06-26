Gradle Tips#1-tasks

[原文链接](http://trickyandroid.com/gradle-tip-1-tasks/)

以这篇博客开始，我将写一系列关于Gradle的文章，用来记录接触Gradle构建脚本以来我所理解的Gradle。

今天要讲的就是Gradle tasks以及task的配置和运行。可能有的读者还不了解Gradle task，用真实的例子来展示应该更容易被理解。下面的代码展示了三个Gradle task，稍后会讲解这三者的不同。
```groovy
	task myTask {
	    println "Hello, World!"
	}

	task myTask {
	    doLast {
	        println "Hello, World!"
	    }
	}

	task myTask << {
	    println "Hello, World!"
	}
```

我的目的是创建一个task，当它执行的时候会打印出来"Hello, World!"。当我第一次创建task的时候，我猜测应该是这样来写的：
```groovy
	task myTask {
	    println "Hello, World!"
	}
```
现在，试着来执行这个myTask，在命令行输入gradle myTask，打印如下：
```shell
	user$ gradle myTask
	Hello, World!
	:myTask UP-TO-DATE
```

这个task看起来起作用了。它打印了"Hello, World!"。
但是，它其实并没有像我们期望的那样。下面我们来看看为什么。在命令行输入gradle tasks来查看所有可用的tasks。
```shell
	user$ gradle tasks
	Hello, World!
	:tasks

	------------------------------------------------------------
	All tasks runnable from root project
	------------------------------------------------------------

	Build Setup tasks
	-----------------
	init - Initializes a new Gradle build. [incubating]
	..........
```

等等，为什么"Hello, World!"打印出来了？我只是想看看有哪些可用的task，并没有执行任何自定义的task！
原因其实很简单，Gradle task在它的生命周期中有两个主要的阶段：配置阶段 和 执行阶段。
可能我的用词不是很精确，但这的确能帮助我理解tasks。

Gradle在执行task之前都要对task先进行配置。那么问题就来了，我怎么知道我的task中，哪些代码是在配置过程中执行的，哪些代码是在task执行的时候运行的？答案就是，在task的最顶层的代码就是配置代码，比如：
```groovy
	task myTask {
	    def name = "Pavel" //<-- 这行代码会在配置阶段执行
	    println "Hello, World!"////<-- 这行代码也将在配置阶段执行
	}
```

这就是为什么我执行gradle tasks的时候，会打印出来"Hello, World!"-因为配置代码被执行了。但这并不是我想要的效果，我想要"Hello, World!"仅仅在我显式的调用myTask的时候才打印出来。为了达到这个效果，最简单的方法就是就是使用Task#doLast()方法。
```groovy
	task myTask {
	    def text = 'Hello, World!' //configure my task
	    doLast {
	        println text //this is executed when my task is called
	    }
	}
```

现在，"Hello, World!"仅仅会在我执行gradle myTask的时候打印出来。Cool，现在我已经知道如何配置以及使task做正确的事情。还有一个问题，最开始的例子中，第三个task的<<符号是什么意思？
```groovy
	task myTask2 << {
	    println "Hello, World!" 
	}
```

这其实只是doLast的一个语法糖版本。它和下面的写法效果是一样的：
```groovy
	task myTask {
	    doLast {
	        println 'Hello, World!' //this is executed when my task is called
	    }
	}
```

但是，这种写法所有的代码都在执行部分，没有配置部分的代码，因此比较适合那些简小不需要配置的task。一旦你的task需要配置，那么还是要使用doLast的版本。

Happy Gradling