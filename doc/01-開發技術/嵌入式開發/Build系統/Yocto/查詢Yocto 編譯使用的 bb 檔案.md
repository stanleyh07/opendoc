
在 Yocto 專案中，可以使用以下方法查詢在編譯特定映像檔（例如 `rity-demo-image`）時所使用的 `.bb`（BitBake）檔案：

1. **使用 BitBake 的 `-g` 選項**  
   透過以下指令，可以生成一組關於依賴和元資料的圖形文件：
   ```bash
   bitbake -g rity-demo-image
   ```
   這會生成以下文件：
   - `pn-buildlist`：列出所有參與的配方（`.bb` 文件）。
   - `task-depends.dot` 和 `package-depends.dot`：可以用來檢視依賴關係（需要進一步使用[[02-開發工具與流程/通用工具/Dot 圖形工具]]進行解析）。

2. **BitBake 的 `--dry-run` 模式**  
   您可以執行指令檢查執行時所涉及的配方：
   ```bash
   bitbake --dry-run rity-demo-image
   ```
   它不會實際執行編譯，但會顯示 BitBake 的解決和執行計劃。

3. **查詢 Task Logs 或 Build Logs**  
   當 BitBake 實際執行後，可以檢查 log 文件來驗證哪些 `.bb` 配方參與了編譯過程。這些檔案位於 `build/tmp/log` 資料夾中。

4. **使用 devtool**  
   Devtool 可以提供更多對配方的細節管理，您可以檢視特定配方的細節：
   ```bash
   devtool search rity-demo-image
   ```

試試看這些方法，應該可以幫助您清楚地看到 `rity-demo-image` 所依賴的 `.bb` 配方！如果需要更深入的解析，請讓我知道！ 😊


