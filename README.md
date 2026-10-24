<p align="center">
    <img src="resources/svg/cn.xfangfang.wiliwili.svg" alt="logo" height="120" width="120"/>
</p>

<h1 align="center">wiliwili for iOS</h1>

<p align="center">
  B 站第三方客户端的 <b>iOS 移植版</b> · 支持 iPhone 与 iPad · 最低 iOS 14.0
</p>

<p align="center">
  <img alt="iOS" src="https://img.shields.io/badge/iOS-14.0%2B-black?logo=apple&logoColor=white">
  <img alt="arm64" src="https://img.shields.io/badge/arch-arm64-blue">
  <img alt="unsigned" src="https://img.shields.io/badge/signing-unsigned-lightgrey">
  <img alt="license" src="https://img.shields.io/badge/license-GPL--3.0-green">
</p>

<p align="center">
  <a href="#关于本项目"><b>关于本项目</b></a> |
  <a href="#功能"><b>功能</b></a> |
  <a href="#安装"><b>安装</b></a> |
  <a href="#已知限制"><b>已知限制</b></a> |
  <a href="#从源码构建"><b>从源码构建</b></a> |
  <a href="#致谢"><b>致谢</b></a>
</p>

---

## 关于本项目

> **这是 [xfangfang/wiliwili](https://github.com/xfangfang/wiliwili) 的非官方 iOS 移植版。**
>
> 本仓库的代码来自原作者 **xfangfang**，我在此基础上做了 iOS 平台适配与二次开发。
> 它不是上游项目的一部分，上游**不支持、也不接受** iOS 移植（上游维护者已明确表示
> Android / iOS 不在讨论范围内），因此 **iOS 相关的问题请提到本仓库，不要去打扰原作者**。

| | |
|---|---|
| **原作者** | xfangfang |
| **作者主页** | https://github.com/xfangfang |
| **上游仓库** | https://github.com/xfangfang/wiliwili |
| **上游支持平台** | Nintendo Switch / PSVita / PS4 / Windows / macOS / Linux |
| **本仓库** | https://github.com/bugggboy/wiliwili （iOS 移植） |
| **许可证** | GPL-3.0（与原项目一致，本仓库同样以 GPL-3.0 分发） |

**如果你觉得这个客户端做得好，功劳属于原作者，请先去上游仓库点一个 Star。**
本项目只做了让它能在 iOS 上编译和运行的工作，界面、交互逻辑、B 站接口封装全部来自上游。

### 本移植版相对上游的改动

1. **构建系统**：新增 iOS 交叉编译链路（CMake + `ios.toolchain.cmake` + 部署目标 14.0），
   并把 `ios_bundle()` 的调用参数修正为所锁定 borealis 版本的真实签名（上游 yoga 分支的
   六参数写法与其子模块的五参数签名不匹配，会导致 iOS 打包失败）。
2. **依赖供给**：上游在 iOS 上使用 SPM 自动解析 mpv/FFmpeg，本移植版改用 CMake，
   因此以 pkg-config 方式接入一组为 arm64-ios 预编译的静态库（清单见「技术栈」一节）。
3. **应用打包**：补齐 Info.plist 的 `MinimumOSVersion` / `UIDeviceFamily` / `CFBundleSupportedPlatforms`，
   并把 `Images.xcassets` 转成 iOS 可识别的标准命名图标（上游的 `ios_bundle()`
   是把 xcassets 原样拷进 bundle，没有经过 `actool`，iOS 读不到图标）。
4. **兼容性修复**：把 MoltenVK 固定到 1.2.11（1.4.x 用较新的 iOS SDK 构建，
   会引入 iOS 14 的 `libc++.1.dylib` 中不存在的 libc++ 符号，导致 iOS 14 启动即崩）。

除以上改动外，应用层业务代码未做修改。

---

## 功能

界面与操作逻辑与上游保持一致，针对触摸屏做了适配。

- **首页**：推荐 / 热门 / 番剧 / 影视 / 直播 等分区横向滑切
- **搜索**：热搜、视频、番剧、影视，支持筛选
- **动态**：关注 UP 主的最新投稿
- **直播**：关注的主播与系统推荐
- **播放页**：视频、番剧、电影、纪录片、综艺，支持**弹幕**与评论
- **个人页**：扫码登录、历史记录、我的收藏、我的追番、我的追剧
- **主题**：深色 / 浅色主题，跟随系统自动切换
- **多语言**：简体中文、繁体中文、日语、韩语、英语

触屏操作：单指点击选择、上下滑动翻页、左右滑动切分区、播放页点击呼出控制栏。

---

## 安装

本移植版**不签名**，无法通过 App Store 或 TestFlight 安装，需要设备满足以下条件之一：

| 方式 | 要求 |
|---|---|
| **TrollStore**（推荐） | 设备已安装 TrollStore（建议 TrollStore 1.x / 2.x，视系统版本而定） |
| 自签工具 | 使用 AltStore / Sideloadly / 爱思助手等，需 7 天重签（免费账号） |

**系统要求**

- iOS **14.0** 及以上（部署目标为 14.0，已在 iOS 14.7.1 与 iOS 15 上验证）
- **arm64** 设备（iPhone 6s 及以后、iPad 第 5 代及以后）
- 支持 iPhone 与 iPad

**下载**

前往 [Releases](https://github.com/bugggboy/ios-app/releases/tag/wiliwili-ios-preview)
下载 `wiliwili-unsigned.ipa`。

在设备上用 Safari 打开下载链接，下载完成后选择「分享 → TrollStore → Install」即可。

> 构建产物目前发布在移植工程的仓库
> [bugggboy/ios-app](https://github.com/bugggboy/ios-app) 的 Release 中，
> 本仓库只存放源码。

**登录**

个人页点击「扫码登录」，用手机上的 B 站 App 扫描屏幕上的二维码即可。

---

## 已知限制

这是移植版，以下限制来自「未签名 + 无 Apple 开发者账号」这一前提，不是可修可不修的小问题：

- **没有 App Store 版本**，只能侧载；证书过期或用免费账号自签时需重新安装
- **不支持系统推送**：需要推送证书，未签名构建拿不到
- **不支持 App 内购**、iCloud 同步、App Groups、HealthKit 等依赖 provisioning profile 的能力
- **未做真机全量回归**：目前验证过 iOS 14.7.1（iPad）与 iOS 15（iPhone）的启动与主要功能，
  其他系统版本与机型可能存在差异
- 播放器依赖软件/硬件解码组合，极少数编码格式可能无画面（可尝试切换渲染方式）
- 首次启动会解压内置资源，耗时略长，属正常现象

---

## 技术栈

上游的图形与播放架构原样保留，仅在依赖供给方式上做了 iOS 适配。

| 层 | 组件 | 说明 |
|---|---|---|
| UI 框架 | [borealis](https://github.com/xfangfang/borealis)（上游定制分支） | nanovg 绘制，iOS 后端走 SDL2 + OpenGL ES |
| 窗口 / 输入 | SDL2 | 触摸事件映射为上游的输入模型 |
| 播放核心 | **mpv 0.41.0** | 经 [MPVKit](https://github.com/mpvkit/MPVKit) 提供的 arm64-ios 静态库 |
| 解码 | **FFmpeg**（随 MPVKit 提供） | avcodec / avformat / avfilter / avutil / swscale / swresample / avdevice |
| 渲染 | **libplacebo 7.351.0** + **MoltenVK 1.2.11** + **shaderc 2026.1.0** | gpu-next 路径；MoltenVK 固定 1.2.11 以兼容 iOS 14 |
| 字幕 / 弹幕 | **libass 0.17.5** + harfbuzz / freetype / fribidi / unibreak | 弹幕与字幕渲染 |
| 网络 | **GnuTLS 3.8.11** + nettle / hogweed / gmp | HTTPS |
| 其他 | dav1d 1.5.3（AV1）、libdovi 3.3.2（杜比视界）、uavs3d 1.2.1（AVS3）、libbluray 1.4.0、uchardet 0.0.8、lcms2 | 按需链接 |

**构建产物指标**（Release 配置）

| 项 | 值 |
|---|---|
| 部署目标（`LC_BUILD_VERSION.minos`） | **14.0** |
| 架构 | arm64（单一架构） |
| 主二进制大小 | 约 63 MB（依赖全部静态链接） |
| IPA 大小 | 约 28 MB |
| 签名 | 无（`_CodeSignature` 不存在） |
| Bundle ID | `com.bugggboy.wiliwili` |

---

## 从源码构建

iOS 构建**必须**在 macOS 上进行（需要 Xcode 与 iPhoneOS SDK）。
本移植版的 CI 由 GitHub Actions 的 `macos-26` runner 完成，工作流见
[`bugggboy/ios-app`](https://github.com/bugggboy/ios-app) 仓库的 `.github/workflows/wiliwili-ios.yml`。

核心步骤：

```bash
# 1. 拉取源码（含子模块）
git clone --recursive -b yoga https://github.com/bugggboy/wiliwili.git
cd wiliwili

# 2. iOS 上资源需要编进二进制，先构建 host 版资源生成器
cd library/borealis && bash build_libromfs_generator.sh && cd ../..
cp library/borealis/libromfs-generator .

# 3. 准备 mpv / FFmpeg / libplacebo 等静态库与 mpv.pc（详见 CI 工作流）
#    关键点：所有库的 MinOS 必须 ≤ 14.0，否则 iOS 14 会因缺少 libc++ 符号而闪退

# 4. 配置并编译
cmake -S . -B build-ios -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=library/borealis/library/cmake/ios.toolchain.cmake \
  -DPLATFORM=OS64 -DARCHS=arm64 -DDEPLOYMENT_TARGET=14.0 \
  -DPLATFORM_IOS=ON -DDISABLE_WEBP=ON \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,-framework,Metal -Wl,-framework,MetalKit -Wl,-framework,QuartzCore -Wl,-framework,IOSurface -Wl,-framework,UIKit"
cmake --build build-ios -j$(sysctl -n hw.ncpu)

# 5. 校验部署目标与未签名状态
otool -l build-ios/wiliwili.app/wiliwili | awk '/minos/{print; exit}'
```

打包成 IPA：

```bash
rm -rf out && mkdir -p out/Payload
cp -R build-ios/wiliwili.app out/Payload/
ditto -c -k --sequesterRsrc --keepParent out/Payload wiliwili-unsigned.ipa
```

### 两个容易踩的坑

1. **`pkg-config` 会丢弃 `-framework`**
   `pkg_check_modules()` 把这些参数放进 `LDFLAGS_OTHER`，而 wiliwili 只读取
   `MPV_STATIC_LIBRARIES`，所以在 `mpv.pc` 里写 `-framework IOSurface` 是**不生效**的，
   必须改用 `CMAKE_EXE_LINKER_FLAGS` 以 `-Wl,-framework,X` 形式传入。
2. **必须屏蔽宿主机的包管理路径**
   iOS 交叉编译时如果 `find_library()` 抓到 Homebrew 的 macOS 版库，
   会出现 `building for 'iOS', but linking in dylib built for 'macOS'`。
   需要设置 `CMAKE_IGNORE_PATH=/opt/homebrew/lib;/usr/local/lib;…`。

---

## 致谢

- **[xfangfang](https://github.com/xfangfang)** —— wiliwili 的原作者。整个应用的设计、
  界面、业务逻辑与 B 站接口封装都出自他手，本项目只是把它搬到了 iOS 上。
- **[borealis](https://github.com/xfangfang/borealis)** —— 上游自研的跨平台 UI 框架，
  其 iOS 后端（SDL2 + OpenGL ES）是这次移植得以成立的基础。
- **[mpv](https://github.com/mpv-player/mpv) / [FFmpeg](https://ffmpeg.org/)** —— 播放与解码。
- **[MPVKit](https://github.com/mpvkit/MPVKit)** —— 提供了 arm64-ios 的 mpv / FFmpeg 预编译产物。
- **[libplacebo](https://github.com/haasn/libplacebo) / [MoltenVK](https://github.com/KhronosGroup/MoltenVK)** —— 渲染。
- **[libass](https://github.com/libass/libass)** —— 字幕与弹幕。
- 以及所有上游依赖库的作者。

---

## 免责声明

本项目为**第三方非官方客户端**，与哔哩哔哩（bilibili）官方无任何关联，
仅供个人学习与交流使用。所有视频内容与版权归原平台及创作者所有。
请勿将本项目用于商业用途。

---

## 许可证

[GPL-3.0](./LICENSE)，与上游一致。

根据 GPL-3.0 的要求：本仓库是对原项目的修改版本，修改内容已在上文
「本移植版相对上游的改动」中说明；原始版权声明与许可证文件均完整保留。
任何分发本移植版的行为同样须遵循 GPL-3.0，并提供完整源码。
