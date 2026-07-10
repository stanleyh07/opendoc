如何提供 bash script 中暫停直到按下任意鍵

```
#!/bin/bash

pause() {
	echo "Press any key to continue..."
	read -n 1 -s
}

pause
echo "Continuing..."

```
