---
title: Linux 5G 網卡連線設定完整指南
tags:
  - linux
  - network
  - 5g
  - modem
created: 2026-08-19
modified: 2026-08-19
aliases:
  - 5G網卡
  - mmcli
  - nmcli gsm
---

# Linux 5G 網卡連線設定完整指南

本文件是一份整合了 `mmcli`（硬體底層控制）與 `nmcli`（系統網路管理）的完整實戰指南，記錄在 Linux 環境下從零開始設定 5G/4G 網卡並成功連線上網的標準流程。

---

## 🎯 核心原則

在 Linux 環境下操作 5G/4G 網卡的最佳實務是：

> [!IMPORTANT] 分層管理
> **先用 `mmcli` 喚醒並準備好硬體，接著用 `nmcli` 建立具備彈性的連線設定檔，最後交由系統自動管理撥號與斷線重連。**

* **硬體層 (`mmcli`)**：負責將網卡上電、切換卡槽、解鎖 SIM 卡。這部分通常只需要在初次設定或排錯時手動執行。
* **系統層 (`nmcli`)**：負責建立虛擬的撥號設定檔（Profile）。只要設定好 **Type (`gsm`)** 與 **APN (`internet`)**，並且不綁死特定的硬體介面名稱，NetworkManager 就會自動處理後續的撥號邏輯，這也是確保 5G 網卡在 Linux 下能夠穩定運作的關鍵。

---

## 📋 標準操作流程

以下從零開始到成功連線上網的完整流程，請確保具備 `sudo` 權限。

### 步驟一：查詢硬體編號

找出 Modem（數據機）與 SIM 卡的 Index。首先列出系統抓到的 5G 網卡，找出它的硬體編號：

```bash
mmcli -L
```

輸出結果會顯示類似 `/org/freedesktop/ModemManager1/Modem/0`。這裡的 **`0`** 就是數據機編號（Modem Index）。在後續的 `mmcli` 指令中，都會以 `0` 作為範例。

### 步驟二：切換 SIM 卡槽

僅限雙卡設備，單卡設備可略過。如果你的 5G 模組支援雙卡雙待，請先指定要使用的卡槽（通常是 1 或 2）：

```bash
sudo mmcli -m 0 --set-primary-sim-slot=1
```

> [!NOTE] 注意
> 切換卡槽後，Modem 通常會短暫重啟初始化，請等待約 3-5 秒再進行下一步。

### 步驟三：啟動網卡並解鎖 SIM 卡

將網卡從休眠或停用狀態喚醒（Enable）：

```bash
sudo mmcli -m 0 -e
```

如果你的 SIM 卡有設定 PIN 碼鎖定，必須輸入 PIN 碼解鎖。注意這裡要指定 SIM 卡的編號（通常也是 0）：

```bash
sudo mmcli -i 0 --pin=0000
```

> [!NOTE] 注意
> 若你的 SIM 卡無 PIN 碼，請略過解鎖指令。

### 步驟四：建立 5G 網路連線設定檔

透過 `nmcli` 建立一個名為 `My5G` 的行動網路連線設定，並指定電信商的 APN（例如 `internet`）：

```bash
sudo nmcli connection add type gsm con-name "My5G" apn "internet"
```

> [!TIP] 省略 ifname 以獲得最佳系統相容性
> **強烈建議省略 `ifname` 參數**，讓系統自動尋找可用的數據機，避免重開機後網卡代號（如 `wwan0` 變成 `wwan1`）跳動導致連線失效。
>
> 某些舊版 Linux 若強制要求介面名稱，則可用 `ifname "*"` 代替。

### 步驟五：設定開機自動連線（推薦）

設定這個連線檔為自動連線，未來只要插著網卡開機或從休眠喚醒，系統就會自動幫你撥號：

```bash
sudo nmcli connection modify "My5G" connection.autoconnect yes
```

> [!TIP] 建議
> 若 SIM 卡有 PIN 碼，也建議將 PIN 碼存入設定中，以免開機卡在解鎖階段：
>
> ```bash
> sudo nmcli connection modify "My5G" gsm.pin "0000"
> ```

### 步驟六：啟動連線與驗證網路

手動啟用剛剛建立好的連線：

```bash
sudo nmcli connection up "My5G"
```

看到 `Connection successfully activated` 後，檢查系統是否順利取得電信商配發的 IP：

```bash
ip a
```

最後，透過 PING 測試外網是否通暢，若有回應即代表設定大功告成：

```bash
ping 8.8.8.8
```

---

## 🔗 相關文件

- [[Ubuntu 設定 Wi-Fi 熱點與網路分享完整指南]] — 將 5G 網卡的連線分享給其他設備（熱點/NAT 設定）
- [[Wi-Fi 安全協議與加密設定完整指南]] — Wi-Fi 網卡安全協議設定
