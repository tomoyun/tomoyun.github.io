---
title: Android App运行V8引擎入门
date: 2017-10-08 19:32:40
categories:
- Android开发
tags:
- Android
- V8
- js
---

V8是Google开源的高性能JavaScript引擎，采用C++编写，使用于Chrome浏览器和Node.js。Android版本2.2以后的WebKit也使用了V8作为JS Engine。
在Android App开发中，一般只能通过WebView来和JS代码进行交互，无法直接调用V8引擎。 这样的开发方式限制较大，且性能差。所以本文尝试在App中自己集成V8引擎来执行JS代码。

## 编译V8引擎源码
官网[V8 wiki](https://github.com/v8/v8/wiki)页面提供了完整的流程文档，上面详细描述了源码编译的准备和过程，照着做就行。
对于国内的开发者来说，编译源码第一个麻烦的问题就是网络。Google网站上不去，源码下不下来，也就啥也做不了。
除此之外，V8最新已经使用GN代替GYP来进行编译。在mac上对V8源码进行编译后发现，编译mac平台使用的v8成功，交叉编译Android arm平台的v8失败。只要在gn args中指定Android系统，编译就会出错。
```
target_os = "android"
```
查询相关资料也没有找到对应的解决办法。怀疑官方gn编译方式好像并不支持在mac上交叉编译Android平台的v8，采用老的gyp的编译方式应该可以成功。由于感人的网速和一些其他的原因，就不再继续往下折腾了，直接用[NativeScript](https://github.com/NativeScript/android-runtime)中已经编译好的V8。

## Android App集成V8
在Android App中集成V8的步骤如下:
1. 新建Android App项目，勾选C++ support，注意最新的NDK开发采用CMake脚本编译。
2. 将NativeScript/android-runtime项目中runtime/src/main/libs文件夹拷贝至新建项目的对应位置。
3. 将NativeScript/android-runtime项目中runtime/src/main/include文件夹拷贝至新建项目的对应位置。
4. 修改CMake编辑脚本和build.gradle文件，这将在下一节中具体说明。
5. 实现JNI接口，调用V8引擎执行JS示例代码。

## CMake编译脚本修改


## Hello V8 App


