
這是一份針對 **「本地 AI 服務（Ollama）加上 API Key 認證」** 的最終完整實作指南。這套方案將 AI 運算引擎（Ollama）、安全門衛（LiteLLM）與資料庫（Postgres）徹底分離，實現生產級的管理彈性。

---

## 第一階段：環境準備

請在你的主機上建立一個資料夾來存放所有設定檔：

```bash
mkdir my-ai-cloud && cd my-ai-cloud
```

### 1. 建立模型設定檔 `litellm_config.yaml`

**解釋：** 這個檔案是 LiteLLM 的靈魂。它告訴代理層有哪些「虛擬門牌」（`model_name`）以及它們對應到 Ollama 內部的哪一個真實模型。

```yaml
model_list:
  - model_name: my-best-model    # 對外公佈的模型名稱（可自訂）
    litellm_params:
      model: ollama/llama3       # 內部 Ollama 實際的模型名稱
      api_base: "http://ollama:11434"

  - model_name: coding-assistant # 另一個對外名稱
    litellm_params:
      model: ollama/codellama
      api_base: "http://ollama:11434"

general_settings:
  master_key: sk-master-admin-888 # 最高管理員密鑰（請自訂）
```

---

## 第二階段：編寫 Docker 部署檔

建立 `docker-compose.yml` 檔案。

**解釋：** * **隔離性**：`ollama` 服務沒有設定 `ports`，因此它在外部 IP 是不可見的，只有 `litellm` 找得到它。

- **持久化**：可以改用 `volumes` 確保你的模型與 API Key 資料在容器重啟後不會消失。    

```yaml
services:
  # 1. 資料庫：儲存你生成的每一組 API Key 與使用量資料
  db:
    image: postgres:16
    container_name: litellm-db
    environment:
      POSTGRES_DB: litellm
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: your_secure_db_password # 請自訂資料庫密碼
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  # 2. Ollama：模型運算引擎（完全在內網運行）
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ./ollama_data:/root/.ollama
    restart: unless-stopped
    # 這裡不寫 ports，對外隱藏

  # 3. LiteLLM Proxy：唯一的安全門衛
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm-proxy
    ports:
      - "4000:4000" # 只開放這個埠給外部
    environment:
      - DATABASE_URL=postgresql://admin:your_secure_db_password@db:5432/litellm
      - LITELLM_MASTER_KEY=sk-master-admin-888
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    depends_on:
      - db
      - ollama
    command: [ "--config", "/app/config.yaml" ]
    restart: unless-stopped
```

---

## 第三階段：啟動與初始化

### 1. 啟動容器

```bash
docker compose up -d
```

### 2. 下載模型（最關鍵的一步）

**解釋：** 剛啟動的 Ollama 是空的。你必須手動進入容器下載你在設定檔中提到的模型。

```bash
docker exec -it ollama ollama run llama3
docker exec -it ollama ollama run codellama
```

_(下載完畢後可按 `Ctrl + C` 或 `Ctrl + D` 退出容器)_

---

## 第四階段：管理與發放 API Key (UI 操作)

**解釋：** 雖然我們有 `master_key`，但不建議直接把這把鑰匙給別人，因為它可以刪除你的所有設定。我們應該為每個使用者生成「專屬虛擬 Key」。

1. **進入後台**：打開瀏覽器訪問 `http://你的伺服器IP:4000/ui`。
    
2. **登入**：在彈出的視窗輸入你的 `sk-master-admin-888`。
    
3. **生成虛擬 Key**：
    
    - 點擊 **"Virtual Keys"** -> **"+ New Key"**。
        
    - 輸入名稱（例如：`ai-plan-generator-key`）。
        
    - 點擊 **"Generate Key"**。
        
4. **複製金鑰**：複製那串 `sk-xxxx...`，這就是你要填入程式或交給別人的 API Key。
    

---

## 第五階段：外部使用範例 (Python)

你的 `ai-plan-generator` 服務現在只需要專注於透過 LiteLLM 獲取回應。

```python
import os
from openai import OpenAI

# 這是你從 UI 生成的虛擬 Key
MY_API_KEY = "sk-從UI複製出來的金鑰"

client = OpenAI(
    api_key=MY_API_KEY,
    base_url="http://你的伺服器IP:4000/v1" # 記得指向 LiteLLM
)

response = client.chat.completions.create(
    model="my-best-model", # 使用你在 YAML 中自訂的名稱
    messages=[{"role": "user", "content": "你好，請給我一個減肥計畫"}]
)

print(response.choices[0].message.content)
```

---

## 👨‍🔧 日常維護指引

