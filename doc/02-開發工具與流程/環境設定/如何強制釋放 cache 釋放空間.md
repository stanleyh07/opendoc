使用 free 可以看到下面內容
```
               total        used        free      shared  buff/cache   available
Mem:         8010512      621220     6174852        2584     1214440     7108040
Swap:              0           0           0

```

其中可以看到 buff/cache 會佔有一些空間，當記憶體空間不足時，如何強制其釋放出來，
可以依照下面方式釋放

1. **清除 PageCache:**   
    ```bash
    sudo sync; echo 1 > /proc/sys/vm/drop_caches
    ```
    
2. **清除 Dentries 和 Inodes**：
    ```bash
    sudo sync; echo 2 > /proc/sys/vm/drop_caches
    ```
    
3. **清除所有缓存（包括 PageCache、Dentries 和 Inodes）**：
   ```bash
    sudo sync; echo 3 > /proc/sys/vm/drop_caches
    ```

結果會如下
```
               total        used        free      shared  buff/cache   available
Mem:         8010512      616784     6753268        2588      640460     7114676
Swap:              0           0           0
```