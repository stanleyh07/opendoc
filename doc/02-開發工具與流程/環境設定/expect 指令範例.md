`expect` 提供了一系列指令來幫助自動化交互式任務。以下是一些常用的 `expect` 指令及其用法示例：

### 常用指令

1. **spawn**
    
    - 用於啟動一個進程。
    
    ```tcl
    spawn ssh user@host
    ```
    
2. **expect**
    
    - 等待特定的輸出。
    
    ```tcl
    expect "*password:"
    ```
    
3. **send**
    
    - 向進程發送輸入。
    
    ```tcl
    send "password\r"
    ```
    
4. **exp_continue**
    
    - 在多次匹配中繼續執行。
    
    ```tcl
    exp_continue
    ```
    
5. **send_user**
    
    - 向用戶輸出信息。
    
    ```tcl
    send_user "Login successful\n"
    ```
    
6. **exit**
    
    - 退出 expect 腳本。
    
    ```tcl
    exit
    ```
    
7. **eof**
    
    - 表示 expect 腳本結束。
    
    ```tcl
    expect eof
    ```
    
8. **set**
    
    - 定義變量。
    
    ```tcl
    set timeout 20
    ```
    
9. **puts**
    
    - 輸出變量。
    
    ```tcl
    puts "Variable value: $variable"
    ```
    

### 示例

#### 1. 自動化 SSH 登錄並執行命令

這個腳本會自動登錄到遠程主機並執行 `df -h` 命令來顯示磁盤使用情況。

```tcl
#!/usr/bin/expect

set timeout 20
set host "192.168.56.103"
set user "username"
set password "password"

spawn ssh $user@$host
expect "*password:"
send "$password\r"
expect "*$ "
send "df -h\r"
expect "*$ "
send "exit\r"
expect eof
```

#### 2. 自動化 SCP 文件傳輸

這個腳本會自動將本地文件傳輸到遠程主機。

```tcl
#!/usr/bin/expect

set timeout 20
set host "192.168.56.103"
set user "username"
set password "password"
set local_file "/path/to/local/file"
set remote_path "/path/to/remote/directory"

spawn scp $local_file $user@$host:$remote_path
expect "*password:"
send "$password\r"
expect eof
```

#### 3. 自動化交互式安裝

這個腳本會自動回答安裝過程中的提示。

```tcl
#!/usr/bin/expect

set timeout -1
spawn ./install_script.sh
expect "Do you agree to the terms? (yes/no)"
send "yes\r"
expect "Enter installation directory:"
send "/opt/install_dir\r"
expect eof
```

[這些示例展示了 `expect` 如何在不同的場景中自動化交互式任務，從而提高效率和減少手動操作](https://www.cnblogs.com/saneri/p/10819348.html)