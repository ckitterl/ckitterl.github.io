---
layout: post
title:  "gradle plugin入门"
categories: gradle
---

gradle的核心部分仅仅提供了基本的功能和通用的机制，比如`task`, `plugin`, `project`的概念实现，`groovy`/`kotlin`语言的实现接口。我们所熟知的功能其实大部分都是由plugin提供的。比如[JavaCompile](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.compile.JavaCompile.html)提供了java代码的构建，提供了java代码构建过程中的参数配置的代码块，源文件的目录结构的配置和默认约定等。具体的说，gradle plugin可以为我们提供以下的几个功能
- 扩展了gradle的模型，增加新的DSL元素，让我们可以为插件动态的提供配置信息
- 可以为工程提供能力，比如定义一个task提供打印依赖树的能力
- 为其他插件的必填配置提供默认的值，比如在一个工程内的所有模块都用到的底层依赖提供统一的版本号


## gradle plugin的类型

- Script plugin
- Binary plugin

Script plugin就是简单的一个gradle文件，可以是在本地，也可以是一个远程的url。Script plugin是自解析的；也就是说，它一经引用就会被解析执行。一般是一些陈述性质的脚本。比如在另外的gradle里定义的ext代码块
```gradle
apply from '{replace with the uri of your script file}'
```

Binary plugin指的是实现了Plugin接口的类，这个类可以是在脚本里，也可以是打包好的jar文件，或者是在`buildSrc`模块里，或者是一个单独发布的Plugin。一般我们说的插件也就是这里的Binary plugin。这也是我们下面我们要介绍的Plugin类型

## gradle plugin使用

一般我们是使用发布到仓库的的plugin。有两种方式引入

- 在`plugins` 代码块里定义plugin(`plugins id`)
- 在主代码里声明(`apply plugin`)

### 在`plugins`代码块中定义plugin

```groovy
plugins {
    id «plugin id»
    id «plugin id» version «plugin version» [apply «false»]
}
```

这里有两种定义方式
1. 内置的plugin，比如我们熟知的`java`
2. 外部依赖的plugin，需要声明版本

#### 延迟apply
默认情况下在plugins代码块里声明的plugin都会被立即apply，也就是Plugin类里的apply函数是否会被立即执行。可以通过在当前plugin id 声明后面添加`apply fasle`来让plugin延迟apply。比如

```groovy
// build.gradle
plugins {
    id 'com.example.hello' version '1.0.0' apply false
    id 'com.example.goodbye' version '1.0.0' apply false
}
```

```groovy
// hello-a/build.gradle
plugins {
    id 'com.example.hello'
}
```

```groovy
// hello-b/build.gradle
plugins {
    id 'com.example.hello'
}
```

```groovy
// hello-c/build.gradle
plugins {
    id 'com.example.goodbye'
}
```

甚至于你可以通过再重新定义plugin的版本，在子模块里使用自己认为更好的plugin版本

