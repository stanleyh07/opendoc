### 📌 `/proc/config.gz` 的來源與生成方式

- **編譯時嵌入**  
    當你編譯 Linux kernel 時，會使用 `.config` 檔案來定義所有內核選項。  
    如果在內核設定中啟用了 **`CONFIG_IKCONFIG=y`**，編譯過程就會把 `.config` 內容嵌入到內核映像檔 (`vmlinux`) 裡。
    
- **壓縮存放**  
    內核會將 `.config` 內容以 **gzip 格式壓縮**，存放在內核的特殊區段中。
    
- **Proc 節點呈現**  
    如果同時啟用了 **`CONFIG_IKCONFIG_PROC=y`**，內核在執行時會透過 **procfs** 暴露這份壓縮的設定檔，形成 `/proc/config.gz` 這個節點。  
    這樣使用者就能直接透過 `zcat /proc/config.gz` 或 `zgrep` 來查看編譯時的內核設定。
    

---

### ⚙️ 條件限制

- 並非所有發行版都會提供 `/proc/config.gz`，例如 Ubuntu 與部分發行版預設沒有啟用這兩個選項。
- 如果 `/proc/config.gz` 不存在，可以改從：
    - `/boot/config-$(uname -r)`
    - `/lib/modules/$(uname -r)/build/.config`  
        找到對應的設定檔。

---

### 🔧 其他提取方式

- 使用 **`scripts/extract-ikconfig`** 工具，可以從 `vmlinux` 或模組中解析出嵌入的 `.config`。
- 在除錯環境下，透過 `gdb` 載入 `vmlinux-gdb.py`，並使用 `lx-configdump` 指令，也能 dump 出內核設定。

---

### 📖 總結

`/proc/config.gz` 的生成流程是：  
**編譯時嵌入 → 內核壓縮儲存 → 執行時透過 procfs 暴露**。  
它本質上就是編譯內核時的 `.config` 檔案副本，前提是啟用了 **`CONFIG_IKCONFIG`** 與 **`CONFIG_IKCONFIG_PROC`**。
