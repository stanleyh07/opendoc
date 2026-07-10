使用 `devtool` 添加新的配方是非常方便的。以下是具體步驟：

1. **設置工作環境**：
    
    - 確保你已經設置好 Yocto 環境並且已經進入了構建目錄。
2. **創建工作區**：
    
    - 如果你還沒有創建工作區，可以使用以下命令：
        
        ```sh
        devtool create-workspace <workspace-directory>
        ```
        
3. **添加新的配方**：
    
    - 使用 `devtool add` 命令來添加新的配方。這裡有兩種常見的方法：
        - **從外部源代碼添加**：
            
            ```sh
            devtool add <recipe-name> <source-url>
            ```
            
            例如：
            
            ```sh
            devtool add hello https://ftp.gnu.org/gnu/hello/hello-2.12.tar.gz
            ```
            
        - **從本地源代碼添加**：
            
            ```sh
            devtool add <recipe-name> <local-source-directory>
            ```
            
            例如：
            
            ```sh
            devtool add hello /path/to/local/source
            ```
            
4. **編輯配方**：
    
    - 配方添加後，你可以使用 `devtool edit-recipe` 命令來編輯配方：
        
        ```sh
        devtool edit-recipe <recipe-name>
        ```
        
5. **構建配方**：
    
    - 編輯完成後，可以使用 `devtool build` 命令來構建配方：
        
        ```sh
        devtool build <recipe-name>
        ```
        
6. **安裝和測試**：
    
    - 構建完成後，你可以使用 `devtool deploy-target` 命令將構建的包部署到目標設備進行測試：
        
        ```sh
        devtool deploy-target <recipe-name>
        ```
        

這些步驟應該能幫助你使用 `devtool` 添加和管理新的配方