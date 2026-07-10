initrd 為 ramdisk，可為 flash loader 的 RAMDISK 或是開機過程中 kernel 的 RAMDISK
製作方式有兩種

# 第一種利用 cpio 工具製作
### 利用 cpio 製作 initrd 的步驟如下

1. **創建一個目錄**：這個目錄將會成為你的 initrd 的根目錄。例如：
    
    ```bash
    mkdir /tmp/initrd
    cd /tmp/initrd
    ```
    
2. **添加所需的文件和目錄**：你需要將所有需要的文件和目錄添加到這個目錄中。例如：
    
    ```bash
    mkdir dev proc sys lib mnt
    ```
    
3. **創建設備文件**：例如：
    
    ```bash
    mknod dev/console c 5 1
    mknod dev/null c 1 3
    ```
    
4. **複製可執行文件**：例如：
    
    ```bash
    cp -a /usr/local/src/busybox-1.5.1/_install/{bin,sbin} .
    ```
    
5. **生成 init 文件**：例如：
    
  ```bash
    cat > init << EOF
    #!/bin/sh
    PATH="/bin:/sbin"
    mount -nt sysfs sysfs /sys
    mount -nt proc proc /proc
    EOF
```

6. **使用 cpio 和 gzip 創建 initrd 鏡像**：例如：
    
    ```bash
    find . | cpio --create -H newc > ../initrd.img
    gzip  -9 ../initrd.img
    ```

### 將 initrd 還原回原來的內容

1. 解壓縮 initrd
```
mv initrd initrd.gz
gzip -d initrd.gz
```

2. 還原
```
mkdir tmp
cd tmp
cpio -idmv < ../initrd

```

# 第二種為使用正常 filesystem 去製作

製作內容

```
# 製作映像檔案
dd if=/dev/zero of=../initrd.img bs=512k count=5

# 格式化
mkfs.ext2 -F -m0 ../initrd.img

# 載入映像檔案並複製檔案
mount -t ext2 -o loop ../initrd.img /mnt
cp -r * /mnt

# 卸載
umount /mnt

# 壓縮檔案
gzip -9 ../initrd.img

```
