Hi3559v200 SDK中内置的mpp sample位于middleware、amp、reference等不同位置。在SDK中编写编译存在编译时间长，代码不分离等问题，所以，我采用cmake使应用程序能够脱离SDK，在SDK外面进行独立编译。

- 主要内容
  - [如何使用](#how-to-use)
  - [关于link library](#about-sdk-library)

# how to use
- [cmake代码](https://github.com/LyuOnLine/test_examples/tree/master/hisi3559/cmake_for_apps_standalone)已经上传至github，使用很简单，只要简单改写CMakeLists.txt，加入自己要编译的代码即可。


```
git clone https://github.com/LyuOnLine/test_examples
cp -r test_examples/hisi3559/cmake_for_apps_standalone/* 工程目录/

# 修改CMakeLists.txt，增加要编译代码
add_executable(test
  src/main.cpp
  ...
)
target_link_libraries(test PRIVATE
  hisi_svp              # svp相关库
  hisi_mpp              # mpp相关库
)
```

- 另外，cmake代码依赖HI3559_SDK_ROOT环境变量，编译前，需要设置如下:

```
cd Hi3559V200_MobileCam_SDK_V1.0.1.5
export HI3559_SDK_ROOT=$(pwd)
```

# about sdk library

[cmake/hisi_sdk.cmake](https://github.com/LyuOnLine/test_examples/blob/master/hisi3559/cmake_for_apps_standalone/cmake/hisi_sdk.cmake)已经在hisi3559v200 SDK目录，查找3559的各种库并以interface target的形式提供使用。

在引用时，可视需要包含以下库:

```
hisi_mpp   # media process platform系统相关库
hisi_svp   # Smart Vision Platform相关库
hisi_rtpserver  # rtp server
```
