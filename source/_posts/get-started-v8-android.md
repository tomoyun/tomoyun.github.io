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
<!--more-->

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
为了将V8引擎集成到Android App中，需要修改CMakeLists.txt编译脚本。
[Android NDK开发: CMake入门](http://cfanr.cn/2017/08/26/Android-NDK-dev-CMake-s-usage/)

设置V8头文件路径
```
include_directories(${PROJECT_SOURCE_DIR}/src/main/cpp/include)
include_directories(${PROJECT_SOURCE_DIR}/src/main/cpp/include/libplatform)
```

导入V8引擎静态库，链接生成项目使用的共享库.so
```
add_library(v8_base STATIC IMPORTED)
set_target_properties(v8_base PROPERTIES IMPORTED_LOCATION ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_base.a)

add_library(v8_libplatform STATIC IMPORTED)
set_target_properties(v8_libplatform PROPERTIES IMPORTED_LOCATION ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_libplatform.a)

add_library(v8_libbase STATIC IMPORTED)
set_target_properties(v8_libbase PROPERTIES IMPORTED_LOCATION ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_libbase.a)

add_library(v8_nosnapshot STATIC IMPORTED)
set_target_properties(v8_nosnapshot PROPERTIES IMPORTED_LOCATION ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_nosnapshot.a)

add_library(v8_libsampler STATIC IMPORTED)
set_target_properties(v8_libsampler PROPERTIES IMPORTED_LOCATION ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_libsampler.a)

target_link_libraries( # Specifies the target library.
                       native-lib

                       v8_base

                       v8_libplatform

                       v8_libbase

                       v8_nosnapshot

                       v8_libsampler

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

同时需要在gradle文件中配置CMake参数
```
externalNativeBuild {
    cmake {
        arguments "-DANDROID_STL=c++_static"
        cFlags "-Wno-error=format-security", "-g"
        cppFlags "-std=c++11", "-fexceptions"
    }
}
```
这样就可以成功将V8编译到我们的共享库中，剩下的就只是在JNI中用C++调用V8引擎。

## Hello V8 App
在JNI中通过C++调用V8引擎没有什么特别，这里直接拿官方的例子实现运行JS脚本，代码如下：
``` C++
Platform* platform = platform::CreateDefaultPlatform();
V8::InitializePlatform(platform);
V8::Initialize();
// Create a new Isolate and make it the current one.
Isolate::CreateParams create_params;
create_params.array_buffer_allocator =
        v8::ArrayBuffer::Allocator::NewDefaultAllocator();
Isolate* isolate = Isolate::New(create_params);
{
    Isolate::Scope isolate_scope(isolate);
    // Create a stack-allocated handle scope.
    HandleScope handle_scope(isolate);
    // Create a new context.
    Local<Context> context = Context::New(isolate);
    // Enter the context for compiling and running the hello world script.
    Context::Scope context_scope(context);
    // Create a string containing the JavaScript source code.
    Local<String> source =
            String::NewFromUtf8(isolate, "'Hello' + ', V8!!!'",
                                NewStringType::kNormal).ToLocalChecked();
    // Compile the source code.
    Local<Script> script = Script::Compile(context, source).ToLocalChecked();
    // Run the script to get the result.
    Local<Value> result = script->Run(context).ToLocalChecked();
    // Convert the result to an UTF8 string and print it.
    String::Utf8Value utf8(result);
    //printf("%s\n", *utf8);
}
// Dispose the isolate and tear down V8.
isolate->Dispose();
V8::Dispose();
V8::ShutdownPlatform();
delete platform;
delete create_params.array_buffer_allocator;
```
有两个地方值得注意：
1. 网上有不少V8的例子都是通过Isolate::GetCurrent(),这样得到的都是NULL。最新的V8不会创建默认的Isolate，需要用户自己来New。这方面请参考最新的V8官方文档
2. 执行JS代码建议放在单独的WorkThread中执行。

## 结尾
以上我们成功将高性能的V8引擎移植到了Android App中，利用它可以执行各种各样的JS代码，来完成一些原本很难实现的功能。

