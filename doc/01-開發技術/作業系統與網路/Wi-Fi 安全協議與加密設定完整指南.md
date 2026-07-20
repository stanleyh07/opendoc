---
title: Wi-Fi 安全協議與加密設定完整指南
tags:
  - Wi-Fi
  - WPA3
  - WPA2
  - SAE
  - PMF
  - 加密
  - NetworkManager
  - Ubuntu
created: 2026-07-16
modified: 2026-07-16
---

## 📝 簡介

本文件詳細說明 Wi-Fi 安全協議（WPA、WPA2、WPA3）的技術原理、NetworkManager 安全設定參數的完整選項、如何檢查裝置支援、以及在 Ubuntu 上的設定方法。

## 🔐 Wi-Fi 安全協議演進

| 協議 | 版本 | 金鑰管理 | 加密演算法 | 狀態 |
|------|------|---------|-----------|------|
| WPA | 1 | TKIP | RC4 | 已淘汰 |
| WPA2 | 2 | PSK/802.1X | AES-CCMP | 廣泛使用 |
| WPA3 | 3 | SAE/802.1X | AES-CCMP/GCMP | 最新標準 |

### WPA2 vs WPA3 差異

| 特性 | WPA2 | WPA3 |
|------|------|------|
| 金鑰交換 | 4-Way Handshake | SAE (Simultaneous Authentication of Equals) |
| 離線字典攻擊 | 易受攻擊 | 防護 |
| 前向保密 | ✗ | ✓ |
| 管理幀保護 | 選擇性 | 強制 |
| 密碼長度 | 8 字元 | 8 字元（但更安全） |

---

## 📋 NetworkManager 安全設定參數完整說明

### wifi-sec.key-mgmt（金鑰管理方式）

| 值 | 說明 | 適用場景 |
|-----|------|---------|
| `none` | WEP 或無密碼保護 | 不推薦 |
| `wpa-psk` | WPA2 + WPA3 Personal | 家用/小型辦公 |
| `sae` | WPA3 Personal only | 最高安全性 |
| `wpa-eap` | WPA2 + WPA3 Enterprise | 企業環境 |
| `ieee8021x` | Dynamic WEP | 舊系統 |
| `owe` | Opportunistic Wireless Encryption | 開放式加密 |

### wifi-sec.proto（協議版本）

| 值 | 說明 |
|-----|------|
| `wpa` | 允許 WPA 協議（舊版，不推薦） |
| `rsn` | 允許 WPA2/RSN 協議（推薦） |
| 未指定 | 兩者都允許 |

### wifi-sec.pmf（Protected Management Frames，管理幀保護）

| 值 | 數值 | 說明 |
|-----|-----|------|
| `default` | 0 | 使用全域預設值（若未設定全域值，預設為 optional） |
| `disable` | 1 | 禁用 PMF（解決 iPhone 相容性問題） |
| `optional` | 2 | 若裝置支援則啟用 PMF |
| `required` | 3 | 必須啟用 PMF，若不支援則連線失敗（WPA3 必要） |

### wifi-sec.pairwise（配對加密演算法）

| 值 | 說明 |
|-----|------|
| `tkip` | TKIP 加密（較舊，不安全） |
| `ccmp` | AES 加密（推薦） |

### wifi-sec.group（群組加密演算法）

| 值 | 說明 |
|-----|------|
| `wep40` | WEP 40-bit |
| `wep104` | WEP 104-bit |
| `tkip` | TKIP |
| `ccmp` | AES（推薦） |

### 加密套件與金鑰管理的關係

```
WPA2-PSK:
├── key-mgmt: wpa-psk
├── proto: rsn
├── pairwise: ccmp
├── group: ccmp
└── pmf: 0 (disable) 或 2 (optional)

WPA3-SAE:
├── key-mgmt: sae
├── proto: rsn
├── pairwise: ccmp
├── group: ccmp
└── pmf: 3 (required)

WPA2/WPA3 混合模式:
├── key-mgmt: wpa-psk (或 sae)
├── proto: rsn
├── pairwise: ccmp
├── group: ccmp
└── pmf: 2 (optional)
```

---

## 🔍 如何檢查裝置是否支援 WPA3

### 檢查網卡支援

```bash
# 檢查網卡是否支援 SAE
iw phy | grep -i "sae"
# 應顯示：Device supports SAE with AUTHENTICATE command

# 檢查支援的加密套件
iw list | grep -i "ccmp\|gcmp"
# 應包含 CCMP-128 或 GCMP-256

# 完整檢查 WPA3 相關功能
iw phy | grep -i "sae\|wpa3\|pmf\|ieee80211w"
```

