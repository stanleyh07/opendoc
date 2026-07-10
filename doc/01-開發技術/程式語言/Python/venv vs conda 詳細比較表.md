
## 📊 venv vs conda 詳細比較表

|面向|venv|conda|
|---|---|---|
|**工具來源**|Python 標準庫內建 (≥3.3)|Anaconda/Miniconda 提供|
|**定位**|輕量級虛擬環境工具，專注於 Python 套件隔離|跨語言環境與套件管理器，支援 Python 以外的依賴|
|**套件管理**|使用 `pip`|使用 `conda`，也可混合 `pip`|
|**依賴範圍**|僅限 Python 套件|Python + C/C++、R、系統庫等|
|**環境建立**|`python -m venv myenv`|`conda create --name myenv python=3.9`|
|**啟用環境**|- Linux/macOS: `source myenv/bin/activate`  <br>- Windows: `.\myenv\Scripts\Activate.ps1`|`conda activate myenv`|
|**退出環境**|`deactivate`|`conda deactivate`|
|**安裝套件**|`pip install requests`|`conda install numpy` 或 `pip install requests`|
|**環境匯出**|`pip freeze > requirements.txt`|`conda env export > environment.yml`|
|**環境重建**|`pip install -r requirements.txt`|`conda env create -f environment.yml`|
|**環境更新**|手動修改 `requirements.txt` 再安裝|`conda env update -f environment.yml --prune`|
|**適合場景**|- 純 Python 專案  <br>- 輕量需求  <br>- CI/CD pipeline|- 資料科學、機器學習  <br>- 跨語言依賴  <br>- 快速安裝大型套件|
|**優勢**|- 內建，無需額外安裝  <br>- 輕量、簡單|- 支援跨語言依賴  <br>- 提供預編譯套件，安裝速度快  <br>- 可完整匯出/重建環境|
|**限制**|- 僅限 Python  <br>- 需自行處理系統依賴|- 需安裝 Anaconda/Miniconda  <br>- 環境較重|

---

## 🎯 總結

- **venv** → 適合純 Python 專案，簡單輕量，搭配 `requirements.txt`。
- **conda** → 適合資料科學、跨語言依賴，能用 `environment.yml` 快速複製環境。
