1. 使用 apt list --installed 去取的安裝的資訊
   並過濾，只取 package name
2. 利用 apt download 去載 package，不安裝
3. 最後程式內容如下
```
#!/bin/bash

# Query installed packages
installed_packages=$(apt list --installed | cut -d'/' -f1 | uniq)

# Download each installed package
for package in $installed_packages; do
    apt download "$package"
done

echo "All installed packages have been downloaded."

```

