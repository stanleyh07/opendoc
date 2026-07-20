---
title: iPhone 與 Apple 裝置連線 Linux Wi-Fi 熱點問題完整指南
tags:
  - Wi-Fi
  - 熱點
  - iPhone
  - Apple
  - WPA3
  - PMF
  - NetworkManager
  - Ubuntu
created: 2026-07-15
modified: 2026-07-15
---

## 📝 簡介

本文件詳細記錄 iPhone 與 Apple 裝置在連線 Linux Wi-Fi 熱點時常見的相容性問題。當 Ubuntu 使用 `wpa_supplicant 2.10+` 建立熱點時，由於預設啟用 WPA3/SAE 支援，會導致 iPhone 等 Apple 裝置無法連線。本文件提供技術分析、受影響裝置清單、以及解決方案。

## 🔍 問題現象

### 典型症狀

- iPhone / iPad / MacBook 無法連線到 Ubuntu 建立的 Wi-Fi 熱點
- 錯誤訊息可能是「無法加入網路」或「密碼錯誤」
- 但 Ubuntu、Windows、Android 裝置可以正常連線
- 使用其他 Ubuntu 電腦的 Wi-Fi 可以連接該熱點

### 環境條件

- Ubuntu 22.04 或更新版本
- wpa_supplicant 2.10 或更新版本
- 熱點使用 WPA2/WPA3 Mixed Mode（預設行為）

## 🔬 技術分析

### 根本原因

這是 `wpa_supplicant 2.10` 的已知問題。該版本會自動啟用 WPA3/SAE 支援，導致熱點廣播包含 SAE 的加密資訊。iPhone 看到 SAE 後嘗試使用 WPA3 連線，但某些 iPhone 型號的 WPA3 實作有問題，導致連線失敗。

### 技術細節

```
wpa_supplicant 2.10 的行為：
├── 啟用 SAE (WPA3) 支援
│   └── 自動啟用 802.11w (PMF)
│       └── 熱點廣播：RSN(PSK,PSK-SHA256,SAE/AES/AES)
│
└── iPhone 看到 SAE
    └── 嘗試使用 WPA3 連線
        └── 但某些 iPhone 的 WPA3 實作有問題
            └── 連線失敗 ✗

禁用 PMF 後：
├── wpa_supplicant 不啟用 SAE
│   └── 熱點廣播：RSN(PSK,PSK-SHA256/AES/AES)
│
└── iPhone 使用 WPA2-PSK 連線
    └── 連線成功 ✓
```

### 核心問題：RSN IE AKM 選擇 Bug

**問題描述：**

當路由器使用 WPA2/WPA3 Transition Mode 時，RSN Information Element (IE) 會同時廣播多個 AKM (Auth Key Management) suite：

```
RSN IE 廣播的 AKM：
├── AKM 2 (PSK) - WPA2
├── AKM 6 (PSK/SHA256) - WPA2 with SHA256
└── AKM 8 (SAE) - WPA3
```

**iOS 的 Bug：**

iOS 的 `_performAssociation` 函數在選擇 AKM 時有缺陷：

```c
// iOS 的 AKM 選擇邏輯（簡化）
int selected_akm = 0;
for (each akm in IE_KEY_RSN_AUTHSELS) {
    if (akm is in known_range) {
        selected_akm = akm;  // 選擇已知的 AKM
    } else {
        // 問題：遇到未知 AKM 時，回退到原始值比較
        // 導致較新的未知 AKM (SAE=8) 覆蓋較早的已知 AKM (PSK=2)
        selected_akm = akm;
    }
}

// 最終選擇的 AKM 在 switch 中命中 default 分支
switch (selected_akm) {
    case 1: // WPA-EAP
    case 2: // PSK
    case 3: // FT-EAP
    case 4: // FT-PSK
    case 5: // WPA-EAP-SHA256
    case 6: // PSK-SHA256
        // 正常處理
        break;
    default:
        return error -0xF3C;  // ← Apple 裝置在這裡失敗
}
```

**各 iOS 版本支援範圍：**

| iOS 版本 | Switch Cases | 最大支援 AKM | 問題 |
|----------|--------------|--------------|------|
| 3.x - 5.x | 1, 2 | 2 | 無法處理任何 WPA3 AKM |
| 6.x - 7.x | 1, 2, 3, 4 | 4 | 無法處理 PSK-SHA256 和 SAE |
| 8.x - 12.x | 1, 2, 3, 4, 5, 6 | 6 | 無法處理 SAE (AKM=8) |
| 13+ | 未知 | 未知 | 可能仍有問題 |

**結論：** iOS 8-12 的最大支援 AKM 值只有 6，完全無法處理 SAE (AKM=8)。

