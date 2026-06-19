# Orange Pi 4 Pro (A733) Agent Skill · 香橙派 4 Pro 智能体技能

<div align="center">

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Orange%20Pi%204%20Pro-orange)](https://www.orangepi.org)
[![SoC](https://img.shields.io/badge/SoC-Allwinner%20A733-red)](https://www.allwinnertech.com)
[![Claude%20Code](https://img.shields.io/badge/Claude%20Code-skill-6C4DF6)](https://claude.ai/code)

**Deploy Claude Code on Orange Pi 4 Pro — code, debug, and deploy directly on the edge**

**将 Claude Code 部署到香橙派 4 Pro — 在边缘设备上直接编码、调试和部署**

*Claude Code On-Device 板载开发 · GPIO 实时控制 · NPU 边缘推理 · 秒级开发迭代*

</div>

---

### 🚀 一键部署

```bash
# 在 Orange Pi 4 Pro 上执行（或任意 PC）
git clone https://github.com/HBConline/orangepi4pro-skill.git
mkdir -p ~/.agents/skills/orangepi4pro && cp -r orangepi4pro-skill/* ~/.agents/skills/orangepi4pro/
mkdir -p ~/.claude/skills && ln -sf ~/.agents/skills/orangepi4pro ~/.claude/skills/orangepi4pro
```

> 安装后直接在 Claude Code 中说需求：`/orangepi4pro 写一个 GPIO 控制 LED 程序`

---

## 📋 Overview · 概述

> **EN:** Deploy Claude Code directly on the Orange Pi 4 Pro and turn it into an **AI-powered edge development workstation**. This skill gives Claude Code intimate knowledge of every pin, driver, command, and hardware quirk — so you can develop GPIO apps, deploy NPU models, configure Linux services, and debug hardware **in seconds** with natural language. No more context-switching between datasheets, manuals, and terminals.
>
> **中文：** 将 Claude Code 直接部署到香橙派 4 Pro 上，将其变为 **AI 驱动的边缘开发工作站**。此技能让 Claude Code 深度掌握每个引脚、驱动、命令和硬件特性——你可以用自然语言**在数秒内**完成 GPIO 应用开发、NPU 模型部署、Linux 服务配置和硬件调试。不再需要反复查阅数据手册、用户手册和终端命令。

### 🎯 Core Use Case · 核心场景

> 💬 **你（自然语言）：**
> *"写一个边缘 AI 安防系统：MIPI 摄像头 + NPU 行人检测，检测到人时触发蜂鸣器警报、MQTT 推送到手机，加 Flask Web 实时监控，systemd 开机自启"*

---

#### Step 1 ── MIPI 摄像头驱动加载

**Claude 做的事：** 从 skill 获取 OV13850/IMX219 摄像头信息 — 知道驱动模块是 `vin_v4l2`、设备节点是 `/dev/video8`、依赖 `python3-opencv` 和 `libopencv-dev`。

```bash
# Claude 在板上直接执行
sudo modprobe vin_v4l2          # 加载 MIPI 摄像头驱动
ls /dev/video*                   # 确认 /dev/video8 出现
sudo apt install -y python3-opencv libopencv-dev python3-pybind11
```

| 技能提供的知识 | 来源 |
|--------------|------|
| MIPI-CSI 是 4-lane 接口 | SKILL.md §1.4 硬件规格 |
| 驱动模块名 `vin_v4l2`，设备节点 `/dev/video8` | SKILL.md §3.29 |
| OV13850 13MP / IMX219 8MP 两个型号都支持 | SKILL.md §3.29 |

---

#### Step 2 ── C++ NPU 推理引擎生成

**Claude 做的事：** 从 skill 获取 NPU 的完整工具链 — Docker 镜像版本 `ubuntu-npu:v2.0.10`、Pegasus 四步转换管线、7 个推理示例中的 YOLOv5s。直接在板上生成 `CMakeLists.txt` 和 C++ 源码，然后板载编译。

```cpp
// ai_guard.cpp — Claude 自动生成的核心推理代码
#include <opencv2/opencv.hpp>
#include "v4l2_camera.h"    // MIPI 摄像头封装 (来自 /opt/v4l2_opencv_demo)
#include "rknn_api.h"        // NPU 推理 API

int main() {
    // 1. 打开 MIPI 摄像头
    V4L2Camera cam("/dev/video8", 640, 480);

    // 2. 加载 YOLOv5s NBG 模型
    rknn_context ctx;
    rknn_init(&ctx, "yolov5s.nb", 0, 0, NULL);

    // 3. 初始化 GPIO (蜂鸣器)
    wiringPiSetup();
    pinMode(2, OUTPUT);       // Pin 7 = wPi 2

    // 4. MQTT 客户端
    mosquitto_lib_init();
    struct mosquitto *mqtt = mosquitto_new("ai-guard", true, NULL);
    mosquitto_connect(mqtt, "192.168.1.100", 1883, 60);

    while (true) {
        cv::Mat frame = cam.capture();
        // ... NPU 推理 → 检测 person 类别 ...
        if (person_detected) {
            digitalWrite(2, HIGH);  // 蜂鸣器响
            // MQTT 推送 JSON
            mosquitto_publish(mqtt, NULL, "home/camera/alert",
                strlen(json), json, 0, false);
            sleep(5);
            digitalWrite(2, LOW);   // 蜂鸣器停
        }
    }
}
```

```bash
# Claude 在板上编译（利用板载 aarch64 GCC）
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4                        # A733 八核并行编译
```

| 技能提供的知识 | 来源 |
|--------------|------|
| NPU 3 TOPS，Pegasus 量化管线 (`import → quantize → inference → export`) | §3.35.1 |
| YOLOv5s 推理示例路径 `ai-sdk/examples/yolov5` | §3.35.2 |
| COCO 80 类标签中 `person = 0` | §8.4（示例代码中的 CLASSES 数组） |
| wiringPi 初始化 `wiringPiSetup()` | §3.17 |

---

#### Step 3 ── GPIO 警报 + MQTT 消息推送

**Claude 做的事：** 知道 GPIO Pin 7 对应 wPi 编号 2、电压 3.3V；生成 MQTT 连接代码；格式化为 Home Assistant 可直接消费的 JSON。

```
gpio mode 2 out                  # wPi 2 = Pin 7，设为输出
gpio write 2 1                   # 高电平 → 有源蜂鸣器响 (3.3V)
gpio write 2 0                   # 低电平 → 蜂鸣器停
```

```json
{
  "device": "ai-guard",
  "event": "person_detected",
  "confidence": 0.87,
  "bbox": [120, 45, 380, 520],
  "timestamp": "2026-01-09T14:32:15",
  "image": "/tmp/alert_20260109_143215.jpg"
}
```

```bash
# Claude 自动安装 Python MQTT 库（备用 Python 方案也有）
pip3 install paho-mqtt
```

| 技能提供的知识 | 来源 |
|--------------|------|
| Pin 7 = wPi 2，3.3V 电平（⚠ 不能接 5V） | §3.15, §3.17.1 |
| MQTT Broker 连接字符串模板 | §8.6（Docker Compose 示例） |
| Home Assistant 可以消费这个 JSON → 手机推送 | §3.23（Home Assistant 安装） |

---

#### Step 4 ── Flask Web 实时监控面板

**Claude 做的事：** 生成一个带图标的 Web 页面 — 采集摄像头帧 → JPEG 编码 → 浏览器 MJPEG 流展示 → 同时显示告警历史表格。

```python
# web_monitor.py — Claude 自动生成
from flask import Flask, Response, render_template_string
import cv2, time, json
from collections import deque

app = Flask(__name__)
alerts = deque(maxlen=50)  # 最近 50 条告警
cam = cv2.VideoCapture(0)  # 或 V4L2Camera("/dev/video8", 640, 480)

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE, alerts=list(alerts))

@app.route('/stream')
def stream():
    def generate():
        while True:
            ret, frame = cam.read()
            if ret:
                _, jpg = cv2.imencode('.jpg', frame, [cv2.IMWRITE_JPEG_QUALITY, 70])
                yield (b'--frame\r\nContent-Type: image/jpeg\r\n\r\n' + jpg.tobytes() + b'\r\n')
            time.sleep(0.05)
    return Response(generate(), mimetype='multipart/x-mixed-replace; boundary=frame')

# 访问 http://<开发板IP>:8080 — 实时画面 + 告警日志
```

| 技能提供的知识 | 来源 |
|--------------|------|
| Flask Web 服务在板子上运行，端口 8080 | §8.5（Web 控制 GPIO 示例） |
| OpenCV `cv2.VideoCapture` 打开摄像头 | §3.24（OpenCV 安装和版本） |
| JPEG MJPEG 流推送方案 | §3.12.4（mjpg-streamer 参考） |

---

#### Step 5 ── systemd 开机自启服务

**Claude 做的事：** 创建 systemd unit 文件，配置 `multi-user.target`、自动重启、硬件权限 — 让安防系统上电即运行。

```ini
# /etc/systemd/system/ai-guard.service — Claude 自动生成并写入
[Unit]
Description=Edge AI Security Guard
After=network.target

[Service]
Type=simple
User=orangepi
WorkingDirectory=/home/orangepi/ai-guard
ExecStart=/home/orangepi/ai-guard/build/ai_guard
Restart=always
RestartSec=5
# 硬件访问权限
DeviceAllow=/dev/video8 rw
DeviceAllow=/dev/gpiomem rw
DeviceAllow=/dev/mem rw
AmbientCapabilities=CAP_SYS_RAWIO

[Install]
WantedBy=multi-user.target
```

```bash
# Claude 在板上执行
sudo cp ai-guard.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable ai-guard     # 开机自启
sudo systemctl start ai-guard       # 立即启动
sudo systemctl status ai-guard      # 验证运行状态
```

| 技能提供的知识 | 来源 |
|--------------|------|
| systemd unit 模板，含 `DeviceAllow` 硬件权限 | §8.7 |
| `AmbientCapabilities=CAP_SYS_RAWIO` 让普通用户访问 GPIO | §8.7 |
| `orangepi` 用户权限配置 | §3.4.1（默认用户名） |

---

#### 完整架构流程

```
                       ┌────────────────────────────┐
                       │  👤 你（中文自然语言）      │
                       │  "写一个边缘 AI 安防系统..." │
                       └─────────────┬──────────────┘
                                     │
               ┌─────────────────────┼─────────────────────┐
               │  🟠 Orange Pi 4 Pro (Debian / Ubuntu)     │
               │  89×56mm · A733 2.0GHz · 3 TOPS NPU      │
               │──────────────────────────────────────────│
               │  🤖 Claude Code  ←  orangepi4pro skill    │
               │  获取：引脚映射 · 驱动状态 · NPU 管线     │
               │                                          │
               │  ┌────────────────────────────────────┐  │
               │  │  ①  MIPI 摄像头驱动                 │  │
               │  │  ───────────────────               │  │
               │  │  # modprobe vin_v4l2                │  │
               │  │  # ls /dev/video8      确认设备     │  │
               │  │  # apt install python3-opencv \     │  │
               │  │    libopencv-dev python3-pybind11   │  │
               │  │  📡 4-lane MIPI-CSI → 就绪          │  │
               │  └─────────────────┬──────────────────┘  │
               │                    │                     │
               │  ┌─────────────────▼──────────────────┐  │
               │  │  ②  C++ NPU 推理引擎（板载编译）    │  │
               │  │  ─────────────────────────────      │  │
               │  │  生成：ai_guard.cpp + CMakeLists    │  │
               │  │  ┌────────────────────────────┐    │  │
               │  │  │ V4L2Camera cam("/dev/       │    │  │
               │  │  │   video8", 640, 480);       │    │  │
               │  │  │ rknn_init(&ctx,              │    │  │
               │  │  │   "yolov5s.nb", ...);        │    │  │
               │  │  │ if (person) {                │    │  │
               │  │  │   digitalWrite(2, HIGH);     │    │  │
               │  │  │   mqtt_publish(json);        │    │  │
               │  │  │ }                            │    │  │
               │  │  └────────────────────────────┘    │  │
               │  │  # cmake .. && make -j4             │  │
               │  └─────────────────┬──────────────────┘  │
               │                    │                     │
               │     ┌──────────────┼──────────────┐      │
               │     │              │              │      │
               │  ┌──▼──────┐ ┌────▼─────┐ ┌──────▼──┐  │
               │  │ ③a GPIO  │ │ ③b MQTT  │ │ ③c Web  │  │
               │  │ ──────── │ │ ──────── │ │ ─────── │  │
               │  │ Pin7     │ │ topic:   │ │ Flask   │  │
               │  │ wPi2 = 1 │ │ home/    │ │ :8080   │  │
               │  │ 蜂鸣器🔊 │ │ camera/  │ │ MJPEG流 │  │
               │  │          │ │ alert    │ │         │  │
               │  └─────────┘ └──────────┘ └─────────┘  │
               │                                          │
               │  ┌────────────────────────────────────┐  │
               │  │  ④  Flask Web 监控页面              │  │
               │  │  ───────────────────               │  │
               │  │  /      → MJPEG 实时画面            │  │
               │  │  /api   → 告警历史 JSON             │  │
               │  │  http://&lt;ip&gt;:8080 ← 手机/PC    │  │
               │  └────────────────────────────────────┘  │
               │                                          │
               │  ┌────────────────────────────────────┐  │
               │  │  ⑤  systemd 开机自启               │  │
               │  │  ───────────────────               │  │
               │  │  [Service]                          │  │
               │  │  ExecStart=.../ai_guard             │  │
               │  │  Restart=always  RestartSec=5       │  │
               │  │  DeviceAllow=/dev/video8 rw         │  │
               │  │  ───────────────────               │  │
               │  │  # systemctl enable --now ai-guard  │  │
               │  │  # systemctl status ai-guard ● 运行 │  │
               │  └────────────────────────────────────┘  │
               │                                          │
               │    ✅ 10 分钟：自然语言 → 系统上线       │
               │                                          │
               └───────────┬───────────┬──────────────────┘
                           │           │
         ┌─────────────────┘           └─────────────────┐
         │                                               │
 ┌───────▼───────┐   ┌───────────┐   ┌──────────────────▼──────────────┐
 │  MIPI Camera  │   │  40-Pin   │   │  Wi-Fi 6 + Ethernet             │
 │  ──────────── │   │  GPIO     │   │  ─────────────────────────────  │
 │  OV13850/219  │   │  ──────── │   │  wlan0 · end0 · nmcli           │
 │  /dev/video8  │   │  Pin  7   │   │                                 │
 │  4-lane CSI   │   │  Pin 11   │   │  ▶ MQTT Broker :1883           │
 │               │   │  Pin 13   │   │  ▶ Flask Web :8080             │
 │               │   │  Pin 15   │   │  ▶ Home Assistant :8123        │
 └───────┬───────┘   └─────┬─────┘   └─────────────────┬───────────────┘
         │                 │                           │
 ┌───────▼───────┐   ┌─────▼─────┐   ┌─────────────────▼──────────────┐
 │  3 TOPS NPU   │   │  Buzzer   │   │  📱 + 🖥️  最终输出               │
 │  ──────────── │   │  蜂鸣器    │   │  ─────────────────────────────  │
 │  YOLOv5s      │   │  ──────── │   │  📱 Home Assistant 自动化推送  │
 │  Pegasus→NBG  │   │  3.3V 有源│   │  🖥️ Flask Web :8080 实时画面   │
 │               │   │  H=响 L=停│   │  📊 MQTT JSON 第三方集成       │
 └───────────────┘   └───────────┘   └────────────────────────────────┘
```

> ⚡ **10 分钟，一次对话：** Claude Code 自动查阅原理图引脚映射 → 加载内核驱动 → 生成 C++/Python 源码 → 板载编译 → 配置 Web 服务 → 写入 systemd 单元 → 系统上线。**不翻阅手册、不搜索引脚、不调试驱动。**

### Why This Skill? · 为什么需要这个技能？

| Without Skill · 无技能 | With Skill · 有技能 |
|---|---|
| ❌ Agent hallucinates GPIO pinouts · 幻觉 GPIO 引脚 | ✅ Exact pin mappings with wPi numbers · 精确引脚映射 |
| ❌ Guesses Linux kernel driver status · 猜测驱动状态 | ✅ Complete driver compatibility matrix · 完整驱动兼容矩阵 |
| ❌ Unknown NPU deployment steps · 不知 NPU 部署流程 | ✅ Full RKNN model pipeline (7 models) · 完整 7 模型管线 |
| ❌ Incorrect device tree overlay names · 错误的 dtbo 名 | ✅ Verified dtbo configs for every interface · 验证过的配置 |
| ❌ No awareness of board-specific quirks · 不了解硬件坑 | ✅ Known issues: SSD compat, PD limitations · 已知兼容问题 |

---

## 🧠 Knowledge Domains · 知识领域

### 🖥️ Hardware Engineering · 硬件工程

#### Processor · 处理器

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **SoC** | Allwinner A733 全志 A733 |
| **CPU Architecture** | 2×Arm Cortex-A76 + 6×Arm Cortex-A55 (big.LITTLE) |
| **Max Clock** | 2.0 GHz |
| **Process Node** | 12nm |
| **Co-processor** | RISC-V E902 @ 200MHz |
| **GPU** | Imagination BXM-4-64 |
| **GPU API** | OpenGL ES 3.2, Vulkan 1.3, OpenCL 3.0 |
| **NPU** | 3 TOPS INT8, dedicated AI accelerator |
| **NPU Framework** | RKNN SDK v2.0.x (Pegasus quantization → NBG export) |
| **Video Decode** | 8K@24fps H.265/VP9/AVS2 |
| **Video Encode** | 4K@30fps H.265/H.264 |
| **Package** | 全志 A733, 12nm, 2.0GHz 主频 |

#### Memory & Storage · 内存与存储

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **RAM Type** | LPDDR5 |
| **Max RAM** | Up to 16GB 最高支持 16GB |
| **eMMC (Optional)** | 16GB / 32GB / 64GB / 128GB socket module · 可选焊接模块 |
| **SPI Flash** | 128Mbit default, 256Mbit option · 128Mb 默认，256Mb 可选 |
| **M.2 Socket** | M-Key, PCIe 3.0 x1, NVMe SSD · M.2 M-Key, PCIe 3.0 x1 |
| **uSD Card Slot** | Supports up to 128GB, Class 10+ · 最大支持 128GB |
| **NVMe SSD Compat.** | ✅ Fancun 樊想 / Guancun 广存 · ⚠️ Kingston (v1.0.6+) |
| **Boot Priority** | TF card > eMMC > SPIFlash+NVMe · TF 卡优先 |

#### Connectivity · 无线连接

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **Wi-Fi** | Wi-Fi 6 (802.11ax), 2.4GHz / 5GHz dual-band |
| **Bluetooth** | BT 5.4 with BLE (Bluetooth Low Energy) |
| **Module** | Combo module (Wi-Fi+BT integrated) |
| **Antenna** | Onboard + external antenna option |

#### Ethernet · 以太网

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **Speed** | 10/100/1000 Mbps Gigabit |
| **PHY Chip** | Motorcomm YT8531CA (板载) |
| **PoE** | Supported via PoE HAT · 支持 PoE 供电 |
| **USB Ethernet** | RTL8152B (100M) / RTL8153 (1G) tested compatible |

#### Display · 显示输出

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **HDMI** | 1×HDMI TX 2.0, up to 4K@60fps · 最高 4K@60fps |
| **HDMI CEC** | Supported |
| **MIPI DSI** | 1×4-lane MIPI-DSI (up to 1920×1200) |
| **LCD** | 10.1" MIPI LCD kit (1280×800, with touch panel) |
| **Dual Display** | HDMI + MIPI DSI simultaneous · 可同时工作 |
| **HDMI→VGA** | Via external converter, plug-and-play · 转换器免驱 |

#### Camera · 摄像头

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **MIPI CSI-1** | 2-lane MIPI-CSI (CAM1) |
| **MIPI CSI-2** | 4-lane MIPI-CSI (CAM2) |
| **Supported Sensors** | OV13850 (13MP MIPI) / IMX219 (8MP MIPI) |
| **CAM1 Status** | Driver OK, 3A auto-tuning not adapted |
| **CAM2 Status** | Driver OK, 3A auto-tuning not adapted (photo distorted, video OK on Android) |
| **USB Camera** | UVC protocol supported, tested with fswebcam + mjpg-streamer |
| **MIPI Device Node** | `/dev/video8` (after `modprobe vin_v4l2`) |

#### USB · USB 接口

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **USB 3.0** | 1×USB Type-A 3.0, 5Gbps |
| **USB 2.0** | 3×USB Type-A 2.0 Host |
| **USB OTG** | Blue USB port — Host/Device switchable (for ADB flashing) |
| **USB Power** | Type-C 5V DC IN — No PD negotiation · 不支持 PD 协商 |

#### Audio · 音频

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **Codec** | ES8323 |
| **Headphone Jack** | 3.5mm, audio input + output · 3.5mm 耳机，支持音频输入输出 |
| **Speaker** | Single onboard speaker output · 单声道板载喇叭 |
| **Microphone** | Single onboard MIC · 单声道板载麦克风 |
| **HDMI Audio** | Via allwinnerhdmi card0 · 通过 HDMI 输出 |
| **Headphone Audio** | Via sndi2s4 card1 (I2S) |
| **Android Audio** | ❌ Headphone/MIC/Speaker not supported on Android 13 |

#### GPIO & Expansion · 扩展接口

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **40-Pin Header** | Raspberry Pi compatible layout · 兼容树莓派布局 |
| **GPIO Count** | 28 GPIO pins · 28 个 GPIO 引脚 |
| **GPIO Voltage** | 3.3V (NOT 5V tolerant!) · 不耐 5V！ |
| **I2C** | 4 buses: I2C0 / I2C1 / I2C2 / I2C3 |
| **SPI** | SPI3 (CS0, MOSI, MISO, CLK) |
| **UART** | UART7 (default); UART8 on Android · 默认 UART7 |
| **PWM** | 8 channels: SPWM0-2 + PWM0-0/2/3/4/5/6/7 |
| **Power Pins** | 3.3V (Pin 1, 17) / 5V (Pin 2, 4) / GND (multiple) |
| **Enable Method** | `sudo orangepi-config` → System → Hardware → DT overlay |
| **C Library** | wiringOP (next branch, wPi numbering) |
| **Python Library** | wiringOP-Python (next branch) |
| **Android** | Preinstalled wiringOP APK with GUI · 预装 wiringOP APK |

#### Debug & Buttons · 调试与按键

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **Debug UART** | 3-pin header, 3.3V TTL, 115200 baud · 3Pin 调试串口 |
| **BOOT Button** | 1×, for maskrom/firmware recovery mode |
| **RESET Button** | 1×, hardware reset |
| **PWR ON Button** | 1×, short press = shutdown, long press = power on |
| **LED Indicators** | Red (power, hardware-controlled) + Green (status, software-controlled) |

#### Physical · 物理参数

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **PCB Size** | 89mm × 56mm |
| **Weight** | 58g |
| **Mounting Holes** | 4×, diameter 3.0mm |
| **PCB Layers** | 6-layer PCB |
| **Operating Temp** | 0°C ~ 70°C (recommended with heatsink) |

#### Power · 电源

| Parameter · 参数 | Specification · 规格 |
|---|---|
| **Input** | USB Type-C, 5V DC only · 仅 5V |
| **Current** | 3A minimum recommended, 4A optional · 推荐 3A，可选 4A |
| **PD Support** | ❌ NO PD negotiation · 不支持 PD 协商！ |
| **⚠️ Warning** | **DO NOT use >5V adapter — WILL burn the board!** · 严禁使用大于 5V 电源！ |
| **Alternative Power** | 40-pin 5V + GND pins (backup method) |
| **Power Stability** | Most boot failures/reboots caused by insufficient PSU · 启动不稳多为电源问题 |
| **PoE** | Supported via PoE HAT · 支持 PoE 供电模块 |

#### Onboard Sensors · 板载传感器

| Parameter · 参数 | Path · 路径 |
|---|---|
| **CPU_L Temp** | `/sys/class/thermal/thermal_zone0/temp` (cpul_thermal_zone) |
| **CPU_B Temp** | `/sys/class/thermal/thermal_zone1/temp` (cpub_thermal_zone) |
| **GPU Temp** | `/sys/class/thermal/thermal_zone4/temp` (gpu_thermal_zone) |
| **NPU Temp** | `/sys/class/thermal/thermal_zone5/temp` (npu_thermal_zone) |
| **DDR Temp** | `/sys/class/thermal/thermal_zone6/temp` (ddr_thermal_zone) |
| **Watchdog** | Hardware watchdog, preinstalled `watchdog_test` |
| **ChipID** | `/sys/class/sunxi_info/sys_info` → sunxi_serial (unique per chip) |

#### OS Support · 系统支持

| OS · 系统 | Kernel · 内核 | Desktop · 桌面 | Status |
|---|---|---|---|
| Ubuntu 22.04 Jammy | Linux 5.15 | XFCE / GNOME | ✅ |
| Debian 12 Bookworm | Linux 5.15 | XFCE / GNOME | ✅ |
| Debian 11 Bullseye | Linux 5.15 | XFCE | ✅ (GPU + HW codec exclusive) |
| Android 13 | Linux 5.15 | AOSP | ✅ (no headphone/MIC/speaker) |

#### GPU & Video Codec Availability · GPU 和硬编解码支持

| Feature · 功能 | Ubuntu | Debian 12 | Debian 11 | Android 13 |
|---|---|---|---|---|
| **GPU Acceleration** | ❌ | ❌ | ✅ | ✅ |
| **HW Video Decode** | ❌ | ❌ | ✅ (Gstreamer omxh264dec) | ✅ |
| **HW Video Encode** | ❌ | ❌ | ✅ (H.265/H.264 4K@30fps) | ✅ |
| **NPU** | ✅ | ✅ | ✅ | ✅ |

### ⚡ 40-Pin GPIO Ecosystem · 40Pin 引脚生态

> **EN:** 28 GPIO pins at 3.3V · wiringOP (C) with pull-up/pull-down control · wiringOP-Python with full SPI/I2C/UART/PWM bindings · SPI3 · I2C0~I2C3 · UART7 · 8 PWM channels · `orangepi-config` device tree overlay management
>
> **中文：** 28 个 3.3V GPIO 引脚 · wiringOP (C 语言) 支持上下拉电阻控制 · wiringOP-Python 完整 SPI/I2C/UART/PWM 绑定 · SPI3 · I2C0~I2C3 · UART7 · 8 路 PWM · `orangepi-config` 设备树叠加管理

### 🐧 Embedded Linux · 嵌入式 Linux

> **EN:** Ubuntu 22.04 Jammy / Debian 12 Bookworm / Debian 11 Bullseye (Linux 5.15) · 25+ driver compatibility matrix · Auto-login · TF card rootfs expansion (auto/manual/shrink) · nmcli/nmtui static IP · SSH · ADB · create_ap WiFi hotspot · Bluetooth (bluez/bluetoothctl) · First-boot auto-config · HDMI/VGA · ES8323 audio · OV13850/IMX219 MIPI cameras · 10.1" MIPI LCD · Docker · Home Assistant · OpenCV · BT Panel · QT 5.15 · ROS2 Humble · Kernel modules
>
> **中文：** Ubuntu 22.04 Jammy / Debian 12 Bookworm / Debian 11 Bullseye（均 Linux 5.15）· 25+ 项驱动兼容矩阵 · 自动登录 · TF 卡 rootfs 扩容（自动/手动/缩小）· nmcli/nmtui 静态 IP · SSH · ADB · create_ap WiFi 热点 · 蓝牙 (bluez/bluetoothctl) · 首次启动自动联网 · HDMI/VGA · ES8323 音频 · OV13850/IMX219 MIPI 摄像头 · 10.1寸 MIPI LCD · Docker · Home Assistant · OpenCV · 宝塔面板 · QT 5.15 · ROS2 Humble · 内核模块

### 🧪 Edge AI — NPU · 边缘 AI 推理

> **EN:** Docker-based RKNN SDK (v2.0.10) on PC · Model import → quantization (uint8) → inference → NBG export · 7 on-device inference examples: YOLOv5s (object detection) · ResNet50 (classification) · Struct2Depth (depth estimation) · Transformer CLS · YOLACT (detection + segmentation) · DeepSpeech2 (speech recognition) · ChineseOCR (text recognition)
>
> **中文：** PC 端 Docker RKNN SDK (v2.0.10) · 模型导入 → uint8 量化 → 推理 → NBG 导出 · 7 个板端推理示例：YOLOv5s 目标检测 · ResNet50 图像分类 · Struct2Depth 深度估计 · Transformer CLS · YOLACT 检测+分割 · DeepSpeech2 语音识别 · ChineseOCR 文字识别

### 🤖 Android 13 (AOSP)

> **EN:** 25+ subsystem feature matrix (with known limitations: no headphone audio/MIC/speaker) · USB OTG Host↔Device mode switching · ADB via USB cable + Network (port 5555) · wiringOP Android App for GPIO/UART/SPI/I2C/PWM · Longan (bootloader+kernel) + AOSP compilation · 200GB+ disk / 16GB+ RAM
>
> **中文：** 25+ 项功能适配矩阵（已知限制：不支持耳机/MIC/喇叭）· USB OTG Host↔Device 模式切换 · ADB 数据线 + 网络 (5555 端口) · wiringOP Android App 测试 GPIO/UART/SPI/I2C/PWM · Longan (bootloader+kernel) + AOSP 编译 · 200GB+ 磁盘 / 16GB+ 内存

### 🔧 Linux SDK · 系统构建

> **EN:** orangepi-build (armbian-based) · 12 cross-compilation toolchains (auto-downloaded from Tsinghua mirror) · Build pipeline: u-boot v2018.05 → kernel 5.15.147 → rootfs → full `.img` · Gitee mirror support (`GITEE_SERVER=yes`) · Repeatable one-liner build commands
>
> **中文：** orangepi-build（基于 armbian）· 12 个交叉编译工具链（从清华镜像自动下载）· 构建流程：u-boot v2018.05 → 内核 5.15.147 → rootfs → 完整 `.img` · 支持 Gitee 国内镜像 (`GITEE_SERVER=yes`) · 可复现的一行构建命令

### 📐 Hardware Schematics · 硬件原理图与 PCB

> **EN:** Complete board-level design files included — full schematic PDF (SoC pinout, power tree, all peripherals) + TOP & BOT DXF PCB layouts (PCB V1.3.2, 2026-01-09) · 4 files · 4.5 MB · Enables circuit tracing, enclosure design, PCB re-spin, repair
>
> **中文：** 包含完整板级设计文件——全电路原理图 PDF（SoC 引脚、电源树、全部外设）+ TOP/BOT DXF PCB 版图（PCB V1.3.2, 2026-01-09）· 4 个文件 · 4.5 MB · 可用于电路追踪、外壳设计、PCB 改版、维修焊接

---

## 🚀 Quick Start · 快速开始

### Installation · 安装

```bash
# Via npx skills
npx skills add HBConline/orangepi4pro-skill -g -y

# Or manual install · 或手动安装
git clone https://github.com/HBConline/orangepi4pro-skill.git
mkdir -p ~/.agents/skills/orangepi4pro
cp -r orangepi4pro-skill/ ~/.agents/skills/
ln -sf ~/.agents/skills/orangepi4pro ~/.claude/skills/orangepi4pro
```

### Usage · 使用

```
/orangepi4pro 怎么把 Ubuntu 烧录到 eMMC？
/orangepi4pro GPIO pin 7 没反应怎么排查？
/orangepi4pro 在 NPU 上部署 YOLOv5s
```

Or ask any Orange Pi 4 Pro question — the skill auto-activates via semantic matching.

或直接问任何香橙派 4 Pro 相关问题——技能通过语义匹配自动激活。

---

## 💬 Examples · 使用示例

### 🔰 Beginner · 入门

<details>
<summary><b>Q: 第一次拿到板子，怎么烧录系统？</b></summary>

```
/orangepi4pro 我是新手，刚拿到 Orange Pi 4 Pro，怎么把 Ubuntu 烧录到 TF 卡？
```

> **Agent 会给出：** balenaEtcher 完整操作步骤 (Windows + Ubuntu PC)、镜像下载地址、TF 卡规格要求 (≥8GB Class 10)、烧录后校验方法、首次启动自动扩容说明

</details>

<details>
<summary><b>Q: First time setup — how to flash the system?</b></summary>

```
/orangepi4pro I just got my Orange Pi 4 Pro. How do I flash Debian to a TF card from my Windows PC?
```

> **Agent responds with:** balenaEtcher step-by-step, Win32DiskImager alternative (with SD Card Formatter), image download URLs, sha256 verification commands, boot procedure

</details>

### ⚡ GPIO & Wiring

<details>
<summary><b>Q: GPIO 引脚没反应，怎么排查？</b></summary>

```
/orangepi4pro 我接了 pin7 上的 LED，但 gpio write 2 1 没反应，怎么排查？
```

> **Agent 会给出：** `gpio readall` 检查引脚映射 → `gpio mode 2 out` 确认模式 → `gpio write 2 1` 控制电平 → 检查 wiringOP 是否正确安装 (next 分支) → 用万用表测量 3.3V → 确认引脚电压是 3.3V 而非 5V

</details>

<details>
<summary><b>Q: How to use SPI on 40-pin header?</b></summary>

```
/orangepi4pro I need to connect an SPI device to the 40-pin. Which pins are for SPI and how do I enable it?
```

> **Agent responds with:** SPI3 pinout (MOSI=19, MISO=21, CLK=23, CS0=24) → `sudo orangepi-config` enable steps → `/dev/spidev3.0` node check → `spidev_test` loopback test → wiringOP-Python `spidev_test.py` example

</details>

### 🌐 Networking · 网络

<details>
<summary><b>Q: 想把开发板做成 WiFi 热点？</b></summary>

```
/orangepi4pro 怎么把 Orange Pi 4 Pro 做成 WiFi 热点，让手机连上来上网？
```

> **Agent 会给出：** NAT 模式 + Bridge 模式完整命令 → 5GHz / 隐藏 SSID 选项 → 自定义网关地址 `-g 192.168.2.1` → `create_ap` 语法参数详解 → 前提条件（以太网已联网）

</details>

<details>
<summary><b>Q: How to auto-connect WiFi on first boot without a monitor?</b></summary>

```
/orangepi4pro I don't have an HDMI monitor. How do I configure WiFi before first boot so it connects automatically?
```

> **Agent responds with:** `orangepi_first_run.txt` template editing on Ubuntu PC → all config variables (`FR_net_wifi_enabled=1`, `FR_net_wifi_ssid`, `FR_net_wifi_key`) → TF card mount steps → static IP variant → file deletion behavior after first boot

</details>

### 🧪 Edge AI · 边缘 AI

<details>
<summary><b>Q: 在 NPU 上跑 YOLOv5s 目标检测？</b></summary>

```
/orangepi4pro 怎么在 A733 的 NPU 上跑 YOLOv5s 做目标检测？从模型转换到板端推理的完整流程。
```

> **Agent 会给出：** PC 端 Docker 环境搭建 → `docker_images_v2.0.x` 加载 → ai-sdk 解压 → `pegasus_import → quantize → inference → export_ovx` 四步转换 → `.nb` 文件部署到开发板 → `cmake && make` 编译 → `./yolov5 ../model/v3/yolov5.nb ../input_data/dog.jpg` 推理

</details>

<details>
<summary><b>Q: Run Chinese OCR on the NPU?</b></summary>

```
/orangepi4pro I need to do Chinese text recognition using the NPU. How to deploy ChineseOCR?
```

> **Agent responds with:** complete `chineseocr` build pipeline → 4 model files (dbnet, angle_net, crnn_lstm, keys.txt) → cmake/make steps → inference command with all flags → expected output

</details>

### 🏗️ System Build · 系统编译

<details>
<summary><b>Q: 自定义编译 Linux 内核？</b></summary>

```
/orangepi4pro 我想修改内核配置然后编译自己的 Linux 镜像，完整流程是什么？
```

> **Agent 会给出：** Ubuntu 22.04 环境要求（不支持 WSL！）→ `orangepi-build` clone (GitHub/Gitee) → `KERNEL_CONFIGURE=no` 跳过菜单 → u-boot 编译 (1 分钟) → kernel 编译 (10 分钟) → rootfs 编译 → 完整镜像生成 (19 分钟) → 输出路径 → 快捷一行命令

</details>

### 🔧 Troubleshooting · 故障排查

<details>
<summary><b>Q: 板子不断重启？</b></summary>

```
/orangepi4pro 开发板上电后不断重启，HDMI 也没有显示，怎么排查？
```

> **Agent 会给出：** ① 换 5V/3A 电源适配器和 Type-C 线（最常见原因！）→ ② 检查 TF 卡是否烧录成功 → ③ 接串口看 boot log（波特率 115200）→ ④ 确认镜像版本与硬件匹配 → ⑤ 禁用 PD 充电器（不支持协商）

</details>

<details>
<summary><b>Q: NVMe SSD won't boot?</b></summary>

```
/orangepi4pro I flashed the system to NVMe SSD following the SPIFlash+NVMe method but it won't boot. What could be wrong?
```

> **Agent responds with:** SSD brand check (only Fancun 樊想 / Guancun 广存 tested!) → v1.0.6 fixed Kingston issue → verify `sudo parted --script /dev/nvme0n1 mklabel gpt mkpart primary ext4 65536s 100%` → confirm SPI Flash bootloader was flashed (`nand-sata-install` option 4) → remove TF card before power-on

</details>

### 📷 Camera & Display · 摄像头与显示

<details>
<summary><b>Q: 怎么用 MIPI 摄像头？</b></summary>

```
/orangepi4pro 我买了 OV13850 MIPI 摄像头，怎么在 Orange Pi 4 Pro 上用 OpenCV 打开它？
```

> **Agent 会给出：** `sudo modprobe vin_v4l2` 加载驱动 → `/dev/video8` 设备节点 → C++ 代码：`V4L2Camera cam("/dev/video8", 640, 480)` → Python 代码：`v4l2cam.V4L2Camera("/dev/video8", 640, 480)` → 编译 `make clean && make` → 运行 `./cam_test /dev/video8`

</details>

---

## 📊 Skill Statistics · 技能统计

| Metric · 指标 | Value · 数值 |
|---|---|
| Source Material · 源材料 | 263-page official manual (v1.4) + full PCB schematics · 官方手册 + 电路原理图 |
| Skill Size · 技能大小 | 83 KB · 2,900+ lines · 12 chapters |
| Chapters Covered · 覆盖章节 | 7 manual chapters + 5 extended chapters + 2 appendices |
| Shell Commands · Shell 命令 | 150+ verified commands · 条验证命令 |
| Configuration Files · 配置文件 | 20+ file paths with full syntax · 完整语法路径 |
| Pin Mappings · 引脚映射 | Complete 40-pin matrix (GPIO/SPI/I2C/UART/PWM) + power pins |
| AI Models · AI 模型 | 7 NPU models with full C++ build pipeline · 完整 C++ 编译管线 |
| Compatibility Matrices · 兼容矩阵 | Kernel driver · Android feature · SSD · GPU/Codec (4 OS × 4 features) |
| Schematics · 原理图 | 4 files (4.5 MB): PDF schematic + TOP/BOT DXF + DXF preview |
| PCB Version · PCB 版本 | V1.3.2 (2026-01-09) · Full power tree, SoC pinout, 40pin GPIO mapping |

---

## 🏗️ Architecture · 架构

```
orangepi4pro/
├── SKILL.md                                    # 2,827-line knowledge base · 11 chapters
├── README.md                                   # Bilingual documentation · 14 HW tables
├── OrangePi_4_Pro_A733_用户手册_v1.4(1).pdf     # Original 263-page official user manual
└── schematics/                                 # Hardware design files · 硬件设计文件
    ├── OPI 4 PRO V1_3_2_20260109.pdf           # Full schematic (multi-page PDF, 1.35 MB)
    ├── OPi_4_Pro_V1_3_2-TOP.dxf                # PCB top layer layout (DXF, 1.24 MB)
    ├── OPi_4_Pro_V1_3_2-BOT.dxf                # PCB bottom layer layout (DXF, 1.71 MB)
    └── OPi_4_Pro_V1_3_2-DXF.pdf                # PCB layout preview (PDF, 325 KB)
```

### 📐 Schematics Included · 含完整原理图

> **PCB Version:** V1.3.2 · **Date:** 2026-01-09
>
> This skill ships the **complete board-level hardware design files**, enabling:
> - **Circuit tracing** — Follow every signal from SoC to connector
> - **Custom enclosure design** — Precise mechanical dimensions from DXF
> - **PCB re-spin / derivative design** — Full schematic as reference
> - **Repair & rework** — Component placement, test points, pin mapping
> - **GPIO source tracing** — Trace 40-pin signals back to SoC GPIO banks
>
> **本技能包含完整板级硬件设计文件**，可用于：电路追踪 / 外壳设计 / PCB 改版 / 维修焊接 / GPIO 来源追踪

**Design Philosophy · 设计理念：** Reference-Type Agent Skill — prioritizes searchability and precision over brevity. Every pin number, device node path, driver status, and build command is directly accessible without hallucination risk.

**参考型智能体技能**——优先考虑可检索性和精确性而非简洁性。每个引脚号、设备节点路径、驱动状态和构建命令均可直接获取，消除 AI 幻觉风险。

---

## 🔍 Key Design Decisions · 核心设计决策

| # | Decision · 决策 | Rationale · 理由 |
|---|---|---|
| 1 | **Monolithic SKILL.md · 单体文件** | Maximizes context coherence; agent avoids multi-file lookups for cross-domain queries · 最大化上下文一致性 |
| 2 | **Mirrors Manual Structure · 遵循手册结构** | Ensures consistency with canonical source; trivial cross-referencing · 与官方源一致 |
| 3 | **Exact Commands · 精确命令** | Shows actual prompts (`orangepi@orangepi:~$` vs `root@orangepi:~#`) so agent knows execution context · 智能体知晓执行环境 |
| 4 | **Known Limitations Explicit · 标注明知限制** | Prevents agent from suggesting unsupported configs (e.g., Kingston SSD, PD negotiation) · 防止推荐不兼容配置 |

---

## 📝 License · 许可证

MIT License — free to use, modify, and distribute. 自由使用、修改和分发。

**Source Data · 源数据：** The encoded knowledge is derived from the publicly available Orange Pi 4 Pro User Manual v1.4 by Shenzhen Xunlong Software Co., Ltd. 知识来源于深圳市迅龙软件有限公司公开发布的 Orange Pi 4 Pro 用户手册 v1.4。

---

<div align="center">

**Built with** `superpowers:writing-skills` **&amp; TDD methodology**
*"No skill without a failing test first." · 「未经失败测试，不写任何技能。」*

</div>
