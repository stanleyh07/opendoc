如何使用不同帳號登入到 docker 
1. 需該 docker image 先建立要登入的帳號
2. 使用下面指令去開啟 docker 容器
```
docker run -it <容器ID或是名稱> --user <用戶名或是 UID> /bin/bash
```

使用指定帳號從已存在的 container去開啟新視窗

```

docker exec -u <user> -it <container_name> /bin/bash

-u <user>: 指定 login 新視窗的帳號 (該帳號必須已經被建立)

ex.
	docker  -u rk -it rk2 /bin/bash

or 
	docker -u root -it rk2 /bin/bash
```

