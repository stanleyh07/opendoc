---
title: OverlayFS 架構與應用指南
tags:
  - linux
  - overlayfs
  - embedded
  - ota
  - rootfs
  - version-control
created: 2026-07-10
modified: 2026-07-10
aliases:
  - OverlayFS rootfs
  - OTA RootFS 架構
  - OverlayFS 版本管理
---

# OverlayFS 架構與應用指南

本文件涵蓋 OverlayFS 在嵌入式 Linux 中的兩大應用場景：**OTA 系統更新架構** 與 **開發版控管理**。

---

## 第一部分：OverlayFS 核心概念

OverlayFS 的核心邏輯是將多個目錄疊加組合，向使用者呈現一個虛擬的合併目錄。

### 目錄角色

| 層級 | 角色 | 說明 |
|------|------|------|
| **`lowerdir`** | 底層（唯讀） | 乾淨的系統基底（Golden Image） |
| **`upperdir`** | 上層（可讀寫） | 所有新增、修改、刪除的變更 |
| **`workdir`** | 工作目錄 | 核心內部原子操作暫存區，須與 `upperdir` 同分割區 |
| **`merged`** | 合併層 | 最終呈現給系統的根目錄 |

---

## 第二部分：OTA 更新架構

利用 OverlayFS 與 `initramfs`，實現唯讀系統與用戶資料分離，確保 OTA 更新時保留使用者設定。

### 磁碟分割規劃

| 分割區 | 屬性 | 功用 |
|--------|------|------|
| **Boot** (`/dev/mmcblk0p1`) | 可讀寫 | Bootloader, Kernel, Device Tree, initramfs |
| **RootFS** (`/dev/mmcblk0p2`) | **唯讀** | OverlayFS 的 `lowerdir` |
| **Data** (`/dev/mmcblk0p3`) | **可讀寫** | OverlayFS 的 `upperdir` 與 `workdir` |

### initramfs 啟動腳本

```bash
#!/bin/sh

# 1. 掛載虛擬檔案系統
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

# 2. 建立掛載點
mkdir -p /mnt/lower /mnt/data /mnt/overlay

# 3. 掛載底層系統
mount -o ro /dev/mmcblk0p2 /mnt/lower
mount -o rw /dev/mmcblk0p3 /mnt/data

# 4. 建立 OverlayFS 目錄
mkdir -p /mnt/data/upper /mnt/data/work

# 5. 組裝 OverlayFS
mount -t overlay overlay -o lowerdir=/mnt/lower,upperdir=/mnt/data/upper,workdir=/mnt/data/work /mnt/overlay

# 6. 轉移虛擬檔案系統
mount --move /sys /mnt/overlay/sys
mount --move /proc /mnt/overlay/proc
mount --move /dev /mnt/overlay/dev

# 7. 切換根目錄
exec switch_root /mnt/overlay /sbin/init
```

### `mount --move` 與 `switch_root` 說明

- **保留狀態**：保留核心早期建立的裝置節點，避免硬體失明
- **釋放記憶體**：允許 `switch_root` 刪除舊的 initramfs
- **原子操作**：在核心掛載樹中瞬間改變指標位置

### OTA 更新流程

1. 背景下載全新 RootFS 映像檔
2. 直接寫入 RootFS 分割區（或切換 A/B Partition）
3. 重新開機
4. initramfs 掛載新系統層，疊加既有 Data 分割區，完成無痛升級

---

## 第三部分：開發版控管理

使用 OverlayFS 實現輕量化的 rootfs 版本控制，替代 Yocto/Buildroot 重量級方案。

### 專案目錄結構

```
my_project_repo/            # git init 在此
├── .gitignore
├── build_env.sh            # 一鍵掛載/解掛載腳本
├── rootfs_base/            # 唯讀底層 (Lower)
├── work/                   # 工作目錄 (被 Git 忽略)
├── merged/                 # 合併層 (被 Git 忽略)
└── overlay/                # 變更層 (Git 追蹤)
    ├── etc/network/interfaces
    └── usr/bin/my_app
```

### `.gitignore`

```
/work/
/merged/
*.tar.gz
```

### 開發流程

```bash
# 掛載
BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
mkdir -p "${BASE_DIR}/work" "${BASE_DIR}/merged"
sudo mount -t overlay overlay \
  -o lowerdir="${BASE_DIR}/rootfs_base",upperdir="${BASE_DIR}/overlay",workdir="${BASE_DIR}/work" \
  "${BASE_DIR}/merged"

# 開發完成後提交
sync
sudo umount -l "${BASE_DIR}/merged"
cd "${BASE_DIR}/overlay"
git add .
git commit -m "feat: update configuration"
```

### 部署同步（rsync 黃金參數）

```bash
sudo rsync -avxHAX --progress /path/to/merged/ /path/to/target_deploy_dir/
```

| 參數 | 作用 |
|------|------|
| `-a` | 封存模式，保留權限、符號連結、裝置檔案 |
| `-x` | 不跨越檔案系統邊界，避免誤同步 `/proc` `/sys` |
| `-H` | 保留硬連結（BusyBox 等依賴硬連結節省空間） |
| `-A` | 保留 ACL |
| `-X` | 保留擴充屬性（SELinux labels, capabilities） |
