git如何在不進行 checkout 的情況下，進行分支更新
比如目前有 branch new、develop、master 三個分支，
當前 checkout 到 develop，
若想將 new 的 commit 更新到 master 分支上，

```
git fetch . new:master
```

fetch 當前 repository，將 new fetch 到 master 上

若需要更新到某一個 commit 上，比如
```
* 4e2e895082 (HEAD -> digi_develop) Fixed ili251x do not work properly
* 42ee65515f Change display from dsi0 to dsi1 for digi LCM
* 76495d8346 Change GMAC reset pin to gpio2_c4
* ccfec92683 (Schematic_R0.2) Modify device tree for schematic R0.2
* 6d76fc1d68 (origin/ubuntu22) Fixed firefox doesn't work
* 0166045462 Add custom version info in uname
* 10d56310f9 Optimize the mechanism for creating Ubuntu rootfs
* 85896172dd Remove unneccessary files
```

要在不 checkout 的情況下，將 Schematic_R0.2 更新到 76495d8346 commit 上

```
git branch -f Schematic_R0.2 76495d8346
```

相當於重新定義 branch 到指定的 commit 上


不更新 workspace 下，切換 branch，將 branch 切換到 develop

```
git symbolic-ref HEAD refs/heads/develop
```

