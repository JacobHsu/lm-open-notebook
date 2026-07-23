> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](quick-start-cloud.md)

# 快速上手 - 雲端 AI 供應商(5 分鐘)

使用 **Anthropic、Google、Groq 或其他雲端供應商** 讓 Open Notebook 跑起來。跟 OpenAI 一樣簡單,但有更多選擇。

## 前置需求

1. **安裝 Docker Desktop**
   - [下載連結](https://www.docker.com/products/docker-desktop/)
   - 已經裝好了?直接跳到步驟 2

2. **API 金鑰**,取自你選擇的供應商:
   - **OpenRouter**(100+ 種模型,單一金鑰):https://openrouter.ai/keys
   - **Anthropic(Claude)**:https://console.anthropic.com/
   - **Google(Gemini)**:https://aistudio.google.com/
   - **Groq**(速度快,有免費額度):https://console.groq.com/
   - **Mistral**:https://console.mistral.ai/
   - **DeepSeek**:https://platform.deepseek.com/
   - **xAI(Grok)**:https://console.x.ai/

## 步驟 1:建立設定檔(1 分鐘)

建立一個新資料夾 `open-notebook`,並加入這個檔案:

**docker-compose.yml**:
```yaml
services:
  surrealdb:
    image: surrealdb/surrealdb:v2
    command: start --user root --pass password rocksdb:/mydata/mydatabase.db
    ports:
      # 僅限 localhost —— 資料庫使用預設帳密,絕對不要把這個 port 對外開放(0.0.0.0)
      - "127.0.0.1:8000:8000"
    volumes:
      - ./surreal_data:/mydata
    # 移除了 healthcheck,因為 v2 映像檔太精簡,沒有 wget/curl 可用
    restart: always

  open_notebook:
    image: lfnovo/open_notebook:v1-latest
    pull_policy: always
    ports:
      - "8502:8502"  # Web UI
      - "5055:5055"  # API
    environment:
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string
      - SURREAL_URL=ws://surrealdb:8000/rpc
      - SURREAL_USER=root
      - SURREAL_PASSWORD=password
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=open_notebook
    volumes:
      - ./notebook_data:/app/data
    depends_on:
      - surrealdb
    restart: always

```

**編輯這個檔案:**
- 把 `change-me-to-a-secret-string` 換成你自己的密鑰(任何字串都可以)

---

## 步驟 2:啟動服務(1 分鐘)

在 `open-notebook` 資料夾裡開啟終端機:

```bash
docker compose up -d
```

等待 15-20 秒讓服務啟動。

---

## 步驟 3:進入 Open Notebook(即時)

開啟瀏覽器:
```
http://localhost:8502
```

你應該會看到 Open Notebook 的介面!

---

## 步驟 4:設定你的 AI 供應商(1 分鐘)

1. 前往 **Settings** → **API Keys**
2. 點擊 **Add Credential**
3. 選擇你的供應商(例如 Anthropic、Google、Groq、OpenRouter)
4. 取個名字,貼上你的 API 金鑰
5. 點擊 **Save**
6. 點擊 **Test Connection** —— 應該顯示成功
7. 點擊 **Discover Models** → **Register Models**

你的供應商模型現在已經可用了!

> **多個供應商**:你可以新增任意多個供應商的憑證,對每個供應商重複這個步驟即可。

---

## 步驟 5:設定你的模型(1 分鐘)

1. 前往 **Settings**(齒輪圖示)
2. 進入 **Models**
3. 選擇你供應商的模型:

| 供應商 | 建議模型 | 備註 |
|----------|-------------------|-------|
| **OpenRouter** | `anthropic/claude-3.5-sonnet` | 可存取 100+ 種模型 |
| **Anthropic** | `claude-3-5-sonnet-latest` | 推理能力最佳 |
| **Google** | `gemini-3.5-flash` | 上下文長、速度快 |
| **Groq** | `llama-3.3-70b-versatile` | 極速 |
| **Mistral** | `mistral-large-latest` | 強力的歐洲選項 |

4. 點擊 **Save**

---

## 步驟 6:建立第一個 Notebook(1 分鐘)

1. 點擊 **New Notebook**
2. 名稱:「My Research」
3. 點擊 **Create**

---

## 步驟 7:加入內容並對話(2 分鐘)

1. 點擊 **Add Source**
2. 選擇 **Web Link**
3. 貼上任何文章網址
4. 等待處理完成
5. 前往 **Chat** 開始提問!

---

## 驗證清單

- [ ] Docker 正在執行
- [ ] 可以連上 `http://localhost:8502`
- [ ] 供應商憑證已設定並測試成功
- [ ] 模型已註冊
- [ ] 已建立一個 notebook
- [ ] 對話功能正常

**都打勾了嗎?** 可以開始研究了!

---

## 供應商比較

| 供應商 | 速度 | 品質 | 上下文長度 | 費用 |
|----------|-------|---------|---------|------|
| **OpenRouter** | 依模型而定 | 依模型而定 | 依模型而定 | 依模型而定(100+ 種) |
| **Anthropic** | 中等 | 極佳 | 200K | $$$ |
| **Google** | 快 | 非常好 | 1M+ | $$ |
| **Groq** | 極速 | 好 | 128K | $(有免費額度) |
| **Mistral** | 快 | 好 | 128K | $$ |
| **DeepSeek** | 中等 | 非常好 | 64K | $ |

---

## 疑難排解

### 「Model not found」錯誤

1. 前往 **Settings** → **API Keys**
2. 對你的憑證點擊 **Test Connection**
3. 若有效,點擊 **Discover Models** → **Register Models**
4. 確認你有該模型的額度/存取權限

### 「Cannot connect to server」

```bash
docker ps  # 檢查所有服務是否都在執行
docker compose logs  # 查看日誌
docker compose restart  # 重啟所有服務
```

### 各供應商常見問題

**Anthropic**:確認金鑰以 `sk-ant-` 開頭
**Google**:要用 AI Studio 的金鑰,不是 Cloud Console 的
**Groq**:免費額度有速率限制;有需要可升級

---

## 費用估算

每 1K token 的大約費用:

| 供應商 | 輸入 | 輸出 |
|----------|-------|--------|
| Anthropic(Sonnet) | $0.003 | $0.015 |
| Google(Flash) | $0.0001 | $0.0004 |
| Groq(Llama 70B) | 有免費額度 | - |
| Mistral(Large) | $0.002 | $0.006 |

實際費率請以供應商官網為準。

---

## 下一步

1. **加入你的內容**:PDF、網頁連結、文件
2. **探索更多功能**:Podcast、內容轉換、搜尋
3. **完整文件**:[查看所有功能](../3-USER-GUIDE/index.md)

---

**需要協助?** 加入我們的 [Discord 社群](https://discord.gg/37XJPXfz2w)!
