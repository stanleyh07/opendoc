---
title: Jetson AGX Orin GPIO 完整指南
tags:
  - jetson
  - gpio
  - pinmux
  - embedded
  - nvidia
  - tegra
created: 2026-08-22
modified: 2026-08-22
aliases:
  - Jetson GPIO
  - AGX Orin Pinmux
  - Tegra234 GPIO
---

# Jetson AGX Orin GPIO 完整指南

本文件以 **Jetson AGX Orin（Tegra234）** 搭配 **L4T R36.5.x（JetPack 6.x）** 為基準，完整整理 GPIO 從硬體架構、Pinmux 配置、Device Tree 設定到使用者空間操作的所有知識。檔案路徑皆對應本機 BSP 目錄 `~/source/nvidia/Linux_for_Tegra/`。

> [!NOTE] 基礎概念
> Push-pull / Open-drain / Pull-up / Pull-down 等通用 GPIO 電氣概念，詳見 [[GPIO 狀態與模式]]，本文件聚焦 Tegra 平台特有的配置方式。

---

## 1. GPIO 硬體架構

### 1.1 兩個 GPIO 控制器

Tegra234 有兩個獨立的 GPIO 控制器，在 device tree 中定義於 `source/hardware/nvidia/t23x/nv-public/tegra234.dtsi`：

| 控制器 | Device Tree Label | 位址 | 相容字串 | Linux 裝置 | 說明 |
|--------|------------------|------|----------|-----------|------|
| Main GPIO | `gpio: gpio@2200000` | `0x02200000` | `nvidia,tegra234-gpio` | `/dev/gpiochip0` | 一般用途腳位，port A~AG |
| AON GPIO | `gpio_aon: gpio@c2f0000` | `0x0c2f0000` | `nvidia,tegra234-gpio-aon` | `/dev/gpiochip1` | Always-On 區塊腳位，port AA~GG |

兩者同時也是 interrupt-controller，可作為中斷來源。

### 1.2 Port 與腳位命名規則

每個控制器下以 **port（字母）+ pin（編號 0~7）** 命名，定義於 `source/hardware/nvidia/t23x/nv-public/include/kernel/dt-bindings/gpio/tegra234-gpio.h`：

```c
/* Main controller ports：A, B, C, D, E, F, G, H, I, J, K, L, M, N,
   P, Q, R, X, Y, Z, AC, AD, AE, AF, AG */
#define TEGRA234_MAIN_GPIO(port, offset) \
    ((TEGRA234_MAIN_GPIO_PORT_##port * 8) + offset)

/* AON controller ports：AA, BB, CC, DD, EE, GG */
#define TEGRA234_AON_GPIO(port, offset) \
    ((TEGRA234_AON_GPIO_PORT_##port * 8) + offset)
```

- 寫法範例：`TEGRA234_MAIN_GPIO(H, 0)` 即 `PH.00`；`TEGRA234_AON_GPIO(BB, 1)` 即 `PBB.01`
- 每個 port 固定 8 支腳，Main 的 SoC GPIO ID = `port_id × 8 + offset`

### 1.3 SFIO 與 GPIO 多工

Tegra 每支腳位都是**多工腳**，可在以下兩種角色間切換：

```
             ┌─── SFIO（Special Function I/O）：交給硬體外設
   Pin ──┤    │   如 UART、SPI、I2C、PWM、CAN、EQOS…
             │
             └─── GPIO：由軟體直接控制 in/out
```

- **SFIO 模式**：腳位由對應硬體控制器驅動，軟體不可直接讀寫電位
- **GPIO 模式**：腳位從 pinmux「脫離」外設功能，由 GPIO 控制器接管
- 切換由 **Pinmux**（見第 2 節）在 bootloader 或 kernel 階段決定；`nvidia,function = "gp"` 表示設定為一般 GPIO

---

## 2. Pinmux 詳解

### 2.1 Pinmux 屬性總表

Pinmux dtsi 中每支腳位的節點格式（摘自 `bootloader/generic/BCT/tegra234-mb1-bct-pinmux-p3701-0000.dtsi`）：

