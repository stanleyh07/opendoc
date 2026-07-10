# 輕量級 Git 伺服器架設與安全配置指南

本文件彙整了在 Linux 系統上建立專用 Git Server 的完整流程。從帳號建立、SSH 金鑰管理腳本、獨立的安全配置，到自訂儲存庫路徑與權限異常的故障排除，提供具體且立即可用的操作步驟。

## 1. 建立 Git 專用帳號與安全限制

為確保伺服器安全，Git 帳號的 Shell 將被限制為 `git-shell`，使其僅能進行 Git 相關操作，無法登入終端機。

```bash
# 更新並安裝 git
sudo apt-get update && sudo apt-get install git -y

# 確認 git-shell 路徑並加入系統允許的 shell 列表
which git-shell | sudo tee -a /etc/shells

# 建立 git 使用者，並將登入 shell 設為 git-shell
sudo useradd -m -s $(which git-shell) git
```

## 2. 初始化 SSH 金鑰設定

為 git 帳號建立專屬的 SSH 目錄，並嚴格限制權限。此帳號不設定密碼，僅允許 SSH Key 登入。

```bash
# 建立目錄與 authorized_keys 檔案
sudo mkdir -p /home/git/.ssh
sudo touch /home/git/.ssh/authorized_keys

# 設定正確的權限與擁有者 (極度重要，設定錯誤會導致 SSH 拒絕連線)
sudo chmod 700 /home/git/.ssh
sudo chmod 600 /home/git/.ssh/authorized_keys
sudo chown -R git:git /home/git/.ssh
```

## 3. 建立 SSH Key 管理腳本

為簡化日常管理，我們建立一個全域腳本 `add-git-key`，讓管理者可以快速將開發者的公鑰加入 Git 帳號中。

```bash
sudo tee /usr/local/bin/add-git-key > /dev/null << 'EOF'
#!/bin/bash
if [ "$#" -ne 2 ]; then
    echo "用法: sudo add-git-key <開發者名稱> "<ssh-rsa ... 公鑰字串>""
    exit 1
fi

AUTH_FILE="/home/git/.ssh/authorized_keys"

# 寫入註解與公鑰，並確保權限正確
echo "# Added for $1" >> "$AUTH_FILE"
echo "$2" >> "$AUTH_FILE"
chown git:git "$AUTH_FILE"
chmod 600 "$AUTH_FILE"

echo "✅ 已成功將 $1 的公鑰加入 Git 伺服器。"
EOF

# 賦予腳本執行權限
sudo chmod +x /usr/local/bin/add-git-key
```

### 移除 SSH 公鑰腳本

```bash
sudo tee /usr/local/bin/remove-git-key > /dev/null << 'EOF'
#!/bin/bash
if [ "$#" -ne 1 ]; then
    echo "用法: sudo remove-git-key <開發者名稱>"
    exit 1
fi

AUTH_FILE="/home/git/.ssh/authorized_keys"
COMMENT="# Added for $1"

if ! grep -qF "$COMMENT" "$AUTH_FILE"; then
    echo "❌ 找不到開發者 '$1' 的公鑰記錄。"
    exit 1
fi

# 移除開發者的註解行與下一行的公鑰
sed -i "/^$(sed 's/[&/\]/\\&/g' <<< "$COMMENT")$/{N;d;}" "$AUTH_FILE"

echo "✅ 已成功移除 $1 的公鑰。"
EOF

# 賦予腳本執行權限
sudo chmod +x /usr/local/bin/remove-git-key
```

**使用範例：**
```bash
# 新增公鑰
sudo add-git-key "Alice" "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... alice@laptop"

# 移除公鑰（依開發者名稱）
sudo remove-git-key "Alice"
```

## 4. 強化 SSH 伺服器安全設定

透過 `sshd_config.d` 目錄，為 git 帳號加入專屬的 SSH 限制，徹底禁用密碼與通道轉發，且不影響系統上的其他一般使用者。

> [!warning] ⚠️ 關鍵注意
> 在使用 Include 目錄設定 Match 規則時，檔案末尾必須加上 `Match All` 重置作用域，否則將導致後續的主設定檔規則錯誤地只套用於 git 帳號。

```bash
# 建立獨立的 SSH 設定檔 (以 99 開頭確保在排序後方載入)
sudo tee /etc/ssh/sshd_config.d/99-git.conf > /dev/null << 'EOF'
Match User git
    PasswordAuthentication no
    PubkeyAuthentication yes
    AllowTcpForwarding no
    AllowAgentForwarding no
    X11Forwarding no

# 必須加上這行！用來重置 Match 範圍
Match All
EOF

# 測試設定檔語法是否正確
sudo sshd -t

# 確認無誤後重新啟動 SSH 服務
sudo systemctl restart sshd
```

## 5. 自訂儲存庫存放路徑 (軟連結與短網址)

為達成以 `git clone git@<IP>:repos/rk3562/manifests.git` 這樣的短網址進行 Clone，同時將實體檔案存放在外部硬碟 (例如 `/media/ssd2`) 中，可利用軟連結來實作：

```bash
# 1. 在 SSD 建立實體目錄並賦予 git 擁有權
sudo mkdir -p /media/ssd2/repos/rk3562
sudo chown -R git:git /media/ssd2/repos

# 2. 在 git 家目錄下建立軟連結
sudo -u git ln -s /media/ssd2/repos /home/git/repos

# 3. 在 SSD 目錄中初始化 Bare 儲存庫
sudo -u git git init --bare /media/ssd2/repos/rk3562/manifests.git
```

## 6. 排除 Git 目錄擁有權安全警告 (CVE-2022-24765)

當存取掛載於外部路徑 (如 `/media/ssd2`) 的儲存庫時，可能會觸發 Git 的 `dubious ownership` 安全保護，導致無法存取。

由於我們已經嚴格限制了 git 帳號的權限，最直接的解法是將該帳號可存取的所有目錄加入信任清單：

```bash
# 以 git 身分寫入全域設定，信任所有目錄
sudo -u git git config --global --add safe.directory '*'

# (選用) 確保上層掛載目錄允許 git 使用者穿透
sudo chmod 755 /media/ssd2
```

完成上述步驟後，用戶即可順利在客戶端透過短網址執行 `git clone`、`push` 與 `pull` 操作。

若需要簡單的 webui，可以參閱 [[02-開發工具與流程/Git/Cgit on Docker 部署指南]]
