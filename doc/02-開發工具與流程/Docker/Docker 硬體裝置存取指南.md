---
title: Docker 硬體裝置存取指南
tags:
  - docker
  - hardware
  - gpio
  - audio
  - usb
  - x11
created: 2026-07-10
modified: 2026-07-10
aliases:
  - Docker 硬體穿透
  - Docker GPIO
  - Docker USB
---

# Docker 硬體裝置存取指南

---

## 音訊裝置映射

```bash
docker run -it \
  --device /dev/snd \
  -v /etc/asound.conf:/etc/asound.conf:ro \
  --group-add $(getent group audio | cut -d: -f3) \
  your-image
```

若使用 PulseAudio，需掛載 Pulse Socket：

```bash
  -v /run/user/$(id -u)/pulse/native:/run/user/$(id -u)/pulse/native
```

---

## GPIO 裝置映射

```bash
docker run -it \
  --device /dev/gpiochip0 \
  -v /sys/class/gpio:/sys/class/gpio:rw \
  --group-add $(getent group gpio | cut -d: -f3) \
  your-image
```

---

## USB 與通用硬體裝置

### 方式一：指定 USB 裝置

```bash
docker run -it \
  --privileged \
  -v /dev/bus/usb:/dev/bus/usb/ \
  your-image
```

### 方式二：全部裝置映射

```bash
docker run -it \
  --privileged \
  -v /dev:/dev \
  your-image
```

### 方式三：USB 儲存裝置

當 USB storage 被主機 mount 後，也可在容器內存取：

```bash
docker run -it \
  --privileged \
  -v /media/$USER:/media/nvidia:slave \
  your-image
```

---

## X11 圖形介面顯示

### 方式一（簡潔）

```bash
xhost +local:docker

docker run -it --rm \
  --env="DISPLAY=${DISPLAY}" \
  --env="QT_X11_NO_MITSHM=1" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  your-image
```

### 方式二（含 Xauthority 認證）

```bash
# 設定 Xauthority
xauth_list=$(xauth list $DISPLAY | grep 'MIT-MAGIC-COOKIE-1' | head -1)
xauth add $xauth_list
xauth nlist $DISPLAY | sed -e 's/^..../ffff/' | xauth -f /tmp/.docker.xauth nmerge -

docker run -it --rm \
  -e DISPLAY=$DISPLAY \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
  -v /tmp/.docker.xauth:/tmp/.docker.xauth:ro \
  --device /dev/dri \
  your-image
```

---

## 完整範例：Audio + GPIO 同時映射

```bash
docker run -it \
  --device /dev/snd \
  -v /etc/asound.conf:/etc/asound.conf:ro \
  -v /run/user/$(id -u)/pulse/native:/run/user/$(id -u)/pulse/native \
  --device /dev/gpiochip0 \
  -v /sys/class/gpio:/sys/class/gpio:rw \
  --group-add $(getent group audio | cut -d: -f3) \
  --group-add $(getent group gpio | cut -d: -f3) \
  --user $(id -u):$(id -g) \
  your-image-name
```

---

## 驗證

```bash
# 測試音訊
aplay -l
speaker-test -t wav -c 2

# 測試 GPIO
gpiodetect
echo 17 > /sys/class/gpio/export

# 測試 X11
apt install -y x11-apps
xclock
```

---

## 注意事項

- 若遇權限錯誤，可先用 `--privileged` 測試（**不建議**生產環境使用）
- 容器內需安裝對應函式庫（`alsa-utils`、`libgpiod-dev` 等）
- 若主機啟用 SELinux/AppArmor，需添加 `--security-opt` 參數
- X11 使用 `xhost +local:docker` 會放寬存取權限，請評估安全風險
