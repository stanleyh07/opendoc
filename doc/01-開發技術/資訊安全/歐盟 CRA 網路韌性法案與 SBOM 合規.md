---
title: 歐盟 CRA 網路韌性法案與 SBOM 合規
tags:
  - security
  - compliance
  - cra
  - sbom
  - spdx
  - cyclonedx
  - vex
  - supply-chain
created: 2026-09-11
modified: 2026-09-11
aliases:
  - CRA
  - Cyber Resilience Act
  - 網路韌性法案
  - SBOM 合規
  - 軟體物料清單
---

# 歐盟 CRA 網路韌性法案與 SBOM 合規

> [!NOTE] 文件定位
> 本文整理歐盟《網路韌性法案》（Cyber Resilience Act, CRA）對軟體產品的核心合規要求，屬於**跨平台通用**的法規與概念知識，不限於特定硬體平台。
> - 實際 SBOM 產生與漏洞掃描工具操作，見 [[SBOM 產生與漏洞掃描工具]]。
> - NVIDIA Jetson 平台的落地實作，見 [[NVIDIA Jetson CRA SBOM 實作指南]]。

---

## 1. 背景

歐盟《網路韌性法案》（Cyber Resilience Act, CRA）要求所有具備數位元素的產品在進入歐盟市場時，必須符合嚴格的資安合規標準。法規涵蓋產品的整個生命週期，從設計、開發、上市到售後維護，皆須具備可證明的安全管理能力。

針對軟體層面，CRA 的核心要求包含下列三項：

1. **提供標準化 SBOM（軟體物料清單）**：必須提供符合國際標準（如 SPDX 或 CycloneDX）的軟體物料清單，精確記錄所有開源與第三方元件。
2. **漏洞追蹤與修補**：需持續比對 NVD 等漏洞資料庫（CVE），並具備在期限內派發更新（OTA）以修補高危險漏洞的能力。
3. **無未評估之已知漏洞**：若系統中存在漏洞但無法被利用，需提供 VEX（Vulnerability Exploitability eXchange）文件進行免責聲明。

> [!IMPORTANT] 核心精神
> CRA 不僅要求「產出一份清單」，更要求建立**可持續運作的管理流程**：知道產品裡有什麼（SBOM）、知道有什麼風險（漏洞掃描）、知道如何處置（修補或 VEX 聲明），並具備派發修補的能力（OTA）。

---

## 2. SBOM 標準格式

SBOM（Software Bill of Materials）是產品的軟體成分清單，需以機器可讀的標準格式呈現。業界主流標準有二：

| 標準 | 維護組織 | 常見格式 | 特性 |
|------|----------|----------|------|
| **SPDX** | Linux Foundation | `spdx`、`spdx-json`、`tag-value` | 歷史悠久、法遵與授權資訊完整，Yocto 內建支援 |
| **CycloneDX** | OWASP | `cyclonedx`、`cyclonedx-json` | 以資安與供應鏈風險為導向，輕量、易整合 |

兩種格式皆為國際公認，選擇時可依工具鏈支援度與主管機關要求決定。實務上常同時保留兩種格式以因應不同審查需求。

> [!TIP] 格式選擇建議
> - 追求**編譯期自動生成**與授權合規：優先 SPDX（Yocto `create-spdx` 原生輸出）。
> - 追求**容器與應用層掃描**與漏洞導向：CycloneDX 通常整合更順暢。
> - 不確定時，兩者都輸出，成本不高。

---

## 3. VEX（漏洞利用交換）

VEX（Vulnerability Exploitability eXchange）用於聲明「某個已知漏洞在特定產品中是否可被利用」。

當掃描工具（如 Trivy、OSV-Scanner）掃出底層套件漏洞時，若該套件的特定功能在你的設備中並未被呼叫（例如某個網路通訊協定漏洞，但設備為封閉內網運行），可撰寫 VEX 文件說明處置狀態，向主管機關證明「該漏洞已評估，且在目前的系統架構下無法被利用」，從而豁免緊急更新要求。

常見的 VEX 狀態分類：

| 狀態 | 意義 |
|------|------|
| `not_affected` | 已評估，確認不受影響（通常需附理由，如 `vulnerable_code_not_in_execute_path`） |
| `affected` | 確認受影響，需修補 |
| `fixed` | 已有修補版本 |
| `under_investigation` | 仍在調查中 |

> [!WARNING] VEX 不是忽略漏洞的藉口
> VEX 必須基於**技術評估**提出正當理由，並保留評估紀錄。主管機關可能要求說明判斷依據，不可濫用。

---

## 4. OTA 更新與生命週期要求

CRA 強制要求設備在生命週期內必須具備漏洞修補能力，因此「能派發更新」是合規的必要條件。

- 需具備安全的 OTA（Over-the-Air）更新機制，能將修補程式派送到已出貨的設備。
- 為避免更新失敗導致設備變磚，業界普遍採用 **A/B 系統分區**（A/B Rootfs）備援機制：新版本寫入非使用中的槽位，驗證成功後才切換；若寫入逾時或核心載入失敗，系統可自動 Rollback 至前一個安全版本。

> [!NOTE] 與平台的關聯
> 不同平台的 OTA 實作方式各異。NVIDIA Jetson 採用 L4T 的 A/B Rootfs 機制，詳見 [[NVIDIA Jetson CRA SBOM 實作指南]]、[[Jetson OTA Update]] 與 [[Nvidia Jetson AB Partition 切換]]。

---

## 5. 合規落地流程（通用）

1. **盤點成分**：對系統 rootfs、容器映像與原始碼專案產生 SBOM（SPDX / CycloneDX）。
2. **掃描漏洞**：將 SBOM 與 CVE / OSV 資料庫比對，列出已知漏洞。
3. **評估與處置**：
   - 可修補者：納入 OTA 更新排程。
   - 不可利用者：撰寫 VEX 聲明並保留評估依據。
4. **持續維運**：建立 CI/CD 自動化流程，於每次版本建置時重新產生 SBOM 並掃描。

> [!TIP] 自動化是關鍵
> 手動盤點無法應付頻繁改版。應將 SBOM 產生與漏洞掃描整合進 CI/CD，讓合規成為流程的一部分，而非額外負擔。

---

## 6. 相關文件

- [[SBOM 產生與漏洞掃描工具]]：Trivy、Syft、OSV-Scanner、Yocto `create-spdx` 的操作說明。
- [[NVIDIA Jetson CRA SBOM 實作指南]]：以 Jetson 平台為例的完整實作路徑。
- [[Jetson OTA Update]]、[[Nvidia Jetson AB Partition 切換]]：Jetson 的更新與備援機制。
