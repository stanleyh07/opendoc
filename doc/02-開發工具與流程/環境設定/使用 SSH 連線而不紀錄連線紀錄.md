在 Linux 下使用 SSH 連線而不紀錄連線歷史，可以通過以下幾種方法來實現：

1. **使用 `-o` 選項**： SSH 提供了許多選項來控制連線行為。你可以使用 `-o` 選項來設置 `UserKnownHostsFile` 和 `StrictHostKeyChecking` 參數，避免將主機密鑰記錄到 `~/.ssh/known_hosts` 文件中。例如：
    
    ```bash
    ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no user@hostname
    ```
    
2. **修改 SSH 配置文件**： 你也可以在 SSH 配置文件（通常是 `~/.ssh/config` 或 `/etc/ssh/ssh_config`）中添加以下配置來全局應用這些設置：
    
    ```plaintext
    Host *
        UserKnownHostsFile /dev/null
        StrictHostKeyChecking no
    ```
    
3. **使用 `sshpass` 工具**： 如果你需要自動輸入密碼，可以使用 `sshpass` 工具來避免記錄連線歷史。這個工具可以在命令行中提供密碼，而不需要手動輸入：
    
    ```bash
    sshpass -p 'your_password' ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no user@hostname
    ```
    

這些方法可以幫助你在使用 SSH 連線時避免記錄連線歷史