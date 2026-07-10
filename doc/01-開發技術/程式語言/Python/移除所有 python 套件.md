
要完全移除所有 Python 套件並重新安裝，你可以按照以下步驟來操作。請注意，這會刪除所有使用 `pip` 安裝的套件，請先備份重要的 Python 環境！

### **步驟 1️⃣：移除所有 Python 套件**
執行以下指令，卸載所有 `pip` 安裝的套件：
```sh
pip3 freeze | xargs pip3 uninstall -y
```
這將讀取所有已安裝的套件並逐一卸載。

### **步驟 2️⃣：清理 `site-packages`**
如果某些套件未完全移除，手動刪除 Python `site-packages` 目錄：
```sh
rm -rf /home/arbor/.local/lib/python3.10/site-packages/*
rm -rf ~/.cache/pip
```

### **步驟 3️⃣：重新安裝 `pip`**
確保 `pip` 自身完整：
```sh
python3 -m ensurepip --default-pip
pip3 install --upgrade pip
```

### **步驟 4️⃣：重新安裝必要套件**
如果你的開發環境需要特定的套件，重新安裝：
```sh
pip3 install -U setuptools wheel
```
然後重新安裝 `genio-tools`：
```sh
pip3 install -U genio-tools
```

### **步驟 5️⃣：驗證安裝**
確認 Python 環境是否正確：
```sh
python3 --version
pip3 --version
pip3 list
genio-config
```

執行完這些步驟後，你應該能夠重新使用 `genio-flash`，如果仍有問題，請提供錯誤訊息！🚀
