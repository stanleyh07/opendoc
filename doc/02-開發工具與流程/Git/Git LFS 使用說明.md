以下是一份完整的 **Git LFS 使用說明文檔**，以 Markdown 格式撰寫，涵蓋安裝、設定、處理既有檔案，以及範例 `.gitattributes`。

---
## 1. 為什麼需要 Git LFS

- GitHub 對單一檔案有 **100MB 上限**，超過會拒絕 push。
- Git LFS（Large File Storage）透過「指標檔」管理大檔案，真正的檔案存放在 LFS 伺服器上。
- 適合用來管理模型檔案、影像、壓縮檔等大型二進位檔。

---

## 2. 安裝 Git LFS

- **Linux**
    
    ```bash
    sudo apt install git-lfs
    git lfs install
    ```
    
- **macOS**
    
    ```bash
    brew install git-lfs
    git lfs install
    ```
    
- **Windows** 安裝 Git LFS 套件後執行：
    
    ```bash
    git lfs install
    ```
    

---

## 3. 設定追蹤規則

Git LFS 透過 `.gitattributes` 檔案來指定哪些檔案要交由 LFS 管理。

### 📂 目錄規則

追蹤 `models/` 目錄下的所有檔案：

```bash
git lfs track "models/*"
```

`.gitattributes` 會新增：

```
models/* filter=lfs diff=lfs merge=lfs -text
```

### 📄 特定檔案規則

追蹤單一檔案：

```bash
git lfs track "models/best_model.pt"
git lfs track "models/large_checkpoint.bin"
```

`.gitattributes` 會新增：

```
models/best_model.pt filter=lfs diff=lfs merge=lfs -text
models/large_checkpoint.bin filter=lfs diff=lfs merge=lfs -text
```

### 🎯 混合範例

同時追蹤整個目錄與特定檔案：

```
models/* filter=lfs diff=lfs merge=lfs -text
models/best_model.pt filter=lfs diff=lfs merge=lfs -text
models/large_checkpoint.bin filter=lfs diff=lfs merge=lfs -text
```

> 📌 `.gitattributes` 必須 commit，讓其他人 clone 時也能正確使用 LFS。

---

## 4. 已存在檔案的處理方式

### 方案 A：只影響未來 commit

- 適合不想改動歷史紀錄的情況。
- 步驟：
    
    ```bash
    git add .gitattributes
    git commit -m "Track models directory with LFS"
    git push origin main
    ```
    
- 舊的 commit 中的大檔案仍然存在，只有未來新增或修改的檔案會用 LFS 管理。

---

### 方案 B：重寫歷史紀錄（完整轉換）

- 適合需要讓 **整個 repo 歷史都改用 LFS** 的情況。
- 步驟：
    
    ```bash
    git lfs migrate import --include="models/*"
    git push origin main --force
    ```
    
- 這會將歷史紀錄中的大檔案轉換成 LFS 物件。
- ⚠️ **注意**：強制推送會改變 commit 歷史，團隊成員需要重新 clone 或 rebase。

---

## 5. 推送到 GitHub

完成設定後，推送即可：

```bash
git push origin main
```

GitHub 會接收 LFS 物件，repo 中只存指標檔。

---

## 6. 注意事項

- GitHub LFS 單檔上限為 **2GB**。
- 團隊成員必須安裝 Git LFS，否則 clone 下來的檔案只會是指標檔。
- 建議在 `.gitattributes` 中明確列出需要 LFS 的檔案類型，避免遺漏。
- 若要轉換舊檔案，務必先在分支測試，確認無誤再合併到主要分支。

---

## 7. 範例 `.gitattributes`

以下是一個適合 ML 專案的範例設定：

```
# 追蹤整個 models 目錄
models/* filter=lfs diff=lfs merge=lfs -text

# 追蹤特定模型檔案
models/best_model.pt filter=lfs diff=lfs merge=lfs -text
models/large_checkpoint.bin filter=lfs diff=lfs merge=lfs -text

# 常見 ML 模型檔案類型
*.pt filter=lfs diff=lfs merge=lfs -text
*.bin filter=lfs diff=lfs merge=lfs -text
*.onnx filter=lfs diff=lfs merge=lfs -text
*.h5 filter=lfs diff=lfs merge=lfs -text
```

---

✅ 這份文檔提供了 **兩種操作方案**：

- **方案 A**：只影響未來 commit，安全但舊檔案仍存在。
- **方案 B**：重寫歷史紀錄，乾淨但需要強制推送。

並附上範例 `.gitattributes`，方便直接套用。