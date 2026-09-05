# LSFG-Android-Plus — 基于 lsfg-vk 管线的 Android 帧生成

[![Discord](https://img.shields.io/discord/1496212333595463780?label=Discord&logo=discord)](https://discord.gg/CkuumNJ7s4)

> **[FrankBarretta/LSFG-Android](https://github.com/FrankBarretta/LSFG-Android) 的中文汉化增强版。**
> English documentation: [README.md](README.md)。

LSFG-Android 将 [`lsfg-vk`](https://github.com/PancakeTAS/lsfg-vk) 的 Vulkan
帧生成管线带到了 Android 上。由于 Android 12+ 禁止在非调试进程中加载外部代码，
本项目无法像 Linux 的 implicit layer 那样钩住其他应用的交换链，因此改为对
`MediaProjection` 屏幕捕获流做帧插值，再把生成的帧合成到悬浮于目标游戏之上的
系统悬浮窗中。Adreno 7xx 级别及更新的 GPU 已可端到端跑通帧生成。

## 本 fork（Plus）与原版的区别

上游功能全部保留，在此基础上增加了：

- **系统要求 Android 14 及以上** —— minSdk 从 29（Android 10）提高到
  34（Android 14）；特权捕获/计时路径面向 Android 14–16（含 ColorOS），
  不支持更低版本。
- **简体中文汉化 + 英文国际化** —— 将约 125 条原先硬编码的界面文案抽取为资源。
  应用跟随系统语言：中文系统显示中文，其余显示英文。
- **触摸时暂停插帧** —— 悬浮窗运行期间，长按或点按屏幕即暂停插帧，松开后恢复。
  Shizuku/Root 捕获模式监听 `/dev/input` 事件流，检测精确（长按、点按均支持）；
  MediaProjection 模式回退为 1px 透明小窗接收悬浮窗外的触摸（仅支持点按）。
  **玩游戏时必须关闭**——否则游戏中每次触摸都会暂停插帧。
- **旋转屏幕后悬浮窗毫秒级恢复** —— native 渲染循环收到停止请求后立即退出，
  不再以 500ms 超时逐帧排空积压队列；framegen 将计算管线持久化到
  `VkPipelineCache` 文件（`framegen_pcache_<uuid>.bin`），约 2 秒的首次编译
  成本只付一次，之后重新初始化不再重编。
- **小窗 / 自由窗口时自动结束会话** —— 监视器检测目标应用窗口离开全屏
  （窗口边界 ≠ 屏幕最大边界，这是 ColorOS 上可靠的判定信号）后自动结束会话
  并提示，因为 UID 过滤截屏本来就只能显示目标窗口，其余区域是黑的。
- **首页悬浮窗权限入口** —— 绿/橙色权限状态指示，一键跳转系统悬浮窗权限页，
  首次启动权限缺失时自动弹窗提示。
- **首页界面重构** —— 捕获模式、节奏预设移到首页做成可选卡片；删除冗余说明卡；
  修复切换帧倍率导致悬浮窗卡死的问题（重新初始化时回退到当前活动目标包）。

## 仓库结构

| 路径 | 说明 |
|---|---|
| [`LSFG-Android-Application/`](LSFG-Android-Application/) | Android Studio 工程 —— Kotlin + Jetpack Compose UI、JNI/C++ 渲染循环，即用户直接使用的 app。 |
| [`lsfg-vk-android/`](lsfg-vk-android/) | 子模块。基于 [`lsfg-vk`](https://github.com/PancakeTAS/lsfg-vk) 1.0.0 的分支，叠加了 Android 专属补丁（基于 AHardwareBuffer 的图像共享、`createContextFromAHB`、`waitIdle`）。所有补丁均以 `#ifdef __ANDROID__` 保护，原 Linux 构建路径不受影响。 |

Android app 通过 CMake `add_subdirectory()` 直接引用子模块中的 `framegen/`，
不随仓库分发预编译 `.so` —— 构建 app 时会透明地为 `arm64-v8a` 和 `x86_64`
编译 framegen 库。

## 功能一览

- **帧生成（LSFG_3_1 / LSFG_3_1P）**：通过 app Vulkan 会话与 framegen 内部设备
  之间的 AHardwareBuffer 共享，全程在 GPU 上运行。
- **游戏内实时设置抽屉**：倍率（2×–8×）、flow scale（0.25–1.0）、性能/HDR 模式、
  抗伪影、bypass、带 slack 调节的垂直同步对齐、节奏预设、目标 FPS 上限、
  队列深度、EMA 抖动平滑。多数参数即时重建 native 上下文；bypass / 节奏 /
  Shizuku 计时支持热应用，不会中断会话。
- **按应用自动悬浮窗**：选定目标应用后，目标进入前台时悬浮窗自动就位。
  两种入口形态：可拖动的启动圆点，或屏幕可配置边缘的图标按钮。
- **首次启动教程**：引导完成无障碍设置（触摸穿透服务和 Android 13+ 侧载应用
  需要的"受限设置"解除）。
- **全透明度触摸穿透**：悬浮窗可寄宿于 `SYSTEM_ALERT_WINDOW`，也可在用户启用
  `LsfgAccessibilityService` 后寄宿为 `TYPE_ACCESSIBILITY_OVERLAY` —— 后者是
  触摸过滤严格的 OEM 机型的可选方案。
- **捕获源**：MediaProjection（默认，每次会话的可见画面都来自它）和 Shizuku
  度量模式（额外提供一条按目标 UID 过滤的特权计时旁路，用于节奏诊断，
  Shizuku 缓冲永远不进入可见视频路径）。
- **后处理管线**：NNAPI 的 NPU 预设（锐化、细节增强、色度清理、游戏清晰）、
  GPU 放大阶段、CPU 增强（LUT、鲜艳度、饱和度、暗角）。
- **帧率 HUD**：真实/总 FPS 计数、帧时间曲线、节奏诊断。
- **崩溃报告**：同时捕获 Java/Kotlin 未捕获异常和 native 信号（SIGSEGV、
  SIGABRT 等）并回溯栈，一键通过 `ACTION_SEND` 分享错误报告。
- **Vulkan 交换链输出路径**：在 CPU blit 兜底方案之上提供高效呈现。
- **旋转与沉浸模式感知**的悬浮窗组件。

> [!IMPORTANT]
> 需要一份合法购买的 Lossless Scaling。本仓库**不**随附、下载或打包
> `Lossless.dll`。用户通过存储访问框架自行选择自己的 DLL，app 在设备端提取
> shader 到私有存储后即删除 DLL 副本。本项目不以任何形式分发 Lossless Scaling
> 的资产。

## 构建

```sh
cd LSFG-Android-Application
./gradlew :app:assembleDebug         # 或 :app:assembleRelease
```

APK 输出位于
`LSFG-Android-Application/app/build/outputs/apk/debug/app-debug.apk`，
用 `adb install` 安装。

工具链：Android Studio Ladybug+、NDK 27.0.12077973、CMake 3.22.1、JDK 17、
C++20。ABI：`arm64-v8a`（正式）与 `x86_64`（仅模拟器）。
`minSdk=34`（Android 14），`targetSdk=35`（Android 15）。**系统要求 Android 14 及以上。**

CMake 通过 JNI 源码中的相对路径 `../../../../../lsfg-vk-android` 自动解析
子模块。请保持两个文件夹与本仓库布局一致地并排放置——移动或重命名任一文件夹
都会破坏 native 构建。

补丁版 `lsfg-vk` 的独立 Linux 构建见
[`lsfg-vk-android/README.md`](lsfg-vk-android/README.md)。Android 专属补丁在
非 Android 目标上均为空操作，上游构建命令原样可用。

## 平台限制（读一遍即可）

在未 root 的 Android 上**不存在 Linux Vulkan implicit layer 机制的等价物**。
Android 12+ 明确禁止向非调试进程加载外部代码，因此本应用无法钩住其他应用的
Vulkan 交换链。帧生成改为运行在 `MediaProjection` 屏幕捕获流上，结果合成在
目标之上的系统悬浮窗中。

相比 Linux Vulkan 层会增加约 50–80ms 延迟——这是平台限制，不是 bug。
`MediaProjection` 每次会话启动都需要用户明确授权，并会显示持久系统指示条。
`SYSTEM_ALERT_WINDOW` + 屏幕捕获 + `AccessibilityService` 的组合违反 Google
Play 政策，因此本应用只能以侧载 APK 形式分发。安装 Magisk 模块向
`/system/etc/vulkan/implicit_layer.d/` 放入 Vulkan implicit layer 是复刻
Linux 体验的唯一现实路径，但不在本项目范围内。

## 子目录级 README

- [`LSFG-Android-Application/README.md`](LSFG-Android-Application/README.md) ——
  app 架构、功能拆解、设备要求、native 模块布局。
- [`lsfg-vk-android/README.md`](lsfg-vk-android/README.md) —— framegen 库、
  Android 补丁集、以及相对上游 `lsfg-vk` 1.0.0 的精确差异。

## 致谢

本项目的存在离不开以下工作：

- **[PancakeTAS](https://github.com/PancakeTAS) 及 lsfg-vk 贡献者** ——
  原始 [`lsfg-vk`](https://github.com/PancakeTAS/lsfg-vk) Vulkan 帧生成层的作者，
  是整个移植工作的根基。
- **THS / Lossless Scaling** —— Lossless Scaling 帧生成 shader 的原作者。
  shader 由用户从自己合法购买的 `Lossless.dll` 副本在设备端提取，
  本项目从不分发。
- **[FrankBarretta](https://github.com/FrankBarretta)** —— Android 移植
  （上游仓库）：JNI/Vulkan 胶水层、基于 AHardwareBuffer 的图像共享、
  MediaProjection 捕获管线、悬浮窗/前台服务、无障碍悬浮窗触摸穿透、Compose UI、
  设置抽屉、按应用自动悬浮窗、可拖动启动圆点、首次启动教程、Shizuku 集成、
  帧率 HUD、崩溃报告、Vulkan 交换链输出，以及
  [`lsfg-vk-android`](lsfg-vk-android/) 子模块中 `#ifdef __ANDROID__` 下的
  上游补丁。

native 构建使用的第三方库：[`volk`](https://github.com/zeux/volk)、
[`pe-parse`](https://github.com/trailofbits/pe-parse)、DXVK 的 `dxbc`
转换器，以及用于特权计时旁路的 [Shizuku](https://github.com/RikkaApps/Shizuku)。

如果你 fork 本项目或在其之上构建，请保留上游 `lsfg-vk` 署名和 LSFG-Android
移植署名（下方许可证同样有此要求）。

## 许可证

本仓库的顶层文件以 **MIT License** 发布 —— 见 [`LICENSE`](LICENSE)。
两个主要子目录各自携带许可证，在其各自目录树内优先于根 MIT 许可证：

| 子目录 | 许可证 | 文件 |
|---|---|---|
| [`LSFG-Android-Application/`](LSFG-Android-Application/) | **Custom License — 禁止上架 Play 商店，禁止商业使用** | [`LSFG-Android-Application/LICENSE`](LSFG-Android-Application/LICENSE) |
| [`lsfg-vk-android/`](lsfg-vk-android/) | **MIT**（继承自上游 `lsfg-vk`） | [`lsfg-vk-android/LICENSE.md`](lsfg-vk-android/LICENSE.md) |

如果你整体再分发本仓库，请同时附上全部三份许可证文件，并遵守各子树最严格的
条款——特别地，LSFG-Android app 不得发布到 Google Play 或任何其他商业应用
商店，亦不得用于商业用途。

`Lossless.dll` 为 THS / Lossless Scaling 所有，本项目**绝不**分发。
