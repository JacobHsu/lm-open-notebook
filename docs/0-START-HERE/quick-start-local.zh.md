> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](quick-start-local.md)

# 快速上手 - 本機與私有部署(5 分鐘)

使用 **100% 本機 AI**(透過 Ollama)讓 Open Notebook 跑起來。不需要雲端 API 金鑰,完全私有。

**已經裝好 Ollama 了嗎?** 請改看 [外部 Ollama 指南](quick-start-external-ollama.zh.md)。

## 前置需求

1. **安裝 Docker Desktop**
   - [下載連結](https://www.docker.com/products/docker-desktop/)
   - 已經裝好了?直接跳到步驟 2

2. **本機 LLM** - 選一個:
   - **Ollama**(推薦):[下載連結](https://ollama.ai/)
   - **LM Studio**(圖形介面替代方案):[下載連結](https://lmstudio.ai)

## 步驟 1:選擇你的部署方式(1 分鐘)

### 本機(同一台電腦)
所有服務都跑在你的機器上。適合測試/學習使用。

### 遠端伺服器(樹莓派、NAS、雲端主機)
在另一台電腦上執行,從別的裝置連線使用。需要額外的網路設定。

---

## 步驟 2:建立設定檔(1 分鐘)

建立一個新資料夾 `open-notebook-local`,並加入這個檔案:

**docker-compose.yml**:
```yaml
services:
  surrealdb:
    image: surrealdb/surrealdb:v2
    command: start --user root --pass password rocksdb:/mydata/mydatabase.db
    user: root
    ports:
      # 僅限 localhost —— 資料庫使用預設帳密,絕對不要把這個 port 對外開放(0.0.0.0)
      - "127.0.0.1:8000:8000"
    volumes:
      - ./surreal_data:/mydata

  open_notebook:
    image: lfnovo/open_notebook:v1-latest
    pull_policy: always
    ports:
      - "8502:8502"  # Web UI(React 前端)
      - "5055:5055"  # API(必要!)
    environment:
      # 憑證加密金鑰(必要)
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string

      # 資料庫(必要)
      - SURREAL_URL=ws://surrealdb:8000/rpc
      - SURREAL_USER=root
      - SURREAL_PASSWORD=password
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=open_notebook

      # Ollama(當你像本範例一樣透過 Docker 執行 Ollama 時,此項為必要)
      - OLLAMA_API_BASE=http://ollama:11434
    volumes:
      - ./notebook_data:/app/data
    depends_on:
      - surrealdb
    restart: always

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_models:/root/.ollama
    restart: always
    # 選用:若有 GPU 可設定 GPU 支援
    #deploy:
    #  resources:
    #    reservations:
    #      devices:
    #        - driver: nvidia
    #          count: 1
    #          capabilities: [gpu]

```

**編輯這個檔案:**
- 把 `change-me-to-a-secret-string` 換成你自己的密鑰(任何字串都可以)

---

## 步驟 3:啟動服務(1 分鐘)

在 `open-notebook-local` 資料夾裡開啟終端機:

```bash
docker compose up -d
```

等待 10-15 秒讓所有服務啟動。

---

## 步驟 4:下載模型(2-3 分鐘)

Ollama 需要至少一個語言模型。挑一個:

```bash
# 最快最小(建議用於測試)
docker exec open-notebook-local-ollama-1 ollama pull mistral

# 或者:品質更好但較慢
docker exec open-notebook-local-ollama-1 ollama pull neural-chat

# 或者:品質更好,但需要更多 VRAM
docker exec open-notebook-local-ollama-1 ollama pull llama2
```

這會下載模型(依你的網路速度,約需 1-5 分鐘)。

---

## 步驟 5:進入 Open Notebook(即時)

開啟瀏覽器:
```
http://localhost:8502
```

你應該會看到 Open Notebook 的介面。

---

## 步驟 6:設定 Ollama 供應商(1 分鐘)

1. 點左側欄「管理」底下的 **模型**(頁面標題是「使用您自己的 API 金鑰設定 AI」)
2. 往下捲找到 **Ollama** 卡片,點擊 **新增設定**
3. 填寫:
   - **設定名稱**:取個名字,例如「Local Ollama」
   - **基礎 URL**:`http://ollama:11434`(容器內連容器,見下方「其他做法」)
4. 點擊 **新增設定** 儲存
5. 儲存後,該筆設定同一列會出現 **Test** 和 **Models** 兩個按鈕(這兩個按鈕沒有中文化,是介面本身的顯示,不是翻譯漏掉)
6. 點 **Test** —— 右下角會跳出「連線成功」提示
7. 點 **Models** —— 會開啟「探索模型 - Ollama」視窗,勾選要用的模型(例如 `llama3`),按 **新增 (N)** 完成註冊

> 如果你是照本頁「本機部署」用 docker-compose 把 Ollama 也跑在同一個 compose 網路裡,基礎 URL 用 `http://ollama:11434` 沒問題;如果你的 Ollama 是另外裝在主機上(不在這個 compose 檔裡),要改用 `http://host.docker.internal:11434`,詳見 [外部 Ollama 指南](quick-start-external-ollama.zh.md)。

---

## 步驟 7:設定預設模型(1 分鐘)

不用換頁,**跟步驟 6 同一頁**,捲回最上方:

1. **聊天模型**:選 `ollama/llama3`(或是你剛剛註冊的其他模型)
2. **嵌入模型**:選 `ollama/nomic-embed-text`(沒有的話用 Ollama 拉一個:`ollama pull nomic-embed-text`,再回來這頁重新整理)
3. 選好即自動儲存(沒有另外的「Save」按鈕)

---

## 步驟 8:建立第一個 Notebook(1 分鐘)

1. 點擊 **New Notebook**
2. 名稱:「My Private Research」
3. 點擊 **Create**

---

## 步驟 9:加入本機內容(1 分鐘)

1. 點擊 **Add Source**
2. 選擇 **Text**
3. 貼上文字或本機文件內容
4. 點擊 **Add**

---

## 步驟 10:與你的內容對話(1 分鐘)

1. 前往 **Chat**
2. 輸入:「What did you learn from this?」
3. 點擊 **Send**
4. 看著本機的 Ollama 模型回應你!

---

## 驗證清單

- [ ] Docker 正在執行
- [ ] 可以連上 `http://localhost:8502`
- [ ] Ollama 憑證已設定並測試成功
- [ ] 模型已註冊
- [ ] 已建立一個 notebook
- [ ] 用本機模型對話成功

**都打勾了嗎?** 你已經擁有一個完全**私有、離線**的研究助理!

---

## 本機部署的優點

- **沒有 API 費用** - 永久免費
- **不需要網路** - 真正的離線能力
- **隱私優先** - 資料永遠不會離開你的機器
- **不用訂閱** - 沒有每月帳單

**取捨:** 比雲端模型慢(取決於你的 CPU/GPU)

---

## 疑難排解

### 「ollama: command not found」

Docker 容器名稱可能不一樣:
```bash
docker ps  # 找出 Ollama 容器名稱
docker exec <container_name> ollama pull mistral
```

### 模型下載卡住

檢查網路連線並重啟:
```bash
docker compose restart ollama
```

然後重新執行下載模型的指令。

### 「Address already in use」錯誤

```bash
docker compose down
docker compose up -d
```

### 效能低落

檢查 GPU 是否可用:
```bash
# 顯示可用的 GPU
docker exec open-notebook-local-ollama-1 ollama ps

# 在 docker-compose.yml 中啟用 GPU
```

然後重啟:`docker compose restart ollama`

### 新增更多模型

```bash
# 列出可用模型
docker exec open-notebook-local-ollama-1 ollama list

# 下載額外的模型
docker exec open-notebook-local-ollama-1 ollama pull neural-chat
```

---

## 下一步

**現在已經跑起來了:**

1. **加入自己的內容**:PDF、文件、文章(見 3-USER-GUIDE)
2. **探索更多功能**:Podcast、內容轉換、搜尋
3. **完整文件**:[查看所有功能](../3-USER-GUIDE/index.md)
4. **擴大規模**:部署到硬體更好的伺服器上以獲得更快的回應
5. **模型效能比較**:試試不同模型,找出你偏好的速度/品質平衡點

## 替代方案:用 LM Studio 取代 Ollama

**偏好圖形介面?** LM Studio 對非技術使用者更友善:

1. 下載 LM Studio:https://lmstudio.ai
2. 開啟應用程式,從模型庫下載一個模型
3. 前往「Local Server」分頁,啟動伺服器(port 1234)
4. 在 Open Notebook 中,前往 **Settings** → **API Keys**
5. 點擊 **Add Credential** → 選擇 **OpenAI-Compatible**
6. 輸入 base URL:`http://host.docker.internal:1234/v1`
7. 輸入 API key:`lm-studio`(佔位用,隨便打)
8. 點擊 **Save**,然後 **Test Connection**
9. 在 Settings → Models 中設定,選擇你的 LM Studio 模型

**注意**:LM Studio 跑在 Docker 之外,請用 `host.docker.internal` 連線。

---

## 更進一步

- **切換模型**:隨時可在 Settings → Models 中更改
- **新增更多模型**:
  - Ollama:執行 `ollama pull <model>`,然後從憑證重新偵測模型
  - LM Studio:從應用程式的模型庫下載
- **部署到伺服器**:同一份 docker-compose.yml 到哪都能用
- **混合雲端使用**:保留部分本機模型,同時加入雲端供應商憑證處理複雜任務

---

## 常見模型選擇

| 模型 | 速度 | 品質 | VRAM | 適合場景 |
|-------|-------|---------|------|----------|
| **mistral** | 快 | 良好 | 4GB | 測試、一般用途 |
| **neural-chat** | 中等 | 更好 | 6GB | 均衡,推薦使用 |
| **llama2** | 慢 | 最佳 | 8GB+ | 複雜推理 |
| **phi** | 非常快 | 尚可 | 2GB | 硬體資源有限 |

---

**需要協助?** 加入我們的 [Discord 社群](https://discord.gg/37XJPXfz2w) —— 很多使用者都在跑本機部署!
