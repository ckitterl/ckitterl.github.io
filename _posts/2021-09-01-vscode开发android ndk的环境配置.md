---
layout: post
title:  "用vscode+cmake开发android ndk开发环境"
categories: cmake android vscode
---

vscode 现在对于android ndk的开发支持已经蛮完善了。这里介绍一下如何在vscode使用cmake开发android ndk

## 插件安装

- [c/c++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
- [CMake](https://marketplace.visualstudio.com/items?itemName=twxs.cmake)
- [CMake Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)

## 目标

我们的目标是可以根据不同的`ABI`和`buildType`构建出不同的DSO(dynamic shared object, 即so文件)。
> 在官方文档里，已经介绍了如何用命令行构建不同ABI和buildType的DSO，这里针对Cmake Tools将命令行中的参数填入配置项。

## 配置文件

为了能够让vscode能够调用cmake生成构建文件并调用ninja进行构建，我们需要三个配置文件

- variants
- cmake-tool-kits
- settings

### variants
variants的相关配置可以放到`cmake-variants.yaml`或者`cmake-variants.json`，这两者只是格式不一样，效果和配置规则是一样的。这里我们采用`yaml`的文件格式。 这个文件可以放到`工程根目录`或者`.vscode/`目录里。

下面是一个配置的例子
```yaml
buildType:
  default: debug
  description: My option
  choices:
    debug:
      short: Debug
      long: Build With debugging informationg
      buildType: Debug

    release:
      short: Release
      long: Optimize the resulting binaryies
      buildType: Release

abi:
  default: armeabi-v7a
  description: abi
  choices:
    armeabi-v7a:
      short: armeabi-v7a
      long: General for abi of armeabi-v7a
      settings:
        ANDROID_ABI: armeabi-v7a
    arm64-v8a:
      short: arm64-v8a
      long: General for abi of arm64-v8a
      settings:
        ANDROID_ABI: arm64-v8a
```

这里定义了两个选择项`buildType`和`abi`(名字根据自己爱好变更)。buildType定义了构建类型（debug/release）相关的配置参数（这里只是加了一个buildType —— 这个buildType是choices选项里的buildType，是variants配置里定义好的一个属性，并能更改 —— 用于后续构建参数)，定义了abi类型
（注意这里的settings, 我们添加了ANDROID_ABI这个字段，这里的设置将作为-D{SETTING_KEY}={SETTING_VALUE）传给CMake）

> 如果项目组都用vscode，建议将这个配置文件添加到版本管理中，方便项目组其他人员引入

### cmake-tool-kits.json
打开命令面板，输入`CMake: Edit User-Local CMake Kits`
![](assets/image/cmake-setting-cmake-tool-kits.png)
`cmake-tool-kits.json`配置文件是全局的。这个配置文件配置了构建用的工具链和通用设置。
```json
  {
    "name": "Clang android",
    "compilers": {
      "C": "/Users/mk/Library/Android/sdk/ndk/21.1.6352462/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang",
      "CXX": "/Users/mk/Library/Android/sdk/ndk/21.1.6352462/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang++"
    },
    "environmentVariables": {
      "ANDROID_NDK": "/Users/mk/Library/Android/sdk/ndk/21.1.6352462"
    },
    "toolchainFile": "${env:ANDROID_NDK}/build/cmake/android.toolchain.cmake",
    "cmakeSettings":{
      "ANDROID_NATIVE_API_LEVEL": 23,
      "ANDROID_TOOLCHAIN": "clang",
      "ANDROID_NDK":"${env:ANDROID_NDK}",
      "CMAKE_LIBRARY_OUTPUT_DIRECTORY":"output"
    }
  }
```
> 需要在这里设置compilers必须设置，默认的几个都是用的系统clang或者gcc，构建会失败。然后其余的设置都是通用的，所以也都设置在这里的。


### settings.json
打开命令面板，输入`settings`，选择`Perferences: Open User Settings`
![](assets/image/cmake-settings.png)

这个就是普通的vscode配置文件,这里配置的是跟项目相关的，包括`cmake构建结果输出的位置`和`是否在打开项目的时候是否立即进行构建`


```json
{
    "cmake.configureOnOpen": true,
    "cmake.buildDirectory": "${workspaceFolder}/build/${variant:buildType}/${variant:abi}"
}
```

### Additional
如果发现android相关的头文件引用和相关符号无法识别，可以在.vscode文件夹中增加`c_cpp_properties.json`的配置文件，并增加以下配置。
加完配置后就可以在右下角的配置里选择name对应的配置项（我们这里是Android）

这个配置文件是给`C/C++`插件用的，所以需要安装`C/C++`插件才会起作用

```json
{
    "configurations": [
        {
            "name": "Android",
            "includePath": [
                 "${workspaceFolder}",
                 "/Users/mk/Library/Android/sdk/ndk/21.3.6528147/toolchains/llvm/prebuilt/darwin-x86_64/sysroot/usr/include/aarch64-linux-android/**"
            ],
            "defines": [],
            "compilerPath": "${env:ANDROID_NDK}/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang++",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64",
            "configurationProvider": "ms-vscode.cmake-tools"
        }
    ],
    "version": 4
}
```



