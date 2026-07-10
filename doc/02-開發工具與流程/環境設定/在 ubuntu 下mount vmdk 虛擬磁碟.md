如何在 ubuntu 下 mount vmdk 檔案的虛擬磁碟

1. **安裝必要工具**：首先，確保你安裝了 `qemu-utils`，它包含了 `qemu-nbd` 工具。
```bash
sudo apt-get update
sudo apt-get install qemu-utils
```

2. **載入 NBD 模組**：`nbd` 模組允許使用網路區塊裝置。
```bash
sudo modprobe nbd max_part=8
```

3. **連接 VMDK 檔案**：使用 `qemu-nbd` 將 VMDK 檔案連接到本地裝置。
```bash
sudo qemu-nbd -c /dev/nbd0 /你的/路徑/檔案.vmdk
```
_請將_ `/你的/路徑/檔案.vmdk` _替換為實際的 VMDK 檔案路徑。_

4. **查看分區資訊**：確定 VMDK 檔案中的分區。
```bash
sudo fdisk -l /dev/nbd0
```
這將顯示 VMDK 包含的分區，例如 `/dev/nbd0p1`、`/dev/nbd0p2` 等。

5. **建立掛載點**：如果還沒有掛載目錄，建立一個。
```bash
sudo mkdir /mnt/vmdk
```

6. **掛載分區**：將需要的分區掛載到剛剛建立的目錄。
```bash
sudo mount /dev/nbd0p1 /mnt/vmdk

# 使用唯讀方式更安全，避免不小修改到內容
sudo mount -o ro /dev/ndb0p1 /mnt/vmdk

```
_確保將_ `/dev/nbd0p1` _更改為你實際想要掛載的分區。_

7. **存取檔案**：現在，你可以像存取本地檔案一樣，在 `/mnt/vmdk` 目錄中瀏覽 VMDK 檔案的內容了。

8. **完成後卸載**：
 - **卸載分區**：
```bash
sudo umount /mnt/vmdk
```
- **斷開 bnd 裝置 :
```bash
sudo qemu-nbd -d /dev/nbd0
```

**注意事項**：

- **權限問題**：在整個過程中，需要使用 `sudo` 來獲得必要的權限。    
- **資料安全**：操作虛擬磁碟時，務必小心，以防資料損壞。建議在修改之前備份重要資料。    
- **多分區情況**：如果 VMDK 有多個分區，重複第 6 步，掛載其他需要的分區。    

**額外提示**：

掛載 VMDK 檔案可以讓你直接存取虛擬機的檔案系統，非常適合資料恢復、檔案提取或檢查虛擬機內部配置。如果你對虛擬化技術感興趣，Ubuntu 提供了豐富的工具，如 **KVM**、**VirtualBox** 以及 **VMware Workstation**，都值得一試！

另外，熟悉 `qemu-nbd` 後，你還可以探索掛載其他格式的虛擬磁碟，如 QCOW2。同樣的思路，可以拓展到更多的虛擬化應用中。