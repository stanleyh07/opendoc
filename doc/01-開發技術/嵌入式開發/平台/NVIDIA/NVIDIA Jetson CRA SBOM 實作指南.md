---
title: NVIDIA Jetson CRA SBOM 實作指南
tags:
  - nvidia
  - jetson
  - sbom
  - cra
  - yocto
  - meta-tegra
  - ota
  - security
created: 2026-09-11
modified: 2026-09-11
aliases:
  - Jetson SBOM
  - Jetson CRA 實作
  - JetPack SBOM
---

# NVIDIA Jetson CRA SBOM 實作指南

> [!NOTE] 文件定位
> 本文聚焦 NVIDIA Jetson 平台因應歐盟《網路韌性法案》（CRA）的**平台專屬**實作路徑。
> - 法規背景、SBOM 標準、VEX 與 OTA 合規要求見 [[歐盟 CRA 網路韌性法案與 SBOM 合規]]。
> - Trivy、Syft、OSV-Scanner、Yocto `create-spdx` 的通用操作見 [[SBOM 產生與漏洞掃描工具]]。

---

## 1. Jetson 平台的 SBOM 現況

NVIDIA 官方目前並**沒有**針對整個 JetPack SDK（包含 L4T、CUDA、cuDNN 等）提供單一且現成可下載的 SBOM 檔案。

由於 Jetson 底層是基於 Ubuntu（Linux for Tegra, L4T），其軟體堆疊複雜，開發者必須依據自身的「系統架構」與「開發階段」，自行盤點並產出對應的 SBOM。

因此實作上沒有單一萬用解，需依下列三個維度選擇機制：

| 維度 | 適用情境 | 使用工具 |
|------|----------|----------|
| **系統與容器層級** | 維持既有 Ubuntu / Docker 架構，需快速產出報告並整合 CI/CD | Trivy |
| **底層原始碼與專案層級** | 內部開發的底層邏輯、硬體暫存器存取模組，或重度依賴 Rust / C++ 專案 | Syft + OSV-Scanner |
| **量產環境** | AGX Orin、Thor 等搭載於工業設備，準備進入最終量產 | Yocto + `meta-tegra` |

> [!TIP] 三種機制可並存
> 開發階段可先用 Trivy / Syft 快速盤點，量產階段再切換到 Yocto 建置期生成 SBOM。工具細節見 [[SBOM 產生與漏洞掃描工具]]。

---

## 2. 機制一：系統與容器層級（Trivy）

適用於維持既有 Ubuntu / Docker 架構的 Jetson 開發。Trivy 可解析 Jetson 本機的 Ubuntu Rootfs 套件（APT），以及 Docker 容器映像。

- **本機 Rootfs 掃描**：對整個 Jetson 系統產出 SPDX 報告。
- **容器映像掃描**：若邊緣 AI 服務以 Docker Compose 運行，直接掃描映像可避免混入無關系統套件，精確度最高。

安裝方式、指令與輸出格式詳見 [[SBOM 產生與漏洞掃描工具#2. Trivy]]。

---

## 3. 機制二：底層原始碼與專案層級（Syft + OSV-Scanner）

適用於針對內部開發的特定底層邏輯、硬體暫存器存取模組，或重度依賴 Rust / C++ 編譯的專案（例如自訂驅動、推論模組）。

Syft 能深度遍歷目錄結構，精準解析 `Cargo.lock`（Rust）或 `CMakeLists.txt`（C++）等依賴檔，並匯出帶有 CPE 資訊的報告；再結合 OSV-Scanner 與全球漏洞資料庫交叉比對。

安裝方式、指令與輸出格式詳見 [[SBOM 產生與漏洞掃描工具#3. Syft]] 與 [[SBOM 產生與漏洞掃描工具#4. OSV-Scanner]]。

---

## 4. 機制三：量產環境標準做法（Yocto + meta-tegra）

適用於搭載於工業設備的 AGX Orin 或 Thor 等型號，準備進入最終量產，需嚴格符合歐盟 CRA 「可重製性」與「更新派發」要求。

### 4.1 為何量產不直接沿用 Ubuntu

直接在產品上運行完整版 Ubuntu 會夾帶龐大且不必要的冗餘套件，這將導致漏洞掃描產生極高的「偽陽性」，讓資安團隊陷入無止盡的漏洞修補地獄。

業界標準是轉向 **Yocto Project**，並搭配 NVIDIA 官方維護的 `meta-tegra` 層來建置精簡版的客製化 Linux，讓系統只包含真正需要的元件。

### 4.2 編譯期自動生成 SBOM

Yocto 內建 SBOM 生成機制（`create-spdx`），能在 Bitbake 編譯 Jetson system Image 時，精準記錄每一行被編譯進系統的程式碼與底層驅動。

`conf/local.conf` 的完整設定與輸出位置見 [[SBOM 產生與漏洞掃描工具#5. Yocto 內建 SBOM 生成（create-spdx）]]。

### 4.3 獲取產出檔案

當執行 `bitbake core-image-minimal`（或自訂的 image 名稱）完成後，Yocto 會在 `tmp/deploy/images/<machine>/` 目錄下自動產生對應整個系統映像檔的 `.spdx.json` 檔案。

這是提交給歐盟審查最純淨、最具公信力的版本。

> [!TIP] 與既有 Yocto 知識的關聯
> Yocto build 環境建置、bb 檔案查詢、重新編譯等操作，見知識庫 Yocto 系列文章（如 [[基於 Yocto 的 build-dir 建立內部安裝套件]]、[[查詢Yocto 編譯使用的 bb 檔案]]）。

---

## 5. 延伸合規：VEX 與 OTA 更新

產出 SBOM 只是合規的第一步，設備的長期營運還需要以下架構支撐。

### 5.1 VEX 漏洞聲明

當 Trivy 或 OSV-Scanner 掃出底層套件漏洞時，如果該套件的特定功能在你的設備中並未被呼叫（例如某個網路通訊協定漏洞，但設備為封閉內網運行），需撰寫 VEX 文件。

這能向主管機關證明「該漏洞已評估，且在目前的系統架構下無法被利用」，從而豁免緊急更新的要求。VEX 概念見 [[歐盟 CRA 網路韌性法案與 SBOM 合規#3. VEX（漏洞利用交換）]]。

### 5.2 A/B 分區與 OTA 更新

CRA 強制要求設備生命週期內必須具備漏洞修補能力。在 Jetson 平台上，需確實啟用 L4T 的 A/B Rootfs 備援機制，確保在派發更新以修補 CVE 漏洞時，若寫入逾時或核心載入失敗，系統能自動 Rollback 至前一個安全版本，避免設備變磚。

- OTA 更新流程與指令：[[Jetson OTA Update]]
- A/B 分區切換原理：[[Nvidia Jetson AB Partition 切換]]

---

## 6. 相關文件

- [[歐盟 CRA 網路韌性法案與 SBOM 合規]]：法規背景與核心要求。
- [[SBOM 產生與漏洞掃描工具]]：Trivy、Syft、OSV-Scanner、Yocto `create-spdx` 操作。
- [[Jetson OTA Update]]、[[Nvidia Jetson AB Partition 切換]]：更新與備援機制。
