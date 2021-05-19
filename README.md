## 编译过程

### 下载ffmpeg源代码
直接下载最新版本的ffmpeg源代码
```shell
git clone https://git.ffmpeg.org/ffmpeg.git ffmpeg
```

### ndk
使用NDKr21来进行编译

#### 下载地址
- [mac NDKr21](https://dl.google.com/android/repository/android-ndk-r21-darwin-x86_64.zip)
- [windows NDKr21](https://dl.google.com/android/repository/android-ndk-r21-windows-x86_64.zip)
- [linux NDKr21](https://dl.google.com/android/repository/android-ndk-r21-linux-x86_64.zip)

### 编写编译脚本
在网络上查找的大多数实例都是说要使用低版本的NDK来编译，但是对于为什么要采用低版本的NDK进行编译，人云亦云。本着知其然，亦要知其所以然的目的，查到了这篇文章 [FFMPEG-Android利用ndk\(r20\)编译最新版本ffmpeg4.2.1](https://juejin.cn/post/6844903945496690696)
真实的原因在于：CLANG是更加高效的C/C++的编译工具，Google在ndk17以后，把GCC移除了 ，全面推行使用CLANG。于是导致编译失败，说是找不到各种文件。实际上只是编译环境的配置错误。

#### 完整的编译脚本
在ffmpeg的根目录上，创建build_ffmpeg_android.sh文件。然后使用vim或者其他编辑器进行编辑。
```shell
#!/bin/bash

PREFIX=./android-build

#此处需要更改为自己的ndk目录
NDK=/Users/rain/ndk/ndk21
#这个是mac上的环境，Linux和window上的路径不一样
TOOLCHAIN=${NDK}/toolchains/llvm/prebuilt/darwin-x86_64
API=29

COMMON_OPTIONS="\
    --target-os=android \
    --disable-static \
    --enable-shared \
    --enable-small \
    --disable-programs \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-doc \
    --disable-symver \
    --disable-asm \
    "

function build_android {
    echo "Compiling FFmpeg for armeabi-v7a"
    # armeabi-v7a
    ./configure \
    --libdir=${PREFIX}/libs/armeabi-v7a \
    --incdir=${PREFIX}/includes/armeabi-v7a \
    --pkgconfigdir=${PREFIX}/pkgconfig/armeabi-v7a \
    --cc=${TOOLCHAIN}/bin/armv7a-linux-androideabi$API-clang \
    --cxx=${TOOLCHAIN}/bin/armv7a-linux-androideabi$API-clang++ \
    --arch=arm \
    --cpu=armv7-a \
    --cross-prefix="${TOOLCHAIN}/bin/arm-linux-androideabi-" \
    --sysroot="${TOOLCHAIN}/sysroot/" \
    --extra-ldexeflags=-pie \
    ${COMMON_OPTIONS}
    make clean
    make -j8 && make install
    echo "Compiling FFmpeg for armeabi-v7a finished."

    echo "Compiling FFmpeg for arm64-v8a"
    # arm64-v8a
    ./configure \
    --libdir=${PREFIX}/libs/arm64-v8a \
    --incdir=${PREFIX}/includes/arm64-v8a \
    --pkgconfigdir=${PREFIX}/pkgconfig/arm64-v8a \
    --arch=aarch64 \
    --cpu=armv8-a \
    --cc=${TOOLCHAIN}/bin/aarch64-linux-android$API-clang \
    --cxx=$TOOLCHAIN/bin/aarch64-linux-android$API-clang++ \
    --cross-prefix="${TOOLCHAIN}/bin/aarch64-linux-android-" \
    --sysroot="${TOOLCHAIN}/sysroot/" \
    --extra-ldexeflags=-pie \
    ${COMMON_OPTIONS}
    make clean
    make -j8 && make install
    echo "Compiling FFmpeg for arm64-v8a finished."

    echo "Compiling FFmpeg for X86_64"
    ./configure \
    --libdir=${PREFIX}/libs/x86_64 \
    --incdir=${PREFIX}/includes/x86_64 \
    --pkgconfigdir=${PREFIX}/pkgconfig/x86_64 \
    --arch=x86_64 \
    --cpu=x86_64 \
    --cc=${TOOLCHAIN}/bin/x86_64-linux-android$API-clang \
    --cxx=$TOOLCHAIN/bin/x86_64-linux-android$API-clang++ \
    --cross-prefix="${TOOLCHAIN}/bin/x86_64-linux-android-" \
    --sysroot="${TOOLCHAIN}/sysroot/" \
    --extra-ldexeflags=-pie \
    ${COMMON_OPTIONS}
    make clean
    make -j8 && make install
    echo "Compiling FFmpeg fro X86_64 finished."

    echo "Compiling FFmpeg for X86_64"
    ./configure \
    --libdir=${PREFIX}/libs/x86 \
    --incdir=${PREFIX}/includes/x86 \
    --pkgconfigdir=${PREFIX}/pkgconfig/x86 \
    --arch=x86 \
    --cpu=i686 \
    --cc=${TOOLCHAIN}/bin/i686-linux-android$API-clang \
    --cxx=$TOOLCHAIN/bin/i686-linux-android$API-clang++ \
    --cross-prefix="${TOOLCHAIN}/bin/i686-linux-android-" \
    --sysroot="${TOOLCHAIN}/sysroot/" \
    --extra-ldexeflags=-pie \
    ${COMMON_OPTIONS} 
    make clean
    make -j8 && make install
    echo "Compiling FFmpeg fro X86_64 finished."
}

build_android
```

#### 编译完成之后
![e363252e1cf84e0eb857141a427c1fb0.png](evernotecid://DB157826-C608-46AF-989E-E110D704789D/appyinxiangcom/15575473/ENResource/p104)

## 集成到Android studio
在这一小节中，通过一个简单的示例，来将ffmpeg中的各个模块集成到Android studio中。以及在集成过程中遇到的坑：D。

### 创建一个native C++工程
![a0734e828c1095647668731d6581e239.png](evernotecid://DB157826-C608-46AF-989E-E110D704789D/appyinxiangcom/15575473/ENResource/p105)
一路下一步就行。

#### CMakeLists.txt
在app目录下创建一个CMakeLists.txt。这样做是为了能够让CMakeLists.txt中的路径更加清晰。Android studio默认创建的CMakeLists是放在cpp目录下面的。

本次构建的内容如下：
```cmake
cmake_minimum_required(VERSION 3.4.1)

find_library(log-lib log)
find_library(android-lib android)

add_library(native-lib SHARED src/main/cpp/native-lib.cpp)

set(JNI_LIBS_DIR ${CMAKE_SOURCE_DIR}/src/main/nativeLibs)
include_directories(${JNI_LIBS_DIR}/includes)

add_library(avutil
        SHARED
        IMPORTED )
set_target_properties(avutil
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libavutil.so )

add_library(swresample
        SHARED
        IMPORTED )
set_target_properties(swresample
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libswresample.so )

add_library(swscale
        SHARED
        IMPORTED )
set_target_properties(swscale
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libswscale.so )

add_library(avcodec
        SHARED
        IMPORTED )
set_target_properties(avcodec
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libavcodec.so )

add_library(avformat
        SHARED
        IMPORTED )
set_target_properties(avformat
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libavformat.so )

add_library(avfilter
        SHARED
        IMPORTED )
set_target_properties(avfilter
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libavfilter.so )

add_library(avdevice
        SHARED
        IMPORTED )
set_target_properties(avdevice
        PROPERTIES IMPORTED_LOCATION
        ${JNI_LIBS_DIR}/${ANDROID_ABI}/libavdevice.so )

target_link_libraries(native-lib
        avutil swresample swscale avcodec avformat avfilter avdevice
        ${log-lib} ${android-lib} )
```

#### 整个项目的目录结构

![1363f1d9f759d75a4ef9c464a9a3e3c8.png](evernotecid://DB157826-C608-46AF-989E-E110D704789D/appyinxiangcom/15575473/ENResource/p106)

#### 遇到的坑

> More than one file was found with OS independent path

这个问题困扰了很久，主要是由于我的Android studio版本是4.1, 项目中使用的gradle版本是4.1.2。然后问题就出现了。由于我这个Hello项目是参考的别人的代码结构，之前是将ffmpeg中的预编译库是放在`src/main/jniLibs`中的，问题的根源就在这里。

官方指导：[自动打包 CMake 使用的预构建依赖项](https://developer.android.com/studio/releases/gradle-plugin#cmake-imported-targets)

具体参考文章：[Android Studio带C++项目提示More than one file was found with OS independent path问题修正](https://blog.k-res.net/archives/2592.html)

参考以上两篇文章，将jniLibs改为nativeLibs，然后更改CMakeLists中的路径，这个问题就解决了。感谢！


