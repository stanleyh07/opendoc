當我們用下面指令可以 build 出 yocto image
``` bash
	source oe-init-build-env build

	LANG=en_US.UTF-8 LANGUAGE=en_US.en LC_ALL=en_US.UTF-8 \
		bitbake core-image-minimal -C rootfs

```

如何得知有那些類似 core-image-minimal 參數可以使用，去 build 出不同的 image
查詢下面路徑
```bash
ls -al poky/meta/recipes-core/images/
```
一個 bb 檔案就是一種配置去編譯一種 image

或是檢查有那些 recipes
```bash
bitbake-layers show-recipes | grep -A 3 core-image
```

(grep -A 3，代表找到後多印 3 行, 也可使用 -B 列印之前，-C 前後都印)
