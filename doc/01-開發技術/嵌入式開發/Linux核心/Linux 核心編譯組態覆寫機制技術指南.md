
在嵌入式 Linux 開發、驅動程式整合或硬體移植過程中，維護一份乾淨、可擴充且易於升級的核心組態檔（Kernel Configuration）是至關重要的。Linux 核心內建的 **Configuration Fragments（組態片段）** 機制，允許開發人員在不更動原始 `defconfig` 的前提下，透過「疊加（Override）」的方式動態生成最終的 `.config`。

## 1. 為什麼要使用組態覆寫機制？

傳統開發模式中，若要修改核心功能，通常會直接修改 `arch/<arch>/configs/defconfig` 或直接在產生的 `.config` 上做更動。然而這會帶來以下痛點：

- **版本升級困難**：當核心版本升級、 upstream 有新的 `defconfig` 時，客製化的修改極易產生衝突。
    
- **維護混亂**：多個不同的硬體專案（Boards）或功能需求（如 Debug 版 vs Release 版）需要各自複製整份幾千行的組態檔。
    

**組態片段機制** 則如同配置檔案的覆寫（Override），其核心思想是：**保留原廠/標準的 `defconfig` 不動，只將「相對於 `defconfig` 的變更」記錄在獨立的微型設定檔中，由核心編譯系統自動進行合併。**

## 2. 客製化組態片段的撰寫規範

組態片段檔案（例如 `aaa.config`）的語法與標準的 `.config` 完全一致，但它**不需要包含整份核心配置**，僅需條列出需要「新增」、「修改」或「關閉」的選項。**(組態片段檔案必須是 .config 作為附檔名才會生效)**

### 撰寫範例：
```
# 範例檔案內容：用來覆寫的客製化設定

# 1. 啟用原本未開啟的功能（y 代表編譯進核心，m 代表編譯為模組）
CONFIG_DEBUG_INFO=y
CONFIG_MY_NEW_DRIVER=m

# 2. 修改原本已存在的參數或數值
CONFIG_LOCALVERSION="-custom-v1"
CONFIG_HZ=250

# 3. 關閉原本開啟的功能（必須嚴格遵循 "# CONFIG_XXX is not set" 的標準格式）
# CONFIG_SND_PROC_FS is not set
```

> ⚠️ **注意**：若要關閉某個選項，最保險且正規的寫法是 `# CONFIG_XXX is not set`。請勿只寫 `CONFIG_XXX=n`，因為部分舊版核心的 Kconfig 語法不支援這種寫法。

## 3. 方法一：使用 Kbuild 原生 `make` 指令（最推薦）

核心的 Kbuild 系統直接支援在 `make` 命令後方帶入多個設定目標，後方的目標會自動覆寫前方的目標。

### 3.1 進階結構：客製化子目錄分類法

為了維持核心原始碼目錄的整潔，我們可以自由在 `configs` 底下建立子目錄（例如 `override/`）來進行模組化管理。

#### 目錄結構示意：
```
arch/arm64/configs/
├── defconfig                  # 核心原生基礎組態
└── override/                  # 自行建立的客製化目錄（名稱可自訂）
    ├── aaa.config             # 基礎客製化覆寫檔
    └── wifi.config            # 無線網路功能片段檔
```

#### 指令範例：
```bash
# 語法：make ARCH=<架構> <基礎defconfig> <子目錄/客製化config_1> ...
make ARCH=arm64 defconfig override/aaa.config override/wifi.config
```

### 3.2 為什麼可以指定子目錄？（Makefile 原理）

深入解析 Linux 核心中的 `scripts/kconfig/Makefile`，其底層核心邏輯如下：
```Makefile
configfiles=$(wildcard $(srctree)/kernel/configs/$@ $(srctree)/arch/$(SRCARCH)/configs/$@)
```

當你輸入 `override/aaa.config` 時，Makefile 的萬用字元會自動展開並尋找 `arch/arm64/configs/override/aaa.config`。如果檔案存在，便會將其完整路徑丟給合併腳本處理，因此建立子目錄是完全合法的。

## 4. 方法二：使用內建腳本 `merge_config.sh`

如果你的客製化組態片段**完全不想放在核心原始碼目錄內**（例如放在獨立的 BSP Git 專案倉庫中），或者檔名不想遵循 `.config` 的副檔名限制，則適合直接呼叫核心提供的工具腳本。

### 4.1 指令語法

`merge_config.sh` 位於核心的 `scripts/kconfig/` 目錄下。
```bash
# 語法：ARCH=<架構> ./scripts/kconfig/merge_config.sh [基礎config路徑] [覆寫config路徑1] ...
ARCH=arm64 ./scripts/kconfig/merge_config.sh \
    arch/arm64/configs/defconfig \
    /path/to/external/my_project_aaa.cfg \
    /path/to/external/my_project_wifi.cfg
```

### 4.2 先生成再合併模式

另一種常見的自動化腳本寫法是先透過標準 make 生成 `.config`，再使用該腳本強行併入客製化修改：
```bash
# 1. 先產生標準 .config
make ARCH=arm64 defconfig

# 2. 將外部的客製化設定直接疊加到當前的 .config 上
ARCH=arm64 ./scripts/kconfig/merge_config.sh .config /path/to/external/aaa_config
```

## 5. 核心開發中的關鍵注意事項（避坑指南）

在實際的自動化編譯（CI/CD）或日常開發中，不當使用覆寫機制可能會導致組態遺失，請務必留意以下三點：

### 1. 順序具備高度敏感性（必須由左至右）

編譯系統是採取後者覆寫前者（Last-match wins）的原則。

- ❌ **錯誤示範**：`make ARCH=arm64 override/aaa.config defconfig`
    
    _(這會導致最後執行的 `defconfig` 把你的客製化設定全部洗掉)_
    
- o **正確示範**：`make ARCH=arm64 defconfig override/aaa.config`
    

### 2. 務必全程帶上 `ARCH=...` 環境變數

當組態片段合併完成後，系統會在背景自動呼叫 `make oldconfig` 來重新解析整個 Kconfig 樹狀結構並補齊相依性。

- 如果漏掉了 `ARCH=arm64`，系統會預設使用 **Host 端（通常是電腦的 x86_64）** 的架構去解析。
    
- 這會導致所有 ARM64 特有的硬體暫存器、平台驅動的 `CONFIG_*` 選項因為在 x86 上「不合法」，而被編譯系統**自動刪除**。
    

### 3. 使用原生 `make` 時的副檔名限制

- **方法一（Make 模式）**：檔案副檔名**強制必須是 `.config`**（例如 `aaa.config`），關鍵在於點（`.`）後面的 `config` 字樣，否則 Makefile 會認不出這個目標。
    
- **方法二（Script 模式）**：使用 `merge_config.sh` 則沒有副檔名限制，任何純文字檔（如 `.cfg`、`.txt`）皆可順利解析。
    

## 6. 結論與最佳實踐

組態片段（Configuration Fragments）是現代 Linux 核心工程化開發的標準標配。

在實際維護專案時，建議將指令整合進你的自動化編譯腳本中：
```bash
# 1. 一行指令合併基礎與多個客製化片段
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig override/aaa.config override/wifi.config

# 2. 開始多核心平行編譯
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

這能確保你的核心原始碼樹（Kernel Tree）保持絕對的乾淨，同時又具備極高的維護彈性。