### WPA3 支援條件

| 層級 | 必要條件 | 說明 |
|------|---------|------|
| 晶片 | 支援 SAE 密碼學運算 | SHA-256、ECC |
| 韌體 | 啟用 SAE 功能 | 韌體字串需包含 `sae` |
| 驅動 | 支援 `NL80211_FEATURE_SAE` | `iw phy` 顯示支援 SAE |
| 軟體 | wpa_supplicant ≥ 2.10 或 iwd | Ubuntu 22.04 已內建 |

### 常見支援 WPA3 的網卡晶片

| 晶片廠牌 | 支援 WPA3 的驅動 | 備註 |
|---------|------------------|------|
| Intel | iwlwifi (Intel AX200/AX201/AX210) | ✓ 完整支援 |
| Qualcomm/Atheros | ath9k, ath10k, ath11k | ✓ 支援 |
| MediaTek | mt76 | ✓ 支援 |
| Realtek | rtl8xxxu, rtw88, rtw89 | 部分支援 |
| Broadcom | brcmfmac | 有限支援 |

### SAE 支援的技術層級

```
SAE 支援涉及四個層級：
┌─────────────────────────────────────────────┐
│           使用者空間軟體 (Software)           │
│     wpa_supplicant / iwd / NetworkManager   │
├─────────────────────────────────────────────┤
│           驅動程式 (Driver)                   │
│         iwlwifi / ath11k / brcmfmac          │
├─────────────────────────────────────────────┤
│           韌體 (Firmware)                    │
│      網卡內部的微控制器程式                      │
├─────────────────────────────────────────────┤
│           晶片硬體 (Hardware)                 │
│         支援 SAE 密碼學運算的電路               │
└─────────────────────────────────────────────┘

即使硬體支援，若韌體或驅動未啟用 SAE，仍無法使用 WPA3。
```

---

## ⚙️ Ubuntu 上的 WPA3 設定方法

### 方法一：建立全新的 WPA3 熱點

```bash
# 1. 建立熱點（先用 WPA2）
sudo nmcli device wifi hotspot ifname wlan0 ssid "您的Wi-Fi名稱" password "您的密碼"

# 2. 修改為 WPA3
sudo nmcli connection modify "Hotspot" wifi-sec.key-mgmt sae
sudo nmcli connection modify "Hotspot" wifi-sec.proto rsn
sudo nmcli connection modify "Hotspot" wifi-sec.pairwise ccmp
sudo nmcli connection modify "Hotspot" wifi-sec.group ccmp
sudo nmcli connection modify "Hotspot" wifi-sec.pmf required

# 3. 重啟熱點
sudo nmcli connection down "Hotspot"
sudo nmcli connection up "Hotspot"
```

### 方法二：將現有 WPA2 熱點升級為 WPA3

```bash
# 1. 停止熱點
sudo nmcli connection down "Hotspot"

# 2. 修改安全設定
sudo nmcli connection modify "Hotspot" wifi-sec.key-mgmt sae
sudo nmcli connection modify "Hotspot" wifi-sec.pmf required

# 3. 重啟熱點
sudo nmcli connection up "Hotspot"
```

### 方法三：WPA2/WPA3 混合模式（相容性最佳）

```bash
# 設定為混合模式（裝置可選擇 WPA2 或 WPA3）
sudo nmcli connection modify "Hotspot" wifi-sec.key-mgmt wpa-psk
sudo nmcli connection modify "Hotspot" wifi-sec.pmf optional

# 重啟熱點
sudo nmcli connection down "Hotspot"
sudo nmcli connection up "Hotspot"
```

### 驗證 WPA3 設定

```bash
# 檢查熱點的安全設定
nmcli connection show "Hotspot" | grep wifi-security

# 應顯示：
# 802-11-wireless-security.key-mgmt: sae
# 802-11-wireless-security.pmf: 3 (required)
```

### WPA3 注意事項

| 項目 | 說明 |
|------|------|
| 密碼長度 | WPA3 最少 8 個字元 |
| 裝置相容性 | 舊裝置可能不支援 WPA3，需使用混合模式 |
| iPhone 問題 | 若 iPhone 無法連線，可暫時禁用 PMF |
| 效能 | WPA3 會稍微增加 CPU 使用率 |

---

## ⚠️ Ubuntu GUI 重置設定問題

### 問題描述

使用 Ubuntu 設定 UI 啟動熱點時，會重置網路安全設定，將 WPA3 設定改回 WPA2-PSK。這是因為 GUI 在啟動熱點時，會重新寫入 `/etc/NetworkManager/system-connections/Hotspot.nmconnection` 檔案。