```dts
hdmi_cec_pgg0 {
    nvidia,pins = "hdmi_cec_pgg0";          /* 腳位名稱 */
    nvidia,function = "hdmi";               /* 功能選擇：外設名稱或 "gp"（GPIO） */
    nvidia,pull = <TEGRA_PIN_PULL_NONE>;    /* 上/下拉：NONE / PULL_DOWN / PULL_UP */
    nvidia,tristate = <TEGRA_PIN_DISABLE>;  /* 三態（浮接）：ENABLE=高阻抗 */
    nvidia,enable-input = <TEGRA_PIN_ENABLE>; /* 輸入緩衝：輸出腳也要能讀回時開啟 */
    nvidia,io-high-voltage = <TEGRA_PIN_ENABLE>; /* 高壓模式（3.3V 輸出準位） */
    nvidia,lpdr = <TEGRA_PIN_DISABLE>;      /* Low Power Drive（低功耗驅動） */
};
```

| 屬性 | 可用值 | 意義 |
|------|--------|------|
| `nvidia,function` | 外設名稱（`i2c3`、`spi1`…）或 `gp` | 選擇 SFIO 功能；`gp` = 一般 GPIO |
| `nvidia,pull` | `TEGRA_PIN_PULL_NONE` / `PULL_DOWN` / `PULL_UP` | 內建上/下拉電阻 |
| `nvidia,tristate` | `ENABLE` / `DISABLE` | ENABLE 時輸出呈高阻抗（浮接） |
| `nvidia,enable-input` | `ENABLE` / `DISABLE` | 啟用輸入緩衝器；做輸入或需讀回確認時必須 ENABLE |
| `nvidia,io-high-voltage` | `ENABLE` / `DISABLE` | 啟用 3.3V 輸出準位（僅部分支援腳位） |
| `nvidia,lpdr` | `ENABLE` / `DISABLE` | 低功耗驅動模式 |
| `nvidia,lock` | `ENABLE` / `DISABLE` | 鎖定 pinmux 暫存器防誤改 |
| `nvidia,park` | `PARKED` / `NORMAL` | 將腳位停泊至固定狀態 |
| `nvidia,open-drain` | `ENABLE` / `DISABLE` | 開汲極輸出（部分腳位支援） |

> [!IMPORTANT] 常見陷阱
> - 設定為**輸入**時：`tristate = ENABLE` + `enable-input = ENABLE`
> - 設定為**輸出**時：`tristate = DISABLE` + `enable-input` 依需求（要讀回實際電位就開）
> - `pull` 只在腳位浮接或開汲極時有意義；push-pull 輸出腳不需要 pull

### 2.2 Pad Voltage（腳位電壓域）

每個 pad group 有自己的供電電壓（1.2V / 1.8V / 3.3V），由 **padvoltage dtsi** 設定：

- 檔案位置：`bootloader/generic/BCT/tegra234-mb1-bct-padvoltage-p3701-0000.dtsi`
- 在板卡 conf 中對應變數 `PMC_CONFIG`
- **硬體設計時就必須確定**：外部電路電平必須與 pad 電壓一致，否則可能損壞 IO

### 2.3 GPIO 中斷設定（gpioint）

`bootloader/generic/BCT/tegra234-mb1-bct-gpioint-p3701-0000.dts` 定義哪些 GPIO port 具備中斷能力，透過 conf 變數 `GPIOINT_CONFIG` 引用。

---

## 3. GPIO 設定的三個階段（生命週期）

同一支腳位的行為由三個階段依序決定，後者可覆寫前者：

```
┌────────────────┐   ┌────────────────────┐   ┌──────────────────┐
│ MB1 / Bootloader│ → │ Kernel Device Tree │ → │  使用者空間        │
│ BCT pinmux/gpio │   │ pinctrl / gpio-hog │   │ gpiod/Jetson.GPIO │
│ （燒錄時固化）    │   │ （隨 dtb 更新）      │   │ （runtime 動態）   │
└────────────────┘   └────────────────────┘   └──────────────────┘
```

