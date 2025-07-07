# FFmpeg 6.0 源码编译 支持RTMP_H265 Code id=12 

## 概述

1. 使用WSL 静态编译ffmpeg 源码 支持RTMP_H265 Code id=12 （国内标准）
2. 主要解决在静态编译过程中遇到的 `ERROR: x265 not found using pkg-config` 问题。
3. 实现RTMP协议传输H.265视频流的完整解决方案

## 环境准备

### 系统环境
- Ubuntu 22.04 LTS (WSL2)
- gcc 11.4.0
- pkg-config 0.29.2

### 依赖安装

```bash
# 更新系统包管理器
sudo apt update

# 安装基础编译工具
sudo apt install -y build-essential pkg-config yasm nasm git

# 安装第三方编码库
sudo apt install -y libx264-dev libx265-dev libvpx-dev libopus-dev libnuma-dev

# 安装其他可能需要的依赖
sudo apt install -y cmake libssl-dev zlib1g-dev libbz2-dev
```

## FFmpeg 源码获取与切换版本

```bash
# 克隆 FFmpeg 源码
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg

# 切换到 6.0 版本
git checkout release/6.0
```

## 修改源码支持RTMP_H265 Code id=12

### 背景说明

RTMP协议原生只支持H.264视频编码，对于H.265的支持需要通过扩展实现。Code id=12是国内标准中定义的H.265编码标识符。

### 获取支持H.265的RTMP补丁

```bash
# 克隆支持RTMP H.265的补丁代码
git clone https://github.com/runner365/ffmpeg_rtmp_h265

# 查看补丁文件
ls ffmpeg_rtmp_h265/
```

### 应用补丁

```bash
# 进入FFmpeg源码目录
cd FFmpeg

# 备份原始文件
cp libavformat/flv.h libavformat/flv.h.backup
cp libavformat/flvdec.c libavformat/flvdec.c.backup
cp libavformat/flvenc.c libavformat/flvenc.c.backup

# 复制修改后的文件
cp ../ffmpeg_rtmp_h265/flv.h libavformat/
cp ../ffmpeg_rtmp_h265/flvdec.c libavformat/
cp ../ffmpeg_rtmp_h265/flvenc.c libavformat/

# 验证文件是否正确复制
ls -la libavformat/flv*
```


## 编译配置

### 基础配置（动态链接）

```bash
./configure \
  --enable-gpl \
  --enable-libx264 \
  --enable-libx265 \
  --enable-libvpx \
  --enable-libopus
```

### 静态编译配置（推荐）

```bash
./configure \
  --enable-gpl \
  --enable-static \
  --disable-shared \
  --enable-libx264 \
  --enable-libx265 \
  --enable-libvpx \
  --enable-libopus \
  --pkg-config-flags="--static"
```

## 问题诊断与解决

### 问题现象

在进行静态编译配置时，遇到以下错误：

```
ERROR: x265 not found using pkg-config
```

### 问题分析

通过查看编译日志 `ffbuild/config.log`，发现真正的错误原因：

```bash
tail -50 ffbuild/config.log | grep -A 10 -B 10 x265
```

错误信息显示：
```
/usr/bin/ld: cannot find -lgcc_s: No such file or directory
```

### 根本原因

x265 的 pkg-config 配置文件 `/usr/lib/x86_64-linux-gnu/pkgconfig/x265.pc` 中的 `Libs.private` 字段包含了静态链接时不可用的库：

```ini
Libs.private: -lstdc++ -lm -lgcc_s -lgcc -lgcc_s -lgcc -lrt -ldl -lnuma
```

其中 `-lgcc_s -lgcc -lgcc_s -lgcc` 在静态链接时会导致链接失败。

### 解决方案

#### 1. 备份原始配置文件

```bash
sudo cp /usr/lib/x86_64-linux-gnu/pkgconfig/x265.pc /usr/lib/x86_64-linux-gnu/pkgconfig/x265.pc.backup
```

#### 2. 修改 x265.pc 文件

```bash
sudo nano /usr/lib/x86_64-linux-gnu/pkgconfig/x265.pc
```

将 `Libs.private` 行修改为：

```ini
prefix=/usr
exec_prefix=${prefix}
libdir=/usr/lib/x86_64-linux-gnu
includedir=${prefix}/include

Name: x265
Description: H.265/HEVC video encoder
Version: 3.5
Libs: -L/usr/lib/x86_64-linux-gnu -lx265
Libs.private: -lstdc++ -lm -lrt -ldl -lnuma
Cflags: -I${includedir}
```

**关键变化**：从 `Libs.private` 中移除 `-lgcc_s -lgcc -lgcc_s -lgcc`

#### 3. 验证修复效果