- **新增模型**：
    
    1. 使用 `docker exec` 幫 Ollama 下載新模型。
        
    2. 修改 `litellm_config.yaml` 加入新模型區塊。
        
    3. 執行 `docker compose restart litellm`。
        
- **管理使用者**：
    
    1. 隨時登入 UI 介面，你可以看到誰使用了多少 Token（流量統計）。
        
    2. 如果有人濫用服務，直接在 UI 點擊 **"Revoke"** 即可撤銷該 API Key。
        

這套方案既維持了 Ollama 的本地高效運算，又具備了雲端服務般的安全控管。

**這份步驟是否已經涵蓋了你所有的疑慮？如果啟動時遇到 `500 Internal Server Error`，通常是模型名稱對不起來**


---
## 額外補充資訊

# 📘 LiteLLM 模型定義與路由指南

在 LiteLLM 中，模型定義的核心邏輯是：**「讓代理層知道要用哪種協議（供應商）去跟哪個後端（模型）溝通。」**

## 1. 模型定義基本語法

在 `litellm_config.yaml` 中，每個模型由兩個名稱組成：

- **`model_name` (對外門牌)**：你給外部程式呼叫時用的名字（自訂）。
    
- **`model` (內部路由)**：告知 LiteLLM 真實的供應商與模型路徑。
    

---

## 2. 路由規則：供應商前綴 (Prefix)

LiteLLM 使用 `供應商/模型名稱` 的格式來決定呼叫邏輯。

### A. 必須加前綴的情況

當模型來源不是 OpenAI 官方時，**強烈建議**加上前綴。

|**供應商類型**|**前綴範例**|**實際寫法範例**|
|---|---|---|
|**本地端 Ollama**|`ollama/`|`ollama/llama3`, `ollama/gpt-oss:120b`|
|**雲端 DeepSeek**|`deepseek/`|`deepseek/deepseek-chat`|
|**Google Gemini**|`gemini/`|`gemini/gemini-1.5-pro`|
|**Anthropic Claude**|`anthropic/`|`anthropic/claude-3-5-sonnet`|

### B. 不必加前綴的情況 (預設值)

若未加前綴，LiteLLM 預設將其視為 **OpenAI** 協議。

- 範例：`gpt-4o`, `gpt-3.5-turbo`
    

---

## 3. 為什麼「相容 OpenAI」的服務也建議加前綴？

即使像 DeepSeek 或某些本地服務號稱「完全相容 OpenAI API」，加上正確前綴（如 `deepseek/`）仍有三大不可取代的好處：

1. **Tokenizer (分詞器) 精準度**：
    
    - 不同廠商計算 Token 的方式不同。加上前綴後，LiteLLM 會使用該廠商專屬的分詞器，確保你的 **Usage (使用量統計)** 與帳單金額一致。
        
2. **參數過濾 (Parameter Mapping)**：
    
    - 有些廠商不支援 `logprobs` 或特定的 `tool_choice`。加上前綴後，LiteLLM 會自動過濾掉不支援的參數，防止 API 回傳 `400 Bad Request` 錯誤。
        
3. **錯誤訊息翻譯**：
    
    - LiteLLM 會將各家廠商奇形怪狀的錯誤訊息（例如：餘額不足、觸發敏感詞），統一翻譯成標準的 OpenAI 錯誤格式，方便你的 Flask 程式處理例外（Exception）。
        

---

## 4. 實戰設定範例 (`litellm_config.yaml`)

```yaml
model_list:
  # 範例 1：本地 Ollama 模型
  - model_name: my-local-ai
    litellm_params:
      model: ollama/llama3
      api_base: "http://ollama:11434"

  # 範例 2：雲端 DeepSeek (雖然相容 OpenAI，但建議加前綴)
  - model_name: ds-chat
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: "os.environ/DEEPSEEK_API_KEY"

  # 範例 3：自訂相容 OpenAI 的服務 (例如 LocalAI)
  - model_name: custom-service
    litellm_params:
      model: openai/custom-model-name # 指定使用 openai 協議
      api_base: "https://your-custom-endpoint.com"
      api_key: "your-api-key"
```

---

## 5. 如何檢查設定是否正確？

1. **UI 檢查**：登入 `http://localhost:4000/ui`，在 **Models** 頁面確認模型狀態是否為綠色。
    
2. **Playground 測試**：直接在 UI 的 **Playground** 選擇該模型發送測試訊息。
    
3. **指令檢查**：
      
    ```bash
    # 查看 LiteLLM 目前識別出的所有可用模型
    curl http://localhost:4000/v1/models -H "Authorization: Bearer sk-your-master-key"
    ```
    
---

**💡 總結小撇步**：

永遠記得：`model_name` 是給**人**看的標籤，`model`（含前綴）是給 **LiteLLM** 看的指令。


