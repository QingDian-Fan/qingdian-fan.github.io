---
title: AOP之AspectJ基本使用
tags: 其他
permalink: android-source/dc-other-4
key: android-source-dc-other-4
sidebar:
  nav: android-source
---

# 基本介绍

## AOP

### 概念

AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，它可以通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。

### 作用

AOP可以做一些事件的拦截或者做某些事件的预处理。例如：我们经常出现View的连续多次点击，正常我们需要在这块记录第一次的点击时间，与下面做对比如果是在限制的时间内点击的，都忽略掉。这个事例我们就可以采用AOP进行处理，并且进行了对点击事件和限制条件进行隔离，降低业务逻辑的耦合性，提高程序的可重用型和开发效率。

### 实现方式

AOP实现主要分为 静态 和 动态 两种

- 静态方式：在编译期，切面直接以字节码方式编译到目标字节码文件中，生成静态的AOP代理类（主要有：AspectJ等）
- 动态方式：在运行期，为接口动态生成代理类（主要有：Spring AOP、动态代理、自定义类加载器等）

## AspectJ

### 概念

AspectJ是一个面向切面的框架，它扩展了Java语言。AspectJ定义了AOP语法，它有一个专门的编译器用来生成遵守Java字节编码规范的Class文件。简单来说，AspectJ是AOP的一种实现框架。

### 原理

执行AspectJ的时候，我们需要使用ajc编译器，对Aspect和需要织入的Java Source Code进行编译，得到字节码后，可以使用java命令来执行。

ajc编译器会首先调用javac将Java源代码编译成字节码，然后根据我们在Aspect中定义的pointcut找到相对应的Java Byte Code部分，使用对应的advice动作修改源代码的字节码。最后得到了经过Aspect织入的Java字节码，然后就可以正常使用这个字节码了

### 优点

- 功能强大：可以实现形形色色的功能，开发者可以有很大的操作空间。

- 非侵入式：可以在不修改原有程序的前提下，对原有逻辑进行修改和监控。

- 性能强大：在编译期间进行代码修改和注入，对于运行期没有性能损耗。

- 友好：使用java代码来操作字节码，不必对字节码有过多的了解就可以进行修改，对开发者非常友好。

# 使用

## 集成AspectJ

1. 在根目录的build.gradletian添加以下代码：

```
classpath 'org.aspectj:aspectjtools:1.9.6'
classpath 'org.aspectj:aspectjweaver:1.9.6'
```

2. 在module的build.gradle中添加一下代码

```
dependencies {
	implementation 'org.aspectj:aspectjrt:1.9.6'
}

//aop aspectj
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

android.applicationVariants.all { variant ->

        def fullName = ""
        output.name.tokenize('-').eachWithIndex { token, index ->
            fullName = fullName + (index == 0 ? token : token.capitalize())
        }

        JavaCompile javaCompile = variant.javaCompiler

        MessageHandler handler = new MessageHandler(true)
        javaCompile.doLast {
            String[] javaArgs = ["-showWeaveInfo",
                                 "-1.8",
                                 "-inpath", javaCompile.destinationDir.toString(),
                                 "-aspectpath", javaCompile.classpath.asPath,
                                 "-d", javaCompile.destinationDir.toString(),
                                 "-classpath", javaCompile.classpath.asPath,
                                 "-bootclasspath", project.android.bootClasspath.join(
                    File.pathSeparator)]

            String[] kotlinArgs = ["-showWeaveInfo",
                                   "-1.8",
                                   "-inpath", project.buildDir.path + "/tmp/kotlin-classes/" + fullName,
                                   "-aspectpath", javaCompile.classpath.asPath,
                                   "-d", project.buildDir.path + "/tmp/kotlin-classes/" + fullName,
                                   "-classpath", javaCompile.classpath.asPath,
                                   "-bootclasspath", project.android.bootClasspath.join(
                    File.pathSeparator)]

            new Main().run(javaArgs, handler)
            new Main().run(kotlinArgs, handler)

            def log = project.logger
            for (IMessage message : handler.getMessages(null, true)) {
                switch (message.getKind()) {
                    case IMessage.ABORT:
                    case IMessage.ERROR:
                    case IMessage.FAIL:
                        log.error message.message, message.thrown
                        break
                    case IMessage.WARNING:
                    case IMessage.INFO:
                        log.info message.message, message.thrown
                        break
                    case IMessage.DEBUG:
                        log.debug message.message, message.thrown
                        break
                }
            }
        }
    }

}
```

