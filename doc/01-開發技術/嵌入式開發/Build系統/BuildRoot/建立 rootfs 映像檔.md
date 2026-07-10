建立 rootfs image，特別是處理與 Host PC 不同平台的 rootfs，需要透過 qemu 的虛擬機。
因此需要安裝下面的 package
1. qemu 相關 package
   `apt install qemu qemu-user qemu-user-static`
2. 辨識載入相對應的虛擬機機制的 package，參考[[01-開發技術/嵌入式開發/Build系統/BuildRoot/使用 chroot 進入虛擬機的機制]]
   `apt install binfmt-support`
3. 若是在 docker 上處理，則 docker 內，以及 docker 的 host 都需要安裝
   

