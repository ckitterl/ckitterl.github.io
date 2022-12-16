---
layout: post
title:  "Dagger的概念以及module的引入"
categories: dagger
---

## 依赖注入（Dependency injection）
依赖注入是软件工程中的一种设计模式，也是实现控制反转的其中一种技术。这种模式能让一个对象接收它所依赖的其他对象。`依赖`是指接收方所需的对象。`注入`是指将`依赖`传递给接收方的过程。在`注入`之后，接收方才会调用该`依赖`。此模式确保了任何想要使用给定服务(`依赖`)的对象(`接收方`像是client)不需要知道如何创建这些服务, 也不知道外部代码（注入器）是如何提供接收方所需的服务。
> 编程语言层次下，“接收方”为对象和 class，“依赖”为变量。在提供服务的角度下，“接收方”为客户端，“依赖”为服务。

该设计的目的是为了分离关注点，分离接收方和依赖，从而提供松耦合以及代码重用性。

传统编程方式，客户对象自己创建一个服务实例并使用它。这带来的缺点和问题是：

- 如果使用不同类型的服务对象，就需要修改、重新编译客户类。
- 客户类需要通过配置来适配服务类及服务类的依赖。如果程序有多个类都使用同一个服务类，这些配置就会变得复杂并分散在程序各处。
- 难以单元测试。本来需要使用服务类的 mock 或 stub，在这种方式下不太可行。

依赖注入可以解决上述问题：

- 使用接口或抽象基类，来抽象化依赖实现。
- 依赖在一个服务容器中注册。客户类构造函数被注入服务实例。框架负责创建依赖实例并在没有用户时销毁它。

以下图示就是一个典型的DI模式
![](/assets/image/dependency-injection.jpeg)

我们的Dagger就是一个用来实现DI Pattern的框架。

## 解决的问题
可以参考这个[slides](https://docs.google.com/presentation/d/1fby5VeGU9CN8zjw4lAb2QPPsKRxx6mSwCe9q7ECNSJQ/pub?start=false&loop=false&delayms=3000&slide=id.p)
## 概念
Dagger实现依赖注入的原理是在编译阶段通过程序中的注解构建一个依赖关系图，然后根据依赖关系图来生成对象工程，并在需要的地方进行注入。而我们要做的是在合适的地方进行注解，从而帮助Dagger生成这张依赖关系图。
### Inject
在这张依赖关系图中，我们首先需要声明哪些对象需要注入，以及这些对象如何生成。首先声明哪些对象需要注入，只要使用Inject关键字对该对象进行注解即可
```Kotlin
    // 在App类中声明一个成员变量tea，并使用Inject注解告诉Dagger这个对象需要注入
    @Inject
    lateinit var tea: Tea
```
然后要告诉Dagger框架如何初始化该对象。也只要在类的构造函数中添加Inject注解
```Kotlin
class BlackTea @Inject constructor(): Tea() {
    ...
}
```
这就是Inject注解的所以使用方法。接下来就是要将两者联系起来，就靠`module`和`component`来实现了

### module
`module`也被翻译成模块，它提供了注入对象的。Dagger通过它来获知如何构建一个依赖对象。有两种方式
- Provides
- Binds


```Kotlin
    @Provides
    fun provideTea(): Tea {
        return Tea.make(Tea.Type.BLACK);
    }
```
provides通过字面意思就是提供者，提供了某个对象如何构建。在这个函数里，我们可以自由的选择如何生成这个对象。函数返回类型告诉Dagger这个函数提供何种返回对象，可以是一个具体类，也可以是一个接口

```Kotlin
@Binds
abstract fun bindTea(BlackTea blackTea): Tea;
```
Binds的意思是将一个父类（函数返回对象）和一个子类（函数参数）进行绑定，当需要父类类型的对象时就使用子类的默认构造函数（如果有多个构造函数用@Inject注解指定）生成具体对象。Binds类型函数必须是一个抽象方法或者试接口类里的一个方法

可以看出来，Provides类型的函数可定制程度比较高，而Binds类型的函数比较简洁。

module根据情况可以是一个普通类，也可以是一个抽象类，甚至可以是一个接口类。下面是一个典型的module类
```Kotlin
@Module
class BeverageModule {
    @Provides
    fun provideTea(): Tea {
        return Tea.make(Tea.Type.BLACK);
    }
}
```

### component
`component`或者叫做容器。容器里包含了`module`以及依赖的对象，并提供了`module`里的对象的生命周期。在注解中添加需要
包含的module列表，这样子的话，这个module里提供的对象的声明周期就归这个Component管理了。
```Kotlin
@Component(modules = [BeverageModule::class])
interface AppComponent {
    fun inject(app: App)
    fun makeTea(): Tea

```

#### 关于生命周期
这里的生命周期怎么理解呢？一般使用Dagger的话会在某个类中调用Dagger框架中的模板方法生成Component对象。需要使用Component包含的
module所提供的对象的类就需要通过持有Component对象的类获取Component对象，然后显示告知Dagger框架此类需要使用Component所包含的module
提供的对象来给类中用Inject注解的对象赋值。类结构大概如下所示
![](/assets/image/dagger-use-case-1.png)

从这个结构中我们就可以大致了解，App类持有Component对象，所以这里的Component对象的生命周期也就由App类决定了。

在Android应用中，这个App类对应的就是Application类，而Business类就是各个Activity类。Activity类通过getApplication并访问其中的
Component对象来完成注入。

## 结尾
有了这三个基本的工具后，我们就能帮助Dagger框架搭建一个完整的依赖关系图了。整个结构如下所示
![](/assets/image/dagger-use-case-2.png)

