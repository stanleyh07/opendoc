Linux kernel 會產生 version magic 字串並加到 kernel image 以及 module ko 檔上，
當載入 module 時，會比對字串是否對應，如果沒應匹配，便無法載入。

但 scripts/setlocalversion 會使用 git 去產生附加於 kernel 版本後的字串，
若 放在 rootfs 下的 module 未同步與 kernel 更新，便會導致載入失敗。

解決方法是
可以在 kernel 目錄下，建立一個 .scmversion 空檔案，
根據 setlocalversion 中，會先檢查是否有 .scmversion 檔案，
若有，則以其內容代替，而不再使用 git 產生字串，
因此版本就會被固定成 kernel 版本 (Makefile 中定義的版本)

若要檢查 linux image 的 version magic string 
```
strings arch/arm64/boot/Image | grep 'Linux version'
```

若要檢查 module 的 version magic string
```
modinfo bcmdhd.ko | grep vermagic
```