### SHA256 Cipher Suite 問題

**技術背景：**

- WPA3 啟用時會同時啟用 802.11w (PMF)
- PMF 啟用後會使用 SHA256 cipher suite
- Apple 裝置對 SHA256 的支援不完整

**具體表現：**

```
當 AP 廣播：
RSN: * Group cipher: CCMP
     * Pairwise ciphers: CCMP
     * Authentication suites: PSK PSK/SHA-256 00-0f-ac:8
     * Capabilities: MFP-capable (0x008c)

Apple 裝置的反應：
├── 看到 PSK/SHA-256
├── 嘗試使用 SHA256 cipher suite
├── 但實作不完整，導致連線失敗
└── 不是回退到 PSK，而是直接失敗
```

**影響範圍：**
- 不只是 Apple 裝置
- 某些 Android 裝置（特定版本範圍）也有同樣問題
- IoT 裝置（Roomba、智慧家電等）也受影響

### PMF (Protected Management Frames) 問題

**802.1X Enterprise 環境：**

```
AP 設定：
├── WPA2 with PMF required
├── RSN AKM type: "WPA (SHA256)" (AKM=5)
└── Management Frame Protection: Required

Apple 裝置的反應：
├── 看到 "WPA (SHA256)"
├── 錯誤識別為 WPA-PSK 網路
├── 發送 Association Request 時沒有 RSN 資訊
└── 連線失敗
```

**WPA2-PSK 環境：**

| PMF 設定 | Apple 裝置行為 |
|---------|---------------|
| PMF Optional | 正常連線 |
| PMF Required | 可能失敗 |
| PMF Disabled | 正常連線 |

### 問題根源綜合分析

根據深入分析，WiFi 連線問題的真正原因並非「Linux driver 沒有正確支援 PMF」，而是以下三個因素的組合：

#### 1. 主要問題：iOS 的 AKM 選擇 Bug

- iOS 的 `_performAssociation` 函數在選擇 AKM（Auth Key Management）時有缺陷
- 當 RSN IE 同時廣播 PSK（AKM=2）和 SAE（AKM=8）時，iOS 可能錯誤選擇 SAE
- iOS 8-12 的最大支援 AKM 值只有 6，完全無法處理 SAE（AKM=8）
- 這是 **Apple 裝置端的問題**，不是 Linux driver 的問題

#### 2. 觸發條件：wpa_supplicant 2.10+ 的預設行為

- wpa_supplicant 2.10 自動啟用 WPA3/SAE 支援
- 啟用 SAE 時自動啟用 802.11w（PMF）
- 熱點廣播包含 SAE 的 RSN IE：`RSN(PSK,PSK-SHA256,SAE/AES/AES)`

#### 3. PMF 的角色

- PMF 是 IEEE 802.11w 標準，用於保護管理幀
- 當 PMF 啟用時，會使用 SHA256 cipher suite
- **Apple 裝置對 SHA256 的支援不完整**
- 在 802.1X Enterprise 環境中，PMF Required 可能導致 Apple 裝置錯誤識別網路類型

#### Linux driver 相關線索

文件中提到 `wpa_supplicant 2.11 brcmfmac 修復`，這是一個 Red Hat Bugzilla 連結，暗示 Broadcom 的 Linux WiFi driver（brcmfmac）可能有相關問題，但需要進一步調查。

#### 結論

**主要問題在於：**
1. **Apple 裝置**：iOS 的 AKM 選擇 Bug 和不完整的 WPA3/SHA256 實作
2. **Linux 側**：wpa_supplicant 2.10+ 預設啟用 SAE/PMF 的行為

**PMF 的角色**是觸發條件，而不是根本原因。禁用 PMF 可以解決問題，是因為這會阻止 wpa_supplicant 啟用 SAE，讓熱點只廣播 WPA2 相容的 RSN IE。

這也解釋了為什麼解決方案（禁用 PMF 或使用純 WPA2 模式）能夠有效解決問題。

### 802.11r Fast Transition 問題

**問題現象：**

- 在 WPA2/WPA3 Mixed Mode 下啟用 802.11r
- 某些 Apple 裝置無法連線
- 即使禁用 PMF 也無法解決

**受影響裝置：**
- iPhone XR
- iPhone 11 系列
- iPad 8th generation
- MacBook Air M1

**解決方案：**
- 禁用 802.11r Fast Transition
- 或使用純 WPA2 模式

### macOS 特定問題

**WPA3-SAE 認證失敗：**

