# 让 KDE Wayland 使用 NVIDIA 独显并启用 Edge 硬件加速

本教程基于 KDE Plasma + Wayland 桌面，教你如何强制 KWin 使用 NVIDIA 独显渲染，并让 Edge 走 Wayland 原生模式 + VA-API 硬解。
硬件环境：**GTX 1650 Ti + Intel UHD** 笔记本，实测有效。

## 问题描述

* KWin 在 Wayland 下**不读取 `prime-select` 或 `nvidia-smi` 的设置**，而是按 DRM 枚举顺序选择第一块显卡。
* 绝大多数笔记本中 Intel 核显总是 `card0`，因此 KWin 默认用核显渲染，NVIDIA 独显被闲置。
* 结果是 Edge 等应用无法调用独显，视频播放、WebGL 等也没有硬件加速。

## 处理过程

### 1. 确认显卡对应的 DRM 设备

```bash
for i in /sys/class/drm/card*/device; do
  echo "Card: $(basename $(dirname $i))"
  [ -f "$i/vendor" ] && echo "  Vendor: $(cat $i/vendor)"
  [ -f "$i/device" ] && echo "  Device: $(cat $i/device)"
  echo "---"
done
```

* `Vendor: 0x8086` → Intel 核显（通常 `card0`）
* `Vendor: 0x10de` → NVIDIA 独显（通常 `card1`）

### 2. 创建环境变量脚本强制 KWin 使用 NVIDIA

```bash
mkdir -p ~/.config/plasma-workspace/env
nano ~/.config/plasma-workspace/env/gpu.sh
```

写入：

```bash
#!/bin/bash
export KWIN_DRM_DEVICES=/dev/dri/card1:/dev/dri/card0
```

> 更稳定的写法：使用 `by-path` 路径，避免 `cardX` 编号变动。先 `ls -l /dev/dri/by-path/` 查看，例如：
>
> ```bash
> export KWIN_DRM_DEVICES=/dev/dri/by-path/pci-0000:01:00.0-card:/dev/dri/by-path/pci-0000:00:02.0-card
> ```

### 3. 赋予执行权限并重启

```bash
chmod +x ~/.config/plasma-workspace/env/gpu.sh
```

注销重登或重启。

### 4. 验证 KWin 渲染器

```bash
qdbus6 org.kde.KWin /KWin supportInformation | grep -i "renderer"
```

期望输出：`OpenGL renderer string: NVIDIA GeForce GTX 1650 Ti/PCIe/SSE2`

同时 `nvidia-smi` 进程列表中应出现 `kwin_wayland`。

### 5. 强制 Edge 使用 Wayland 原生模式

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/microsoft-edge.desktop ~/.local/share/applications/
nano ~/.local/share/applications/microsoft-edge.desktop
```

在 `Exec=` 行末尾添加：

```
--ozone-platform=wayland --enable-features=UseOzonePlatform
```

### 6. 启用 Edge VA-API 视频硬解

在 Edge 地址栏 `edge://flags` 中启用：

* `#enable-vaapi` → Enabled
* `#enable-accelerated-video-decode` → Enabled
* `#enable-zero-copy` → Enabled（可选）

### 7. 安装 NVIDIA VA-API 驱动并设置环境变量

```bash
# Debian/Ubuntu
sudo apt install nvidia-vaapi-driver libva-utils
# Arch
sudo pacman -S libva-nvidia-driver libva-utils
```

编辑 `/etc/environment`，加入：

```
LIBVA_DRIVER_NAME=nvidia
NVD_BACKEND=direct
```

重启后验证：

```bash
vainfo
```

应显示 NVIDIA 驱动及 H264/HEVC 等支持。

### 8. 验证 Edge GPU 状态

访问 `edge://gpu`，确认：

* Window System: Wayland
* Canvas / WebGL / Video Decode: Hardware accelerated

## 命令速查

| 目的 | 命令 |
| --- | --- |
| 查看 KWin 渲染器 | `qdbus6 org.kde.KWin /KWin supportInformation \| grep -i renderer` |
| 查看 NVIDIA 状态 | `nvidia-smi` |
| 查看 DRM 设备 | `ls -l /dev/dri/by-path/` |
| 验证 VA-API | `vainfo` |
| 查看 Edge GPU 状态 | `edge://gpu` |

## 注意事项

* **功耗与续航**：KWin 全程使用独显会显著增加待机功耗，电池续航下降，风扇更易启动。
* **黑屏回退**：若配置后无法进入桌面，按 `Ctrl+Alt+F2` 切到 TTY，删除 `~/.config/plasma-workspace/env/gpu.sh` 后重启即可恢复。
* **按需切换**：若更在意续航，可改回核显优先，仅对特定应用使用 `prime-run` 或 `__NV_PRIME_RENDER_OFFLOAD=1` 调用独显，但 Wayland 显式同步体验会打折扣。

## 总结

KWin 在 Wayland 下按 DRM 枚举顺序选卡，核显恒为 `card0` 会抢占渲染；关键解法是用 `KWIN_DRM_DEVICES` 环境变量把独显排到前面。Edge 侧则同时搞定两件事——`--ozone-platform=wayland` 走原生 Wayland，VA-API（`nvidia-vaapi-driver` + `LIBVA_DRIVER_NAME=nvidia`）开启视频硬解，最后用 `edge://gpu` 确认全面硬件加速。
