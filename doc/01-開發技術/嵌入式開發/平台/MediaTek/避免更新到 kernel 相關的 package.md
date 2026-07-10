可以使用下面指令去避免更新到 kernel 相關的 package,

下面是我檢查出來相關會被更新到的 package

```
apt list --upgradable | grep -E 'mtk|genio'

linux-buildinfo-5.15.0-1029-mtk/jammy-updates,jammy-security 5.15.0-1029.33 arm64 [upgradable from: 5.15.0-1029.33]
linux-headers-mtk/jammy-updates,jammy-security 5.15.0.1041.44 arm64 [upgradable from: 5.15.0.1029.31]
linux-image-mtk/jammy-updates,jammy-security 5.15.0.1041.44 arm64 [upgradable from: 5.15.0.1029.31]
linux-modules-5.15.0-1029-mtk/jammy-updates,jammy-security 5.15.0-1029.33 arm64 [upgradable from: 5.15.0-1029.33]
linux-mtk-headers-5.15.0-1029/jammy-updates,jammy-security 5.15.0-1029.33 all [upgradable from: 5.15.0-1029.33]
linux-mtk/jammy-updates,jammy-security 5.15.0.1041.44 arm64 [upgradable from: 5.15.0.1029.31]
linux-firmware-mediatek-genio/unknown 4.3-0ubuntu1~0oem1 arm64 [upgradable from: 4-0ubuntu1~22.04.1ubuntu1]
oem-baoshan-genio-desktop-meta/unknown 0.4 all [upgradable from: 0.3]
```

使用下面指令去鎖住這個版本的 package

```
sudo apt-mark hold [package name]
```

ex.
```
sudo apt-mark hold linux-buildinfo-5.15.0-1029-mtk \
                   linux-headers-mtk linux-image-mtk \
                   linux-modules-5.15.0-1029-mtk linux-mtk-headers-5.15.0-1029 \
                   linux-mtk linux-firmware-mediatek-genio \
                   oem-baoshan-genio-desktop-meta
```

可以是使用下面指令檢查有那些 package 被 hold

```
sudo apt-mark showhold
```

之後再去更新

```
sudo apt upgrade
```

這樣就不會更新到那幾個 package, 只會顯示提醒這些 package 會被維持舊的版本