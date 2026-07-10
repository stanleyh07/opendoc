
**使用 `partclone` 的高效能（跳過空白區塊），直接生成一個標準的、可掛載的 Raw Image，或者直接複製到另一顆硬碟，而不產生專有的備份格式。**

---

## 1. 核心概念與優勢

在備份 Linux 分區時，我們通常面臨兩種選擇：

1. **`dd`**：可以直接掛載，但會複製所有空白區塊，速度慢且佔空間。
    
2. **`partclone` (預設)**：只複製有資料的區塊（快），但產出的是專有格式 (`.pcl`)，無法直接掛載。
    

**本指南的方案結合了兩者優點：** 使用 `partclone` 的智慧讀取技術，配合 `--restore_raw_file` 參數，產出與 `dd` 相同的 Raw Image 格式，但速度與效率卻是 `partclone` 的水準。

---

## 2. 應用場景 A：備份為可掛載的印象檔 (Image File)

這是將分區備份成一個檔案（如 `backup.img`），該檔案可以像隨身碟一樣被掛載讀取。

### 2.1 執行指令

假設來源分區為 `/dev/nvme0n1p1`，目標檔案為 `backup.img`。

Bash

```
# 建議先建立一個稀疏檔案容器（可選，但建議）
truncate -s 100G backup.img  # 大小建議等於或略大於來源分區

# 執行備份指令
sudo partclone.ext4 -c -s /dev/nvme0n1p1 -o backup.img -W -F -L partclone.log
```

### 2.2 參數詳解

|**參數**|**全名**|**功能說明**|
|---|---|---|
|**`-c`**|`--clone`|啟動複製模式。|
|**`-s`**|`--source`|指定來源設備 (Source)。|
|**`-o`**|`--output`|指定輸出目標 (Output)。|
|**`-W`**|**`--restore_raw_file`**|**核心參數**。強制輸出為原始 Raw 格式，而非 Partclone 特殊格式。|
|**`-F`**|`--force`|**強制執行**。當來源分區（如根目錄 `/`）正在掛載使用中時，必須加上此參數才能略過檢查。|

### 2.3 驗證與掛載

備份完成後，請務必按照以下步驟驗證：

1. **檢查檔案類型**：
    
    Bash
    
    ```
    file backup.img
    ```
    
    - ✅ **成功**：顯示 `Linux rev 1.0 ext4 filesystem data...`
        
    - ❌ **失敗**：顯示 `Partclone image...`（代表漏了 `-W` 參數）
        
2. **修復檔案系統 (重要)**：
    
    由於使用了 `-F` 強制備份運作中的系統，印象檔內的檔案系統會標記為 "Dirty"（不一致）。掛載前建議修復：
    
    Bash
    
    ```
    e2fsck -fy backup.img
    ```
    
3. **掛載映像檔**：
    
    Bash
    
    ```
    mkdir -p /mnt/backup
    sudo mount -o loop backup.img /mnt/backup
    ```
    

---

## 3. 應用場景 B：直接複製到實體硬碟 (Disk to Disk)

這是將系統直接「搬家」到另一顆硬碟（如 `/dev/sda1`），常用於更換硬碟或製作實體備份碟。

### 3.1 執行指令

假設來源為 `/dev/nvme0n1p1`，目標硬碟分區為 `/dev/sda1`。

Bash

```
sudo partclone.ext4 -c -s /dev/nvme0n1p1 -o /dev/sda1 -W -F -L partclone.log
```

### 3.2 ⚠️ 重大風險提示 (必讀)

1. **資料覆蓋風險**：
    
    目標分區 `/dev/sda1` 上的**所有資料將被永久清除**並覆蓋。
    
2. **容量限制**：
    
    目標分區的容量 **必須 >=** 來源分區的總容量。
    
    - 即使來源只用了 10GB，但分區大小是 100GB，目標分區也至少要有 100GB。
        
3. **UUID 衝突 (開機危機)**：
    
    複製後的 `/dev/sda1` 會擁有與來源 **完全相同** 的 UUID。如果重新開機時兩顆硬碟都插著，BIOS/OS 可能會混淆，導致開機失敗或掛載錯誤。
    

### 3.3 UUID 衝突解決方案

複製完成後，請立即修改目標分區的 UUID：

Bash

```
# 生成隨機 UUID 並寫入目標分區
sudo tune2fs -U random /dev/sda1
```

---

## 4. 常見問題排除 (Troubleshooting)

### Q1: 出現 `This is not partclone image` 錯誤？

- **原因**：通常是因為指令混用了 `-r` (restore) 且來源指定為物理設備，導致 Partclone 誤以為你要從一個壞掉的備份檔還原。
    
- **解法**：請確保使用 `-c` (clone) 模式，或者使用 `partclone.dd` 指令格式。
    

### Q2: 備份出來的檔案大小為何看起來很大？

- **原因**：使用 `ls -lh` 看到的是「邏輯大小」（等於來源分區大小）。
    
- **解法**：使用 `du -sh backup.img` 查看「實際佔用空間」。因為 Partclone 跳過了空白區塊，且 Linux 支援稀疏檔案 (Sparse File)，實際佔用空間通常遠小於邏輯大小。
    

### Q3: 掛載時出現 `Structure needs cleaning`？

- **原因**：這是因為強行備份了掛載中的根目錄 (`-F`)。
    
- **解法**：請執行 `e2fsck -fy backup.img` 進行修復後再掛載。
    

---

## 5. 總結比較表

|**方案**|**工具與參數**|**速度**|**空間佔用**|**產出格式**|**可直接掛載?**|**備註**|
|---|---|---|---|---|---|---|
|**傳統備份**|`dd`|慢|極大 (含空白區塊)|Raw Image|✅ 是|最笨重但最通用|
|**標準 Partclone**|`partclone ... -c -s ... -o ...`|**快**|小 (僅有效資料)|.pcl (專有)|❌ 否|需還原才能用|
|**本指南方案**|`partclone ... -W`|**快**|中 (視稀疏支援)|**Raw Image**|**✅ 是**|**最佳平衡解**|
