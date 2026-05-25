# RustDesk 工程解析（代码级）

本文基于当前仓库代码，对 RustDesk 的整体架构、支持功能、关键流程、编解码能力以及前端实现进行归纳，便于快速理解工程全貌。

## 1. 工程总体定位

RustDesk 是一个跨平台远程桌面系统，核心后端使用 Rust 实现，当前主前端为 Flutter，旧版桌面界面仍保留 Sciter 实现但已处于废弃状态。

从仓库结构看，工程大致分为 4 层：

- **核心应用层**：`/home/runner/work/rustdesk/rustdesk/src`
- **平台与基础能力层**：`/home/runner/work/rustdesk/rustdesk/src/platform`、`/home/runner/work/rustdesk/rustdesk/libs`
- **现代前端层**：`/home/runner/work/rustdesk/rustdesk/flutter`
- **旧版桌面前端层**：`/home/runner/work/rustdesk/rustdesk/src/ui`

对应入口和说明可参考：

- `/home/runner/work/rustdesk/rustdesk/README.md`
- `/home/runner/work/rustdesk/rustdesk/Cargo.toml`
- `/home/runner/work/rustdesk/rustdesk/AGENTS.md`

## 2. 主要模块分工

### 2.1 Rust 主程序层

- `src/server.rs`：本地服务端聚合层，统一管理音频、视频、剪贴板、输入等服务
- `src/server/connection.rs`：单连接生命周期、鉴权、权限、文件任务、端口转发、终端等
- `src/client.rs`：发起连接、处理远端会话、编解码接收、输入与剪贴板交互
- `src/rendezvous_mediator.rs`：与 rendezvous/relay 服务器交互，负责注册、打洞、维持在线状态
- `src/flutter.rs`、`src/flutter_ffi.rs`：Flutter 与 Rust 的桥接层

### 2.2 基础能力库

- `libs/hbb_common`：配置、协议、通用工具、网络封装
- `libs/scrap`：屏幕采集、视频编解码、平台采集实现
- `libs/enigo`：跨平台键鼠输入模拟
- `libs/clipboard`：跨平台剪贴板与文件复制粘贴

### 2.3 前端层

- `flutter/lib/desktop/pages`：桌面端页面
- `flutter/lib/mobile/pages`：移动端页面
- `flutter/lib/models`：状态模型、FFI 调用封装、会话数据模型
- `src/ui.rs` 与 `src/ui/*`：旧版 Sciter UI

## 3. 工程支持的核心功能

从页面目录、服务目录和连接对象字段综合看，当前工程支持的核心能力包括：

### 3.1 远程桌面控制

- 屏幕采集与推流：`src/server/video_service.rs`
- 显示服务管理：`src/server/display_service.rs`
- 鼠标/键盘/指针事件处理：`src/server/input_service.rs`
- 多显示器/窗口相关控制：`src/server.rs`、`src/server/connection.rs`

### 3.2 音频传输

- 音频采集与编码：`src/server/audio_service.rs`
- 客户端音频解码与播放入口：`src/client.rs`

### 3.3 剪贴板同步

- 剪贴板服务：`src/server/clipboard_service.rs`
- 平台剪贴板实现：`libs/clipboard`
- Linux 还包含文件复制粘贴能力开关：`Cargo.toml` 中 `unix-file-copy-paste`

### 3.4 文件传输

- 连接层维护读写文件任务：`src/server/connection.rs`
- 客户端文件能力入口：`src/client/file_trait.rs`
- Flutter 页面：`flutter/lib/desktop/pages/file_manager_page.dart`、`flutter/lib/mobile/pages/file_manager_page.dart`

### 3.5 远程终端

- 终端服务：`src/server/terminal_service.rs`
- 终端辅助：`src/server/terminal_helper.rs`
- Flutter 页面：`flutter/lib/desktop/pages/terminal_page.dart`、`flutter/lib/mobile/pages/terminal_page.dart`

### 3.6 端口转发

- 核心实现：`src/port_forward.rs`
- 连接对象中包含端口转发 socket 与地址：`src/server/connection.rs`
- Flutter 页面：`flutter/lib/desktop/pages/port_forward_page.dart`

### 3.7 摄像头查看

- `src/server/video_service.rs` 中 `VideoSource` 区分 `Monitor` 与 `Camera`
- Flutter 页面：`flutter/lib/desktop/pages/view_camera_page.dart`、`flutter/lib/mobile/pages/view_camera_page.dart`

### 3.8 服务器模式 / 被控端模式

- 桌面端页面：`flutter/lib/desktop/pages/server_page.dart`
- 移动端页面：`flutter/lib/mobile/pages/server_page.dart`
- rendezvous 与在线状态维护：`src/rendezvous_mediator.rs`

### 3.9 其他代码中可见的能力

- 2FA：`src/auth_2fa.rs`
- 隐私模式：`src/privacy_mode.rs`
- 局域网发现与 Wake-on-LAN：`src/lan.rs`
- 插件框架开关：`Cargo.toml` 中 `plugin_framework` 与 `src/plugin`
- Windows Flutter 打印服务：`src/server.rs` 中 `printer_service`

