# AGENTS.md — wiliwili iOS 移植版开发指南

本仓库是 [xfangfang/wiliwili](https://github.com/xfangfang/wiliwili) 的 **iOS 移植版**（GPL-3.0）。
上游支持 Switch / PSVita / PS4 / Windows / macOS / Linux，**iOS 由本仓库独立维护**，
上游不接受 iOS 相关补丁，因此 iOS 的问题与改动都在这里处理。

改代码前请先读本文件；涉及上游业务逻辑时，优先查阅上游仓库与
[`AGENTS.md`](https://github.com/xfangfang/wiliwili/blob/yoga/AGENTS.md) 中更完整的约定。

---

## 平台约束（最重要）

| 项 | 值 | 说明 |
|---|---|---|
| 部署目标 | **iOS 14.0** | 不允许在无用例分支保护的情况下调用 15.0+ API |
| 架构 | arm64 | 只出单一架构，不做模拟器切片 |
| 签名 | 无 | 只通过 TrollStore / 自签安装，不能用推送、内购、iCloud、App Groups |
| 图形 | OpenGL ES 3.0 | 走 SDL2 + GLES，无 Metal 自绘路径 |
| 输入 | 触摸 | 手柄/键盘代码保留但 iOS 不启用 |

**新增依赖时必须检查 MinOS**：所有静态库的 `LC_BUILD_VERSION.minos` 必须 ≤ 14.0，
否则会在 iOS 14 上因缺少 `libc++.1.dylib` 符号而启动即崩
（`basic_filebuf` / `basic_ifstream` 等 libc++ 15 才外置的符号最典型）。

---

## 架构分层

| 层 | 目录 | 职责 |
|---|---|---|
| Activity | `wiliwili/include/activity/`、`wiliwili/source/activity/` | 顶层整屏页面 |
| Fragment | `wiliwili/include/fragment/`、`wiliwili/source/fragment/` | Activity 内的子页面 |
| Presenter | `wiliwili/include/presenter/` | 异步取数，Fragment 继承 Presenter |
| View | `wiliwili/include/view/` | 可复用组件（MPVCore、RecyclingGrid、VideoView…） |
| API | `wiliwili/include/api/bilibili/` | B 站接口封装（cpr + nlohmann/json） |
| Utils | `wiliwili/include/utils/` | Intent（导航）、ProgramConfig（配置）、ImageHelper 等 |

UI 框架是上游定制分支的 **borealis**（`library/borealis/`），所有视图继承 `brls::Box`。

---

## 关键约定

### Presenter 的异步安全

每个取数的 Fragment 都继承 Presenter，回调必须用宏保护，避免 view 已销毁后访问：

```cpp
void requestSomething() {
    ASYNC_RETAIN
    bilibili::HTTP::getResultAsync<MyType>(url, params, [ASYNC_TOKEN](auto result) {
        ASYNC_RELEASE              // view 已销毁则提前返回
        brls::Threading::sync([this, result]() {
            // 这里可以安全更新 UI（主线程）
        });
    });
}
```

用 `CHECK_AND_SET_REQUEST` / `UNSET_REQUEST` 防止重复请求。

### XML 驱动的界面

布局在 `resources/xml/`（`activity/`、`fragment/`、`views/`）。**改界面优先改 XML**，
不要用代码堆布局。XML 会被 `libromfs` 编进二进制。

### 资源

`resources/` 下的资源在 iOS 上通过 **libromfs** 编进可执行文件（iOS 没有合适的散装资源目录）。
新增资源后必须重新生成 `libromfs-generator` 并重新编译，否则运行时读不到。

### 触摸

上游的输入模型基于「焦点 + 按键」，iOS 上由 SDL2 把触摸事件映射成该模型。
新增可交互控件时，确保它在焦点体系里可达（`setFocusable` / `registerXMLView`），
否则触摸点不到。

---

## iOS 构建

iOS 只能在 macOS 上构建，CI 用 GitHub Actions `macos-26` runner。
工作流：`bugggboy/ios-app` 仓库的 `.github/workflows/wiliwili-ios.yml`
（源码通过 `actions/checkout` 指向本仓库，因此本仓库的 Actions 无需启用）。

```bash
# 1. 子模块
git clone --recursive -b yoga https://github.com/bugggboy/wiliwili.git
cd wiliwili

# 2. 资源生成器（iOS 强制 USE_LIBROMFS）
cd library/borealis && bash build_libromfs_generator.sh && cd ../..
cp library/borealis/libromfs-generator .

# 3. 依赖静态库 + mpv.pc（MinOS 全部 ≤ 14.0）
#    mpv 0.41.0 / FFmpeg / libplacebo 7.351.0 / libass 0.17.5 / GnuTLS 3.8.11
#    MoltenVK **1.2.11**（不要用 1.4.x）/ shaderc / dav1d / libdovi / uavs3d / libbluray

# 4. 配置
cmake -S . -B build-ios -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=library/borealis/library/cmake/ios.toolchain.cmake \
  -DPLATFORM=OS64 -DARCHS=arm64 -DDEPLOYMENT_TARGET=14.0 \
  -DPLATFORM_IOS=ON -DDISABLE_WEBP=ON \
  -DCMAKE_IGNORE_PATH="/opt/homebrew/lib;/opt/homebrew/include;/usr/local/lib;/usr/local/include" \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,-framework,Metal -Wl,-framework,MetalKit -Wl,-framework,QuartzCore -Wl,-framework,IOSurface -Wl,-framework,UIKit -Wl,-framework,CoreGraphics -Wl,-framework,Foundation -Wl,-framework,CoreVideo -Wl,-framework,IOKit"

cmake --build build-ios -j$(sysctl -n hw.ncpu)
```

### 三个必须记住的坑

1. **`pkg-config` 会丢弃 `-framework`**
   `pkg_check_modules()` 把 `-framework X` 归到 `LDFLAGS_OTHER`，而本工程只读
   `MPV_STATIC_LIBRARIES`，所以写在 `mpv.pc` 里无效，必须用 `CMAKE_EXE_LINKER_FLAGS`。
2. **必须屏蔽 Homebrew 路径**
   否则 `find_library()` 会抓到 macOS 版 dylib，报
   `building for 'iOS', but linking in dylib built for 'macOS'`。
3. **静态库的 MinOS 决定能否在 iOS 14 运行**
   链接器的 `was built for newer 'iOS' version` 只是 **warning**，不会拦你，
   但运行时 dyld 会因缺符号直接崩。加库前用 `otool -l <lib>.a | awk '/minos/'` 确认。

### 打包

```bash
rm -rf out && mkdir -p out/Payload
cp -R build-ios/wiliwili.app out/Payload/
ditto -c -k --sequesterRsrc --keepParent out/Payload wiliwili-unsigned.ipa
```

`ditto` 而不是 `zip`：`zip` 会丢掉符号链接。

Info.plist 需补齐 `MinimumOSVersion` / `UIDeviceFamily`（`[1,2]`，同时支持 iPhone 与 iPad）/
`CFBundleSupportedPlatforms` / `CFBundleIcons`；图标要用 `sips` 从 `1024.png` 生成
标准命名文件放到 `.app` 根（`ios_bundle()` 不做 `actool` 编译）。

---

## 相对上游修改过的文件

改动集中在构建与打包层，业务代码保持与上游一致，便于日后合并上游更新：

| 文件 / 位置 | 改动 |
|---|---|
| `CMakeLists.txt` | `ios_bundle()` 改为 5 参数，匹配所锁定 borealis 版本的函数签名 |
| `README.md` | 改写为 iOS 专用说明 |
| `AGENTS.md` | 本文件，改写为 iOS 专用 |
| `.github/workflows/` | 仅保留 `wiliwili-ios.yml`，其余平台 CI 已移除 |

合上游更新时：`git remote add upstream https://github.com/xfangfang/wiliwili.git && git fetch upstream yoga`，
重点关注 `CMakeLists.txt` 的冲突。

---

## 验证清单

改动后至少确认：

- [ ] `otool -l build-ios/wiliwili.app/wiliwili | awk '/minos/'` 结果是 `14.0`
- [ ] `nm -u <二进制> | c++filt | grep -E "basic_filebuf|basic_ifstream|basic_ofstream"` 为空
- [ ] 产物里没有 `_CodeSignature`、没有 `embedded.mobileprovision`
- [ ] Info.plist 含 `MinimumOSVersion=14.0`、`UIDeviceFamily=[1,2]`
- [ ] 在 iOS 14 与 iOS 15 真机上均能启动
