---
title: Tensorflow在Android上的搭建
date: 2017-10-25 15:31:14
tags: [Tensorflow, Android, 环境搭建]
categories: [AI]
---

#### 第一个坑

```

android_sdk_repository(
    name = "androidsdk",
    api_level = 24,
#    # Ensure that you have the build_tools_version below installed in the
#    # SDK manager as it updates periodically.
    build_tools_version = "23.0.2",
#    # Replace with path to Android SDK on your system
    path = "/home/kk/Pictures/SDKK/android-sdk-linux/",
)
```
注意在取消注释时，最后一个括号前面的‘#’也要去掉。

### sdk安装
一直提示WORKSPACE里面的android_sdk_reporitory有错，提示我应该安装sdk，但是我有将sdk安装包解压，build-tools等也进行了解压，并且设置了正确的path，所以一直没考虑到sdk的安装问题，一直以为是不是路径写错了，因为error最后提示到了“Referenced by /'external/sdk'”类似的字样，所以google(最近被墙的很严重，勉强能用stackoverflow),baidu了很久，把https://docs.bazel.build/versions/master/tutorial/android-app.html中相关内容看了一遍，还是没找到原因。
后来突然想到会不会真是sdk安装的原因？后来查了一下sdk在Linux上的安装方法，果然有一些问题。
解压完sdk包后，进入tools文件夹，用android命令安装sdk,build-tools等工具（其实感觉更像是更新），这样操作后，总算过了这个error。
所以实际原因就是sdk的安装不对。

### CMakeList.txt
    * 编写了CMake的语句
    * 通过CMake语句指向这个.txt所在目录可以生成makefile文件
    
    通过tensorflow_demo的CmakeList.txt文件，分析一下语法：
```
project(TENSORFLOW_DEMO) # 工程名

#最低版本要求3.4.1，否则不能构建
cmake_minimum_required(VERSION 3.4.1)

#设置makefile为开启
set(CMAKE_VERBOSE_MAKEFILE on)

# 得到这个完整文件名的完整路径
# get_filename_component 得到一个完整文件名中的特定部分。
# get_filename_component(<VAR> FileName
                         PATH|ABSOLUTE|NAME|EXT|NAME_WE|REALPATH
                         [CACHE])
#　　将变量#<VAR>设置为路径(PATH)，文件名(NAME)，文件扩展名(EXT)，去掉扩#展名的文件名(NAME_WE)，完整路径(ABSOLUTE)，或者所有符号链接被解析出的完#整路径(REALPATH)。
　　
get_filename_component(TF_SRC_ROOT ${CMAKE_SOURCE_DIR}/../../../..  ABSOLUTE)
get_filename_component(SAMPLE_SRC_DIR  ${CMAKE_SOURCE_DIR}/..  ABSOLUTE)

if (ANDROID_ABI MATCHES "^armeabi-v7a$")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfloat-abi=softfp -mfpu=neon")
elseif(ANDROID_ABI MATCHES "^arm64-v8a")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -ftree-vectorize")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSTANDALONE_DEMO_LIB \
                    -std=c++11 -fno-exceptions -fno-rtti -O2 -Wno-narrowing \
                    -fPIE")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} \
                              -Wl,--allow-multiple-definition \
                              -Wl,--whole-archive -fPIE -v")

file(GLOB_RECURSE tensorflow_demo_sources ${SAMPLE_SRC_DIR}/jni/*.*)
add_library(tensorflow_demo SHARED
            ${tensorflow_demo_sources})
target_include_directories(tensorflow_demo PRIVATE
                           ${TF_SRC_ROOT}
                           ${CMAKE_SOURCE_DIR})
# 执行文件tensorflow_demo依赖的子目录生成的库名字：android,log.....
target_link_libraries(tensorflow_demo
                      android
                      log
                      jnigraphics
                      m
                      atomic
                      z)
```

### Can't find <android/log.h>
![](http://ovwunej09.bkt.clouddn.com/TF01_1.png)
* 差点忘记了include的用法，本来第一反应android/log.h是路径，然后去上一级android文件夹里看了下....并没有log.h这个文件，又搜索看了下include的用法，发现include应该是搜索系统的一些路径。同时，JNI文件的include查找的是NDK的路径，发现自己还没有设置NDK的环境变量，设置了，问题解决。


### kernels:svd_op,failed
![](http://ovwunej09.bkt.clouddn.com/tf01_5.png)
这个问题有人遇到过，error exit 4的错误，内存空间不足
http://blog.csdn.net/cq361106306/article/details/52929468
这里找到两个解决方案
* 命令增加--local_resources 2048,.5,1.0
* 
tensorflow的kernel出现了问题，然后有点怀疑是tensorflow的安装出了问题。
于是又找了一个tensorflow源码安装的教程，补充了一些依赖的安装。
> sudo apt-get install python-numpy swig python-dev
> 
> sudo apt-get install python-numpy python-dev python-pip python-wheel

### '@aws//:aws' failed
又是一个C++ compilation rule的错误，去到提示的文件README.Bugs中没什么发现，

bazel build -c opt --spawn_strategy=standalone --verbose_failures --local_resources 2048,.5,1.0 //tensorflow/tools/pip_package:build_pip_package
