# QtScrcpy 容器画面推送问题 - 交接文档

**日期**: 2026-05-22
**问题**: Docker 容器内 Android 10 无法通过 scrcpy/QtScrcpy 推送画面

---

## 当前环境状态

### ADB 连接状态
```
宿主:  192.168.2.19:5555 (Alpine Linux Docker 主机)
容器:  192.168.2.19:5004 (Android 10 ARM64 容器, arm96 V0.61.0)
```

### arm96 版本
```
V0.61.0
```

### 容器内运行的应用
```
com.delta.app.tt (PID 45645)
```

### 容器网络配置
```
宿主机 iptables DNAT: 0.0.0.0:5004 → 容器:5555
容器内 ADB: setprop service.adb.tcp.port 5555 → adbd 重启
```

### 容器硬件信息
```
CPU: MYT P1 (ARM64, CPU part 0xd81/0xd80, 含 SM3/SM4)
Android: 10 (API 29)
分辨率: 720x1280 (旋转后 1280x720)
SELinux: Enforcing
内核: Linux 6.1.44-14-generic
```

---

## scrcpy v4.0 连接日志

```
scrcpy -s 192.168.2.19:5004
[server] INFO: Device: [OPPO] OPPO PDKM00 (Android 10)
[server] WARN: Audio disabled: it is not supported before Android 11
INFO: Renderer: direct3d11
WARN: Demuxer 'audio': stream explicitly disabled by the device
```

**现象**: 连接成功但无画面输出，窗口黑屏或无显示

---

## 根因分析

### 内核日志关键错误
```
[63970.519675] UDC core: g1: couldn't find an available UDC or it's busy
```

**结论**: Docker 容器环境下，Android 容器的 USB Device Controller (UDC) 不可用。scrcpy-server 依赖 UDC 创建虚拟显示缓冲区，在纯网络连接的容器中无法工作。

### 相关进程状态
- `surfaceflinger` 正常运行
- `android.hardware.graphics.composer@2.1-service` 正常运行
- `gpudervice` 正常运行
- SurfaceFlinger 显示配置正常 (`mState=ON`)

### screencap 测试
```bash
adb -s 192.168.2.19:5004 exec-out screencap -p | base64 | head -c 200
# 结果: 成功返回 PNG 数据 (说明本地渲染正常)
```

---

## 已验证可行的诊断命令

```bash
# 1. ADB 连接
adb connect 192.168.2.19:5004
adb devices -l

# 2. 检查 arm96 版本
adb -s 192.168.2.19:5004 shell "arm96 -v"

# 3. 检查当前 Activity
adb -s 192.168.2.19:5004 shell "dumpsys activity activities | grep mCurrentFocus"

# 4. 检查 SurfaceFlinger
adb -s 192.168.2.19:5004 shell "dumpsys SurfaceFlinger | head -30"

# 5. 截图测试
adb -s 192.168.2.19:5004 exec-out screencap -p > screen.png

# 6. 检查内核日志 (UDC 错误)
adb -s 192.168.2.19:5004 shell "dmesg | tail -20"

# 7. 屏幕录制测试
adb -s 192.168.2.19:5004 shell "screenrecord --time-limit 3 /sdcard/test.mp4"
adb -s 192.168.2.19:5004 pull /sdcard/test.mp4

# 8. 容器内重启 ADB TCP 模式 (如连接断开)
docker exec con4 setprop service.adb.tcp.port 5555
docker exec con4 stop adbd && start adbd
```

---

## 可能的解决方案

### 1. scrcpy 参数调优
```powershell
# 尝试强制 H.264 编码
scrcpy -s 192.168.2.19:5004 --codec=h264 -W

# 或禁用视频重编码，使用原始帧
scrcpy -s 192.168.2.19:5004 --no-video --turn-screen-off
```

### 2. QtScrcpy 配置调整
- 检查 QtScrcpy 是否使用正确的工作模式
- 确认编解码器配置
- 检查缓冲区大小设置

### 3. 屏幕录制替代方案
如果 scrcpy/QtScrcpy 确实无法工作，可用以下替代方案：

```bash
# 截图
adb -s 192.168.2.19:5004 exec-out screencap -p > /path/to/screen.png

# 录屏
adb -s 192.168.2.19:5004 shell "screenrecord --time-limit 10 /sdcard/screen.mp4"
adb -s 192.168.2.19:5004 pull /sdcard/screen.mp4

### 4. VNC 方案 (如容器支持)
```bash
adb -s 192.168.2.19:5004 shell "vncservice start"
# 需要在容器内安装 VNC 服务器
```

---

## 后续建议

1. **在 Windows 侧 QtScrcpy 中测试**不同参数组合
2. **检查 QtScrcpy 的日志输出**，看是否有更详细的错误信息
3. 考虑 **VNC 方案**作为替代（需要在容器内安装 vncserver）
4. 如果需要频繁调试，考虑 **USB ADB 直连** 或 **宿主机安装 scrcpy-server 并转发**

---

## 相关文件路径

- CLAUDE.md: `/home/mc/2T/cx10/CLAUDE.md`
- 加固历史: `/home/mc/2T/cx10/.claude/context/tango_hardening.md`
- 本交接文档: `/home/mc/2T/cx10/.claude/chats/chat29-myt-scrcpy-handoff.md`