| 階段 | 設定內容 | 生效時機 | 修改方式 |
|------|---------|---------|---------|
| MB1/BCT | pinmux 全表、pad voltage、GPIO 初始狀態、gpioint | 上電最早期（kernel 之前） | 改 dtsi + 重燒錄 |
| Kernel DT | 驅動引用的 gpios、gpio-hog、pinctrl 覆寫 | kernel 載入驅動時 | 改 dts/dtbo + 更新 dtb |
| 使用者空間 | 方向、電位、邊緣偵測 | runtime | gpiod / Jetson.GPIO |

> [!WARNING]
> 若腳位在 MB1 階段被 pinmux 成 SFIO（如 UART），使用者空間無法直接當 GPIO 操作——必須先修改 pinmux。

---

## 4. BSP 檔案地圖

以 AGX Orin devkit（P3737 + P3701）為例，關鍵檔案如下：

### 4.1 Bootloader / BCT（MB1 階段）

| 檔案 | 用途 |
|------|------|
| `bootloader/generic/BCT/tegra234-mb1-bct-pinmux-p3701-0000.dtsi` | **AGX Orin devkit 預設 pinmux 表**（SFIO 設定） |
| `bootloader/tegra234-mb1-bct-gpio-p3701-0000.dtsi` | GPIO 初始狀態（被上方 pinmux dtsi `#include`） |
| `bootloader/generic/BCT/tegra234-mb1-bct-padvoltage-p3701-0000.dtsi` | Pad 電壓設定 |
| `bootloader/generic/BCT/tegra234-mb1-bct-gpioint-p3701-0000.dts` | GPIO 中斷設定 |

GPIO 初始狀態 dtsi 格式（三種清單）：

```dts
gpio@2200000 {
    gpio-init-names = "default";
    gpio-init-0 = <&gpio_main_default>;
    gpio_main_default: default {
        gpio-input = <
            TEGRA234_MAIN_GPIO(Z, 1)
            TEGRA234_MAIN_GPIO(G, 0) >;
        gpio-output-high = <
            TEGRA234_MAIN_GPIO(H, 0) >;
        gpio-output-low = <
            TEGRA234_MAIN_GPIO(AC, 0) >;
    };
};
```

### 4.2 板卡設定與引用鏈

`p3737-0000-p3701-0000.conf`（flash.sh 讀取）：

```bash
PINMUX_CONFIG="tegra234-mb1-bct-pinmux-p3701-0000.dtsi"   # pinmux（含 include 的 gpio dtsi）
PMC_CONFIG="tegra234-mb1-bct-padvoltage-p3701-0000.dtsi"  # pad voltage
DTB_FILE=tegra234-p3737-0000+p3701-0000-nv.dtb            # kernel device tree
OVERLAY_DTB_FILE="...,tegra234-p3737-0000+p3701-0000-dynamic.dtbo,..."  # overlay 清單
```

flash.sh 於第 2945 行附近以 `mkfilesoft pinmux_config ...` 將 dtsi 交給 bootgen 組裝進 BCT。

### 4.3 Kernel Device Tree

| 檔案 | 用途 |
|------|------|
| `source/hardware/nvidia/t23x/nv-public/tegra234.dtsi` | SoC 層 GPIO/pinmux 控制器定義 |
| `.../include/kernel/dt-bindings/gpio/tegra234-gpio.h` | port ID 巨集 |
| `.../nv-platform/tegra234-p3737-0000+p3701-xxxx-nv-common.dtsi` | 板級共用設定 |
| `.../overlay/tegra234-p3737-0000+p3701-0000-hdr40.dtso` | 40-pin header overlay 原始碼 |
| `kernel/dtb/tegra234-p3737-0000+p3701-0000-hdr40.dtbo` | 編譯後的 header overlay |

### 4.4 工具

| 工具 | 位置 | 用途 |
|------|------|------|
| `pinmux-dts2cfg.py` | `kernel/pinmux/t19x/` | dtsi → MB1 cfg 轉換（進階/除錯用，R36 flash 流程已自動處理） |
| Jetson-IO | 目標板上執行 `sudo jetson-io` | 圖形化配置 40-pin header |
| Jetson.GPIO deb | `tools/jetson-gpio-common_*.deb`、`python3-jetson-gpio_*.deb` | Python GPIO 函式庫 |

