# 领克900车机 eCarX 方向盘控制功能接入指南

## 概述

将 eCarX SDK + qsbridge 桥接层注入目标 APK，使车机方向盘按键（播放/暂停/上一曲/下一曲）能够控制 App 的音乐播放。
	
功能通过 `AudioManager.dispatchMediaKeyEvent()` 发送媒体按键事件，Android 系统自动路由到当前媒体会话，App 侧无需额外处理。

## 包内容

```
ecarx-bridge-package/
├── AndroidManifest.xml       # 参考 manifest（已含 eCarX 相关声明）
└── smali/classes3/
    ├── com/kingxw/qsbridge/  # 桥接层（12 个文件）
    ├── com/ecarx/            # eCarX SDK（371 个文件）
    ├── ecarx/                # eCarX AIDL 存根（155 个文件）
    └── com/bytedance/        # Lancet 日志框架（3 个文件）
```

### 桥接层文件说明

| 文件 | 作用 |
|------|------|
| `BridgeInit.smali` | ContentProvider 入口，应用启动时初始化桥接 |
| `EcarxBridge.smali` | 桥接核心：反射加载 MediaCenterAPI、注册客户端 |
| `EcarxBridge$1.smali` | onAPIReady 回调代理 |
| `EcarxBridge$1$1.smali` | 注册到主线程的 Runnable |
| `EcarxCallbackSink.smali` | 方向盘事件处理（onPlay/onPause/onNext/onPrevious） |
| `EcarxCallbackSink$1.smali` | 按键分发 Runnable |
| `EcarxState.smali` | 播放状态枚举 |
| `QsMusicClient.smali` | eCarX 音乐客户端实现 |
| `QsMusicPlaybackInfo.smali` | 播放信息封装 |
| `Announce.smali` + `$1` + `$2` | 弹窗界面（桥接层未调用，可忽略） |

### Lancet 框架依赖

eCarX SDK 被 ByteDance Lancet 字节码插桩，以下三个类必须存在：

```
com/bytedance/auto/lite/lancet/LogLancet.smali
com/bytedance/auto/lite/serviceapi/config/log/Level.smali
com/bytedance/sysoptimizer/ReceiverRegisterCrashOptimizer.smali
```

缺少 `Level` 会导致 `NoClassDefFoundError`。

## 接入步骤

### 1. 反编译目标 APK

```powershell
java -jar libs/APKEditor.jar d -i app.apk -o decoded/
```

### 2. 复制 smali 文件

将 `ecarx-bridge-package/smali/classes3/` 下的所有内容复制到 `decoded/smali/classes3/`。

如果目标 APK 是单 dex（反编译后只有 `smali/` 没有 `smali/classes2`），需要将 `classes3` 重命名为 `classes2`：

```
decoded/smali/classes2/com/kingxw/qsbridge/
decoded/smali/classes2/com/ecarx/
decoded/smali/classes2/ecarx/
decoded/smali/classes2/com/bytedance/
```

Android 的 dex 加载要求编号连续，单 dex APK 跳过 `classes2` 直接使用 `classes3` 会导致 ART 忽略该 dex。

### 3. 修改 AndroidManifest.xml

在目标 APK 的 `AndroidManifest.xml` 中添加以下内容：

**权限：**

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="ecarx.oem.permission.OPENAPI_MEDIACENTER_PERMISSION" />
```

**包可见性声明（Android 11+）：**

```xml
<queries>
    <package android:name="com.ecarx.sdk.openapi" />
    <package android:name="com.ecarx.eas.carservice" />
</queries>
```

**eCarX 凭证：**

```xml
<meta-data android:name="eCarX_OpenAPI_AppId" android:value="340394" />
<meta-data android:name="eCarX_OpenAPI_AppKey" android:value="qRmYCC8G2mTAWgwn215S" />
```

**ContentProvider 入口：**

```xml
<provider
    android:name="com.kingxw.qsbridge.BridgeInit"
    android:authorities="${applicationId}.qsbridge"
    android:exported="false" />
```

### 4. 构建 APK

```powershell
java -jar libs/APKEditor.jar b -i decoded/ -o app_rebuilt.apk
```

### 5. 签名

```powershell
java -jar libs/uber-apk-signer.jar --apks app_rebuilt.apk --allowResign --overwrite
```

## 验证

### 日志调试

```bash
adb logcat -s QsBridge
```

关键日志标记：

| 日志 | 含义 |
|------|------|
| `eCarX init called` | 桥接初始化成功 |
| `onAPIReady=true` | eCarX 服务连接成功 |
| `registered token=...` | 注册音乐客户端成功 |
| `wheel onPlay/onPause/onNext/onPrevious` | 方向盘事件到达 |

### 按键代码对照

| 动作 | KeyEvent 代码 |
|------|--------------|
| 播放 | 126 (0x7E) |
| 暂停 | 127 (0x7F) |
| 下一曲 | 87 (0x57) |
| 上一曲 | 88 (0x58) |

模拟器测试：

```bash
adb shell input keyevent 126   # 播放
adb shell input keyevent 127   # 暂停
adb shell input keyevent 87    # 下一曲
adb shell input keyevent 88    # 上一曲
```

## 注意事项

1. **包名无关性** — 方向盘控制不依赖具体包名。`EcarxBridge` 在注册时通过 `context.getPackageName()` 动态获取当前包名传给 eCarX 服务。

2. **ContentProvider 初始化时机** — `BridgeInit` 是一个 `ContentProvider`，在 `Application.onCreate()` 之前执行。所有初始化代码必须轻量，不应在主线程发起网络请求。`BridgeInit.onCreate()` 已包裹 `try/catch`，即使初始化失败也不会导致 App 崩溃。

3. **Lancet 字节码插桩** — 从源 App 复制的 eCarX SDK 已被 Lancet 框架插桩。即使目标 APK 不依赖任何字节跳动 SDK，也必须带上三个 Lancet 框架类，否则初始化时会 `ClassNotFoundException`。

4. **构建工具** — 使用 APKEditor.jar 构建，不要用 apktool（对 Flutter 等现代 APK 还原不完整）。

5. **仅新增文件，不修改原 smali** — 所有改动仅限于 AndroidManifest.xml 新增声明和复制 smali 文件，不修改目标 APK 的任何现有 smali。
