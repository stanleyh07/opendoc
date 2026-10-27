---
title: SBOM 產生與漏洞掃描工具
tags:
  - security
  - sbom
  - trivy
  - syft
  - osv-scanner
  - yocto
  - spdx
  - cyclonedx
created: 2026-09-11
modified: 2026-09-11
aliases:
  - SBOM 工具
  - Trivy
  - Syft
  - OSV-Scanner
  - create-spdx
---

# SBOM 產生與漏洞掃描工具

> [!NOTE] 文件定位
> 本文整理產生 SBOM 與掃描漏洞的**通用工具**操作方式，適用於各種 Linux 平台、容器與原始碼專案。
> - 法規背景與合規要求見 [[歐盟 CRA 網路韌性法案與 SBOM 合規]]。
> - NVIDIA Jetson 平台的實作路徑見 [[NVIDIA Jetson CRA SBOM 實作指南]]。

---

## 1. 工具概覽

| 工具 | 掃描對象 | 輸出格式 | 適用情境 |
|------|----------|----------|----------|
| **Trivy** | 系統 rootfs、Docker 映像 | SPDX、CycloneDX | 既有 Ubuntu / Docker 架構，快速產出報告並整合 CI/CD |
| **Syft** | 原始碼目錄、映像 | SPDX、CycloneDX | 深度解析 `Cargo.lock`（Rust）、`CMakeLists.txt`（C++）等依賴檔 |
| **OSV-Scanner** | SBOM、專案鎖定檔 | 漏洞清單 | 將 SBOM 直接與全球漏洞資料庫比對 |
| **Yocto `create-spdx`** | 建置產出的系統映像 | SPDX | 編譯期即記錄每個被編入系統的元件 |

> [!TIP] 選用策略
> - 系統層 / 容器層：Trivy。
> - 專案原始碼層：Syft + OSV-Scanner。
> - Yocto 量產建置：內建 `create-spdx`。
> 三種機制可並存互補，依系統架構與開發階段選用。

---

## 2. Trivy

Trivy 是開源生態系中最全面的掃描工具之一，能同時解析本機的 Ubuntu Rootfs 套件（APT）以及封裝好的 Docker 容器映像。

### 2.1 安裝（含 ARM64）

```bash
# 安裝必要的相依套件
sudo apt-get install wget apt-transport-https gnupg lsb-release

# 匯入 Trivy 官方 GPG Key 並加入 APT 來源清單
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list

# 更新並安裝
sudo apt-get update && sudo apt-get install trivy
```

> [!WARNING] `apt-key` 已淘汰
> `apt-key` 在新版 Ubuntu 已被標示為淘汰（deprecated）。維護時可改用將 GPG key 放到 `/etc/apt/keyrings/` 並以 `signed-by` 指定的方式，避免未來失效。

### 2.2 掃描系統 Rootfs

掃描整個系統安裝的 deb 套件與全域依賴，產出 SPDX 格式報告。

```bash
# 執行 Rootfs 掃描並匯出 JSON 格式的 SPDX SBOM
sudo trivy rootfs / --format spdx-json --output jetson-rootfs-sbom.spdx.json
```

> [!NOTE] 適用於任何 Linux 系統
> `trivy rootfs /` 不限於 Jetson，任何 Debian / Ubuntu 系 Linux 皆可使用，只是輸出檔名可自行調整。

### 2.3 掃描容器映像

若服務是透過 Docker / Docker Compose 運行，直接掃描映像能避免混入無關的系統套件，精確度最高。

```bash
# 掃描指定的 Docker Image 並匯出 CycloneDX 格式
trivy image my-jetson-ai-service:latest --format cyclonedx --output ai-service-sbom.json
```

---

## 3. Syft

Syft 能深度遍歷目錄結構，精準解析如 `Cargo.lock`（Rust）或 `CMakeLists.txt`（C++）等依賴檔，並匯出帶有 CPE 資訊的報告。適合針對內部開發的特定底層邏輯、硬體暫存器存取模組，或重度依賴 Rust / C++ 編譯的專案。

### 3.1 安裝

```bash
# 透過官方安裝腳本直接將 Syft 下載至 /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sudo sh -s -- -b /usr/local/bin
```

### 3.2 掃描原始碼目錄

```bash
# 進入專案目錄並執行深度掃描
cd /path/to/your/custom_driver_or_inference_project
syft dir:. --output spdx-json=project-sbom.json
```

---

## 4. OSV-Scanner

