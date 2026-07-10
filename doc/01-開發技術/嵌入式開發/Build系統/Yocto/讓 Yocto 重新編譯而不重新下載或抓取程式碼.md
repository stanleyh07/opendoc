
要讓 Yocto 重新編譯而不重新下載或抓取程式碼，可以透過以下幾種方法進行控制：

### 1. **清理特定目標而非全部**
使用 `bitbake` 的清理命令來移除特定目標的已編譯文件，而保留下載好的檔案及原始碼：
```bash
bitbake -c clean <target-recipe>
```
執行後，再重新編譯：
```bash
bitbake <target-recipe>
```

這樣只清理目標的輸出檔案，避免重新下載或拉取原始碼。

---

### 2. **跳過下載階段**
如果你擔心 Yocto 可能會重新下載檔案，可以使用如下方式跳過：
- 指定只執行某些階段，例如從原始碼編譯（編譯階段開始於 `do_compile`）：
  ```bash
  bitbake -c compile <target-recipe>
  ```
- 或執行更高階段，例如打包：
  ```bash
  bitbake -c package <target-recipe>
  ```

這樣可以確保不觸發下載相關的步驟。

---

### 3. **使用 sstate-cache**
Yocto 的 sstate-cache（共享狀態緩存）會記錄已完成的任務。如果這些緩存仍可用且未被清理，Yocto 編譯時會跳過已完成的步驟（如下載和解壓縮）。確保未清理緩存：
```bash
bitbake -c cleansstate <target-recipe>  # 此命令會清理，但不要執行！
```

只需執行正常的編譯命令即可利用緩存：
```bash
bitbake <target-recipe>
```

---

### 4. **避免清理 build 資料夾**
如果你之前執行過會清理整個環境的指令（例如 `bitbake -c cleansstate` 或手動刪除 build 資料夾），Yocto 會重新下載和配置。因此，只要不主動執行清理指令，Yocto 預設會保留下載好的檔案與緩存，並基於現有環境重新編譯。

---