```
測試環境：
├── AP: GL.iNet MT6000 (MediaTek MT7986)
├── 失敗裝置: MacBook Pro M3 Max (macOS 26.1)
└── 成功裝置: iPhone 15 Pro Max (iOS 26.1)

問題：使用 sae_password_file 時失敗
├── wpa_passphrase (單一密碼): ✓ 成功
├── sae_password_file (多密碼): ✗ 失敗
└── sae_password (內聯): ✗ 失敗

錯誤訊息：
daemon.info hostapd: wlan1: STA xx:xx IEEE 802.11: authentication OK (SAE)
daemon.info hostapd: wlan1: STA xx:xx IEEE 802.11: did not acknowledge authentication response
```

**結論：** macOS 的 SAE 實作比 iOS 更嚴格，對某些配置更敏感。

## 📱 受影響裝置

### iPhone 機型

| 機型 | 問題程度 | 備註 |
|------|---------|------|
| iPhone 6 / 6 Plus | 嚴重 | 無法連線 |
| iPhone SE (2016, 1st gen) | 輕微 | 部分 iOS 版本可連線 |
| iPhone SE (2020, 2nd gen) | 嚴重 | 無法連線 |
| iPhone XR | 嚴重 | 無法連線 |
| iPhone 11 系列 | 嚴重 | 無法連線 |
| iPhone 12 mini | 嚴重 | 無法連線 |
| iPhone 14 | 嚴重 | 無法連線 |
| iPhone 15 Pro Max | 輕微 | 某些情況可連線 |

### iPad / Mac 機型

| 機型 | 問題程度 |
|------|---------|
| iPad Air 2 | 嚴重 |
| iPad 8th generation | 嚴重 |
| iPad Pro 11 2nd generation | 嚴重 |
| MacBook Air M1 | 嚴重 |
| MacBook Pro M1 / M3 | 嚴重 |
| iMac Pro | 嚴重 |

## 🔧 解決方案

### 方案一：禁用 PMF（最簡單，推薦）

```bash
# 禁用 Protected Management Frames
sudo nmcli connection modify "您的連線名稱" wifi-sec.pmf disable

# 重啟熱點
sudo nmcli connection down "您的連線名稱"
sudo nmcli connection up "您的連線名稱"
```

**驗證設定：**

```bash
# 確認 PMF 已禁用
nmcli connection show "您的連線名稱" | grep pmf
# 預期輸出：802-11-wireless-security.pmf: 1 (disable)
```

### 方案二：使用純 WPA2 模式

```bash
# 設定純 WPA2 模式
sudo nmcli connection modify "您的連線名稱" wifi-sec.proto rsn
sudo nmcli connection modify "您的連線名稱" wifi-sec.pmf disable

# 重啟熱點
sudo nmcli connection down "您的連線名稱"
sudo nmcli connection up "您的連線名稱"
```

### 方案三：禁用 802.11r Fast Transition

```bash
# 如果有啟用 802.11r，建議禁用
sudo nmcli connection modify "您的連線名稱" 802-11-wireless.ft off

# 重啟熱點
sudo nmcli connection down "您的連線名稱"
sudo nmcli connection up "您的連線名稱"
```

### 方案四：頻段與頻道設定

```bash
# 設定使用 2.4GHz 頻段（相容性最佳）
sudo nmcli connection modify "您的連線名稱" 802-11-wireless.band bg

# 設定使用頻道 6（避開 DFS 和邊緣頻道）
sudo nmcli connection modify "您的連線名稱" 802-11-wireless.channel 6

# 重啟熱點
sudo nmcli connection down "您的連線名稱"
sudo nmcli connection up "您的連線名稱"
```

## ⚠️ 風險與限制評估

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

## 🔗 相關連結

- [Apple 官方 WPA3 支援文件](https://support.apple.com/guide/security/security-features-connecting-wireless-sec8a67fa93d/web)
- [OpenWrt Issue #16728 - Apple 裝置連線問題](https://github.com/openwrt/openwrt/pull/16728)
- [OpenWrt Issue #7858 - SAE Mixed Mode 問題](https://github.com/openwrt/openwrt/issues/7858)
- [LegacyWiFiFix - iOS AKM 選擇 Bug 分析](https://github.com/PlayDay-iOS/LegacyWiFiFix)
- [M1 MacBook Pro 連線問題討論](https://superuser.com/questions/1771752/m1-macbook-pro-cant-join-linux-hotspot-when-other-devices-can)
- [wpa_supplicant 2.11 brcmfmac 修復](https://bugzilla.redhat.com/show_bug.cgi?id=2302577)

## 📚 參考資料

- WPA3 Specification v3.3 - https://www.wi-fi.org/system/files/WPA3%20Specification%20v3.3.pdf
- 802.11w Management Frame Protection
- RSN Information Element (IE) 結構
- AKM Suite Selector 標準

## 🏷️ 相關文件

- [[Ubuntu 設定 Wi-Fi 熱點與網路分享完整指南]]
