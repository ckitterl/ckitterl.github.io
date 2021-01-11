---
layout: post
title:  "使用CMake构建android动态连接口"
categories: android
---
# 使用CMake构建android动态链接库
## CMake简介
CMake是个一个开源的跨平台自动化建构系统。CMake并不直接建构出最终的软件，而是产生标准的建构工程文件（如Unix的Makefile或Windows Visual C++的projects/workspaces），然后再依一般的建构方式使用。在这里，CMake生成的是Ninja的构建工程（Ninja是一个专注于速度的小型构建系统[1]，由Evan Martin于2010年在Chrome团队工作时开发），然后再用Ninja来生成动态链接库

### 一个简单的例子
CMake的配置文件取名为CMakeLists.txt。
```txt
########## CMakeLists.txt ##########

# 最低版本要求，版本要求大于等于3.4.1
cmake_minimum_required(VERSION 3.4.1)

# 定义动态链接库的名字、类型和源码
add_library( # Specifies the name of the library.
             native-lib
             # Sets the library as a shared library.
             SHARED
             # Provides a relative path to your source file(s).
             main.c)

# 定义需要链接的其他库
# Links your native library against one or more other native libraries.
target_link_libraries( # Specifies the target library.
                        native-lib # 这里声明目标库
                        android    # 随后目标库需要链接的库
                        log)       # 可以继续往下加，这里都是NDK里的

```

在当前目录里创建源码
```c
// main.c

#include <android/log.h>
#include <jni.h>

#define TAG "MyApplication"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,TAG ,__VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,TAG ,__VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,TAG ,__VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,TAG ,__VA_ARGS__)
#define LOGF(...) __android_log_print(ANDROID_LOG_FATAL,TAG ,__VA_ARGS__)

#define JNI_API_DEF(f) Java_com_nd_app_factory_imapp0nd_MainActivity_##f

JNIEXPORT jint JNICALL
JNI_API_DEF(show)(JNIEnv *env, jobject thiz) {
  LOGW("hello world");
  return 5;
}

```

在这里需要的文件都生成好了，我们需要调用构建命令
```bash
${ANDROID_SDK_HOME}/cmake/3.6.3155560/bin/cmake \
-H. \
-B./arm64-v8a \
-G"Android Gradle - Ninja" \
-DANDROID_ABI=arm64-v8a \
-DANDROID_NDK=${ANDROID_SDK_HOME}/ndk/21.3.6528147 \
-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=./release/obj/arm64-v8a \
-DCMAKE_MAKE_PROGRAM=${ANDROID_SDK_HOME}/cmake/3.6.3155560/bin/ninja \
-DCMAKE_TOOLCHAIN_FILE=${ANDROID_SDK_HOME}/ndk/21.3.6528147/build/cmake/android.toolchain.cmake \
-DANDROID_NATIVE_API_LEVEL=23 \
-DANDROID_TOOLCHAIN=clang
```
- 第一个参数`H`表示CMakeLists.txt所在的路径。我们这里使用当前路径`.`
- 第二个参数`B`表示生成的构建工程目录，如果不存在CMake会为我们创建.这里这里使用`./arm64-v8a`
- 第三个参数`G`表示我们生成的构建工程是什么，默认应该是make。我们这里需要使用Ninja，所以使用`Android Gradle - Ninja`
- 第四个参数`DANDROID_ABI`是我们要生成的so的ABI
- 第五个参数`DCMAKE_LIBRARY_OUTPUT_DIRECTORY`是ndk路径
- 第六个参数`DCMAKE_LIBRARY_OUTPUT_DIRECTORY`是最终so的输出路径，这里声明后，会最终输出给Ninja工程的配置文件.我们这里使用`./release/obj/arm64-v8a`
- 第七个参数`DCMAKE_MAKE_PROGRAM`表示最终构建的构建工具路径，我们使用Ninja，所以我们这里使用Ninja的路径`${ANDROID_SDK_HOME}/cmake/3.6.3155560/bin/ninja`
- 第八个参数`DCMAKE_TOOLCHAIN_FILE`表示使用的工具链的地址
- 第九个参数`DANDROID_NATIVE_API_LEVEL`是android api
- 第十个参数`DANDROID_TOOLCHAIN`表示android构建工具链

其中从第四个参数开始，都是自定义的，会写入到最终的构建系统的

运行完构建命令后，如果我们sdk和ndk的版本/路径都没有问题，会输出
```bash
 $ ${ANDROID_SDK_HOME}/cmake/3.6.3155560/bin/cmake \
-H. \
-B./arm64-v8a \
-G"Android Gradle - Ninja" \
-DANDROID_ABI=arm64-v8a \
-DANDROID_NDK=${ANDROID_SDK_HOME}/ndk/21.3.6528147 \
-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=./release/obj/arm64-v8a \
-DCMAKE_MAKE_PROGRAM=${ANDROID_SDK_HOME}/cmake/3.6.3155560/bin/ninja \
-DCMAKE_TOOLCHAIN_FILE=${ANDROID_SDK_HOME}/ndk/21.3.6528147/build/cmake/android.toolchain.cmake \
-DANDROID_NATIVE_API_LEVEL=23 \
-DANDROID_TOOLCHAIN=clang

-- Configuring done
-- Generating done
-- Build files have been written to: /Users/mk/security/test/cmakeTest/arm64-v8a
```

然后我们再运行Ninja生成最终的结果
```bash
$ $ANDROID_HOME/cmake/3.6.3155560/bin/ninja -C ./arm64-v8a/
ninja: Entering directory `./arm64-v8a/'
[1/1] Linking C shared library release/obj/arm64-v8a/libnative-lib.so
```

最终我们就得到了libnative-lib.so

## 使用自定义高版本cmake
由于Android Sdk里自带的cmake版本太低，有些API或者子模块可能会不存在，所以，我们有时候会使用自定义的cmake版本。
测试发现新版本的`cmake`（3.19.2）也是支持`${ANDROID_SDK_HOME}/cmake/3.10.2.4988404/bin/ninja`(1.8)的，所以我们可以仅替换掉cmake的路径，以及-G参数（新版本cmake没有 `Android Gradle - Ninja`这个generator里，统一使用`Ninja`）

修改后的命令行变成
```bash
${where-you-install-cmake}/cmake \
-H. \
-B./arm64-v8a \
-G"Ninja" \
-DANDROID_ABI=arm64-v8a \
-DANDROID_NDK=${ANDROID_SDK_HOME}/ndk/21.3.6528147 \
-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=./release/obj/arm64-v8a \
-DCMAKE_MAKE_PROGRAM=${ANDROID_SDK_HOME}/cmake/3.10.2.4988404/bin/ninja \
-DCMAKE_TOOLCHAIN_FILE=${ANDROID_SDK_HOME}/ndk/21.3.6528147/build/cmake/android.toolchain.cmake \
-DANDROID_NATIVE_API_LEVEL=23 \
-DANDROID_TOOLCHAIN=clang
```