```bash
# 测试 pkg-config 配置
pkg-config --libs --static x265

# 应该输出：-lx265 -lstdc++ -lm -lrt -ldl -lnuma
```

#### 4. 重新配置并编译

```bash
make clean

./configure \
  --enable-gpl \
  --enable-static \
  --disable-shared \
  --enable-libx264 \
  --enable-libx265 \
  --enable-libvpx \
  --enable-libopus \
  --pkg-config-flags="--static"

make -j$(nproc)
sudo make install
```

## 验证编译结果

### 检查版本信息

```bash
ffmpeg -version
```

### 验证编码器支持

```bash
ffmpeg -codecs | grep -E '264|265|vpx|opus'
```

### 检查静态链接

```bash
ldd $(which ffmpeg)
```

对于真正的静态编译，应该显示较少的动态库依赖。

## 部署方案

### 静态编译部署

如果成功进行了静态编译，可以直接复制可执行文件到目标机器：

```bash
# 复制到目标机器
scp /usr/local/bin/ffmpeg user@target-machine:/path/to/ffmpeg
```

## 应用场景

编译完成的 FFmpeg 支持以下功能：

### RTMP265 转 RTSP

```bash
# 直接转封装（延迟最低）
ffmpeg -i "rtmp://source/app/stream" -c copy -f rtsp rtsp://target/app/stream

```

### 支持的编码格式

- **视频编码**：H.264 (libx264)、H.265 (libx265)、VP8/VP9 (libvpx)
- **音频编码**：Opus (libopus)、AAC、MP3 等
- **协议支持**：RTMP、RTSP、HTTP、UDP 等

## 常见问题与解决方案

### 问题 1：找不到头文件

```bash
# 确保安装了开发包
sudo apt install libx264-dev libx265-dev libvpx-dev libopus-dev
```

### 问题 2：pkg-config 找不到库

```bash
# 检查 PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig:$PKG_CONFIG_PATH
```

### 问题 3：其他静态链接问题

如果遇到其他库的静态链接问题，可以采用类似的方法修改对应的 `.pc` 文件，移除有问题的库依赖。

## 技术细节分析

### RTMP H.265 Code id=12 实现原理

RTMP协议中，视频编码类型通过Code id标识：
- Code id=7：H.264/AVC
- Code id=12：H.265/HEVC（扩展标准）

修改后的flv.h文件中定义了对Code id=12的支持：

```c
#define FLV_CODECID_H264    7
#define FLV_CODECID_HEVC    12  // H.265/HEVC support
```

### 为什么 x264 可以正常工作而 x265 不行？

通过对比 x264.pc 和 x265.pc 文件：

**x264.pc**:
```ini
Libs.private: -lpthread -lm -ldl
```

**x265.pc**（修改前）:
```ini
Libs.private: -lstdc++ -lm -lgcc_s -lgcc -lgcc_s -lgcc -lrt -ldl -lnuma
```

x265 的配置包含了 `-lgcc_s` 等在静态链接时不可用的库，而 x264 的配置相对简洁，没有这些问题库。

### pkg-config 的工作原理

当使用 `--pkg-config-flags="--static"` 时，FFmpeg 的 configure 脚本会调用：

```bash
pkg-config --libs --static x265
```

这会返回 `Libs` + `Libs.private` 的组合，如果其中包含不存在的静态库，就会导致链接失败。


## 总结

本文档详细介绍了 FFmpeg 6.0 的源码编译过程，重点解决了 x265 在静态编译时的 pkg-config 配置问题，并实现了对RTMP H.265 Code id=12的支持。

### 关键要点：

1. **问题定位**：通过查看 `ffbuild/config.log` 找到真正的错误原因
2. **根本解决**：修改 pkg-config 配置文件，而不是绕过问题
3. **验证方法**：使用 `pkg-config --libs --static` 验证修复效果
4. **H.265支持**：通过应用runner365的补丁实现RTMP H.265支持
5. **应用价值**：编译完成的 FFmpeg 可以很好地支持 RTMP H.265 到 RTSP 的转推功能

### 技术优势：

- **静态编译**：单文件部署，无依赖问题
- **H.265支持**：支持国内标准的Code id=12 ( FFmpeg6.0 默认已经支持 enhance rtmp 265)
- **协议完整**：支持RTMP、RTMPS、RTMPT、RTMPE等协议
- **性能优化**：支持硬件加速和参数调优

这种解决方案不仅适用于 x265，也可以推广到其他第三方库的类似问题，为流媒体应用提供了完整的H.265 RTMP支持方案。

---

**技术支持**：WSL2 + Ubuntu 22.04  
**编译版本**：FFmpeg 6.0 + RTMP H.265 补丁  
**更新日期**：2025年7月
**文档版本**：2.0
