使用 genio1200 HW Video Codec (vpud) 會發生記憶體洩露問題
### 問題描述 :
	當使用 vpud 去撥放影片時，會發現 buff/cache 不斷增加，這是因為 vpud 在解碼影片時，
	會不斷要求 buff，所以會看到 buff/cache 的大小不斷增加，但在這個大小應該在記憶體不足，
	或是被要求的情況下被釋放，例如使用下面指令去強制釋放
	
```
echo 1 > /proc/sys/vm/drop_cache
```
	但結果並未釋放，這是因為 vpud 仍然佔住這個空間導致


### 解決方法 :
	根據 mediatek 的 gitlab 已經解決，但仍未更新到 PPA ubuntu server 上，
	因此無法透過 apt upgrade mediatek-vpud-genio1200 去更新，
	但可以手動更新來解決這個問題


根據 [MediaTek / AIoT / RITY / vpud · GitLab](https://gitlab.com/mediatek/aiot/rity/vpud)
當中的 [commit 445f68ae]([GENIO: vpud: mt8395: Update vpud library (445f68ae) · 提交 · MediaTek / AIoT / RITY / vpud · GitLab](https://gitlab.com/mediatek/aiot/rity/vpud/-/commit/445f68ae159856c161702ee7008902bfef6c48b4))
Author: Macross Chen <macross.chen@mediatek.com>
Date:   Wed Jul 12 19:54:54 2023 +0800

    GENIO: vpud: mt8395: Update vpud library
    
    built from internal SHA:
    dd32cad726065f3098dba15a91ecd67e7cc7045c
    
    update binary to bring:
    dd32cad GENIO: vpud: mt8395: vdec: Fix using uninitialized variables
    5cc440c GENIO: vpud: mt8395: vdec: Fix memory leak
    d3e1de3 GENIO: vpud: mt8395: vdec: Fix wrong output format of logs
    ef5dafa GENIO: vpud: mt8395: vdec: Fix accessing null pointer
    b7caf12 GENIO: vpud: mt8395: vdec: Fix h265_parse return error
    ad7eeff GENIO: vpud: mt8395: vdec: Fix leaked storage

mediatek 在 7/12 的 commit 解決這個問題
因此可以透過這個這個 gitlab 手動下載這個日期之後的版本

```
# 抓取最新的那個版本
git clone https://gitlab.com/mediatek/aiot/rity/vpud

cp -rfp vpud/mt8395/aarch64/libvpud_vcodec.so <rootfs>/usr/lib/libvpud_vcodec.so
cp -rfp vpud/mt8395/aarch64/vpud  <rootfs>/usr/bin/vpud
```

重新開機即可

### 驗證方法 :
	使用撥放影片，並使用 HW video codec 去解碼，
	可以看到記憶體的 buff/cache 會不斷增加，
	然後使用下面指令，並查看是否有釋放出記憶體
	
```
	echo 1 > /proc/sys/vm/drop_cache
```