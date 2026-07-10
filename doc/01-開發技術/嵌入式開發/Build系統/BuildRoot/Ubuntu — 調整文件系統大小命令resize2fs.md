# [《Ubuntu — 調整文件系統大小命令resize2fs》](https://www.cnblogs.com/zhuangquan/p/12532622.html "发布于 2020-03-20 16:37")

**1.resize2fs命令**

```
resize2fs /dev/mmcblk1p8                

# mmcblk1p8是文件系統分隔區  
```


**resize2fs命令可以調整ext2\ext3\ext4文件系統的大小，它可以放大或者縮小沒有掛載的文件系統的大小。 如果文件系統已經掛載，它可以擴大文件系統的大小，前提是內核支援在線調整大小。**

語法格式 : resize2fs <參數> <分隔區>
參數說明 :
-d : 打開調試特性
-f : 強制執行調整大小操作，覆蓋掉安全檢查操作
-F : 開始執行調整大小前，刷新文件系統設備的緩衝區
-p : 打印已完成的百分比進度條
-P : 顯示文件系統的最小值(大寫P)
-M : 將文件系統縮小到最小值

size參數指定所請求的檔案系統的新大小。 如果沒有指定任何單元，那麼size參數的單位應該是文件系統的文件系統塊大小。 size參數可以由下列單位編號之一後綴：“s”、“K”、“M”或“G”，分別用於512位元組扇區、千位元組、兆位元組或千兆位元組。 文件系統的大小可能永遠不會大於分區的大小。 **如果未指定Size參數，則它將預設為分區的大小。**

resize2fs程式不操作分區的大小。 如果希望擴大檔系統，必須首先確保可以擴展基礎分區的大小。 如果您使用邏輯卷管理器LVM（8），可以使用fdisk（8）刪除分區並以更大的大小重新創建它，或者使用lvexport（8）。 在重新創建分區時，請確保使用與以前相同的啟動磁碟圓柱來創建分區！ 否則，調整大小操作肯定無法工作，您可能會丟失整個文件系統。 運行fdisk（8）后，運行resize2fs來調整ext 2文件系統的大小，以使用新擴大的分區中的所有空間。

**此命令的適用範圍：RedHat、RHEL、Ubuntu、CentOS、SUSE、openSUSE、Fedora。**


**2.實例**

顯示sda1最小值

```
[root@localhost ~]# resize2fs -P /dev/sda1

resize2fs 1.41.12 (17-May-2010)

Estimated minimum size of the filesystem: 37540
```


設置sdb1為1k
```
[root@localhost ~]# resize2fs /dev/sdb1 1k

resize2fs 1.41.12 (17-May-2010)

resize2fs: New size smaller than minimum (373)    # 小於最小值，失敗
```



**3.應用場景**

```
df -h
```


![](https://img2020.cnblogs.com/blog/1359524/202003/1359524-20200320200219348-1178348654.png)

可以看到這個的文件系統分區大小是6.4G，但是實際上有15G。

resize2fs /dev/mmcblk1p5

![](https://img2020.cnblogs.com/blog/1359524/202003/1359524-20200320200248096-1268764098.png)

再次查看就會發現分區大小變成15G了。

![](https://img2020.cnblogs.com/blog/1359524/202003/1359524-20200320200258191-1471290460.png)