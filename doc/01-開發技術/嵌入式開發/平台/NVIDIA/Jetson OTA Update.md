---
title: Jetson OTA Update
tags:
  - NVIDIA
  - Jetson
  - OTA
  - A/B
  - rootfs-redundancy
  - deployment
created: 2026-07-10
modified: 2026-08-30
aliases:
  - Jetson OTA
  - OTA 更新
  - l4t_generate_ota_package
  - nv_ota_start
---

# Jetson OTA Update（Image-Based Over-the-Air Update）

Jetson Linux 支援以映像檔為基礎的 OTA 更新。整體流程分為兩階段：

1. **在主機端**使用 `l4t_generate_ota_package.sh` 從「舊版 BSP」與「新版 BSP」產生 OTA 更新包（`ota_payload_package.tar.gz`）。
2. **在目標裝置端**使用 `nv_ota_start.sh` 套用更新包，自動處理分割區配置與 A/B rootfs 切換。

> [!NOTE] 與 A/B Partition 的關係
> OTA 更新會將新 rootfs 寫入非使用中的 slot（如目前由 rootfs_A 開機，則寫入 rootfs_B），完成後切換開機槽位。A/B 切換的基礎操作見 [[Nvidia Jetson AB Partition 切換]]。

---

## 1. 產生 OTA 更新包（主機端）

### 1.1 OTA 更新流程與工具需求

- 在開發主機上設定環境變數 `BASE_BSP`（舊版 BSP）與 `TARGET_BSP`（新版 BSP），並下載、解壓對應版本的 `ota_tools_<rel>_aarch64.tbz2`。
- 使用 `l4t_generate_ota_package.sh` 腳本產生 `ota_payload_package.tar.gz`，再將這個 OTA 更新包及工具推到目標裝置。
- 在目標裝置上執行 `nv_ota_start.sh`，自動處理分割區配置、A/B rootfs 並完成更新。

### 1.2 常見參數與支援清單

- `<target_board>` 只接受：
    
    - jetson-agx-orin-devkit
    - jetson-agx-orin-devkit-industrial
    - jetson-orin-nano-devkit
- `<bsp_version>` 格式必須為：
    
    - R35-5、R35-6、R36-3 或 R36-4
- `--external-device` 與 `-S` 選項僅適用於 jetson-orin-nano-devkit（NVMe 外接裝置）。
    

### 1.3 參數錯誤與修正

- 變數名稱中不允許使用「.」，要把 `R36.4.0` 改為 `R36_4_0`。
- 但即便底線版合法，仍須符合支援清單，因此若 BASE_BSP 指向 L4T r36.3.0，就必須用 `R36-3` 而非 `R36-4`。

### 1.4 最終正確示範指令

```bash
export BASE_BSP=/home/android/nvidia/.../JetPack_6.2_Linux_JETSON_ORIN_NX_TARGETS/Linux_for_Tegra
export TARGET_BSP=/home/android/nvidia/.../JetPack_6.2.1_Linux_JETSON_ORIN_NX_TARGETS/Linux_for_Tegra

cd $TARGET_BSP

sudo -E BASE_BSP=$BASE_BSP \
  ./tools/ota_tools/version_upgrade/l4t_generate_ota_package.sh \
    --external-device nvme0n1 \
    -S 40GiB \
    jetson-orin-nano-devkit \
    R36-3
```

執行後，OTA 更新包會生成於：

```
$TARGET_BSP/bootloader/jetson-orin-nano-devkit/ota_payload_package.tar.gz
```

---

## 2. 在目標裝置上套用 OTA 更新

以下範例示範在目標 Jetson 裝置上如何利用先前產生的 `ota_payload_package.tar.gz` 及 OTA 工具完成映像檔 OTA 更新。

### 2.1 準備 OTA 更新資料

- 將 `ota_payload_package.tar.gz` 複製到裝置上，例如放至 `/ota` 目錄：
    
    ```bash
    sudo mkdir -p /ota
    scp ota_payload_package.tar.gz user@jetson:/ota/
    ```
    
- 下載對應版本的 OTA 工具包（若尚未下載）：
    
    ```bash
    wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/ota_tools_R36.4.0_aarch64.tbz2
    sudo tar xpf ota_tools_R36.4.0_aarch64.tbz2 -C /ota/
    ```
    

### 2.2 解壓並設定執行環境

1. 切換至 OTA 目錄：
    
    ```bash
    cd /ota
    ```
    
2. 將 Payload 及工具解壓：
    
    ```bash
    sudo tar xpf ota_payload_package.tar.gz
    ```
    
3. 進入版本更新腳本所在目錄：
    
    ```bash
    cd tools/ota_tools/version_upgrade
    ```
    

### 2.3 啟動 OTA 更新

1. 使用 `nv_ota_start.sh` 執行 OTA 更新：
    
    ```bash
    sudo ./nv_ota_start.sh /ota/ota_payload_package.tar.gz
    ```
    
2. 腳本自動執行以下步驟：
    - 驗證 Payload 完整性與簽章
    - 備份與更新 UEFI Bootloader 與韌體
    - 調整分割區配置（含 A/B rootfs）
    - 寫入並切換至新 rootfs
3. 更新完成後，裝置會自動重新啟動並切換到新映像。

### 2.4 更新驗證

- 登入重開機後的系統，確認版本號與韌體版本：
    
    ```bash
    head -n1 /etc/nv_tegra_release
    ```
    
- 檢查 boot log 中是否有 OTA 成功訊息：
    
    ```bash
    journalctl -b | grep OTA
    ```
    

---

## 3. 參考資料

- [Software Packages and the Update Mechanism — NVIDIA Jetson Linux Developer Guide 1 documentation](https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/SoftwarePackagesAndTheUpdateMechanism.html#sd-softwarepackagesandtheupdatemechanism)
- [Jetson Linux | NVIDIA Developer](https://developer.nvidia.com/embedded/jetson-linux-r3640)
- 更多 OTA 更新細節，可參考 NVIDIA Jetson Linux 開發指南 OTA 章節。
- 相關知識庫文章：[[Nvidia Jetson AB Partition 切換]]、[[Jetson 系統映像客製化與燒錄完整指南]]