## 4. 核心运行流程

### 4.1 启动阶段

应用启动后，核心会创建本地服务集合。`src/server.rs` 中 `new()` 会把以下服务注册到统一的 `Server` 中：

- audio service
- display service
- clipboard service
- input service（cursor / pos / window focus）
- printer service（Windows + Flutter 场景）

这说明 RustDesk 的设计不是“单功能线程”，而是“一个连接中心 + 多服务订阅广播”的结构。

### 4.2 在线与打洞阶段

`src/rendezvous_mediator.rs` 的 `RendezvousMediator::start_all()` 负责：

- 启动在线注册
- 与 rendezvous 服务器保持通信
- 启动直连服务
- 触发 LAN 监听
- 在需要时进行配置同步与自动更新

这一层是整个系统的“控制面”，负责找到对端、决定是直连还是后续中继。

### 4.3 建立连接与加密握手

`src/server.rs` 的 `create_tcp_connection()` 体现了连接建立时的安全流程：

- 生成会话级公私钥
- 发送带签名的 `SignedId`
- 接收对端公钥材料
- 调用 `tcp::Encrypt::decode()` 建立加密流

这里能看出 RustDesk 并非明文传输，连接建立后会进入加密通道。

### 4.4 客户端发起会话

`src/client.rs` 中 `Client::start()` 是客户端启动远程会话的入口。它负责：

- 根据连接类型发起连接
- 处理直连、UDP/KCP、Relay 等通信路径
- 初始化会话对象
- 配合 UI 层进入后续的远程桌面、文件传输或终端流程

### 4.5 单连接生命周期

`src/server/connection.rs` 的 `Connection` 结构体字段非常丰富，说明单连接承担了大量业务状态，包括：

- 是否已授权
- 键盘、剪贴板、音频、文件权限
- 是否启用隐私模式
- 文件读写任务
- 端口转发 socket
- 语音通话状态
- 跟随远端光标/窗口
- 终端服务与持久化终端

因此它是 RustDesk 实际业务逻辑最重的对象之一。

### 4.6 媒体与输入数据流

一个典型远控会话的数据流大致如下：

1. rendezvous 建立可达路径
2. 客户端与被控端完成安全握手
3. 服务端开始采集显示器或摄像头画面
4. 视频服务进行编码并分发给订阅连接
5. 音频服务采集系统/输入音频并编码发送
6. 客户端解码画面与音频并渲染
7. 客户端输入事件再反向发送给被控端执行
8. 剪贴板、文件、终端等辅助通道并行运行

## 5. 支持的编解码能力

### 5.1 视频编解码

`libs/scrap/src/common/mod.rs` 中定义了视频编码格式：

- `VP8`
- `VP9`
- `AV1`
- `H264`
- `H265`

对应编码名称还区分了：

- `H264RAM(String)`
- `H265RAM(String)`
- `H264VRAM`
- `H265VRAM`

说明视频编解码并不只是一种实现，而是至少包含：

- **VP8 / VP9**：软件编码路径
- **AV1**：软件编码路径
- **H264 / H265**：硬编或平台加速路径
- **VRAM 路径**：直接走显存/GPU 相关能力

相关实现文件：

- `libs/scrap/src/common/codec.rs`：统一编码器/解码器调度
- `libs/scrap/src/common/vpxcodec.rs`：VP8/VP9
- `libs/scrap/src/common/aom.rs`：AV1
- `libs/scrap/src/common/hwcodec.rs`：H264/H265 硬编相关
- `libs/scrap/src/common/vram.rs`：VRAM/GPU 加速路径
- `libs/scrap/src/common/mediacodec.rs`：Android MediaCodec

### 5.2 视频采集与编码特性

`Cargo.toml` 中的视频相关 feature：

- `hwcodec`
- `vram`
- `mediacodec`

含义可以概括为：

- **hwcodec**：开启硬件视频编解码能力
- **vram**：启用基于显存/GPU 的路径
- **mediacodec**：Android 平台走 MediaCodec

同时 `src/server/video_service.rs` 直接依赖：

- `scrap::codec::Encoder`
- `AomEncoderConfig`
- `VpxEncoderConfig`
- `HwRamEncoder`
- `VRamEncoder`

说明视频服务自身就是编解码调度中心。

### 5.3 音频编解码

`src/server/audio_service.rs` 使用：

- `magnum_opus::Encoder`
- `Application::LowDelay`
- `Channels::Stereo`

这表明当前主音频编码是 **Opus**，并且偏实时低延迟。

其中：

- `AUDIO_DATA_SIZE_U8 = 960 * 4`，对应 48kHz 立体声短帧发送
- Linux/Android 使用 `pa_impl`
- 非 Linux/Android 使用 `cpal_impl`

### 5.4 音频重采样能力

`Cargo.toml` 中提供了 3 套音频重采样 feature：

- `use_dasp`（默认）
- `use_rubato`
- `use_samplerate`

