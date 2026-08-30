---
title: Nvidia Jetson AB Partition 切換
tags:
  - NVIDIA
  - Jetson
  - A/B
  - rootfs-redundancy
  - bootloader
  - nvbootctrl
created: 2026-07-10
modified: 2026-08-30
aliases:
  - Jetson AB Partition
  - Jetson A/B rootfs 切換
  - nvbootctrl
  - rootfs redundancy
---

# Nvidia Jetson AB Partition 切換

## 1. 概述

要讓 NVIDIA Jetson 在開機時從 `rootfs_B` 啟動，而不是預設的 `rootfs_A`，可以使用 `nvbootctrl` 工具來切換啟動槽位。這在 Jetson 支援 rootfs redundancy（A/B 檔案系統冗餘）時是可行的，特別是在 Jetson Orin 系列上。

A/B 機制將系統分割為兩個獨立的 slot（`rootfs_A` / `rootfs_B`，對應 GPT 的 `APP` 與 `kernel-dtb` 等 partition），開機時由 bootloader 選定其一啟動。此架構為 **OTA 更新**與**系統故障回退**的基礎。

> [!NOTE] 與其他文件關係
> - A/B partition 在 GPT 中的命名規則與 L4tLauncher 的動態掃描機制，見 [[Jetson AGX Orin DTB 載入機制與判斷邏輯]]。
> - rootfs redundancy 下的 Device Tree Overlay 部署注意事項，見 [[NVIDIA Jetson Device Tree Overlay (DTBO) 完整指南]] 的「A/B Rootfs 注意事項」章節。
> - 透過 OTA 更新自動處理 A/B rootfs 切換，見 [[Jetson OTA Update]]。

---

## 2. 切換到 rootfs_B 開機

### 2.1 步驟

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

## 3. nvbootctrl 常用指令

除了上述基本切換流程，`nvbootctrl` 提供以下常用指令：

| 指令 | 作用 |
|------|------|
| `nvbootctrl get-current-slot` | 顯示目前開機的 slot 編號（0 = rootfs_A，1 = rootfs_B） |
| `nvbootctrl set-active-boot-slot <n>` | 設定下一次開機使用的 slot |
| `nvbootctrl mark-boot-successful` | 將目前 slot 標記為「開機成功」，避免下次被視為失敗而回退 |
| `nvbootctrl is-slot-marked-successful <n>` | 查詢指定 slot 是否已被標記為成功 |
| `nvbootctrl get-slot-suffix` | 取得目前 slot 的後綴（`_a` / `_b`） |
| `nvbootctrl dump-slots-info` | 列出所有 slot 的詳細資訊 |

> [!TIP] 開機回退機制
> A/B 開機有「失敗回退」設計：若某 slot 開機失敗（kernel 無法載入或 rootfs 掛載失敗），bootloader 會自動切換到另一個 slot 開機。系統正常啟動完成後，建議執行 `nvbootctrl mark-boot-successful` 將目前 slot 標記為成功，確保後續更新流程的正確性。

---

## 4. 注意事項

- 若看到錯誤訊息如 `RootFS A/B is not enabled`，代表 Jetson 裝置尚未啟用 A/B 檔案系統冗餘。需要在燒錄映像時啟用這個功能。
- Jetson Linux 支援 rootfs redundancy 的詳細說明可參考 [NVIDIA 官方文件](https://docs.nvidia.com/jetson/archives/r36.2/DeveloperGuide/SD/RootFileSystem.html)。
- 若使用 Jetpack 4.6 或更新版本，也可以參考 RidgeRun 的 [A/B Filesystem Redundancy 教學](https://developer.ridgerun.com/wiki/index.php/How_to_Use_A/B_Filesystem_Redundancy_and_OTA_with_NVIDIA_Jetpack)。
- 啟用加密 rootfs（`*_enc_rootfs_ab.xml`）時，A/B 切換需搭配 EKS/FSKP 金鑰，詳見 [[Jetson AGX Orin 開機流程與客製化指南#3.9 EKS / FSKP（加密金鑰相關）]]。

---

## 5. 相關連結

- [[Jetson 系統映像客製化與燒錄完整指南]]
- [[Jetson AGX Orin 開機流程與客製化指南]]
- [[Jetson AGX Orin DTB 載入機制與判斷邏輯]]
- [[Jetson OTA Update]]
