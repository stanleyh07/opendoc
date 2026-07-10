
## 🛠 安裝 uv

### macOS / Linux

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Windows (PowerShell)

```powershell
irm https://astral.sh/uv/install.ps1 | iex
```

安裝完成後，可以輸入：

```bash
uv --version
```

確認是否安裝成功。

---

## 🚀 使用 uvx

`uvx` 是 uv 的「快速執行工具」，可以直接執行套件而不用先安裝到環境。

### 基本語法

```bash
uvx <套件名稱> [參數]
```

### 常用範例

- **程式碼格式化工具 Black**
    
    ```bash
    uvx black myscript.py
    ```
    
- **Lint 工具 Ruff**
    
    ```bash
    uvx ruff check myscript.py
    ```
    
- **測試框架 Pytest**
    
    ```bash
    uvx pytest tests/
    ```
    
- **HTTP 請求工具 HTTPie**
    
    ```bash
    uvx httpie get https://example.com
    ```
    
- **型別檢查 Mypy**
    
    ```bash
    uvx mypy myscript.py
    ```
    

---

## 📌 小提醒

- 第一次執行會自動下載並快取套件，之後速度會更快。
- 適合臨時測試工具，不會污染你的 Python 環境。
- 如果常用某工具，可以再用 `uv add <套件>` 安裝到專案環境中。

---