產出 SBOM 後，可交由 Google 開源的 OSV-Scanner，直接與全球漏洞資料庫交叉比對。

> [!NOTE] 版本差異（v1 vs v2）
> OSV-Scanner 自 v2 起 CLI 大幅調整：掃描改為 `osv-scanner scan ...` 子命令、SBOM 掃描改為**依檔名判斷格式**、Go 模組路徑加入 `/v2`。以下以 v2 為主。

### 4.1 安裝方式

OSV-Scanner **本身不需要 Go 環境**，官方提供 SLSA3 等級的預編譯 binary 與多種套件管理員安裝方式；只有「從原始碼安裝」才需要 Go。

**方式一：預編譯 Binary（推薦，免 Go）**

從官方 [Releases 頁面](https://github.com/google/osv-scanner/releases) 下載對應平台檔案（資產命名如 `osv-scanner_linux_arm64`）：

```bash
# 以 ARM64 Linux（如 Jetson）為例
curl -sSL -o osv-scanner \
  https://github.com/google/osv-scanner/releases/latest/download/osv-scanner_linux_arm64
chmod +x osv-scanner
sudo mv osv-scanner /usr/local/bin/
osv-scanner --version
```

**方式二：套件管理員**

```bash
brew install osv-scanner              # macOS / Linux (Homebrew)
pacman -S osv-scanner                 # Arch Linux
apk add osv-scanner                   # Alpine Linux
scoop install osv-scanner             # Windows (Scoop)
winget install Google.OSVScanner      # Windows (WinGet)
```

**方式三：Docker（免在本機安裝）**

```bash
docker pull ghcr.io/google/osv-scanner:latest
docker run -v ${PWD}:/src ghcr.io/google/osv-scanner --sbom=/src/project.spdx.json
```

**方式四：從原始碼安裝（需 Go 環境）**

需要 Go 1.26.2 以上，且 v2 模組路徑含 `/v2`：

```bash
go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest
```

> [!WARNING] 舊版 go install 路徑已失效
> 早期文件常見的 `github.com/google/osv-scanner/cmd/osv-scanner@latest`（無 `/v2`）對應 v1，v2 起已變更。

### 4.2 載入 SBOM 進行漏洞比對

```bash
# v2：檔名需符合 SBOM 規格後綴（.spdx.json / .cdx.json），工具依檔名判斷格式
osv-scanner scan --sbom=project.spdx.json

# 等價的 scan source 形式
osv-scanner scan source --sbom=project.spdx.json

# v1（舊版）語法
osv-scanner --sbom=project-sbom.json
```

> [!TIP] 檔名很重要
> v2 不再靠旗標判斷 SBOM 格式，而是**依檔名**（例如 `project.spdx.json`、`project.cdx.json`）決定解析方式。命名時請保留對應後綴。

---

## 5. Yocto 內建 SBOM 生成（create-spdx）

Yocto 內建 SBOM 生成機制，能在 Bitbake 編譯系統 Image 時，精準記錄每一行被編譯進系統的程式碼與底層驅動。請在 Yocto 的 `conf/local.conf` 中加入以下設定：

```bitbake
# 啟用 SPDX SBOM 生成機制 (核心設定)
INHERIT += "create-spdx"

# 設定輸出格式更容易閱讀 (針對開發與除錯)
SPDX_PRETTY = "1"

# 包含建置環境的詳細變數資訊 (符合嚴格稽核需求)
SPDX_INCLUDE_BUILD_VARIABLES = "1"

# 若需要追蹤上游原始碼的 Checksum
SPDX_ARCHIVE_SOURCES = "1"
```

當執行 `bitbake core-image-minimal`（或自訂的 image 名稱）完成後，Yocto 會在 `tmp/deploy/images/<machine>/` 目錄下自動產生對應整個系統映像檔的 `.spdx.json` 檔案。

> [!TIP] 量產環境的首選
> 對準備量產、需符合 CRA「可重製性」與「更新派發」要求的產品，Yocto 建置期生成的 SBOM 是最純淨、最具公信力的版本。搭配 NVIDIA `meta-tegra` 的實務應用見 [[NVIDIA Jetson CRA SBOM 實作指南]]。

---

## 6. 相關文件

- [[歐盟 CRA 網路韌性法案與 SBOM 合規]]：法規背景、SBOM 標準、VEX 與 OTA 要求。
- [[NVIDIA Jetson CRA SBOM 實作指南]]：以 Jetson 平台為例的完整實作路徑。