## 基本语法

### Join Points

连接点,程序中可能作为代码注入目标的特定的点。

### Pointcuts

切入点，是带条件的Join Points，确定切入点位置。

| Pointcuts语法                     | **说明**        |
| --------------------------------- | --------------- |
| execution(MethodPattern)          | 方法执行        |
| call(MethodPattern)               | 方法被调用      |
| execution(ConstructorPattern)     | 构造方法执行    |
| call(ConstructorPattern)          | 构造方法被调用  |
| get(FieldPattern)                 | 读取属性        |
| set(FieldPattern)                 | 设置属性        |
| staticinitialization(TypePattern) | static 块初始化 |
| handler(TypePattern)              | 异常处理        |

Pattern规则如下：

- 下表中中括号为可选项，没有可以不写

| **Pattern**        | **规则（注意空格）**                                         |
| ------------------ | ------------------------------------------------------------ |
| MethodPattern      | [@注解] [访问权限] 返回值类型 [类名.]方法名(参数) [throws 异常类型] |
| ConstructorPattern | [@注解] [访问权限] [类名.]new(参数) [throws 异常类型]        |
| FieldPattern       | [@注解] [访问权限] 变量类型 [类名.]变量名                    |
| TypePattern        | `*` 单独使用事表示匹配任意类型，` ..` 匹配任意字符串，` ..` 单独使用时表示匹配任意长度任意类型，` +` 匹配其自身及子类，还有一个 `...` 表示不定个数。也可以使用` &&，，！`进行逻辑运算。 |

### Advice

| **Advice**      | **说明**                                                     |
| --------------- | ------------------------------------------------------------ |
| @Before         | 在执行JPoint之前                                             |
| @After          | 在执行JPoint之后                                             |
| @AfterReturning | 方法执行后，返回结果后再执行。                               |
| @AfterThrowing  | 处理未处理的异常。                                           |
| @Around         | 可以替换原代码。如果需要执行原代码，<br />可以使用ProceedingJoinPoint#proceed()。 |

# 实战

### 定义注解

```
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.FUNCTION)
annotation class SingleClick constructor(
    /**
     * 快速点击的间隔
     */
    val value: Long = 800
)
```

### Aspect实现具体逻辑

```

/**
 * 如果这个文件是 java文件则只有java代码会生效
 * 如果是kotlin文件则java和kotlin代码都会生效
 */
@Aspect
class SingleClickAspect {
    /**
     * 定义切点，标记切点为所有被@SingleClick注解的方法
     * 自己项目中SingleClick这个类的全路径哦
     */
    @Pointcut("execution(@com.demo.aspectj.SingleClick * *(..))")
    fun methodAnnotated() {
    }

    /**
     * 定义一个切面方法，包裹切点方法
     */
    @Around("methodAnnotated()")
    @Throws(Throwable::class)
    fun aroundJoinPoint(joinPoint: ProceedingJoinPoint) {
        // 取出方法的参数
        var view: View? = null
        for (arg in joinPoint.args) {
            if (arg is View) {
                view = arg
                break
            }
        }
        if (view == null) {
            return
        }
        // 取出方法的注解
        val methodSignature = joinPoint.signature as MethodSignature
        val method = methodSignature.method
        if (method == null || !method.isAnnotationPresent(SingleClick::class.java)) {
            return
        }
        val singleClick = method.getAnnotation(SingleClick::class.java)
        // 判断是否快速点击
        if (!isFastClick()) {
            // 不是快速点击，执行原方法
            joinPoint.proceed()
        }
    }
}
```

### 代码中调用样例

```
var index = 0

@SingleClick
fun onClick(view: View) {
    index++
   (view as TextView).text = "点击：$index"
}
```



代码地址：[Aspectj基本使用](https://github.com/QingDian-Fan/ArchitectureProjects/tree/master/AspectjProject)
