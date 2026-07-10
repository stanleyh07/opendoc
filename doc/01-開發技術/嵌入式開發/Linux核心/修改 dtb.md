
在 Ubuntu 下修改 DTB 檔案並透過 DTBO 替換內容的完整步驟如下：

### 1. 安裝所需工具

首先，您需要安裝 **Device Tree Compiler (dtc)** 工具來進行 DTB 和 DTS 之間的轉換：

```bash
sudo apt-get install device-tree-compiler
```

### 2. 將 DTB 轉換為 DTS

使用 `dtc` 工具將現有的 DTB 檔案反編譯為可讀的 DTS 檔案：

```bash
dtc -I dtb -O dts -o output.dts input.dtb
```

- `input.dtb` 是原始的 DTB 檔案。
- `output.dts` 是生成的 DTS 檔案。

### 3. 修改 DTS 檔案

打開生成的 `output.dts` 檔案，找到根節點 `/`，然後修改 `model` 屬性為：

```dts
/ {
    model = "NVIDIA Jetson Orin Nano 8GB - Super";
};
```

保存修改後的檔案。

### 4. 將 DTS 轉換回 DTB

使用 `dtc` 工具將修改後的 DTS 檔案編譯回 DTB 格式：

```bash
dtc -I dts -O dtb -o new.dtb output.dts
```

- `new.dtb` 是生成的新 DTB 檔案。

### 5. 使用 DTBO 替換內容

如果需要使用 DTBO（Device Tree Blob Overlay）來覆蓋修改內容，請執行以下步驟：

1. 創建一個新的 DTS 檔案，僅包含需要覆蓋的內容，例如：
    
    ```dts
    /dts-v1/;
    /plugin/;
    
    / {
        fragment@0 {
            target-path = "/";
            __overlay__ {
                model = "NVIDIA Jetson Orin Nano 8GB - Super";
            };
        };
    };
    ```
    
    保存為 `overlay.dts`。
    
2. 編譯 DTS 為 DTBO：
    
    ```bash
    dtc -I dts -O dtb -o overlay.dtbo overlay.dts
    ```
    

### 6. 替換或應用新的 DTB/DTBO

- 如果是直接替換 DTB，將 `new.dtb` 複製到系統的對應位置（例如 `/boot` 或其他啟動分區），並更新引導程式的配置。
- 如果是使用 DTBO，將 `overlay.dtbo` 複製到系統的對應位置，並確保引導程式（如 U-Boot）正確加載該檔案。

### 7. 驗證修改

重啟系統後，驗證修改是否生效：

```bash
dtc -I dtb -O dts -o check.dts new.dtb
cat check.dts | grep "model"
```

