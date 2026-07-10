
## 核心機制：Linux 模組版本校驗 (Symbol Versioning)

當 Linux 核心開啟 `CONFIG_MODVERSIONS=y` 時，模組載入不只看版本號，還要比對 **CRC 雜湊值**（即「指紋」）。

### 1. 為什麼修改 IP 選項會影響 PCIe 驅動？

核心是一個緊密耦合的整體。當您開啟 `CONFIG_IP_ADVANCED_ROUTER` 時，核心的全域資料結構（如與中斷、網絡、或裝置管理相關的 `struct`）會發生微調。

- **指紋跳變：** 即使 `pcie-tegra194.ko` 的代碼沒動，但它所依賴的核心函數（如 `_dev_info`）因為環境變量改變，其 CRC 指紋也會隨之跳變。
    
- **一同編譯的意義：** 只有在同一批 `make` 過程中產生的 `Image` 與 `.ko`，其 `Module.symvers` 內的指紋才會完全匹配。
    

---

## 深度解析：Initrd (Initial RAM Disk) 的隱藏陷阱

**這是您目前開不了機的最核心原因。**

### 1. 什麼是 Initrd？

`initrd` 是一個微型檔案系統，在開機的第一階段被 Bootloader 載入到記憶體。它的任務是：

- 在真正的磁碟掛載前，提供必要的驅動（如 **PCIe, NVMe, USB, PHY**）。
    
- **陷阱：** 為了確保能讀到磁碟，`initrd` 內部**打包了一份關鍵驅動模組**。
    

### 2. 「新核心」遇上「舊包裝」

當您更新了 `/boot/Image` 和 `/lib/modules/` 後：

1. **核心啟動：** 系統跑的是您的「新 Image」（`uname` 時間正確）。
    
2. **預載入：** 核心在讀取硬碟上的 `/lib/modules` 之前，會先從 `initrd` 載入 PCIe 驅動。
    
3. **報錯：** `initrd` 裡包的是**舊編譯的模組**，指紋與新核心不符。
    
4. **結果：** PCIe 驅動載入失敗，導致磁碟掛載失敗，最終無法進入桌面。
    

---

## 手動較驗與錯誤檢查方式

### 1. 如何確認 Initrd 裡包了什麼？

如果您懷疑 `initrd` 沒更新，可以在 Jetson 上直接解開它來檢查：

Bash

```
# 在臨時目錄操作
mkdir /tmp/initrd_check && cd /tmp/initrd_check
# 解開目前的 initrd (路徑視您的系統而定，通常是 /boot/initrd.img)
cp /boot/initrd.img ./initrd.gz
gunzip initrd.gz
cpio -idm < initrd
# 檢查裡面的模組日期
ls -l lib/modules/$(uname -r)/kernel/drivers/pci/controller/dwc/pcie-tegra194.ko
```

### 2. 符號指紋手動較驗

- **核心提供的指紋：** `cat /proc/kallsyms | grep __crc__dev_info`
    
- **模組要求的指紋：** `modprobe --dump-modversions <路徑> | grep _dev_info`
    

> 如果兩者十六進制碼不同，系統絕對會拒絕載入。

---

## 系統化解決步驟 (完整版)

### 第一階段：Host PC 正確產出

1. **徹底清理：** `make mrproper` 刪除舊的 `Module.symvers`。
    
2. **同步編譯：** 確保 `Image` 與 `modules` 在同一次指令序列中完成。
    
3. **導出模組：** 使用 `INSTALL_MOD_PATH` 整理好整包 `/lib/modules/`。
    

### 第二階段：Jetson 平台同步與部署

這一步是修復報錯的關鍵，請務必按照順序執行：

|**步驟**|**動作指令**|**目的**|
|---|---|---|
|**1. 備份**|`sudo cp /boot/initrd.img /boot/initrd.img.bak`|預防更新失敗無法啟動。|
|**2. 覆蓋模組**|`sudo rm -rf /lib/modules/$(uname -r)`<br><br>  <br><br>`sudo cp -r <新模組> /lib/modules/`|確保實體硬碟上的驅動是最新的。|
|**3. 更新 Initrd**|**`sudo update-initramfs -c -k $(uname -r)`**|**核心步驟：將新模組打包進開機預載鏡像。**|
|**4. 指定導引**|檢查 `/boot/extlinux/extlinux.conf`|確認 `INITRD` 參數指向您剛更新的檔案。|

---

## 開啟或關閉 `CONFIG_MODVERSIONS` 的影響

|**設定**|**優點**|**缺點 / 風險**|
|---|---|---|
|**開啟 (=y)**<br><br>  <br><br>(建議)|**安全性極高**：防止因為 ABI 不相容導致的記憶體損壞或 Kernel Panic。|對編譯同步要求極嚴，`initrd` 稍有過期即會報錯。|
|**關閉 (=n)**|**相容性高**：只要版本號一致就能載入，適合頻繁微調核心設定的開發期。|**極大風險**：若資料結構偏移改變，模組載入後會導致系統隨機崩潰且難以排查。|

---

## 總結建議

您目前雖然 `uname` 時間正確，但因為 **PCIe 驅動在開機極早期就必須運作**，所以它受限於 `initrd` 內部的舊檔案。

> **最後解決行動：** 請在 Jetson 平台上執行 `sudo update-initramfs -u`（更新目前）或 `sudo update-initramfs -c -k $(uname -r)`（建立新檔），重啟後即可解決 `disagrees about version` 的報錯。
