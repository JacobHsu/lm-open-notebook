> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](single-container.md)

# 單一容器安裝(已淘汰)

> **淘汰通知:** 單一容器 image(`v1-latest-single`)已**淘汰**,將於 v2 移除。請改用 [Docker Compose](docker-compose.zh.md),這是我們對所有使用者建議的安裝方式。單一容器 image 在 v2 發佈前仍會持續更新,但不會有新功能或新文件以它為對象。

全部整合在一個容器中的設定。**比 Docker Compose 更簡單,但彈性較差。**

**適合:** PikaPods、Railway、共享主機、精簡設定

> **替代 Registry:** image 同時提供於 Docker Hub(`lfnovo/open_notebook:v1-latest-single`)與 GitHub Container Registry(`ghcr.io/lfnovo/open-notebook:v1-latest-single`)。

## 前置需求

- 已安裝 Docker(用於本機測試)
- 來自 OpenAI、Anthropic 或其他供應商的 API 金鑰
- 5 分鐘

## 快速設定

### 本機測試(Docker)

```yaml
# docker-compose.yml
services:
  open_notebook:
    image: lfnovo/open_notebook:v1-latest-single
    pull_policy: always
    ports:
      - "8502:8502"  # Web UI(React 前端)
      - "5055:5055"  # API
    environment:
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string
      - SURREAL_URL=ws://localhost:8000/rpc
      - SURREAL_USER=root
      - SURREAL_PASSWORD=root
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=open_notebook
    volumes:
      - ./data:/app/data
    restart: always
```

執行:
```bash
docker compose up -d
```

存取:`http://localhost:8502`

接著設定你的 AI 供應商:
1. 前往 **Settings** → **API Keys**
2. 點選 **Add Credential** → 選擇你的供應商 → 貼上 API 金鑰
3. 點選 **Save**,接著 **Test Connection**
4. 點選 **Discover Models** → **Register Models**

### 雲端平台

**PikaPods:**
1. 點選「New App」
2. 搜尋「Open Notebook」
3. 設定環境變數(至少需要 `OPEN_NOTEBOOK_ENCRYPTION_KEY`)
4. 點選「Deploy」
5. 開啟應用程式 → 到 **Settings → API Keys** 設定你的 AI 供應商

**Railway:**
1. 建立新專案
2. 加入 `lfnovo/open_notebook:v1-latest-single`
3. 設定環境變數(至少需要 `OPEN_NOTEBOOK_ENCRYPTION_KEY`)
4. 部署
5. 開啟應用程式 → 到 **Settings → API Keys** 設定你的 AI 供應商

**Render:**
1. 建立新的 Web Service
2. 使用 Docker image:`lfnovo/open_notebook:v1-latest-single`
3. 在 dashboard 中設定環境變數(至少需要 `OPEN_NOTEBOOK_ENCRYPTION_KEY`)
4. 為 `/app/data` 與 `/mydata` 設定持久化磁碟

**DigitalOcean App Platform:**
1. 從 Docker Hub 建立新的 app
2. 使用 image:`lfnovo/open_notebook:v1-latest-single`
3. 連接埠設為 8502
4. 新增環境變數(至少需要 `OPEN_NOTEBOOK_ENCRYPTION_KEY`)
5. 設定持久化儲存空間

**Heroku:**
```bash
# 使用 heroku.yml
heroku container:push web
heroku container:release web
heroku config:set OPEN_NOTEBOOK_ENCRYPTION_KEY=your-secret-key
```

**Coolify:**
1. 新增服務 → Docker Image
2. Image:`lfnovo/open_notebook:v1-latest-single`
3. 連接埠:8502
4. 新增環境變數(至少需要 `OPEN_NOTEBOOK_ENCRYPTION_KEY`)
5. 啟用持久化 volume
6. Coolify 會自動處理 HTTPS

**EasyPanel:**

Open Notebook 提供 EasyPanel 範本,位於 [`examples/easypanel/`](https://github.com/lfnovo/open-notebook/tree/main/examples/easypanel)。與上述單一 image 選項不同,此範本會建立**兩個服務**——Open Notebook 應用程式與一個獨立的 SurrealDB 實例——並自動為你產生資料庫密碼、加密金鑰,以及(選用的)應用程式密碼。

- **一鍵安裝(建議):** 一旦此範本發佈到官方 [EasyPanel 範本庫](https://github.com/easypanel-io/templates),就可以從「Open Notebook」建立新服務,設定一組應用程式密碼(留空則自動產生),然後部署。
- **手動安裝:** 把 `examples/easypanel/` 複製到 [`easypanel-io/templates`](https://github.com/easypanel-io/templates) 檢出版本中的 `templates/open-notebook`,執行範本測試環境(`npm run dev`),再從產生的 JSON 於你的 EasyPanel 實例中建立範本。

部署完成後,開啟 EasyPanel 網域,在 **Settings → API Keys** 中設定你的 AI 供應商。詳情參見 [`examples/easypanel/README.md`](https://github.com/lfnovo/open-notebook/blob/main/examples/easypanel/README.md)。

---

## 環境變數

| 變數 | 用途 | 範例 |
|----------|---------|---------|
| `OPEN_NOTEBOOK_ENCRYPTION_KEY` | 用於加密憑證的密鑰(必要) | `my-secret-key` |
| `SURREAL_URL` | 資料庫 | `ws://localhost:8000/rpc` |
| `SURREAL_USER` | 資料庫使用者 | `root` |
| `SURREAL_PASSWORD` | 資料庫密碼 | `root` |
| `SURREAL_NAMESPACE` | 資料庫命名空間 | `open_notebook` |
| `SURREAL_DATABASE` | 資料庫名稱 | `open_notebook` |
| `API_URL` | 對外 URL(用於遠端存取) | `https://myapp.example.com` |

AI 供應商的 API 金鑰是在部署完成後,透過 **Settings → API Keys** 介面設定。

---

## 與 Docker Compose 相比的限制

| 功能 | 單一容器 | Docker Compose |
|---------|------------------|-----------------|
| 設定時間 | 2 分鐘 | 5 分鐘 |
| 複雜度 | 最小 | 中等 |
| 服務 | 全部整合 | 分開 |
| 可擴充性 | 有限 | 優秀 |
| 記憶體用量 | 約 800MB | 約 1.2GB |

---

## 後續步驟

與 Docker Compose 設定相同——只是改成透過 `http://localhost:8502`(本機)或你所用平台的 URL(雲端)存取。

1. 到 **Settings → API Keys** 新增你的 AI 供應商憑證
2. **Test Connection** 並 **Discover Models**

完整的安裝後指南請參見 [Docker Compose](docker-compose.zh.md)。
