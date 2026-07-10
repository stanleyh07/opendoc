## 🗂 常見 Linux Kernel Config 與對應檔案系統

|Kernel Config 選項|對應檔案系統|說明|
|---|---|---|
|`CONFIG_EXT4_FS`|ext4|最常用的 Linux 本地檔案系統，支援 journaling、extents。|
|`CONFIG_EXT3_FS`|ext3|舊版 journaling 檔案系統，逐漸被 ext4 取代。|
|`CONFIG_EXT2_FS`|ext2|無 journaling，適合小型或嵌入式系統。|
|`CONFIG_XFS_FS`|XFS|高效能 journaling 檔案系統，適合大容量儲存。|
|`CONFIG_BTRFS_FS`|Btrfs|支援快照、壓縮、子卷，設計為下一代 Linux FS。|
|`CONFIG_F2FS_FS`|F2FS|為 NAND/SSD 儲存優化的檔案系統。|
|`CONFIG_REISERFS_FS`|ReiserFS|舊式 journaling FS，現已少用。|
|`CONFIG_JFS_FS`|JFS|IBM 開發，適合大檔案處理。|
|`CONFIG_NILFS2_FS`|NILFS2|支援持續快照的 log-structured FS。|
|`CONFIG_OVERLAY_FS`|OverlayFS|容器常用的 union 檔案系統。|
|`CONFIG_TMPFS`|tmpfs|記憶體中的暫存檔案系統。|
|`CONFIG_PROC_FS`|procfs|提供系統資訊的虛擬檔案系統。|
|`CONFIG_SYSFS`|sysfs|匯出 kernel 物件資訊。|
|`CONFIG_NFS_FS`|NFS|網路檔案系統，常用於伺服器共享。|
|`CONFIG_CIFS`|CIFS/SMB|Windows 相容的網路檔案系統。|
|`CONFIG_ISO9660_FS`|ISO9660|光碟檔案系統。|
|`CONFIG_UDF_FS`|UDF|DVD/藍光常用檔案系統。|
|`CONFIG_VFAT_FS`|FAT32/VFAT|與 Windows 相容的 FAT 系列。|
|`CONFIG_EXFAT_FS`|exFAT|微軟 exFAT，適合 SD 卡、大容量隨身碟。|

---

## 🔧 如何查看目前系統支援的檔案系統

1. **檢查 kernel config**
    
    ```bash
    zcat /proc/config.gz | grep CONFIG_*FS
    ```
    
    或者：
    
    ```bash
    cat /boot/config-$(uname -r) | grep CONFIG_*FS
    ```
    
2. **檢查已載入的檔案系統模組**
    
    ```bash
    cat /proc/filesystems
    ```
    
3. **檢查模組清單**
    
    ```bash
    lsmod | grep fs
    ```
    

---

## 📌 補充說明

- **核心內建 vs 模組**：某些檔案系統可編譯為核心內建 (`=y`) 或模組 (`=m`)。
- **虛擬檔案系統 (VFS)**：如 `procfs`、`sysfs`、`tmpfs` 並非真正的磁碟檔案系統，而是提供系統資訊或暫存用途。
- **嵌入式/特殊用途**：如 `CONFIG_SQUASHFS` (壓縮唯讀 FS)、`CONFIG_CRAMFS` (小型唯讀 FS)，常用於嵌入式設備。

---
## 📊 Linux 檔案系統比較表

| 檔案系統 (CONFIG)                       | 效能  | 可靠性 | 特性                      | 適用場景                |
| ----------------------------------- | --- | --- | ----------------------- | ------------------- |
| **ext4 (`CONFIG_EXT4_FS`)**         | 中高  | 高   | journaling、extents、延遲分配 | 通用，伺服器與桌機預設         |
| **XFS (`CONFIG_XFS_FS`)**           | 高   | 高   | 高度可擴展、支援大檔案             | 大容量儲存、資料庫           |
| **Btrfs (`CONFIG_BTRFS_FS`)**       | 中   | 中   | 快照、壓縮、子卷、RAID           | 容器、雲端、測試環境          |
| **F2FS (`CONFIG_F2FS_FS`)**         | 高   | 中   | 為 NAND/SSD 優化           | 手機、嵌入式、SSD          |
| **ext2 (`CONFIG_EXT2_FS`)**         | 中   | 低   | 無 journaling            | 小型或嵌入式系統            |
| **ReiserFS (`CONFIG_REISERFS_FS`)** | 中   | 中   | 高效小檔案處理                 | 舊系統，現已少用            |
| **JFS (`CONFIG_JFS_FS`)**           | 中高  | 高   | IBM 開發，低 CPU 使用率        | 大檔案處理               |
| **NILFS2 (`CONFIG_NILFS2_FS`)**     | 中   | 中   | 持續快照、log-structured     | 特殊需求系統              |
| **OverlayFS (`CONFIG_OVERLAY_FS`)** | 中   | 中   | union FS，支援多層 overlay   | 容器 (Docker, Podman) |
| **tmpfs (`CONFIG_TMPFS`)**          | 高   | 中   | 記憶體檔案系統                 | 暫存檔、快取              |
| **procfs (`CONFIG_PROC_FS`)**       | N/A | N/A | 虛擬 FS，提供系統資訊            | 系統管理                |
| **sysfs (`CONFIG_SYSFS`)**          | N/A | N/A | 匯出 kernel 物件資訊          | 驅動程式、系統工具           |
| **NFS (`CONFIG_NFS_FS`)**           | 中   | 中   | 網路檔案系統                  | 伺服器共享               |
| **CIFS/SMB (`CONFIG_CIFS`)**        | 中   | 中   | Windows 相容              | 異質環境共享              |
| **ISO9660 (`CONFIG_ISO9660_FS`)**   | 低   | 高   | 光碟檔案系統                  | CD/DVD              |
| **UDF (`CONFIG_UDF_FS`)**           | 中   | 高   | DVD/藍光檔案系統              | 光碟媒體                |
| **VFAT (`CONFIG_VFAT_FS`)**         | 中   | 中   | FAT32 相容                | USB、SD 卡            |
| **exFAT (`CONFIG_EXFAT_FS`)**       | 中   | 中   | 微軟 exFAT                | 大容量隨身碟、SDXC         |
|                                     |     |     |                         |                     |

---

## 🔍 建議 workflow

- **伺服器/桌機**：ext4 或 XFS
- **容器/雲端**：OverlayFS + Btrfs
- **嵌入式/手機**：F2FS 或 ext2
- **跨平台共享**：NFS、CIFS、VFAT、exFAT