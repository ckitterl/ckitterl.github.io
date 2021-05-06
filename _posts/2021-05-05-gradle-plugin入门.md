---
layout: post
title:  "gradle plugin入门"
categories: gradle-plugin gradle
---

gradle的核心部分仅仅提供了基本的功能和通用的机制，比如`task`, `plugin`, `project`的概念实现，`groovy`/`kotlin`语言的实现接口。我们所熟知的功能其实大部分都是由plugin提供的。比如[JavaCompile](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.compile.JavaCompile.html)提供了java代码的构建，提供了java代码构建过程中的参数配置的代码块，源文件的目录结构的配置和默认约定等。具体的说，gradle plugin可以为我们提供以下的几个功能
- 扩展了gradle的模型，增加新的DSL元素，让我们可以为插件动态的提供配置信息
- 可以为工程提供能力，比如定义一个task提供打印依赖树的能力
- 为其他插件的必填配置提供默认的值，比如在一个工程内的所有模块都用到的底层依赖提供统一的版本号


## gradle plugin的类型

- Script plugin
- Binary plugin

Script plugin就是简单的一个gradle文件，可以是在本地路径，也可以是一个远程的url。Script plugin是自解析的；也就是说，它一经引用就会被解析执行。一般是一些陈述性质的脚本。比如在另外的gradle里定义的ext代码块

Binary plugin指的是实现了Plugin接口的类，这个类可以是在脚本里，也可以是打包好的jar文件，或者是在`buildSrc`模块里，或者是一个单独发布的Plugin。一般我们说的插件也就是这里的Binary plugin。我们的介绍的Plugin开发也是只的Binary plugin

## gradle plugin的使用
一般我们是使用打包好的plugin。有两种方式引入

