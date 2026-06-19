# Orange Pi 4 Pro 官方工具集

> 来源：Orange Pi 官方资料下载页 | 共计 11 个工具

---

## 已打包工具（本目录）

| 工具 | 文件 | 大小 | 用途 | 平台 |
|------|------|------|------|------|
| **balenaEtcher** | *(见下方链接)* | — | Linux 镜像烧录（推荐） | Win/Mac/Linux/Arm64 |
| **Win32DiskImager** | `Win32DiskImager-1.0.0-binary.zip` | 17 MB | Linux 镜像烧录（备选） | Windows |
| **PhoenixCard** | `PhoenixCard4.2.8.zip` | 11 MB | Android 镜像烧录（唯一工具） | Windows |
| **VC++ 2008** | `vcredist_x86.exe` | 4.3 MB | PhoenixCard 前置依赖 | Windows |
| **SDCardFormatter** | `SDCardFormatterv5_WinEN.zip` | 6.2 MB | TF 卡格式化 | Windows |
| **MobaXterm** | `MobaXterm_Portable_v22.2.zip` | 29 MB | 串口调试 + SSH + SFTP | Windows |
| **FileZilla** | `FileZilla_3.62.2_win64_sponsored2-setup.exe` | 12 MB | SFTP 文件传输 | Windows |
| **NoMachine (arm64)** | `nomachine_8.5.3_1_arm64.deb` | 49 MB | 远程桌面（板上安装） | Arm64 Linux |
| **wiringOP 源码** | `wiringOP.tar.gz` | 1.3 MB | GPIO 库（C 语言，next 分支） | Arm64 Linux |
| **wiringOP-Python 源码** | `wiringOP-Python.tar.gz` | 1.1 MB | GPIO 库（Python 绑定，next 分支） | Arm64 Linux |
| **rootcheck.apk** | `rootcheck.apk` | 2.0 MB | Android ROOT 状态验证 | Android |
| **usbcamera.apk** | `usbcamera.apk` | 20 MB | USB 摄像头测试 APP | Android |

---

## 未打包工具（需单独下载）

| 工具 | 大小 | 原因 | 下载位置 |
|------|------|------|---------|
| **ai-sdk.tar.gz** | 1.3 GB | 超出 Git 限制 | [Orange Pi 官网](http://www.orangepi.cn) → Orange Pi 4 Pro → 官方工具 |
| **toolchains.tar.gz** | 2.0 GB | 超出 Git 限制 | orangepi-build 自动下载，或手动从 [清华镜像](https://mirrors.tuna.tsinghua.edu.cn/armbian-releases/_toolchain/) 下载 |
| **balenaEtcher** | ~150 MB | 多平台多文件 | [balena.io/etcher](https://www.balena.io/etcher/) |
| **NoMachine (Win/Mac/amd64)** | 54-78 MB | 仅打包 arm64 | [nomachine.com](https://www.nomachine.com/) |

---

## 工具使用流程图

```
拿到新板子
    │
    ├─ 烧录系统
    │   ├─ Linux 镜像 → balenaEtcher（推荐）或 Win32DiskImager
    │   └─ Android 镜像 → PhoenixCard 4.2.8（唯一选择！）
    │
    ├─ TF 卡格式化 → SDCardFormatter
    │
    ├─ 连接调试
    │   ├─ 串口 → MobaXterm（波特率 115200）
    │   ├─ SSH  → 系统自带 / MobaXterm / FileZilla
    │   └─ 远程桌面 → NoMachine（arm64 deb 装开发板上）
    │
    ├─ GPIO 开发
    │   ├─ C 语言 → wiringOP.tar.gz
    │   └─ Python → wiringOP-Python.tar.gz
    │
    └─ Android 开发
        ├─ ROOT 验证 → rootcheck.apk
        └─ USB 摄像头 → usbcamera.apk
```
