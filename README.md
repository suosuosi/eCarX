# 领克900车机 eCarX 方向盘控制功能+HUD歌曲信息显示接入指南

## 功能概述

本集成包为 App 添加以下车机功能：

| 功能 | 说明 |
|------|------|
| **方向盘控制** | 通过车机物理按键控制播放/暂停/上一曲/下一曲 |
| **歌曲信息显示** | 车机屏幕显示当前歌曲标题、歌手、专辑、封面图 |
| **歌词显示** | 车机屏幕显示当前歌词文本 |
| **播放进度同步** | 车机实时显示播放进度条 |
| **状态恢复** | 车机唤醒 App 时自动恢复播放 |

## 集成包内容

```
ecarx-integration-package/
├── AndroidManifest.xml          # 参考清单（含所有需要添加的声明）
└── smali/classes3/
    ├── com/kingxw/qsbridge/     # 桥接层（18 个文件）
    ├── com/ecarx/               # eCarX SDK（371 个文件）
    ├── ecarx/                   # eCarX AIDL 存根（155 个文件）
    └── com/bytedance/           # Lancet 日志框架（3 个文件，eCarX SDK 依赖）
```

### 桥接层文件说明

| 文件 | 作用 |
|------|------|
| `BridgeInit.smali` | ContentProvider 入口，应用启动时初始化桥接和弹窗 |
| `EcarxBridge.smali` | 核心桥接：初始化 eCarX API、连接 MediaBrowser、进度推送、恢复注册 |
| `EcarxBridge$1.smali` | eCarX 回调代理（onAPIReady） |
| `EcarxBridge$1$1.smali` | 主线程 Runnable |
| `EcarxBridge$2.smali` | 进度定时器（每秒更新播放位置） |
| `EcarxBridge$3.smali` | eCarX 恢复回调代理 |
| `EcarxBridge$4.smali` | MediaController.Callback（监听元数据/播放状态变化） |
| `EcarxBridge$5.smali` | MediaBrowser.ConnectionCallback（连接成功获取 Controller） |
| `EcarxBridge$6.smali` | 歌词获取完成回调 |
| `EcarxCallbackSink.smali` | 方向盘事件处理 |
| `EcarxCallbackSink$1.smali` | 按键分发 Runnable |
| `EcarxState.smali` | 状态容器（标题/歌手/专辑/封面/歌词/进度等） |
| `QsMusicClient.smali` | eCarX 音乐客户端实现 |
| `QsMusicPlaybackInfo.smali` | 播放信息封装（App 名称、封面、歌词等） |
| `QsMediaInfo.smali` | 媒体信息类，用于推送内容到车机 |
| `LyricFetcher.smali` | 歌词获取器（从网易云音乐 API） |
| `LyricFetcher$1.smali` | 歌词获取后台线程 |
| `RecoveryReceiver.smali` | 广播接收器，eCarX 唤醒时恢复播放 |

### Lancet 框架文件

eCarX SDK 被 Lancet 字节码插桩，必须包含以下三个类：

```
com/bytedance/auto/lite/lancet/LogLancet.smali
com/bytedance/auto/lite/serviceapi/config/log/Level.smali
com/bytedance/sysoptimizer/ReceiverRegisterCrashOptimizer.smali
```

缺少 `Level` 会导致：`NoClassDefFoundError: Failed resolution of: Lcom/bytedance/auto/lite/serviceapi/config/log/Level;`

## 集成步骤

### 第一步：反编译目标 APK

```powershell
java -jar libs/APKEditor.jar d -i apk/目标应用.apk -o decoded/目标应用/
```

### 第二步：确定 MediaBrowserService

功能需要通过 `MediaBrowser` 连接 App 的媒体服务来获取歌曲信息。在目标 APK 的 `AndroidManifest.xml` 中找到：

```xml
<service android:exported="true" ...>
  <intent-filter>
    <action android:name="android.media.browse.MediaBrowserService" />
  </intent-filter>
</service>
```

记下 `android:name` 的值（如 `com.example.MusicService`），后续需要修改桥接代码中的 Service 类名。

> 如果 App 没有 `MediaBrowserService`，则方向盘控制仍可工作，但歌曲信息、歌词、进度等 HUD 功能无法使用。

### 第三步：复制文件

```powershell
# 桥接层
robocopy ecarx-integration-package/smali/classes3/com/kingxw/qsbridge/ decoded/目标应用/smali/classes3/com/kingxw/qsbridge/ *.smali

# eCarX SDK（如目标 APK 尚未包含）
robocopy ecarx-integration-package/smali/classes3/com/ecarx/ decoded/目标应用/smali/classes3/com/ecarx/ /E
robocopy ecarx-integration-package/smali/classes3/ecarx/ decoded/目标应用/smali/classes3/ecarx/ /E

# Lancet 框架（如目标 APK 尚未包含）
robocopy ecarx-integration-package/smali/classes3/com/bytedance/ decoded/目标应用/smali/classes3/com/bytedance/ /E
```

> **单 DEX APK 注意事项：** 如果目标 APK 反编译后只有 `smali/` 没有 `smali/classes2`，则需要将上述文件放到 `smali/classes2/` 而不是 `smali/classes3/`。Android ART 加载 DEX 时需要编号连续，跳过 `classes2` 会导致 `classes3.dex` 被忽略。

### 第四步：修改桥接代码中的硬编码

根据目标 App 的信息，修改以下两处：

#### 4.1 Service 类名

