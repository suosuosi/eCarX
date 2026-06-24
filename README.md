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
