---
title: VMware Tools 安裝與設定
tags:
  - vmware
  - ubuntu
  - shared-folders
  - open-vm-tools
created: 2026-07-10
modified: 2026-07-10
aliases:
  - VMware Tools
  - open-vm-tools
  - HGFS 掛載
---

# VMware Tools 安裝與設定

本文件涵蓋 Ubuntu 22.04 與 24.04 在 VMware 環境下的 open-vm-tools 安裝、共用資料夾掛載、自動掛載及權限設定。

---

## 安裝 VMware Tools

現代 Ubuntu 版本使用官方開源版本 **open-vm-tools**，無需傳統 ISO 手動安裝。

### 更新並安裝

```bash
sudo apt update
```

- **桌面版 (Desktop)：**
  ```bash
  sudo apt install open-vm-tools-desktop
  ```
- **伺服器版 (Server/CLI)：**
  ```bash
  sudo apt install open-vm-tools
  ```

安裝後重新啟動虛擬機：

```bash
sudo reboot
```

---

## 手動掛載共用資料夾

### 1. 確認主機端設定

VMware 選單：`虛擬機` → `設定` → `選項` → `共用資料夾` 設為 **「總是啟用」** 並添加路徑。

### 2. 確認資料夾可見

```bash
vmware-hgfsclient
```

應列出你設定的資料夾名稱。

### 3. 建立掛載點並掛載

```bash
sudo mkdir -p /mnt/hgfs
sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
```

---

## 開機自動掛載

### 方式一：透過 `/etc/fstab`（Ubuntu 22.04 推薦）

在 `/etc/fstab` 末尾加入：

```
.host:/    /mnt/hgfs    fuse.vmhgfs-fuse    defaults,allow_other,uid=0,gid=0,_netdev,nofail,x-systemd.requires=vmtoolsd.service    0    0
```

**參數說明：**

| 參數 | 用途 |
|------|------|
| `.host:/` | 掛載 VMware 設定的所有共享資料夾 |
| `fuse.vmhgfs-fuse` | 指定 FUSE 掛載方式 |
| `allow_other` | 允許其他使用者存取 |
| `uid=0,gid=0` | 掛載為 root 擁有者 |
| `_netdev` | 延遲掛載直到網路就緒 |
| `nofail` | 掛載失敗不影響開機 |
| `x-systemd.requires=vmtoolsd.service` | 確保 VMware 工具服務先啟動 |

測試掛載：

```bash
sudo mount -a
ls /mnt/hgfs
```

### 方式二：透過 `/etc/fstab` 搭配 `auto` 關鍵字（Ubuntu 24.04 通用）

```
.host:/	/mnt/hgfs	fuse.vmhgfs-fuse	auto,allow_other	0	0
```

### 方式三：使用 `noauto` 搭配 `rc.local`

若 `auto` 關鍵字無法正常運作，可使用 `noauto` 搭配開機腳本：

1. 在 `/etc/fstab` 使用 `noauto`：
   ```
   .host:/	/mnt/hgfs	fuse.vmhgfs-fuse	noauto,allow_other	0	0
   ```

2. 編輯 `/etc/rc.local`：
   ```bash
   #!/bin/sh
   mount /mnt/hgfs
   ```

3. 設定權限並啟用：
   ```bash
   sudo chown root:root /etc/rc.local
   sudo chmod 0755 /etc/rc.local
   sudo systemctl enable rc-local.service
   ```

---

## 權限問題處理

### 方式一：指定 UID/GID

查詢使用者 ID：

```bash
id
```

修改 `/etc/fstab` 加入 `uid=1000,gid=1000`：

```
.host:/ /mnt/hgfs fuse.vmhgfs-fuse allow_other,uid=1000,gid=1000 0 0
```

重新掛載：

```bash
sudo umount /mnt/hgfs
sudo mount -a
```

### 方式二：放寬目錄權限

```bash
sudo chmod 777 /mnt/hgfs
```

### 方式三：群組管理（較安全）

```bash
sudo groupadd vmshare
sudo chown root:vmshare /mnt/hgfs
sudo chmod 775 /mnt/hgfs
sudo usermod -aG vmshare <你的使用者名稱>
```

需重新登入生效。

---

## 常見狀態檢查

| 項目 | 檢查指令 | 預期結果 |
|------|----------|----------|
| 服務狀態 | `systemctl status open-vm-tools` | `active (running)` |
| 掛載狀態 | `df -h` | `.host:/` 掛載在 `/mnt/hgfs` |
| 核心模組 | `lsmod \| grep vmw` | 顯示 vmw 相關模組 |

---

## 版本差異備註

- **Ubuntu 22.04**：建議使用 `defaults,allow_other,uid=0,gid=0,_netdev,nofail,x-systemd.requires=vmtoolsd.service`
- **Ubuntu 24.04**：`auto` 關鍵字通常可直接運作，若遇問題可改用 `noauto` + `rc.local` 方式
