---
layout: post
title: WebAssembly+ffmpeg浏览器视频处理
tags: ['2019']
---

- ffmpeg 编译成 wasm 供浏览器使用
- 浏览器上传视频后无缝对接 ffmpeg 能力

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504204310_1c9770d8b86dc16b22195b126427996c.png)

## Emscripten

Emscripten 是一个 LLVM 到 JS 的编译器，编译出 JS 文件供浏览器使用，也可以生成 WASM 提供更好的性能体验。

### 环境准备

cmake、git、python2.7

Mac 环境下，只需要通过 Homebrew 安装 cmake 即可

安装 Homebrew：https://brew.sh

```sh
brew install cmake
```

如果因为网络问题无法使用 Homebrew，参考 8000 的这篇 Proxifier 教程：http://8000.oa.com/#/article?id=KB201905220003&from=km_search

### 安装 emsdk


```sh
git clone https://github.com/juj/emsdk && cd emsdk 
./emsdk install sdk-incoming-64bit binaryen-master-64bit 
./emsdk activate sdk-incoming-64bit binaryen-master-64bit

# 配置环境变量，每次需要编译的时候配置一次
source ./emsdk_env.sh
# source /Users/ringcrl/Documents/github/emsdk/emsdk_env.sh

# 校验编译成功
emcc --help
```

## 编译 FFmpeg 到 LLVM

```sh
# 克隆项目
git clone https://github.com/FFmpeg/FFmpeg
cd FFmpeg

# 编译配置
CPPFLAGS="-D_POSIX_C_SOURCE=200112 -D_XOPEN_SOURCE=600" \
emconfigure ./configure --cc="emcc" \
--prefix=$(pwd)/../dist --enable-cross-compile --target-os=none --arch=x86_64 \
--cpu=generic --disable-ffplay --disable-ffprobe \
--disable-asm --disable-doc --disable-devices --disable-pthreads \
--disable-w32threads --disable-network --disable-hwaccels \
--disable-parsers --disable-bsfs --disable-debug --disable-protocols \
--disable-indevs --disable-outdevs --enable-protocol=file

# 编译
make
```

以上编译参数参考：https://github.com/bgrins/videoconverter.js/blob/master/build/build_lgpl.sh#L20

由于需要校验上传的视频的音频流长度和视频流长度，我的编译参数把 `--disable-ffprobe` 去掉了，因为我后面还需要用到 ffprobe 命令。

## 编译 LLVM 到 WebAssembly

```sh
# 这里的 ffmpeg 是上一步编译输出的 LLVM bitcode，注意一定要是 .bc 后缀
cp ffmpeg ffmpeg.bc

# 然后把 ffmpeg.bc 移到一个单独的文件夹作处理
# 我是在 fock 出来的 FFmpeg 项目下新建了 wasmbuild 做实验

# ASSERTIONS 用于启用运行时检查常见内存分配错误
# VERBOSE 显示详细的信息
# TOTAL_MEMORY 控制内存容量，默认的内存容量为 16MB，栈容量为 5MB
# ALLOW_MEMORY_GROWTH Emscripten 堆一经初始化，容量就固定了，该选项支持自动拓展
# WASM 编译到 wasm，默认是 asm.js
# -02 优化等级
# -v 制定 xx.bc 文件
# -o 制定前置钩子、后置钩子，也就是 JS IO 通信使用，传递文件、传入参数，回调拿到产出结果

emcc -s ASSERTIONS=1 \
-s VERBOSE=1 \
-s TOTAL_MEMORY=33554432 \
-s ALLOW_MEMORY_GROWTH=1 \
-s WASM=1 \
-O2 \
-v ffmpeg.bc \
-o ./ffmpeg.js --pre-js ./ffmpeg_pre.js --post-js ./ffmpeg_post.js
```

## ffmpeg 前后置钩子

上一步最后出现了两个文件 `ffmpeg_pre.js` 钩子和 `ffmpeg_post.js`钩子，这里参考了：https://github.com/bgrins/videoconverter.js/blob/master/build/ffmpeg_pre.js 进行编写。

- 命令行运行之前在创建 `/input`，`/output` 文件夹，并将浏览器上传得到的文件放入 `/input`
- 执行 `ffmpeg` 相关命令，在命令中控制输出逻辑，将输出文件放入到 `/output`
- 在运行完成后读取 `/output` 目录，将文件转成 ArrayBuffer 回调给浏览器

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504204323_573a33ca78c64005d5ad700c9d309d06.png)

## 前端调用

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504204329_1c9770d8b86dc16b22195b126427996c.png)

## 源码、体验地址

- Demo 源码地址：https://github.com/ringcrl/FFmpeg/tree/master/wasmbuild
- 内网体验地址：http://saga.oa.com/wasmbuild/

## 参考链接

- [Emscripten 快速入门](https://www.cntofu.com/book/150/zh/ch1-quick-guide/readme.md)
- [videoconverter.js](https://github.com/bgrins/videoconverter.js)
- [使用WebAssembly+FFmpeg实现前端转码](https://zhuanlan.zhihu.com/p/27910351)
- [ffmpeg 文档](https://www.ffmpeg.org/ffmpeg.html)