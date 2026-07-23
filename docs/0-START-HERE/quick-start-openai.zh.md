> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](quick-start-openai.md)

# 快速上手 - OpenAI(5 分鐘)

使用 OpenAI 的 GPT 模型讓 Open Notebook 跑起來。快速、強大、簡單。

## 前置需求

1. **安裝 Docker Desktop**
   - [下載連結](https://www.docker.com/products/docker-desktop/)
   - 已經裝好了?直接跳到步驟 2

2. **OpenAI API 金鑰**(必要)
   - 前往 https://platform.openai.com/api-keys
   - 建立帳號 → 建立新的 secret key
   - 至少儲值 $5 美元的額度
   - 複製金鑰(以 `sk-` 開頭)

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

  open_notebook:
    image: lfnovo/open_notebook:v1-latest
    pull_policy: always
    ports:
      - "8502:8502"  # Web UI
      - "5055:5055"  # API
    environment:
      # 憑證加密金鑰(必要)
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string

      # 資料庫(必要)
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

## 步驟 4:設定你的 OpenAI 供應商(1 分鐘)

1. 前往 **Settings** → **API Keys**
2. 點擊 **Add Credential**
3. 選擇供應商:**OpenAI**
4. 取個名字(例如「My OpenAI Key」)
5. 貼上你的 OpenAI API 金鑰
6. 點擊 **Save**
7. 點擊 **Test Connection** —— 應該顯示成功
8. 點擊 **Discover Models** → **Register Models**

你的 OpenAI 模型現在已經可用了!

---

## 步驟 5:建立第一個 Notebook(1 分鐘)

1. 點擊 **New Notebook**
2. 名稱:「My Research」
3. 點擊 **Create**

---

## 步驟 6:加入一個來源(1 分鐘)

1. 點擊 **Add Source**
2. 選擇 **Web Link**
3. 貼上:`https://en.wikipedia.org/wiki/Artificial_intelligence`
4. 點擊 **Add**
5. 等待處理完成(30-60 秒)

---

## 步驟 7:與你的內容對話(1 分鐘)

1. 前往 **Chat**
2. 輸入:「What is artificial intelligence?」
3. 點擊 **Send**
4. 看著 GPT 根據你的來源內容回答!

---

## 驗證清單

- [ ] Docker 正在執行
- [ ] 可以連上 `http://localhost:8502`
- [ ] OpenAI 憑證已設定並測試成功
- [ ] 已建立一個 notebook
- [ ] 已加入一個來源
- [ ] 對話功能正常

**都打勾了嗎?** 你已經擁有一個完整可用的 AI 研究助理!

---

## 使用不同的模型

在你的 notebook 中,前往 **Settings** → **Models** 選擇:
- `gpt-4o` - 品質最佳(推薦)
- `gpt-4o-mini` - 速度快又便宜(適合測試)

---

## 疑難排解

### 「Port 8502 already in use」

在 docker-compose.yml 中更改 port:
```yaml
ports:
  - "8503:8502"  # 改用 8503
```

然後用 `http://localhost:8503` 連線

### 「API key not working」

1. 前往 **Settings** → **API Keys**
2. 對你的 OpenAI 憑證點擊 **Test Connection**
3. 若失敗,到 https://platform.openai.com 確認你的金鑰
4. 刪除該憑證,用正確的金鑰重新建立一個

### 「Cannot connect to server」

```bash
docker ps  # 檢查所有服務是否都在執行
docker compose logs  # 查看日誌
docker compose restart  # 重啟所有服務
```

---

## 下一步

1. **加入自己的內容**:PDF、網頁連結、文件
2. **探索更多功能**:Podcast、內容轉換、搜尋
3. **完整文件**:[查看所有功能](../3-USER-GUIDE/index.md)

---

## 費用估算

OpenAI 定價(大約):
- **對話**:每 1K token $0.01-0.10
- **Embeddings**:每 1M token $0.02
- **一般使用量**:輕度使用每月 $1-5,重度使用每月 $20-50

實際費率請以 https://openai.com/pricing 為準。

---

**需要協助?** 加入我們的 [Discord 社群](https://discord.gg/37XJPXfz2w)!