打开 `EcarxBridge.smali`，找到 `connectMediaBrowser()` 方法：

```smali
const-string v3, "com.luna.player.impl.playerservice.MediaBrowserPlayerService"
```

改为第二步中记录的 MediaBrowserService 类名。例如：

```smali
const-string v3, "com.ryanheise.audioservice.AudioService"
```

#### 4.2 App 名称

打开 `QsMusicPlaybackInfo.smali`，找到 `getAppName()` 方法：

```smali
const-string v0, "\u6c7d\u6c34\u97f3\u4e50"    # "汽水音乐"
```

改为目标 App 的名称。例如：

```smali
const-string v0, "\u97f3\u6d41"                  # "音流"
```

> **Unicode 编码参考：** 可以使用 `native2ascii` 或在线工具将中文转为 `\uXXXX` 格式。

#### 4.3 Fallback 包名（可选）

打开 `EcarxState.smali`，找到 fallback 包名：

```smali
const-string v0, "com.tencent.wemeet.app"
```

改为目标 App 的包名：

```smali
const-string v0, "com.example.app"
```

### 第五步：修改 AndroidManifest.xml

在目标 APK 的 `AndroidManifest.xml` 中添加以下内容：

#### 权限

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="ecarx.oem.permission.OPENAPI_MEDIACENTER_PERMISSION" />
```

#### 包可见性声明（Android 11+ 必需）

```xml
<queries>
    <package android:name="com.ecarx.sdk.openapi" />
    <package android:name="com.ecarx.eas.carservice" />
</queries>
```

#### eCarX 凭证

```xml
<meta-data android:name="eCarX_OpenAPI_AppId" android:value="340394" />
<meta-data android:name="eCarX_OpenAPI_AppKey" android:value="qRmYCC8G2mTAWgwn215S" />
```

#### ContentProvider 入口

```xml
<provider
    android:name="com.kingxw.qsbridge.BridgeInit"
    android:authorities="${applicationId}.qsbridge"
    android:exported="false" />
```

#### 恢复广播接收器

```xml
<receiver android:name="com.kingxw.qsbridge.RecoveryReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="com.kingxw.qsbridge.RECOVER" />
    </intent-filter>
</receiver>
```

### 第六步：编译 APK

```powershell
java -jar libs/APKEditor.jar b -i decoded/目标应用/ -o output/目标应用.apk
```

### 第七步：签名

```powershell
java -jar libs/uber-apk-signer.jar --apks output/目标应用.apk --ks libs/debug.keystore --ksPass android --ksAlias androiddebugkey --ksKeyPass android --allowResign --overwrite
```

> 如果没有 `debug.keystore`，签名工具会自动生成，或使用正式签名证书。

## 验证方式

### logcat 调试

```bash
adb logcat -s QsBridge
```

关键日志标记：

| 日志 | 含义 |
|------|------|
| `eCarX init called` | 桥接初始化成功 |
| `onAPIReady=true` | eCarX 服务连接成功 |
| `registered token=...` | 注册音乐客户端成功 |
| `MediaBrowser connecting ...` | 开始连接 App 媒体服务 |
| `wheel onPlay/onPause/onNext/onPrevious` | 方向盘事件到达 |
| `connectMediaBrowser fail: ...` | 连接媒体服务失败，检查 Service 类名 |

### 模拟器测试（方向盘按键）

```bash
adb shell input keyevent 126   # 播放
adb shell input keyevent 127   # 暂停
adb shell input keyevent 87    # 下一曲
adb shell input keyevent 88    # 上一曲
```

### 按键代码对照

| 动作 | KeyEvent 代码 |
|------|--------------|
| 播放 | 126 (0x7E) |
| 暂停 | 127 (0x7F) |
| 下一曲 | 87 (0x57) |
| 上一曲 | 88 (0x58) |

## 注意事项

### 1. 包名无关性

方向盘控制不依赖具体包名。`EcarxBridge` 在注册时通过 `context.getPackageName()` 动态获取当前包名传给 eCarX 服务。

### 2. ContentProvider 初始化时机

`BridgeInit` 是一个 `ContentProvider`，在 `Application.onCreate()` 之前执行。`BridgeInit.onCreate()` 已包裹 `try/catch`，即使初始化失败也不会导致 App 崩溃。

### 3. MediaBrowserService 依赖

歌曲信息、歌词、进度显示等功能依赖 App 的 `MediaBrowserService`。如果 App 没有此服务，方向盘控制仍然可用，但 HUD 显示功能无法工作。

### 4. LyricFetcher 独立性

`LyricFetcher` 从网易云音乐 API（`music.163.com`）独立获取歌词，不依赖 App 自身。如果 App 已有歌词数据，可以修改 `LyricFetcher` 或替换为其他歌词源。

### 5. SourceType

桥接使用 `sourceType=6` 向 eCarX 注册。不同车机对 sourceType 的显示策略不同，如果车机不显示歌曲信息，可以尝试改为其他值。

### 6. 构建工具

使用 APKEditor.jar 构建，不要用 apktool（对 Flutter 等现代 APK 还原不完整）。

### 7. 仅新增文件

所有改动仅限于 AndroidManifest.xml 新增声明和复制 smali 文件，不修改目标 APK 的任何现有 smali。

5. **仅新增文件，不修改原 smali** — 所有改动仅限于 AndroidManifest.xml 新增声明和复制 smali 文件，不修改目标 APK 的任何现有 smali。