- 在`plugins` 代码块里定义plugin(`plugins id`)
- 使用`buildscript`代码块里声明(`apply plugin`)

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
gradle先在搜索声明的plugin有没有在核心plugins里；如果没有，就去中心仓库里找，默认是去[官方的中心仓库](https://plugins.gradle.org/m2/)，可以在[plugins.gradle.org](https://plugins.gradle.org)中搜索。如果想用私有仓库，可以在`settings.gradle`中设置
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

> 如果使用nexus仓库，可以常见一个proxy类型的仓库，代理地址填写`https://plugins.gradle.org/m2/`

> pluginManagement块可以声明在settings.gradle或者init.gradle里，有`plugins`,`resolutionStrategy`和`repositories`三个子块, 分别用于:
> - plugins: 声明plugin
> - resolutionStrategy: 允许自定义规则来修改plugin的版本，添加plugin等
> - repositories： 声明plugin的插件仓库地址

> 详情参考[官方文档](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_version_management)

## 使用`buildscript`代码块里声明
```groovy
// build.gradle
buildscript {
    repositories {
        gradlePluginPortal()
    }
    dependencies {
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.8.5'
    }
}

apply plugin: 'com.jfrog.bintray'
```

这种用法就需要显式的调用`apply plugin`来apply一个plugin。上面例子中`apply plugin: 'com.jfrog.bintray'`的`com.jfrog.bintray`是具体的要apply的那个plugin类.

> 可以这样理解，这里依赖的是一个包含了plugin类的jar包（通过gradle的版本管理下载到本地缓存），然后调用apply plugin语句显示的apply。
> 我们也可以自己打包一个jar文件，然后依赖进来，然后再`apply plugin` jar中的类。（使用略）

## `plugins id` vs `apply plugin`
`apply plugin`是一种过时的写法，官方建议使用`plugins id`来代替。但是具体的原因没有看到官方的解释（有看到的麻烦评论一下）。我这里能感受到的只是旧的写法比较啰嗦，`plugins id`只要定义一个id和version就够了。现在两种都还是可以用的


## gradle plugin的开发
gradle plugin的代码结构有三种
- 直接在脚本文件里定义plugin类
- 使用`buildSrc`模块
- 单独的plugin工程

### 直接在脚本文件里定义plugin类
直接在脚本文件里定义plugin类很容易理解，就像下面的build.gradle文件中定义的
```groovy
// build.gradle
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```

执行plugin中添加的任务
```bash
> gradle -q hello
Hello from the GreetingPlugin
```

### buildSrc
`buildSrc`是一个特殊的模块名，是gradle保留的模块名，不能用于自定义模块，不需要在`settings.gradle`中添加到模块列表中（会报错）

#### 模块创建
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
                         └── GreetingPlugin.java

```
> 我们这里选择用`Java`来开发gradle plugin，也可以选择`groovy`或者`kotlin`，代码都是类似的。只是需要在build.gradle里引入对应的语言依赖和相对应构建插件。

由于我们采用了Java来开发，也还没有准备好写单元测试和覆盖率统计，所以这里的`build.gradle`文件可以不做额外声明，是一个空的文件`GreetingPlugin.java`里定义GreetingPlugin类，代码如下
```java
package com.example.plugin;
import org.gradle.api.Plugin;
import org.gradle.api.Project****;

class GreetingPlugin implements Plugin<Project> {
    @Override
    public void apply(Project target) {
		System.out.println("hello GreetingPlugin");
    }
}
```

#### plugin使用
有了上述的buildSrc后，我们就可以直接在其他build.gradle的apply这个plugin了。
```groovy
// module/build.gradle

apply plugin: com.example.plugin.GreetingPlugin

```

> `buildSrc`有性能问题，有建议用`composite builds`代替；但是现在还是可以用的，也是官方建议的一种使用方式

### 单独的工程
目前IDEA没有模板来创建一个gradle plugin的基础工程，所以我们使用gradle init命令来创建
```bash
> gradle init  #选择`basic`, `groovy`, `junit`(低版本的gradle没有第三个选择项)；这些选项也都是默认值
```
初始化完成之后，文件结构如下所示
```bash
── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
```
接下来往空白的`build.gradle`里添加`java-gradle-plugin`这个plugin和`gradleApi()`的依赖，如下
```groovy
/*
 * This file was generated by the Gradle 'init' task.
 *
 * This is a general purpose Gradle build.
 * Learn how to create Gradle builds at https://guides.gradle.org/creating-new-gradle-builds
 */

plugins {
    id 'java-gradle-plugin' // 用java语言来开发gradle插件的插件
}

repositories {
	// 可以添加自定义的仓库地址
}

dependencies {
	// gradle的api依赖,里面包含了plugin的接口声明等
    implementation  gradleApi()
}
```
最后按照`java-gradle-plugin`的代码结构约定创建代码文件
```bash
{project_root)
    └── src
         └── main
              └── java
                   └── com
                        └── example
                             └── plugin
                                  └── GreetingPlugin.java
```
GreetingPlugin类的开发就跟buildSrc是一样的

#### plugin的发布
plugin有两种发布方式：
1. 普通发布
2. plugin id发布

##### 普通发布
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

##### plugin id发布
这种发布方式让gradle可以用plugins id的方式使用plugin
```groovy
plugins {
    id 'java-gradle-plugin'
    id 'maven-publish' // 发布插件
}

// plugins id发布了两个部分，jar包和索引部分，这里声明jar包的包名和版本,这里的版本和plugins id的版本是一致的
group 'com.nd.sdp.plugin.MyFirstPlugin'
version '1.0.0'

gradlePlugin {
    plugins { // 一个工程可以包含多个plugin，所以在这个代码块里可以声明多个plugin节点
        myPlugins { // 这里是一个自定义的节点，可以起自己喜欢的名字，只要不占用保留字
            id = 'com.nd.sdp.plugin.MyFirstPlugin' // 这个就是我们的plugin id
            implementationClass = 'com.nd.sdp.plugin.MyFirstPlugin' // plugin id的实现类
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















