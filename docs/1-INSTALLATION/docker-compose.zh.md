> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](docker-compose.md)

# Docker Compose 安裝(建議)

多容器設定,各服務分開運作。**適合大多數使用者。**

> **替代 Registry:** 所有 image 同時提供於 Docker Hub(`lfnovo/open_notebook`)與 GitHub Container Registry(`ghcr.io/lfnovo/open-notebook`)。如果 Docker Hub 被封鎖,或你偏好 GitHub 原生的工作流程,可改用 GHCR。

## 前置需求

- 已安裝 **Docker Desktop**([下載](https://www.docker.com/products/docker-desktop/))
- **5-10 分鐘**的時間
- 至少一個 AI 供應商的 **API 金鑰**(初學者建議使用 OpenAI)

## 步驟 1:取得 docker-compose.yml(1 分鐘)

**選項 A:從儲存庫下載**
```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
```

**選項 B:使用儲存庫中的官方檔案**

官方的 `docker-compose.yml` 位於我們儲存庫的根目錄:[在 GitHub 上查看](https://github.com/lfnovo/open-notebook/blob/main/docker-compose.yml)

把該檔案複製到你的專案資料夾。

**選項 C:手動建立**

建立一個名為 `docker-compose.yml` 的檔案,內容如下:

```yaml
services:
  surrealdb:
    image: surrealdb/surrealdb:v2
    # 憑證預設為 root:root,方便零設定的本機使用。若要將此實例
    # 對外開放,請在 .env 檔案(參見 .env.example)中設定
    # SURREAL_USER / SURREAL_PASSWORD——這裡與下方的 open_notebook
    # 服務都會套用同一組值,所以兩邊會保持一致。
    # 使用列表(exec)格式,讓每個內插值都維持單一參數——
    # 否則含有空白的密碼會被拆成好幾個參數。
    command: ["start", "--log", "info", "--user", "${SURREAL_USER:-root}", "--pass", "${SURREAL_PASSWORD:-root}", "rocksdb:/mydata/mydatabase.db"]
    user: root  # Linux 上使用 bind mount 時必須設定
    ports:
      # 只綁定在 localhost:open_notebook 服務無論如何都是透過
      # compose 的內部網路連到這裡,host 上的連接埠純粹是為了
      # 本機除錯(例如 Surrealist、`surreal sql`)。若開放在
      # 0.0.0.0,任何能連到這台主機的人都能用預設的 root:root
      # 憑證連進來。
      - "127.0.0.1:8000:8000"
    volumes:
      - ./surreal_data:/mydata
    environment:
      - SURREAL_EXPERIMENTAL_GRAPHQL=true
    restart: always
    pull_policy: always

  open_notebook:
    image: lfnovo/open_notebook:v1-latest
    ports:
      - "8502:8502"  # Web UI
      - "5055:5055"  # REST API
    environment:
      # 必填:換成你自己的密鑰字串
      # 這會用來加密資料庫中的 API 金鑰
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string

      # 資料庫連線。SURREAL_USER / SURREAL_PASSWORD 本機使用時
      # 預設為 root:root;若要對外開放此實例,請在 .env 檔案中
      # 覆寫這兩個值(上方 surrealdb 服務也是用同一組值設定)。
      - SURREAL_URL=ws://surrealdb:8000/rpc
      - SURREAL_USER=${SURREAL_USER:-root}
      - SURREAL_PASSWORD=${SURREAL_PASSWORD:-root}
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=open_notebook
    volumes:
      - ./notebook_data:/app/data
    depends_on:
      - surrealdb
    restart: always
    pull_policy: always
```

**編輯檔案:**
- 把 `change-me-to-a-secret-string` 換成你自己的密鑰(任何字串都可以,例如 `my-super-secret-key-123`)
- (選用)若要使用預設 `root:root` 以外的資料庫憑證,可在 `docker-compose.yml` 旁建立一個 `.env` 檔案,填入 `SURREAL_USER=...` 與 `SURREAL_PASSWORD=...`——兩個服務都會自動讀取(完整格式參見 [.env.example](https://github.com/lfnovo/open-notebook/blob/main/.env.example))

---

## 步驟 2:啟動服務(2 分鐘)

在 `open-notebook` 資料夾中開啟終端機:

```bash
docker compose up -d
```

等待 15-20 秒讓所有服務啟動:
```
✅ surrealdb running on :8000
✅ open_notebook running on :8502 (UI) and :5055 (API)
```

檢查狀態:
```bash
docker compose ps
```

---

## 步驟 3:驗證安裝(1 分鐘)

**API 健康檢查:**
```bash
curl http://localhost:5055/health
# 應回傳:{"status": "healthy"}
```

**前端存取:**
用瀏覽器開啟:
```
http://localhost:8502
```

你應該會看到 Open Notebook 的介面!

---

## 步驟 4:設定 AI 供應商(2 分鐘)

1. 前往 **Settings** → **API Keys**
2. 點選 **Add Credential**
3. 選擇你的供應商(例如 OpenAI、Anthropic、Google)
4. 為它取個名字,貼上你的 API 金鑰
5. 點選 **Save**
6. 點選 **Test Connection**——應該顯示成功
7. 點選 **Discover Models** → **Register Models**

你的模型現在可以使用了!

> **需要 API 金鑰?** 從你選擇的供應商取得:
> - **OpenAI**:https://platform.openai.com/api-keys
> - **Anthropic**:https://console.anthropic.com/
> - **Google**:https://aistudio.google.com/
> - **Groq**:https://console.groq.com/

---

## 步驟 5:建立第一個 Notebook(2 分鐘)

1. 點選 **New Notebook**
2. 名稱:「My Research」
3. 描述:「Getting started」
4. 點選 **Create**

完成!你現在擁有一個完整可用的 Open Notebook 實例。

---

## 設定

### 新增 Ollama(免費本機模型)

不用手動編輯,直接使用我們準備好的範例:

```bash
# 下載 Ollama 範例
curl -o docker-compose.yml https://raw.githubusercontent.com/lfnovo/open-notebook/main/examples/docker-compose-ollama.yml

# 或從儲存庫複製
cp examples/docker-compose-ollama.yml docker-compose.yml
```

完整設定參見 [examples/docker-compose-ollama.yml](../../examples/docker-compose-ollama.yml)。

**手動設定:**把以下內容加進你現有的 `docker-compose.yml`:

```yaml
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_models:/root/.ollama
    restart: always

volumes:
  ollama_models:
```

接著重新啟動並拉取一個模型:
```bash
docker compose restart
docker exec open-notebook-local-ollama-1 ollama pull mistral
```

在 Settings UI 中設定 Ollama:
1. 前往 **Settings** → **API Keys**
2. 點選 **Add Credential** → 選擇 **Ollama**
3. 輸入 base URL:`http://ollama:11434`
4. 點選 **Save**,接著 **Test Connection**
5. 點選 **Discover Models** → **Register Models**

---

## 環境變數參考

| 變數 | 用途 | 範例 |
|----------|---------|---------|
| `OPEN_NOTEBOOK_ENCRYPTION_KEY` | 用於加密憑證的密鑰 | `my-secret-key` |
| `SURREAL_URL` | 資料庫連線 | `ws://surrealdb:8000/rpc` |
| `SURREAL_USER` | 資料庫使用者 | `root` |
| `SURREAL_PASSWORD` | 資料庫密碼 | `root` |
| `SURREAL_NAMESPACE` | 資料庫命名空間 | `open_notebook` |
| `SURREAL_DATABASE` | 資料庫名稱 | `open_notebook` |
| `API_URL` | API 對外 URL | `http://localhost:5055` |
| `OPEN_NOTEBOOK_EMBEDDING_BATCH_SIZE` | 覆寫嵌入批次大小,適用於較嚴格或本機供應商(僅使用 CPU 的本機環境建議設為 `8`) | `50` |

完整清單參見[環境變數參考](../5-CONFIGURATION/environment-reference.md)。

---

## 常見操作

### 停止服務
```bash
docker compose down
```

### 查看日誌
```bash
# 所有服務
docker compose logs -f

# 指定服務
docker compose logs -f api
```

### 重新啟動服務
```bash
docker compose restart
```

### 更新到最新版本
```bash
docker compose down
docker compose pull
docker compose up -d
```

### 移除所有資料
```bash
docker compose down -v
```

---

## 疑難排解

### 「Cannot connect to API」錯誤

1. 確認 Docker 是否正在執行:
```bash
docker ps
```

2. 確認服務是否正在執行:
```bash
docker compose ps
```

3. 檢查 API 日誌:
```bash
docker compose logs api
```

4. 再多等一下——第一次執行時服務可能需要 20-30 秒才會啟動

---

### 連接埠已被佔用

如果出現「Port 8502 already in use」,可以換一個連接埠:

```yaml
ports:
  - "8503:8502"  # 改用 8503
  - "5055:5055"  # API 連接埠維持不變
```

接著到 `http://localhost:8503` 存取

---

### 憑證問題

1. 前往 **Settings** → **API Keys**
2. 在憑證上點選 **Test Connection**
3. 如果失敗,到供應商網站確認金鑰
4. 檢查你的帳號是否還有額度
5. 如有需要,刪除並重新建立憑證

---

### 資料庫連線問題

確認 SurrealDB 是否正在執行:
```bash
docker compose logs surrealdb
```

重設資料庫:
```bash
docker compose down -v
docker compose up -d
```

### 資料庫權限被拒(Linux)

如果在 SurrealDB 日誌中看到 `Permission denied` 或 `Failed to create RocksDB directory`:

```bash
docker compose logs surrealdb | grep -i permission
```

這是因為 SurrealDB 以非 root 使用者執行,但 Docker 建立 bind mount 目錄時是以 root 身分。請在 surrealdb 服務中加入 `user: root`:

```yaml
surrealdb:
  image: surrealdb/surrealdb:v2
  user: root  # 修正 Linux 上 bind mount 的權限問題
  # ...(其餘設定)
```

接著重新啟動:
```bash
docker compose down -v
docker compose up -d
```

---

## 其他設定方式

想找不同的設定方式?可以看看我們的 [examples/](../../examples/) 資料夾:

- **[Ollama 設定](../../examples/docker-compose-ollama.yml)** - 執行本機 AI 模型(免費、私有)
- **[單一容器](../../examples/docker-compose-single.yml)** - 全部整合在一個容器中(已淘汰,將於 v2 移除)
- **[開發環境](../../examples/docker-compose-dev.yml)** - 給貢獻者與開發者使用

每個範例都附有詳細的註解與使用說明。

---

## 後續步驟

1. **新增內容**:來源、notebook、文件
2. **設定模型**:Settings → Models(選擇你偏好的模型)
3. **探索功能**:聊天、搜尋、轉換
4. **閱讀指南**:[使用者指南](../3-USER-GUIDE/index.md)

---

## 正式環境部署

若要部署到正式環境,請參閱:
- [安全性強化](../5-CONFIGURATION/security.md)
- [反向代理](../5-CONFIGURATION/reverse-proxy.md)

---

## 需要協助?

- **Discord**:[社群支援](https://discord.gg/37XJPXfz2w)
- **Issues**:[GitHub Issues](https://github.com/lfnovo/open-notebook/issues)
- **文件**:[完整文件](../index.md)
