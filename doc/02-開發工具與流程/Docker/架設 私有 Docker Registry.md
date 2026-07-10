
在區域網路上架設 **私有 Docker Registry** 的流程，以及可能遇到的問題與解決方法：

---

### **1. 啟動私有 Docker Registry**

使用 Docker 官方 `registry` 映像來啟動私有 registry：

```sh
docker run -d -p 5000:5000 --restart=always --name my_registry registry:2
```

如果希望將映像存儲到本機，則可以掛載存儲目錄：

```sh
docker run -d -p 5000:5000 --restart=always \
  -v /data/registry:/var/lib/registry \
  --name my_registry registry:2
```

---

### **2. Push 映像到私有 Registry**

必須標記含有區域網路資訊(如: 192.168.113.172, loadhost...) 才能推送，注意標記的網路資訊需要別人有可以存取的如: IP, 或是 hostname,  但像 localhost 這種，只能自己 push 或 pull ，其他人就會發生無法 pull 下來 

1. **標記映像**：
    
    ```sh
    docker tag ubuntu:22.04 192.168.113.172:5000/ubuntu:2204
    ```
    
2. **推送映像**：
    
    ```sh
    docker push 192.168.113.172:5000/ubuntu:2204
    ```
    

---

### **3. 在區域網路使用**

在其他主機上拉取映像：

```sh
docker pull 192.168.113.172:5000/ubuntu22:04
```

如果 Registry 只支援 HTTP，可能會遇到錯誤，請參考下方解決方法。

---

### **4. 可能遇到的問題與解決方案**

#### **問題 1：Docker 客戶端要求 HTTPS**

如果 Registry 只支援 HTTP，而 Docker 預設要求 HTTPS，會報錯：

```sh
Get "https://192.168.113.172:5000/v2/": http: server gave HTTP response to HTTPS client
```

**解決方案：**

1. **允許不安全的 Registry**  
    編輯 `/etc/docker/daemon.json`：
    
    ```json
    {
      "insecure-registries": ["192.168.113.172:5000"]
    }
    ```
    
2. **重啟 Docker**：
    
    ```sh
    systemctl restart docker
    ```
    

#### **問題 2：如何刪除已推送的映像**

Docker Registry 沒有直接的 `docker rmi` 指令來刪除已上傳的映像，需透過 API 操作：

1. 找出映像的 Digest：
    
    ```sh
    curl -X GET http://192.168.113.172:5000/v2/my_image/manifests/latest
    ```
    
2. 刪除映像：
    
    ```sh
    curl -X DELETE http://192.168.113.172:5000/v2/my_image/manifests/sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    ```
    
3. 執行 Garbage Collection（清理未引用的映像）：
    
    ```sh
    docker exec -it my_registry bin/registry garbage-collect /etc/docker/registry/config.yml
    ```
    

#### **問題 3：如何啟用 HTTPS**

如果你希望 Registry 使用 HTTPS：

1. **準備 TLS 憑證** (`domain.crt` 和 `domain.key`) 放入 `/certs/`
2. **啟動 Registry 並使用憑證**：
    
    ```sh
    docker run -d -p 5000:5000 --restart=always \
      -v /data/registry:/var/lib/registry \
      -v /certs:/certs \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      --name my_registry registry:2
    ```
    

---

這樣，你就可以在區域網路內成功架設、使用和管理你的 **私有 Docker Registry**！ 🚀 如果有任何其他問題，我很樂意幫助你！