因此音频链路不仅支持编码，还支持根据平台/编译配置切换不同重采样实现。

## 6. 前端实现分析

### 6.1 当前主前端：Flutter

Flutter 是当前主 UI。仓库中：

- `flutter/lib/main.dart`：前端总入口
- `flutter/lib/models/native_model.dart`：Dart 侧 FFI 封装
- `flutter/lib/models/platform_model.dart`：桥接入口聚合
- `flutter/lib/models/model.dart`：会话态 FFI 容器与子模型聚合

`flutter/lib/main.dart` 可以看出桌面端支持多窗口类型：

- RemoteDesktop
- FileTransfer
- ViewCamera
- PortForward
- Terminal

而移动端则走 `runMobileApp()`。

### 6.2 Flutter 与 Rust 的通信方式

Flutter 不是直接实现业务，而是通过 FFI 调 Rust 核心。

### Dart -> Rust

`flutter/lib/models/native_model.dart` 中 `PlatformFFI.init()` 会加载动态库：

- Linux / Android：`librustdesk.so`
- Windows：`librustdesk.dll`
- macOS：`DynamicLibrary.process()`

随后创建 `RustdeskImpl(dylib)`，并通过自动生成的 bridge 调用 Rust 函数。

### Rust -> Dart

`src/flutter_ffi.rs` 提供了主要桥接方法，例如：

- `start_global_event_stream`
- `session_add_sync`
- `session_start`

并定义了 Rust 发往 UI 的事件类型：

- `Event(String)`
- `Rgba(usize)`
- `Texture(usize, bool)`

这说明 Flutter 与 Rust 之间是“FFI 调用 + 事件流回推”的双向桥接。

### 6.3 Flutter 的页面组织

### 桌面端页面

目录：`flutter/lib/desktop/pages`

主要页面包括：

- `desktop_home_page.dart`
- `remote_page.dart`
- `file_manager_page.dart`
- `terminal_page.dart`
- `port_forward_page.dart`
- `view_camera_page.dart`
- `server_page.dart`
- `desktop_setting_page.dart`

### 移动端页面

目录：`flutter/lib/mobile/pages`

主要页面包括：

- `home_page.dart`
- `remote_page.dart`
- `file_manager_page.dart`
- `terminal_page.dart`
- `view_camera_page.dart`
- `server_page.dart`
- `settings_page.dart`
- `scan_page.dart`

### 6.4 Flutter 的状态模型

`flutter/lib/models/model.dart` 中的 `FFI` 类统一持有：

- `ImageModel`
- `FfiModel`
- `CursorModel`
- `CanvasModel`
- `ServerModel`
- `ChatModel`
- `FileModel`
- `UserModel`
- `QualityMonitorModel`
- `RecordingModel`
- `InputModel`
- `TextureModel`
- 多终端 `TerminalModel`

这说明 Flutter 层采用“会话容器 + 多模型”的组织方式，UI 主要负责展示与交互，核心能力仍由 Rust 提供。

### 6.5 旧版前端：Sciter

`src/ui.rs` 明确保留了 Sciter 界面启动逻辑，说明历史版本桌面端曾主要使用 Sciter。

相关文件：

- `src/ui.rs`
- `src/ui/index.html`
- `src/ui/index.tis`
- `src/ui/remote.tis`
- `src/ui/file_transfer.tis`

当前从仓库结构和 README 看，Sciter 已属于废弃路线，Flutter 才是主线。

## 7. 平台实现特点

从 `Cargo.toml` 与 `src/platform`、`libs/scrap` 可以看出项目强烈依赖平台差异化实现。

### Windows

- 支持打印服务、隐私模式、虚拟显示等扩展能力
- 输入与窗口能力更丰富

### Linux

- 同时兼顾 X11 / Wayland
- 音频走 PulseAudio 相关路径
- 存在 D-Bus、uinput、rdp_input 等模块

### macOS

- 有独立平台实现
- Flutter 动态库加载方式不同，避免重复实例化全局对象

### Android

- 支持 `mediacodec`
- Flutter 侧加载 `librustdesk.so`

## 8. 结论

RustDesk 不是单纯的“远程桌面播放器”，而是一个完整的远程协作与远程运维平台，其工程特点可以概括为：

1. **Rust 核心统一承载音视频、输入、文件、终端、网络安全逻辑**
2. **Flutter 作为主前端，通过 FFI 与 Rust 双向通信**
3. **视频编解码支持 VP8 / VP9 / AV1 / H264 / H265，且具备硬编、VRAM、MediaCodec 等路径**
4. **音频主编码为 Opus，并带有多套重采样方案**
5. **支持远程桌面、文件传输、端口转发、终端、摄像头查看、剪贴板、LAN 发现、2FA、隐私模式等完整能力**
6. **整个系统以 rendezvous + 连接层 + 多服务订阅广播的模式组织**

如果把工程理解为一条主链路，可以简化成：

**发现对端 -> 建立安全连接 -> 创建会话 -> 启动音视频/输入/剪贴板等服务 -> Flutter/Sciter 展示与交互**
