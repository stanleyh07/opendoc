# 🧭 `hosts` 檔案用途與修改方式整理

## 📌 什麼是 `hosts` 檔案？

`hosts` 是一個本機 DNS 對應表，用來將主機名稱（domain name）對應到 IP 位址。它的優先順序高於 DNS 查詢，常用於：

- 測試網站部署（例如：將 `example.com` 指向本機 IP）
- 阻擋廣告或惡意網站（例如：將 `ads.example.com` 指向 `127.0.0.1`）
- 暫時繞過 DNS 設定或 CDN
- 建立內部網路名稱解析（例如：`myserver.local`）

---

## 🐧 Linux 下的 `hosts` 檔案

### 📍 路徑

```
/etc/hosts
```

### ✍️ 修改方式

```bash
sudo nano /etc/hosts
```

或使用其他文字編輯器（如 `vim`、`gedit`）。

### 📄 範例內容

```
127.0.0.1   localhost
192.168.1.100   myserver.local
```

### ✅ 注意事項

- 需要 `sudo` 權限才能修改。
- 修改後立即生效，無需重啟。

---

## 🪟 Windows 下的 `hosts` 檔案

### 📍 路徑

```
C:\Windows\System32\drivers\etc\hosts
```

### ✍️ 修改方式

#### 方法一：使用 Notepad（系統管理員身份）

1. 搜尋「Notepad」→ 右鍵 → 以系統管理員身份執行。
2. 開啟 `hosts` 檔案。
3. 編輯並儲存。

#### 方法二：使用 PowerShell（系統管理員身份）

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n127.0.0.1 test.local"
```

#### 方法三：使用 Python（需以管理員身份執行）

```python
hosts_path = r"C:\Windows\System32\drivers\etc\hosts"
entry = "\n127.0.0.1 test.local"

with open(hosts_path, "a", encoding="utf-8") as f:
    f.write(entry)
```

### 📄 範例內容

```
127.0.0.1   localhost
192.168.1.100   myserver.local
```

### ⚠️ 注意事項

- 必須使用系統管理員權限才能修改。
- 若使用非管理員工具，可能會觸發 VirtualStore 虛擬化，導致修改無效。
- 修改後不需重啟，但某些應用程式可能需重新啟動。

---

## 🧪 檢查是否成功修改

### Linux

```bash
cat /etc/hosts
```

### Windows（以系統管理員身份執行）

```powershell
Get-Content "C:\Windows\System32\drivers\etc\hosts"
```
