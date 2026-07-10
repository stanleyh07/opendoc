當你在ubuntu設定啟用自動登入時，會發現開機後第一次開啟 chromium 時，會跳出視窗要求輸入密碼，但未啟用自動登入(即開機需要輸入帳號密碼登入)，則不會跳出要求輸入密碼，這是為什麼呢?

這種清況是由於啟用自動登入，密鑰環(keyring)並未被解鎖，chromium 會在安裝時建立兩組密鑰，因此需要密碼先解鎖密鑰環，才能存取到 chromium 的密鑰

1. 開啟管理密碼AP

   輸入下面指令開啟密鑰管理 AP，或是在尋找 ==passwords and Keys== AP

```
   seahorse
```

2. 開啟後，再 Login 中會看到 chromium 建立兩組密鑰
	- Chrome Safe Storage Control
	- Chrome Safe Storgae
3. 選擇 Login，按右鍵跳出選單後，選擇 ==Change Password==
4. 彈出視窗輸入當前密碼，然後新密碼及確認新密碼都留空
5. 系統可能會警告你留空的風險，按繼續即可