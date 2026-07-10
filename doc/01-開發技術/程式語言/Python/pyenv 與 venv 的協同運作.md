# Python 環境管理深度指南：pyenv 與 venv 的協同運作

當你的系統預設為 **Python 3.8**，而專案需求為 **Python 3.10** 並安裝 `ultralytics` 時，必須區分「解釋器管理」與「套件管理」兩個層次。

---

## 1. 第一層：解釋器管理 (pyenv)

`pyenv` 的任務是讓你的電腦「同時擁有多個 Python 執行檔」。

- **系統現況**：`/usr/bin/python` -> 指向 3.8。
    
- **使用 pyenv 後**：在 `~/.pyenv/versions/3.10.13/bin/python` 產生 3.10 的執行檔。
    
- 自動化關鍵 (Shims)：
    
    pyenv 會在 PATH 環境變數最前面插入一個「墊片 (Shim)」。當你執行 python 時：
    
    1. 系統先呼叫 `pyenv shim`。
        
    2. Shim 讀取目錄下的 `.python-version`。
        
    3. Shim 決定該啟動 3.8 還是 3.10。
        

---

## 2. 第二層：套件隔離環境 (venv)

當 `pyenv` 幫你選好 **3.10** 之後，我們執行 `python -m venv .venv`。這一步至關重要，它做了以下詳細動作：

### `venv` 建立時的詳細內容：

當你執行建立指令後，`.venv` 資料夾內會包含：

- **`bin/` (或 Windows 的 `Scripts/`)**：
    
    - **`python` 符號連結**：這不是複製品，而是一個指向 `pyenv 3.10` 執行檔的連結。
        
    - **`activate` 腳本**：這是一個 Shell 腳本，執行它會暫時修改你的 `PATH`，將 `.venv/bin` 放在最前面。
        
- **`lib/python3.10/site-packages/`**：
    
    - 這是**最核心的部分**。當你執行 `pip install ultralytics` 時，套件會被安裝在這裡，而**不會**裝進 `pyenv` 的全局目錄。
        
- **`pyvenv.cfg`**：
    
    - 這是一個純文字檔，裡面有一行 `include-system-site-packages = false`。這保證了你的虛擬環境是「純淨的」，不會讀取到系統 3.8 裝過的任何套件。
        

---

## 3. 實戰流程：從系統 3.8 到專案 3.10

### 步驟一：確保擁有 3.10 解釋器

Bash

```
pyenv install 3.10.13
cd vision_project
pyenv local 3.10.13  # 此時 python -V 會顯示 3.10
```

### 步驟二：建立與啟動 venv

Bash

```
python -m venv .venv
source .venv/bin/activate
```

**啟動後發生的變化：**

1. 你的 Prompt 前面會出現 `(.venv)`。
    
2. 指令 `which python` 會從 `~/.pyenv/shims/python` 變成 `/your/path/vision_project/.venv/bin/python`。
    
3. 此時安裝任何套件，物理位置都在專案資料夾內。
    

### 步驟三：安裝 Ultralytics

Bash

```
pip install ultralytics
```

---

## 4. 進階細節：Shebang (`#!`) 與路徑搜尋

在你的 `vision_project` 中，如果你有一個 `main.py`：

Python

```
#!/usr/bin/env python
import ultralytics
print("成功使用 3.10 環境")
```

### 執行路徑分析：

1. 若已啟動 venv (source .venv/bin/activate)：
    
    env 找到的 python 是 .venv/bin/python -> 成功執行 3.10 並讀取到套件。
    
2. 若未啟動 venv 但在目錄內：
    
    env 找到的 python 是 pyenv shim -> 讀取 .python-version -> 成功執行 3.10，但會報錯找不到 ultralytics（因為套件裝在 venv 裡）。
    
3. 若擋頭寫死 #!/usr/bin/python：
    
    直接執行系統 3.8 -> 失敗，版本不符且無套件。
    

---

## 5. 總結對照表

|**階段**|**負責工具**|**解決的問題**|**關鍵檔案**|
|---|---|---|---|
|**版本切換**|`pyenv`|解決「系統 3.8」與「專案 3.10」的衝突|`.python-version`|
|**路徑引導**|`pyenv shim`|離開目錄自動變回 3.8 的魔法來源|`~/.pyenv/shims/`|
|**套件隔離**|`venv`|確保 `ultralytics` 不會污染其他 3.10 專案|`.venv/lib/site-packages`|
|**環境啟動**|`activate`|讓 Shell 優先搜尋專案內的 Python 與套件|`.venv/bin/activate`|
