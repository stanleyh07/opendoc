若是有多個 commit 資訊有誤，需要修改資訊，比如修改 author，要如何處理?
使用 git filter-branch 指令可以達到這樣的目的

使用 `git filter-branch --env-filter` 指令時，可以使用並修改以下環境變數：

1. **GIT_AUTHOR_NAME**：作者的名稱。
2. **GIT_AUTHOR_EMAIL**：作者的電子郵件地址。
3. **GIT_AUTHOR_DATE**：提交的日期和時間。
4. **GIT_COMMITTER_NAME**：提交者的名稱。
5. **GIT_COMMITTER_EMAIL**：提交者的電子郵件地址。
6. **GIT_COMMITTER_DATE**：提交的日期和時間。

這些變數允許你在過濾分支時修改提交的元數據。
比如，使用錯誤的帳號 checkin 多個 commit，如何修改?
使用腳本去處理！以下是 `git-author-rewrite.sh` 腳本的詳細說明：

### 腳本內容

```sh
#!/bin/sh

git filter-branch --env-filter '
OLD_EMAIL="舊郵箱"
CORRECT_NAME="新名字"
CORRECT_EMAIL="新郵箱"

if [ "$GIT_COMMITTER_NAME" = "$OLD_NAME" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_NAME" = "$OLD_NAME" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

### 腳本說明

1. **Shebang (`#!/bin/sh`)**: 指定腳本使用 `/bin/sh` 來執行。
    
2. **`git filter-branch --env-filter`**: 使用 `git filter-branch` 命令來重寫 Git 的歷史記錄。`--env-filter` 選項允許我們修改環境變量（如作者和提交者的信息）。
    
3. **環境變量設置**:    
    - `OLD_EMAIL`：舊的郵箱地址。
    - `CORRECT_NAME`：新的名字。
    - `CORRECT_EMAIL`：新的郵箱地址。
4. **條件判斷**:
    
    - 如果提交者的郵箱地址 (`$GIT_COMMITTER_EMAIL`) 與 `OLD_EMAIL` 相同，則修改提交者的名字和郵箱地址。
    - 如果作者的郵箱地址 (`$GIT_AUTHOR_EMAIL`) 與 `OLD_EMAIL` 相同，則修改作者的名字和郵箱地址。
5. **`--tag-name-filter cat`**: 保持標籤名稱不變。
    
6. **`-- --branches --tags`**: 對所有分支和標籤應用這些更改。
    

執行就會進行修改，這個機制也可以在 bare 倉庫中進行，若需要強制推送回遠程倉庫，

**強制推送到遠程倉庫**： 完成修改後，強制推送到遠程倉庫：
    
```sh
git push --force --tags origin 'refs/heads/*'
```

