要讓 NVIDIA Jetson 在開機時從 `rootfs_B` 啟動，而不是預設的 `rootfs_A`，你可以使用 `nvbootctrl` 工具來切換啟動槽位。這在 Jetson 支援 rootfs redundancy（A/B 檔案系統冗餘）時是可行的，特別是在 Jetson Orin 系列上。

### 🧭 步驟：切換到 rootfs_B 開機

1. **確認目前的槽位資訊**
    
    ```bash
    sudo nvbootctrl dump-slots-info
    ```
    
    這會顯示目前啟動的是哪個槽位（slot 0 為 rootfs_A，slot 1 為 rootfs_B）。
    
2. **設定下一次開機使用 rootfs_B**
    
    ```bash
    sudo nvbootctrl set-active-boot-slot 1
    ```
    
    這會將下一次開機的 rootfs 設定為 slot 1（即 rootfs_B）。
    
3. **重新啟動系統**
    
    ```bash
    sudo reboot
    ```
    
4. **驗證是否成功切換** 開機後再次執行：
    
    ```bash
    sudo nvbootctrl dump-slots-info
    ```
    
    確認目前啟動的是 slot 1。
    

---

### 📌 注意事項

- 若你看到錯誤訊息如 `RootFS A/B is not enabled`，代表你的 Jetson 裝置尚未啟用 A/B 檔案系統冗餘。你需要在燒錄映像時啟用這個功能。
- Jetson Linux 支援 rootfs redundancy 的詳細說明可參考 [NVIDIA 官方文件](https://docs.nvidia.com/jetson/archives/r36.2/DeveloperGuide/SD/RootFileSystem.html)。
- 若你使用 Jetpack 4.6 或更新版本，也可以參考 RidgeRun 的 [A/B Filesystem Redundancy 教學](https://developer.ridgerun.com/wiki/index.php/How_to_Use_A/B_Filesystem_Redundancy_and_OTA_with_NVIDIA_Jetpack)。