### 解決方案一：使用 nmcli 命令啟動（推薦）

```bash
# 設定 WPA3 後，使用 nmcli 啟動（不要用 GUI）
sudo nmcli connection down "Hotspot"
sudo nmcli connection up "Hotspot"
```

### 解決方案二：使用不同的連線名稱

```bash
# 刪除預設的 Hotspot 連線
sudo nmcli connection delete "Hotspot"

# 建立新的連線名稱（不叫 "Hotspot"）
sudo nmcli connection add type wifi ifname wlan0 con-name "MyWPA3Hotspot" autoconnect no ssid "您的Wi-Fi名稱" mode ap

# 設定安全參數
sudo nmcli connection modify "MyWPA3Hotspot" 802-11-wireless.mode ap 802-11-wireless.band bg ipv4.method shared
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.key-mgmt sae
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.proto rsn
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.pairwise ccmp
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.group ccmp
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.pmf required
sudo nmcli connection modify "MyWPA3Hotspot" wifi-sec.psk "您的密碼"

# 啟動熱點
sudo nmcli connection up "MyWPA3Hotspot"
```

### 解決方案三：直接編輯設定檔

```bash
# 編輯熱點設定檔
sudo nano /etc/NetworkManager/system-connections/Hotspot.nmconnection

# 修改 [wifi-security] 區塊為：
[wifi-security]
group=ccmp;
key-mgmt=sae
pairwise=ccmp;
proto=rsn;
pmf=3
psk=您的密碼

# 重啟 NetworkManager
sudo systemctl restart NetworkManager

# 使用 nmcli 啟動（不要用 GUI）
sudo nmcli connection up "Hotspot"
```

### 解決方案四：鎖定設定檔權限

```bash
# 先設定好 WPA3 參數
sudo nmcli connection modify "Hotspot" wifi-sec.key-mgmt sae
sudo nmcli connection modify "Hotspot" wifi-sec.pmf required

# 停止熱點
sudo nmcli connection down "Hotspot"

# 鎖定檔案（唯讀）
sudo chattr +i /etc/NetworkManager/system-connections/Hotspot.nmconnection

# 啟動熱點
sudo nmcli connection up "Hotspot"
```

**注意**：若要修改設定，需先解除鎖定：
```bash
sudo chattr -i /etc/NetworkManager/system-connections/Hotspot.nmconnection
```

### 解決方案比較

| 方案 | 難度 | 穩定性 | 推薦度 |
|------|------|--------|--------|
| 使用 nmcli 啟動 | 簡單 | ★★★★ | ✓ 推薦 |
| 改用不同名稱 | 簡單 | ★★★★★ | ✓✓ 最推薦 |
| 編輯設定檔 | 簡單 | ★★★★ | ✓ |
| 鎖定權限 | 中等 | ★★★★ | ✓ |

---

## 🛡️ PMF 風險與限制評估

### 安全性評估

| 防護類型 | PMF 啟用 | PMF 禁用 | 實際影響 |
|---------|---------|---------|----------|
| 離線字典攻擊 | ✓ 防護 | ✗ 無防護 | 攻擊者需物理靠近才能發動 |
| Deauth 攻擊 | ✓ 防護 | ✗ 無防護 | 家庭環境風險很低 |
| 一般竊聽 | ✓ 防護 | ✓ 防護 | WPA2 加密仍然很強 |

### 限制說明

1. **無法使用 WPA3**：禁用 PMF 後，裝置只能使用 WPA2-PSK，無法使用更新的 WPA3 協議。

2. **新裝置回退**：現代 iPhone/Android 支援 WPA3，但會回退到 WPA2 連線。

3. **企業級需求**：若需要最高安全標準（如企業環境），應啟用 PMF 並使用 WPA3。

### 結論

對於家庭或小型辦公環境，WPA2-PSK + AES/CCMP 加密已足夠安全。禁用 PMF 只是關閉 WPA3 支援，對大多數使用場景沒有影響。

---

## 🔗 相關文件

- [[Ubuntu 設定 Wi-Fi 熱點與網路分享完整指南]] - Ubuntu 熱點設定與網路分享
- [[iPhone 與 Apple 裝置連線 Linux Wi-Fi 熱點問題完整指南]] - iPhone/Apple 裝置連線問題的詳細技術分析

## 📚 參考資料

- NetworkManager 官方文件 - https://networkmanager.dev/docs/
- WPA3 Specification - https://www.wi-fi.org/system/files/WPA3%20Specification%20v3.3.pdf
- IEEE 802.11w Management Frame Protection
- nmcli 手冊頁 - `man nmcli`
