可能原因有
1. 檢查 /etc/apt/sources.list 是否正確
2. 檢查 /etc/resove.conf 是否根據當前 network 重新建立
   建立 rootfs 時，會用 host 的 network 建立 /etc/resolve.conf，
   當將 rootfs 更新至裝置，會導致網路設定不同，所以須刪除此檔案，
   並使裝置從重新開機，以便重新建立這個檔案
3. /tmp  目錄未建立或是屬性不正確
   開機時會建立這個目錄，但在建立 rootfs image時可能沒有，
   但在 rootfs 使用 apt 卻需要 /tmp，因此需手動建立這個目錄，
   且屬性必須為 [[02-開發工具與流程/環境設定/drwxrwxrwt 含意]]，使用指令
```
   chmod 1777 /tmp
```
       
其他錯誤

第一種，/dev/null 權限問題
```
/usr/bin/apt-key: 95: cannot create /dev/null: Permission denied
```

檢查 /dev/null 屬性，其必須為
```
crw-rw-rw- 1 root root 1, 3 Nov 22  2023 /dev/null
```

使用下面指令去修改屬性
```
sudo chmod 666 /dev/null
```


第二種，file has an unsupported filetype
``` 
W: [http://us.archive.ubuntu.com/ubuntu/...mmy/InRelease:](http://us.archive.ubuntu.com/ubuntu/dists/jammy/InRelease:) The key(s) in the keyring /etc/apt/trusted.gpg.d/as-repo-public.gpg are ignored as the file has an unsupported filetype.
```

需要建立一些 device node
使用下面 script 進入 rootfs
```
#!/bin/bash

function mnt(){

	echo "MOUNTING......"
	mount -t proc /porc ${1}/proc
	mount -t sysfs /sys ${1}/sys
	mount -o bind /dev ${1}/dev
	#mount -o bind /run ${1}/run
	mount -o bind /dev/pts ${1}/dev/pts
	chroot ${1}
}

function umnt(){

	echo "UNMOUNTING....."
	umount ${1}/proc
	umount ${1}/sys
	umount ${1}/dev/pts
	#umount ${1}/run
	umount ${1}/dev
}

function request_root() {
	if [ $(id -u) -ne 0 ]; then
		echo "This script requires root privilege"
		exit 1
	fi
}

if [ -n "$1" ];
then
	request_root
	mnt $1
	umnt $1
else
	echo ""
	echo "Either 1'st parameters were missing"
	echo ""
	echo "1'st parameter is the full path of ubuntu_bin directory(with trailing '/')"
	echo ""
	echo "For example: ./ch-mount ubuntu_bin"
	echo ""
	echo 1st parameter : ${1}
fi
```