#### plugin的仓库
一旦脚本开始执行，gradle会搜索声明的plugin有没有在核心plugins里(gradle自带的)；
如果没有，就去中心仓库里找，默认是去[官方的中心仓库](https://plugins.gradle.org/m2/)，可以在[plugins.gradle.org](https://plugins.gradle.org)中搜索。

可以在`settings.gradle`中设置plugin的仓库地址
```groovy
// settings.gradle
pluginManagement {
    repositories {
        maven {
            url './maven-repo'
        }
        gradlePluginPortal()
        ivy {
            url './ivy-repo'
        }
    }
}
```

> 如果使用nexus仓库，可以创建一个proxy类型的仓库，代理地址填写`https://plugins.gradle.org/m2/`

> pluginManagement块可以声明在settings.gradle或者init.gradle里，有`plugins`,`resolutionStrategy`和`repositories`三个子块, 分别用于:
> - plugins: 声明plugin
> - resolutionStrategy: 允许自定义规则来修改plugin的版本，添加plugin等
> - repositories： 声明plugin的插件仓库地址

> 详情参考[官方文档](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_version_management)

## gradle plugin的开发

首先在build.gradle中引入Plugin的开发依赖库
```groovy
plugin {
   id 'java-gradle-plugin'
}

dependencies {
    // Use the latest Groovy version for building this library
    implementation  gradleApi()
}
```

然后编写代码，下面是一个用Java编写的简单插件类。当插件被加载的时候，apply函数会自动执行
```Java
import org.gradle.api.Plugin;
import org.gradle.api.Project;

public class MyFirstPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        System.out.println("Hello from the GreetingPlugin");
    }
}
```
> 可能有人会有疑问，如果我一个工程里包含多个Plugin如何识别的。
> 可以这样理解，这里依赖的是一个包含了plugin类的jar包（通过gradle的版本管理下载到本地缓存），然后调用apply plugin语句标明我们要使用哪个Plugin类。
> 我们也可以自己打包一个jar文件，然后依赖进来，然后再`apply plugin` jar中的类。（使用略）
> 至于使用id的时候，一个id只会对应一个plugin实现类，后面会具体讲到

有时候我们会在插件里添加多个不同的任务来完成不同的功能。这时候我们就可以自定义任务类
```Java
import org.gradle.api.DefaultTask;

public class HelloTask extends DefaultTask {

    public void run() {
        System.out.println("Hello from task " + getPath() + "!");
    }
}
```

然后将`HelloTask`这个任务添加到插件中
```Java
public void apply(Project project) {
    ...
    // 在插件的apply函数中添加
    project.getTasks().create("hello", HelloTask.class);
    ...
    // 或者也可以这样
    project.task('taskName') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
    ...
}
        
```

## gradle plugin的发布
发布分为两种。
- 第一种就跟普通的Java Library一样；但是这种发布的Plugin的使用只能使用apply plugin的方式使用。
- 第二种会附带发布到gradle的plugin仓库（[中心仓库](https://plugins.gradle.org/)）。

### 普通发布
gradle plugin也是一个jar包，我们可以类似Java Library或者类似Android Library的方式发布

```groovy
plugins {
	id 'maven'
}


afterEvaluate { project ->
    uploadArchives {
        repositories {
            mavenDeployer {
                configurePOM(pom)

                repository(url: getReleaseRepositoryUrl()) {
                    authentication(userName: getRepositoryUsername(), password: getRepositoryPassword())
                }
                snapshotRepository(url: getSnapshotRepositoryUrl()) {
                    authentication(userName: getRepositoryUsername(), password: getRepositoryPassword())
                }
            }
        }
    }
}
```

发布之后，我们就能够用以`apply plugin`的方式使用了。
```groovy
// build.gradle
buildscript {
	dependencies {
        classpath '{group-id}:{artification}:{version}'
    }
}
```

```groovy
// module/build.gradle
apply plugin: '{plugin-class}'
```

### plugin id发布
这种发布方式让gradle可以用plugins id的方式使用plugin
```groovy
plugins {
    id 'java-gradle-plugin'
    id 'maven-publish' // 发布插件
}

// plugins id发布了两个部分，jar包和索引部分，这里声明jar包的包名和版本,这里的版本和plugins id的版本是一致的
group 'com.example.plugin.MyFirstPlugin'
version '1.0.0'

gradlePlugin {
    plugins { // 一个工程可以包含多个plugin，所以在这个代码块里可以声明多个plugin节点
        myPlugins { // 这里是一个自定义的节点，可以起自己喜欢的名字，只要不占用保留字
            id = 'com.example.plugin.MyFirstPlugin' // 这个就是我们的plugin id
            implementationClass = 'com.example.plugin.MyFirstPlugin' // plugin id的实现类
        }
    }
}

publishing {
    repositories {
        maven {
            url getRepositoryUrl()
            credentials {
                username getRepositoryUsername()
                password getRepositoryPassword()
            }
        }
    }
}

```
这里时候就可以使用`gradle publish`命令进行发布了

> TIP: 构建的插件的时候使用的JDK版本需要和使用插件时候环境的JDK版本需要保持一致，否者会报错；Android Studio一般用的自带的JDK是1.8，而我们一般开发用的IDEA用的自定义版本是11(文章编写的时候)，需要将IDEA的JDK版本降到1.8（建议），或者Android Studio升到11。

## 使用`buildscript`模块

有时候我们的plugin只服务于当前这个工程，并不需要发布到仓库中，知识后我们就可以受用`buildSrc`。
`buildSrc`是一个特殊的模块名，是gradle保留的模块名，不能用于自定义模块，不需要在`settings.gradle`中添加到模块列表中（会报错）

buildSrc的模块创建不管是IDEA还是Android Studio都没有提供模板，我们就按照普通的Java模块创建。
删除掉不必要文件和声明。
```bash
├── build.gradle
└── src
    └── main
        └── java
            └── com
                └── example
                    └── plugin
                         └── MyFirstPlugin.java

```
> 我们这里选择用`Java`来开发gradle plugin，也可以选择`groovy`或者`kotlin`，代码都是类似的。只是需要在build.gradle里引入对应的语言依赖和相对应构建插件。

我们以刚才的MyFirstPlugin类举例。开发完buildSrc模块后就可以在其他模块的build.gradle中依赖具体的某个plugin实现类

```groovy
// module/build.gradle

apply plugin: com.example.plugin.MyFirstPlugin

```

> `buildSrc`有性能问题，有建议用`composite builds`代替；但是现在还是可以用的，也是官方建议的一种使用方式



## 写在最后
plugin的开发作为不管是Java开发也好，android开发也好，应该是一项需要掌握的技能。除了官网意外，google的android插件是一个很好的学习范例
本文提到的代码也发布到github上，欢迎下载测试











