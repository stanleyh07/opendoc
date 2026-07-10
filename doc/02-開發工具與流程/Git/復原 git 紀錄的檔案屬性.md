如何保留修改內容，並復原 git 紀錄的檔案屬性

利用指令取得 git 紀錄的檔案屬性

```sh
git ls-files -s

100644 50a4cd8cac42bf8a81020eb37201eaae4e6f16bd 0	rk356x.prop
100644 9a320dfde73de028e325676093b922935cf245de 0	sepolicy_vendor/file_contexts
100644 5fbc76a78d877c24356666b5ac2ce62f9d42042b 0	sepolicy_vendor/genfs_contexts
100644 ffca0da8fca42b6f37efa1610fabc5bdb72f6d1a 0	sepolicy_vendor/vendor_init.te
100644 3ac4d94fa89517ab9e9d93f5bb6ebaf158251c23 0	wake_lock_filter.xml
100644 6d537925603815895ea5739b82be84bc6f3d8fa3 0	wifi_bt.mk
```
搭配 awk 取的屬性與檔名

```sh
git ls-files -s | awk '{print $1, $4}'

100644 rk3568_t/rk3568_t.mk
100644 rk356x.prop
100644 sepolicy_vendor/file_contexts
100644 sepolicy_vendor/genfs_contexts
100644 sepolicy_vendor/vendor_init.te
100644 wake_lock_filter.xml
100644 wifi_bt.mk
```

過濾掉 link 檔案不需要變更屬性，以及改成 chmod 可以用的參數

```sh
git ls-files -s | awk '$1 != "120000" {print substr($1, 4), $4}'

644 rk356x.prop
644 sepolicy_vendor/file_contexts
644 sepolicy_vendor/genfs_contexts
644 sepolicy_vendor/vendor_init.te
644 wake_lock_filter.xml
644 wifi_bt.mk
```

搭配 xargs 取得參數，並用 chmod 變更屬性

```sh

git ls-files -s | awk '$1 != "120000" {print substr($1, 4), $4}' | xargs -I{} sh -c 'set -- {}; chmod $1 "$2"'
```

這部分會使用 `xargs` 和 `chmod` 根據 Git 紀錄來變更檔案屬性：

- `xargs -I{} sh -c 'set -- {}; chmod $1 "$2"'`：對每一行輸出執行一個新的 `sh` 命令。
    - `-I{}`：指定替換字串 `{}`。
    - `sh -c 'set -- {}; chmod $1 "$2"'`：執行一個新的 `sh` 命令，其中 `set -- {}` 會將 `{}` 替換為輸入行的內容（即檔案名稱和屬性），然後 `chmod $1 "$2"` 會根據屬性 `$1` 變更檔案 `$2` 的屬性。

利用 repo forall 對所有 repository 進行處理

```sh

repo forall -c 'git ls-files -s | awk '\''$1 != "120000" {print substr($1, 4), $4}'\'' | xargs -I{} sh -c '\''set -- {}; chmod $1 "$2"'\'''
```

在你的指令中，反斜線主要用於轉義單引號，使得內部的單引號不會結束外部的單引號字符串：
這裡的反斜線用於轉義內部的單引號，使得 `awk` 和 `sh -c` 的命令可以正確解析。
參考[[01-開發技術/程式語言/Bash/在 Bash 指令中反斜線用法]]

單引號 (`'`) 內的內容會被視為字面值，這意味著所有字符都會被原樣保留，不會進行轉義。因此，如果你需要在單引號內包含單引號，必須先結束當前的單引號字符串，然後使用反斜線轉義單引號，再重新開始一個新的單引號字符串。

例如：

```bash
'awk '\''$1 != "120000" {print $4, substr($1, 4)}'\'
```

這裡的 `'\''` 是這樣解析的：

1. 第一個單引號結束了前面的字符串。
2. 反斜線轉義了單引號，使其成為字面值單引號。
3. 第二個單引號開始了一個新的字符串。

這樣做的目的是讓內部的單引號不會結束外部的單引號字符串。