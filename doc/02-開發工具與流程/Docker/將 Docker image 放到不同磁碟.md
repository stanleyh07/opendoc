參考 [🔖 SSD + Docker - NVIDIA Jetson AI Lab](https://www.jetson-ai-lab.com/tips_ssd-docker.html)

1. 找到要放入的磁區
```bash
nvidia@EAC5k-OrinAGX:~$ lsblk

NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mmcblk0boot0 179:32   0  31.5M  1 disk 
mmcblk0boot1 179:64   0  31.5M  1 disk 
mmcblk1      179:96   0    58G  0 disk /sd
```

2. 確認其為 ext4，不然 format 成 ext4
```bash
sudo mkfs.ext4 /dev/mmcblk1
```

3. 設定開機自動 mounted
   修改 /etc/fstab
   
```bash
nvidia@EAC5k-OrinAGX:~$ lsblk -f /dev/mmcblk1

NAME    FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
mmcblk1 ext4   1.0         be3da290-7ab7-46d9-8292-d0ec0538dab5   24.5G    52% /sd

```

```
# <file system>                  <mount point> <type>  <options>  <dump> <pass>
UUID=be3da290-7ab7-46d9-8292-d0ec0538dab5 /sd ext4    defaults    0      2
```

4. 變更munted 目錄使用者屬性，使可以存取
```
sudo chown ${USER}:${USER} /ssd
```

5. 移植 docker 內容到新位置
``` bash
# 停止 docker
sudo systemctl stop docker

# 搬移資料到新路徑 (假設新路徑為 /sd )
sudo du -csh /var/lib/docker/ && \ 
sudo mkdir /sd/docker && \ 
sudo rsync -axPS /var/lib/docker/ /sd/docker/ && \ 
sudo du -csh /sd/docker/
```

6. 修改 /etc/docker/daemon.json，加入 data-root
```
{ 
	"runtimes":  {
	    "nvidia": { 
	        "path": "nvidia-container-runtime", 
	        "runtimeArgs": [] 
	    } 
	}, 
	"default-runtime": "nvidia", 
	"data-root": "/sd/docker" 
}
```

7. 移除原來路徑並從新啟動
```bash
# 移除原來的路徑
sudo mv /var/lib/docker /var/lib/docker.old

# 重新啟動
sudo systemctl daemon-reload && \ 
sudo systemctl restart docker && \ 
sudo journalctl -u docker
```
