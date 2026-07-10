`cpio` 的指令範例：

1. **建立 cpio 備份檔案**： 假設您有一些 `.txt` 檔案需要備份，您可以使用以下指令將它們備份至 `archive.cpio`：
    
    ```bash
    ls *.txt | cpio -ov > archive.cpio
    ```
    
    這會將列出的 `.txt` 檔案備份到 `archive.cpio` 中。
    
2. **解開 cpio 備份檔案**： 若要解壓縮 `archive.cpio`，您可以使用以下指令：
    
    ```bash
    cpio -iv < archive.cpio
    ```
    
    這將解壓縮檔案並還原到原始目錄結構中。
    
3. **保留檔案變更日期**： 如果您想保留備份檔案的變更日期，可以加上 `-m` 參數：
    
    ```bash
    cpio -idmv < archive.cpio
    ```