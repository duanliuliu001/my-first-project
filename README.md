# my-first-project

test01  
Hello World! 这是我的第一个 GitHub 项目

---

## 树莓派 Debian + USB 摄像头：先把视频画面跑起来

下面这套步骤可以先帮你在树莓派上看到实时视频，后续你再在这个基础上做“掉板功能提醒”（可理解为目标离开/掉落检测）。

### 1) 先确认系统识别到了 USB 摄像头

```bash
ls /dev/video*
v4l2-ctl --list-devices
```

- 如果看到 `/dev/video0`（或 `/dev/video1`），说明设备已识别。
- 如果 `v4l2-ctl` 命令不存在，先安装工具：

```bash
sudo apt update
sudo apt install -y v4l-utils
```

### 2) 用最简单方式直接预览画面（无需写代码）

安装并打开 ffplay 预览：

```bash
sudo apt install -y ffmpeg
ffplay -f v4l2 -framerate 30 -video_size 640x480 /dev/video0
```

如果黑屏或报格式错误，先查支持格式：

```bash
v4l2-ctl -d /dev/video0 --list-formats-ext
```

然后把 `ffplay` 的分辨率和帧率改成摄像头支持的组合。

### 3) 用 Python + OpenCV 打开视频（为后续视觉开发做准备）

安装依赖：

```bash
sudo apt install -y python3-opencv python3-pip
```

新建 `camera_preview.py`：

```python
import cv2

cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
cap.set(cv2.CAP_PROP_FPS, 30)

if not cap.isOpened():
    raise RuntimeError("无法打开摄像头 /dev/video0")

print("按 q 退出预览")
while True:
    ret, frame = cap.read()
    if not ret:
        print("读取帧失败")
        break
    cv2.imshow("USB Camera Preview", frame)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

运行：

```bash
python3 camera_preview.py
```

### 4) 为“掉板提醒”提前做的架构建议

建议把程序拆成 4 个模块，后期维护会轻松很多：

1. **采集模块**：持续读取摄像头帧；
2. **检测模块**：识别“板子”是否还在 ROI（可先用颜色/轮廓，后续再上模型）；
3. **状态机模块**：正常 → 可疑 → 掉板确认（加时间阈值防抖）；
4. **告警模块**：蜂鸣器、日志、MQTT、企业微信/钉钉推送。

一个简单可落地的判定逻辑：
- 连续 `N` 帧（比如 15 帧）都未检测到板子，判定“掉板”；
- 恢复检测到后，发送“恢复正常”事件，避免重复刷屏告警。

### 5) 常见问题排查

- **权限问题**：确认当前用户在 `video` 组：
  ```bash
  groups
  sudo usermod -aG video $USER
  ```
  修改后需重新登录。
- **被其它程序占用**：
  ```bash
  fuser /dev/video0
  ```
- **帧率低**：先降分辨率到 `640x480`，再逐步提高。
- **光照变化大导致误报**：先固定曝光/增益，或加补光。

---

如果你愿意，我下一步可以直接给你一版**“掉板检测最小可运行脚本”**（含 ROI 框选、连续丢失 N 帧告警、保存告警截图）。
