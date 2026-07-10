關於 Linux 密碼複雜度設定整理，我已依照邏輯與操作流程重新編排並補充說明，使內容更清晰：

---

## 🔐 Linux 密碼複雜度與繞過方式全紀錄

### 📌 問題起點

你問到：「使用 `passwd` 設定密碼時，會碰到長度與複雜度限制，有沒有辦法忽略這些限制，強制設定指定密碼？」

---

### ✅ 解法一：直接修改 PAM 密碼策略

1. 編輯 `/etc/pam.d/common-password`：
    
    ```bash
    sudo nano /etc/pam.d/common-password
    ```
    
2. 找到包含 `pam_unix.so` 的那一行，並修改為：
    
    ```bash
    password [success=1 default=ignore] pam_unix.so sha512 minlen=1
    ```
    
    - 移除 `obscure`、`pam_pwquality.so` 或 `pam_cracklib.so`。
    - 設定 `minlen=1`，放寬密碼長度限制。
3. 儲存後即可設定像 `123` 這類簡單密碼。
    

---

### ❓進一步追問：「只想讓某個帳號不受限制，其他仍維持原本複雜度」

#### ✔ 方法一：使用 `chpasswd` 搭配 `--encrypted`

```bash
echo 'username:$(openssl passwd -6 "123")' | sudo chpasswd -e
```

此方法直接將加密密碼寫入 `/etc/shadow`，繞過 PAM 密碼檢查。

#### ✔ 方法二：手動編輯 `/etc/shadow`

產生密碼雜湊：

```bash
openssl passwd -6 "123"
```

然後用 `sudo nano /etc/shadow` 將目標使用者那行的密碼欄位改成該雜湊。

---

### 🔁 追問：「能否暫時修改 `common-password`，設定完密碼後再改回來？」

可以，這是常見的 DevOps 技巧。建議操作流程如下：

#### ⛓ 流程簡述：

1. 備份原始檔案
2. 暫時放寬限制
3. 設定密碼
4. 還原原始密碼策略

---

### 🧪 完整腳本示範

```bash
#!/bin/bash

USERNAME="target_user"         # 修改成你的使用者名稱
PASSWORD="123"                 # 指定密碼
BACKUP="/etc/pam.d/common-password.bak"

# 備份原始密碼策略
sudo cp /etc/pam.d/common-password "$BACKUP"

# 放寬限制：移除 obscure，設定 minlen=1
sudo sed -i 's/obscure//g;s/\(pam_unix.so.*\)/\1 minlen=1/' /etc/pam.d/common-password

# 設定簡單密碼
echo "$USERNAME:$PASSWORD" | sudo chpasswd

# 還原密碼策略
sudo mv "$BACKUP" /etc/pam.d/common-password

echo "✅ 密碼已設定成功並還原原始策略"
```

---

### 🛡 注意事項

- 所有方法都繞過了 PAM 檢查，請在封閉或測試環境中使用。
- `chpasswd -e` 是最安全且不干擾全域策略的方式。
- 修改 `/etc/shadow` 有風險，請務必備份。

---

這份紀錄希望有助於你後續針對 Ubuntu/Linux 密碼策略的管理。  
如果你想進一步自動化或整合至部署流程，我可以幫你設計模組化腳本，甚至搭配 Docker 或 CI/CD 工具整合。

要不要我幫你再封裝成一個 CLI 工具？💡