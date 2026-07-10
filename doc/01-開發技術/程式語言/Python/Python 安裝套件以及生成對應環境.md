## 🐍 使用 `pip`

- **基本方式**
    
    ```bash
    pip freeze > requirements.txt
    ```
    
    - `pip freeze` 會列出所有已安裝套件及版本。
    - 輸出導向到 `requirements.txt` 即可。
- **只針對虛擬環境**  
    如果你在 `venv` 或 `virtualenv` 中執行，這樣匯出的清單只包含該環境的套件。
    

---

## 📦 使用 `pipreqs`

- **用途**：只匯出專案實際有 import 的套件，而不是整個環境。
- 安裝：
    
    ```bash
    pip install pipreqs
    ```
    
- 使用：
    
    ```bash
    pipreqs /path/to/project --force
    ```
    
    - 會在專案目錄生成 `requirements.txt`，只包含程式碼中用到的套件。

---

## 🛠 使用 `conda` (如果你用 Anaconda/Miniconda)

- 匯出完整環境：
    
    ```bash
    conda list --export > requirements.txt
    ```
    
- 或更常見：
    
    ```bash
    conda env export > environment.yml
    ```
    
    - 這會生成 `environment.yml`，比 `requirements.txt` 更完整，包含 Python 版本與 channel。

---

## 📑 使用 `pip-tools`

- 安裝：
    
    ```bash
    pip install pip-tools
    ```
    
- 使用：
    
    ```bash
    pip-compile
    ```
    
    - 會根據 `requirements.in` 生成精確版本的 `requirements.txt`。

---

### ⚖️ 方法比較表

|方法|匯出範圍|特點|
|---|---|---|
|`pip freeze`|全部套件|最常用，簡單直接|
|`pipreqs`|實際 import 套件|避免多餘套件，適合專案交付|
|`conda env export`|Conda 環境|包含 Python 版本與 channel|
|`pip-compile`|依需求檔生成|適合維護大型專案依賴|