---

## 5. 修改 Pinmux 完整流程（硬體 Bring-up）

### 5.1 標準工作流（Excel Pinmux 表）

NVIDIA 官方流程以 Excel 表格為單一真相來源：

1. **下載模板**：至 [Jetson Download Center](https://developer.nvidia.com/embedded/downloads) 搜尋 "Jetson AGX Orin Series Pinmux Config Template"（`.xlsm`）
2. **填寫 Customer Usage 頁籤**：為每支腳位選擇功能（SFIO/GPIO）、pull、tristate 等
3. **產生 dtsi**：使用模板內建的 VBA macro 一鍵產生：
   - pinmux dtsi
   - gpio dtsi（初始狀態）
   - padvoltage dtsi
4. **放置檔案**：將產生的 dtsi 放入 `bootloader/generic/BCT/`（gpio dtsi 慣例放 `bootloader/`）
5. **更新板卡 conf**：修改 `PINMUX_CONFIG`、`PMC_CONFIG` 指向新檔案
6. **重新燒錄**：

```bash
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \
  -c tools/kernel_flash/flash_l4t_external.xml -p "-c bootloader/t186ref/cfg/flash_t234_qspi.xml" \
  --showlogs jetson-agx-orin-devkit external
```

7. **驗證**：使用 NVIDIA 官方 [PinMux Checker](https://developer.nvidia.com/embedded/downloads#?search=pinmux) 網頁工具比對

### 5.2 手工微調單一腳位

若只調整少數腳位，可直接編輯 dtsi：

```dts
/* 例：將 PH.00 設為 GPIO 輸出、無 pull、推挽 */
ph00 {
    nvidia,pins = "ph00";
    nvidia,function = "gp";
    nvidia,pull = <TEGRA_PIN_PULL_NONE>;
    nvidia,tristate = <TEGRA_PIN_DISABLE>;
    nvidia,enable-input = <TEGRA_PIN_ENABLE>;
};
```

並在 gpio dtsi 加入初始狀態：

```dts
gpio-output-low = < TEGRA234_MAIN_GPIO(H, 0) >;
```

### 5.3 pinmux-dts2cfg.py（cfg 產生工具）

R36 的 flash.sh 已自動轉換，但此工具可用於檢查/除錯（用法見 `kernel/pinmux/t19x/README.txt`）：

```bash
cd kernel/pinmux/t19x
python3 pinmux-dts2cfg.py \
    addr_info.txt gpio_addr_info.txt por_val.txt \
    --mandatory_pinmux_file mandatory_pinmux.txt \
    <pinmux.dtsi> <gpio.dtsi> 1.0 > output.cfg
```

---

## 6. Device Tree 階段設定

### 6.1 驅動引用 GPIO

標準 GPIO specifier 為兩個 cell：`<&controller SoC_ID flags>`：

```dts
#include <dt-bindings/gpio/tegra234-gpio.h>

i2c@31e0000 {
    /* 例：觸控面板 reset 腳，低電位有效 */
    reset-gpios = <&gpio TEGRA234_MAIN_GPIO(H, 0) GPIO_ACTIVE_LOW>;
};

/* AON 腳位範例 */
sensor {
    enable-gpios = <&gpio_aon TEGRA234_AON_GPIO(BB, 1) GPIO_ACTIVE_HIGH>;
};
```

flags 常用值：`GPIO_ACTIVE_HIGH` / `GPIO_ACTIVE_LOW` / `GPIO_OPEN_DRAIN`。

### 6.2 gpio-hog（固定用途腳位）

不需驅動、開機即固定的 GPIO 用 hog：

```dts
gpio@2200000 {
    fpga-program-hog {
        gpio-hog;
        gpios = <TEGRA234_MAIN_GPIO(Q, 6) GPIO_ACTIVE_HIGH>;
        output-high;         /* 或 output-low / input */
        line-name = "fpga-program";
    };
};
```

### 6.3 40-Pin Header 配置（hdr40 + Jetson-IO)

AGX Orin devkit 的 40-pin header 預設多為 SFIO（I2C/SPI/UART/PWM）。切換方式：

**方法一：Jetson-IO（互動式，最簡單）**

```bash
sudo jetson-io
# 選 Configure Jetson Nano GPIO / Configure for compatible hardware
# 儲存後重開機生效（自動掛載 hdr40.dtbo）
```

**方法二：手動掛載 overlay**

編輯 `/boot/extlinux/extlinux.conf`，在 `APPEND` 前加入：

```
FDT /boot/dtb/kernel_tegra234-p3737-0000+p3701-0000.dtb
OVERLAYS /boot/tegra234-p3737-0000+p3701-0000-hdr40.dtbo
```

> [!IMPORTANT]
> Overlay 的部署有 **rootfs/extlinux（runtime）** 與 **conf `OVERLAY_DTB_FILE`（燒錄固化至 QSPI）** 兩條路徑，生效條件、initrd/A-B rootfs 影響等完整機制詳見 [[NVIDIA Jetson Device Tree Overlay (DTBO) 完整指南]]。GPIO 相關設定（gpio-hog、pinctrl overlay）兩種方式皆可套用。

---

## 7. 使用者空間操作

### 7.1 找出腳位對應的 line number

Linux 以 **chip + line** 定址，與 SoC 命名不同。查詢方法：

```bash
# 列出所有 GPIO chip
gpiodetect
# gpiochip0 [2200000.gpio] (192 lines)   ← Main
# gpiochip1 [c2f0000.gpio] (32 lines)    ← AON

# 依名稱搜尋（device tree 有 line-name 時）
sudo gpiofind "fpga-program"

# 列出某 chip 所有 line 的名稱與狀態
sudo gpioinfo gpiochip0

# 核心端總覽（含 SFIO 歸屬）
sudo cat /sys/kernel/debug/gpio
```

**換算公式**（Main controller，line = SoC GPIO ID）：

```
PH.00 → port H = 7 → line = 7×8 + 0 = 56
PZ.05 → port Z = 19 → line = 19×8 + 5 = 157
AON PBB.01 → gpiochip1 line = 1×8 + 1 = 9
```

> [!TIP]
> 實務上優先用 `gpioinfo` 看 label，不要心算；部分 port 有保留腳不會出現在 chip 上。

### 7.2 gpiod 工具組（libgpiod v1，R36 預裝）

```bash
# 讀取
sudo gpioget gpiochip0 56

# 寫入（-m exit：設定後離開並保持值）
sudo gpioset -m exit gpiochip0 56=1

# 監看邊緣變化
sudo gpiomon --rising-edge gpiochip0 157
```

> [!WARNING] libgpiod v1 vs v2
> Ubuntu jammy（R36 底層）為 libgpiod **v1**，語法如上。v2（Ubuntu 24.04+）參數格式大幅改變（如 `--chip`、`--line-numbers`），移植腳本時注意版本。

### 7.3 sysfs（舊介面，不建議新專案使用）

```bash
echo 56 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio56/direction
echo 1 > /sys/class/gpio/gpio56/value
echo 56 > /sys/class/gpio/unexport
```

sysfs 已自 kernel 5.10 起標記 deprecated，且無法表達 active-low 等語意。

### 7.4 Jetson.GPIO Python 函式庫

**安裝**（目標板上）：

```bash
# 方法一：安裝 BSP 附帶 deb
sudo dpkg -i tools/jetson-gpio-common_*.deb python3-jetson-gpio_*.deb

# 方法二：apt（需 universe repo）
sudo apt-get install python3-jetson-gpio

# 設定 udev 權限（免 sudo 操作）
sudo groupadd -f -r gpio
sudo usermod -a -G gpio $USER
sudo cp /opt/nvidia/jetson-gpio/etc/99-gpio.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**編號模式**：

| 模式 | 說明 | 範例 |
|------|------|------|
| `GPIO.BOARD` | 40-pin 物理針腳號 | `17` |
| `GPIO.BCM` | 相容 Raspberry Pi BCM 標籤 | `4` |
| `GPIO.TEGRA_SOC` | Tegra 原生命名 | `"PQ.06"` |
| `GPIO.CVM` | 相容 CVV 命名 | `"GPIO09"` |

**基本輸出**：

```python
import Jetson.GPIO as GPIO
import time

GPIO.setmode(GPIO.BOARD)
GPIO.setwarnings(False)

GPIO.setup(7, GPIO.OUT, initial=GPIO.LOW)
try:
    while True:
        GPIO.output(7, GPIO.HIGH)
        time.sleep(0.5)
        GPIO.output(7, GPIO.LOW)
        time.sleep(0.5)
finally:
    GPIO.cleanup()
```

**輸入 + 上拉**：

```python
GPIO.setmode(GPIO.BOARD)
GPIO.setup(11, GPIO.IN, pull_up_down=GPIO.PUD_UP)
print(GPIO.input(11))
GPIO.cleanup()
```

> [!NOTE]
> `pull_up_down` 參數只在該腳位 pinmux 已設為 GPIO 且支援軟體 pull 時有效；若 pinmux 已固定 pull 方向，以此覆蓋可能失敗。

**中斷（edge detection）**：

```python
def callback(channel):
    print(f"Edge detected on {channel}")

GPIO.add_event_detect(11, GPIO.FALLING, callback=callback, bouncetime=100)
# 或阻塞式等待：
GPIO.wait_for_edge(11, GPIO.RISING, timeout=5000)
```

**PWM**：

```python
p = GPIO.PWM(15, 1000)   # 腳位 15、1kHz
p.start(50)              # duty cycle 50%
p.ChangeDutyCycle(75)
p.ChangeFrequency(2000)
p.stop()
```

---

## 8. Kernel 驅動中的 GPIO

撰寫 kernel module 時使用 gpiod API（`linux/gpio/consumer.h`）：

```c
#include <linux/gpio/consumer.h>

struct gpio_desc *rst;

/* 取得（對應 DT 的 "reset-gpios" 屬性） */
rst = devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(rst))
    return PTR_ERR(rst);

gpiod_set_value_cansleep(rst, 1);  /* 拉高（已套用 ACTIVE_LOW 語意） */
int val = gpiod_get_value(rst);

/* 中斷 */
irq = gpiod_to_irq(rst);
ret = devm_request_threaded_irq(dev, irq, NULL, handler,
                                IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                                "mydev-reset", priv);
```

重點：
- API 自動處理 `GPIO_ACTIVE_LOW` 語意，驅動邏輯一律以「有效/無效」思考
- `devm_*` 版本隨裝置釋放自動歸還
- 參考原始碼：`source/kernel/kernel-jammy-src/drivers/gpio/gpio-tegra186.c`（Tegra GPIO 控制器驅動）

---

## 9. 常見任務 Recipes

### Recipe 1：新增一支 GPIO 輸出腳（預設 High，端到端）

1. **Excel pinmux 表**將該腳 function 設 `gp`、pull none、tristate disable → 產生 dtsi
2. gpio dtsi 加入 `gpio-output-high = < TEGRA234_MAIN_GPIO(X, Y) >;`
3. 放置 dtsi 至 `bootloader/generic/BCT/`，更新 conf 的 `PINMUX_CONFIG`
4. 重燒錄（見 5.1 步驟 6）
5. 驗證：`sudo gpioinfo gpiochip0 | grep -A1 "<line>"`

### Recipe 2：按鈕輸入 + 中斷（Python）

```python
import Jetson.GPIO as GPIO

BTN = 11
GPIO.setmode(GPIO.BOARD)
GPIO.setup(BTN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

GPIO.add_event_detect(BTN, GPIO.FALLING, bouncetime=200)
while True:
    if GPIO.event_detected(BTN):
        print("Button pressed!")
```

### Recipe 3：不重燒錄、開機即固定某腳位

用 device tree overlay + gpio-hog（適用無法重燒 QSPI 的場景）：

```dts
/dts-v1/;
/plugin/;
#include <dt-bindings/gpio/tegra234-gpio.h>

/ {
    overlay-1 {
        fragment@0 {
            target = <&gpio>;
            __overlay__ {
                my-signal-hog {
                    gpio-hog;
                    gpios = <TEGRA234_MAIN_GPIO(P, 4) GPIO_ACTIVE_HIGH>;
                    output-high;
                    line-name = "my-signal";
                };
            };
        };
    };
};
```

編譯並部署（Jetson UEFI 於開機階段合併 overlay）：

```bash
dtc -@ -I dts -O dtbo -o my-hog.dtbo my-hog.dts
sudo cp my-hog.dtbo /boot/
# extlinux.conf 加入 OVERLAYS 行後重開機
```

> [!CAUTION]
> hog 只能控制 GPIO 角色；若該腳 pinmux 仍是 SFIO，必須先走第 5 節流程改 pinmux。
> overlay 部署的完整生效條件（Boot Mode、FDT、A/B rootfs）見 [[NVIDIA Jetson Device Tree Overlay (DTBO) 完整指南]]。

---

## 10. 疑難排解與驗證

### 10.1 驗證清單

```bash
# 1. 腳位目前歸誰管？（SFIO 名稱 or 未使用）
sudo cat /sys/kernel/debug/gpio

# 2. line 狀態與方向
sudo gpioinfo gpiochip0

# 3. pinmux 暫存器實際值（對照 TRM）
sudo devmem2 0x02430030        # 例：PH0 的 PINMUX 暫存器位址

# 4. Python 函式庫資訊
python3 -c "import Jetson.GPIO as GPIO; print(GPIO.JETSON_INFO)"
```

### 10.2 常見問題

| 症狀 | 可能原因 | 解法 |
|------|---------|------|
| `gpioset` 寫了沒反應 | 腳位 pinmux 仍為 SFIO | 改 pinmux（第 5 節） |
| `Device or resource busy` | 已被驅動佔用或重複 export | `gpioinfo` 查 consumer；移除佔用者 |
| 讀值永遠為 0/1 | `enable-input` 未開、或外部無驅動源 | 檢查 pinmux `enable-input` 與外部電路 |
| `pull_up_down` 無效 | pinmux 已固定 pull 或腳位不支援 | 回到 pinmux 表修正 |
| 電平異常、IO 發燙 | pad voltage 與外部電路不符 | **立即斷電**，檢查 padvoltage 設定與硬體 |
| 重開機後設定消失 | 只做了 runtime 設定 | 需固化至 pinmux/hog/overlay |
| Docker 內操作失敗 | 未映射裝置 | 加 `--device /dev/gpiochip0`，詳見 [[Docker 硬體裝置存取指南]] |

### 10.3 權限

- gpiod/Jetson.GPIO 需要 `gpio` 群組或 root
- Jetson.GPIO 安裝後務必執行 udev rules 設定（見 7.4），否則每次都要 sudo

---

## 11. 參考資料

- 本機 BSP：`~/source/nvidia/Linux_for_Tegra/`（L4T R36.5.2）
  - pinmux cfg 工具說明：`kernel/pinmux/t19x/README.txt`
  - SoC GPIO 定義：`source/hardware/nvidia/t23x/nv-public/tegra234.dtsi`
- [NVIDIA Jetson Linux Developer Guide — Platform Adaptation: Pinmux](https://docs.nvidia.com/jetson/archives/r36.5/DeveloperGuide/MB/BoardAdaptation.html)
- [NVIDIA Jetson Linux Developer Guide — Kernel Customization: GPIO Changes](https://docs.nvidia.com/jetson/archives/r36.5/DeveloperGuide/HR/KernelCustomization.html)
- [Jetson Download Center](https://developer.nvidia.com/embedded/downloads)（Pinmux Template、PinMux Checker）
- 相關筆記：[[GPIO 狀態與模式]]、[[NVIDIA Jetson Device Tree Overlay (DTBO) 完整指南]]、[[Jetson 系統映像客製化與燒錄完整指南]]、[[Docker 硬體裝置存取指南]]
