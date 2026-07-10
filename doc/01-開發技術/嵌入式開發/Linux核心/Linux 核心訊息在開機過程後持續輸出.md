要讓 Linux 核心訊息在開機過程後持續輸出，可以使用以下方法：

1. **調整 `dmesg` 設定**：
    
    - 使用 `dmesg` 命令來查看核心訊息。可以使用 `dmesg -w` 來持續監控核心訊息的輸出。
    
    ```bash
    dmesg -w
    ```
    
2. **修改 `sysctl` 設定**：
    
    - 編輯 `/etc/sysctl.conf` 文件，添加以下行來確保核心訊息持續輸出到控制台：
    
    ```bash
    kernel.printk = 7 4 1 7
    ```
    
    - 然後運行以下命令來應用更改：
    
    ```bash
    sudo sysctl -p
    ```
    
3. **使用 `journalctl`**：
    
    - `journalctl` 是 systemd 的日誌查看工具，可以用來查看和持續監控系統日誌，包括核心訊息。
    
    ```bash
    journalctl -k -f
    ```
    

這些方法應該能幫助你持續查看核心訊息。

`kernel.printk` 設定用來控制 Linux 核心訊息的輸出級別。這些數字代表不同的日誌級別和行為。以下是每個數字的位置及其意義：

1. **當前日誌級別**：控制當前顯示的訊息級別。數字越小，顯示的訊息越重要。
2. **預設日誌級別**：系統啟動後的預設日誌級別。
3. **最低日誌級別**：允許的最低日誌級別。
4. **啟動時的日誌級別**：系統啟動時的日誌級別。

這些級別的數字範圍從 0 到 7，數字越小，訊息的重要性越高。以下是每個級別的詳細說明：

- **0 (KERN_EMERG)**：緊急訊息，系統無法使用。
- **1 (KERN_ALERT)**：需要立即處理的警報。
- **2 (KERN_CRIT)**：嚴重錯誤。
- **3 (KERN_ERR)**：錯誤訊息。
- **4 (KERN_WARNING)**：警告訊息。
- **5 (KERN_NOTICE)**：正常但重要的訊息。
- **6 (KERN_INFO)**：信息性訊息。
- [**7 (KERN_DEBUG)**：調試訊息](https://www.kernel.org/doc/html/latest/core-api/printk-basics.html)[1](https://www.kernel.org/doc/html/latest/core-api/printk-basics.html)[2](https://lynxbee.com/understanding-debug-levels-for-linux-kernel-printk-a-comprehensive-guide/).

例如，如果你想讓所有訊息都顯示在控制台上，可以將 `kernel.printk` 設定為 `7 4 1 7`，這樣所有級別的訊息都會被顯示。

你可以使用以下命令來查看和修改這些設定：

```bash
# 查看當前設定
cat /proc/sys/kernel/printk

# 修改設定
echo "7 4 1 7" | sudo tee /proc/sys/kernel/printk
```

