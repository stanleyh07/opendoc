---
title: Initrd 與核心映像檔提取指南
tags:
  - linux
  - kernel
  - initrd
  - boot-image
  - android
created: 2026-07-10
modified: 2026-07-10
aliases:
  - initramfs
  - initial ramdisk
  - boot image extraction
---

# Initrd 與核心映像檔提取指南

---

## 第一部分：什麼是 Initrd？

`initrd`（initial ramdisk）是一個在 Linux 開機時用來載入核心模組與初始化系統的映像檔。現代系統多使用 `initramfs`（initial RAM filesystem），兩者概念類似。

### 常見位置

- `/boot/initrd.img-<kernel-version>`
- `/boot/initramfs-<kernel-version>.img`

### 檢視內容

```bash
# 建立工作目錄
mkdir /tmp/initrd
cd /tmp/initrd

# gzip 格式
zcat /boot/initrd.img-$(uname -r) | cpio -idmv

# xz 格式
xzcat /boot/initramfs-$(uname -r).img | cpio -idmv
```

### 使用專用工具

**RHEL/CentOS/Fedora：**
```bash
lsinitrd /boot/initramfs-$(uname -r).img
```

**Debian/Ubuntu：**
```bash
unmkinitramfs /boot/initrd.img-$(uname -r) /tmp/initrd
```

### 重新打包

```bash
find . | cpio -o -H newc | gzip > ../initrd.img
```

> ⚠️ 解壓縮後的檔案只是臨時檔案系統，不要直接修改原始 initrd，否則可能導致系統無法開機。建議在工作目錄編輯後再重新打包。

---

## 第二部分：從磁碟提取核心映像與還原 Initrd

本部分說明如何使用 **The Sleuth Kit (TSK)** 工具組與 **abootimg**，從磁碟裝置中找出核心分區並還原其檔案系統。

### 準備工具

```bash
sudo apt install sleuthkit abootimg
```

### 步驟 1：列出磁碟分區

```bash
mmls /dev/nvme0n1
```

在輸出中尋找標籤為 `boot` 或 `kernel` 的分區，記錄其 **Start Offset**。

### 步驟 2：提取核心分區

```bash
mmcat /dev/nvme0n1 <offset> > kernel_partition.img
```

### 步驟 3：檢查與解開 Boot Image

```bash
file kernel_partition.img
# 應顯示 Android bootimg

abootimg -x kernel_partition.img
```

解開後得到：
- `bootimg.cfg`：映像檔配置資訊
- `zImage`：核心二進位檔
- `initrd.img`：初始 RAM 磁碟

### 步驟 4：還原 Initrd 檔案系統

```bash
# 確認格式
file initrd.img

# 解壓縮
mv initrd.img initrd.img.gz
gunzip initrd.img.gz

# 提取檔案系統
mkdir initrd_root
cd initrd_root
cpio -idm < ../initrd.img
```

### 重新封裝與驗證

```bash
# 打包 ramdisk
find . | cpio -o -H newc | gzip > ../initrd.img

# 重新打包 boot image
abootimg --create new-boot.img -f bootimg.cfg -k zImage -r initrd.img

# 若有 second 或 dtb
abootimg --create new-boot.img -f bootimg.cfg -k zImage -r initrd.img -s second -d dtb

# 驗證
abootimg -i new-boot.img
```

### 注意事項

- `bootimg.cfg` 的參數必須與原始裝置一致，否則可能無法開機
- Ramdisk 目錄結構要正確，避免 init 流程失敗
- 建議先在測試環境驗證再刷入主要設備
