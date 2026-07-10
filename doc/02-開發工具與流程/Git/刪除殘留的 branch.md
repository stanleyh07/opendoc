有時因為特殊操作導致某些 branch 改變，造成原始殘留的 branch 無效，但仍殘留紀錄，
這時已經無法直接使用 git branch 去刪除他。

比如因為重寫 fujisoft_develop 歷史資訊，造成 refs/original/refs/heads/fujisoft_develop 紀錄，但無法刪除或是使用

```sh
* e49e954670 (fujisoft_develop) 
| * 3b154f75f7 (refs/original/refs/heads/fujisoft_develop)
|/
* de3be53ba6 (digi_develop) Fixed speaker pop noise by rockchip's patch
* 12d505dcce (tag: R2.1) Fixed speaker pop noise
* 30cfda1353 (tag: R2.0) Fixed abnormal sound of speaker
* 76495d8346 (Schematic_R0.2) Change GMAC reset pin to gpio2_c4
* 6d76fc1d68 (ubuntu22) Fixed firefox doesn't work
* 5593afce51 (hdmi) Fixed wol unbalanced irq
* 1ea95fa346 (develop) Fixed build error
* 5c20006d46 (HEAD -> master) Add a support a touch panel(ILI2511)
* 170865797a Initial commit

```

要刪除 `refs/original/refs/heads/fujisoft_develop` 這個 branch，可以按照以下步驟進行：

1. **確認當前所在的分支**：確保你不在要刪除的分支上。你可以使用以下命令檢查當前分支：
    
    ```bash
    git branch
    ```
    
2. **刪除指定的引用**：使用 `git update-ref` 命令來刪除 `refs/original/refs/heads/fujisoft_develop`：
    
    ```bash
    git update-ref -d refs/original/refs/heads/fujisoft_develop
    ```
    
3. **清理無用的對象**：使用 `git prune` 命令來清理無用的對象，釋放存儲空間：
    
    ```bash
    git prune
    ```
    

這樣就可以成功地從 Git 倉庫中移除 `refs/original/refs/heads/fujisoft_develop` 引用
