> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](quick-start-external-ollama.md)

# 快速上手 - 外部 Ollama

使用**單獨安裝的 Ollama**(不透過 Docker)來執行 Open Notebook。這樣可以避免 Docker 另外跑一個 Ollama 服務,直接用你自己安裝的本機 Ollama。

## 前置需求

1. **安裝 Docker Desktop**(給 SurrealDB 和 Open Notebook 用)
   - [下載連結](https://www.docker.com/products/docker-desktop/)

2. **單獨安裝 Ollama**
   - [下載連結](https://ollama.ai/)
   - 驗證:執行 `ollama --version`

3. 在 Ollama 中**下載模型**:
   ```bash
   ollama pull mistral
   ollama pull nomic-embed-text
   ```

---

## 步驟 1:啟動 Ollama(1 分鐘)

啟動 Ollama 伺服器:

```bash
# 預設:跑在 http://localhost:11434
ollama serve
```

保持這個終端機開啟,Ollama 會在背景執行。

**選用:用自訂 port 或網路介面啟動 Ollama:**
```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

---

## 步驟 2:建立設定檔(1 分鐘)

建立一個新資料夾 `open-notebook-external-ollama`,並加入以下檔案:

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
    volumes:
      - ./notebook_data:/app/data
    depends_on:
      - surrealdb
    restart: always

```

**注意:** Docker 裡沒有 Ollama 服務 —— 我們使用主機上的 Ollama。

---

## 步驟 3:把 Open Notebook 接到主機的 Ollama(1 分鐘)

當 Open Notebook 跑在 Docker 內部時,無法直接連到主機的 `localhost:11434`。要用這個特殊主機名稱:

| 主機作業系統 | Open Notebook 中的 Ollama URL |
|---------|----------------------------|
| Linux | `http://host.containers.internal:11434` |
| macOS | `http://host.docker.internal:11434` |
| Windows | `http://host.docker.internal:11434` |

---

## 步驟 4:啟動 Open Notebook(1 分鐘)

在 `open-notebook-external-ollama` 資料夾裡開啟終端機:

```bash
docker compose up -d
```

等待 10-15 秒讓服務啟動。

---

## 步驟 5:設定 Ollama 供應商(1 分鐘)

1. 前往 **Settings** → **API Keys**
2. 點擊 **Add Credential**
3. 選擇供應商:**Ollama**
4. 取個名字(例如「Local Ollama」)
5. 輸入 base URL:
   - **Windows/macOS:** `http://host.docker.internal:11434`
   - **Linux:** `http://host.containers.internal:11434`
6. 點擊 **Save**
7. 點擊 **Test Connection** —— 應該顯示成功
8. 點擊 **Discover Models** → **Register Models**

---

## 步驟 6:設定模型(1 分鐘)

1. 前往 **Settings** → **Models**
2. 設定:
   - **Language Model**:`ollama/mistral`(或是你下載的模型)
   - **Embedding Model**:`ollama/nomic-embed-text`
3. 點擊 **Save**

---

## 步驟 7:進入 Open Notebook(即時)

開啟瀏覽器:
```
http://localhost:8502
```

---

## 驗證清單

- [ ] Ollama 正在執行(終端機裡的 `ollama serve`)
- [ ] Docker 正在執行
- [ ] 可以連上 `http://localhost:8502`
- [ ] Ollama 憑證已用主機 URL 設定並測試成功
- [ ] 模型已註冊
- [ ] 對話功能正常

---

## 疑難排解

### 測試 Ollama 憑證時出現「Connection failed」

1. 確認 Ollama 正在執行:
   ```bash
   curl http://localhost:11434/api/version
   ```

2. 檢查防火牆是否允許 port 11434 的本機連線

3. Windows/macOS 使用者,確認容器內可以連到 `host.docker.internal`:
   ```bash
   docker exec <open_notebook_container> curl http://host.docker.internal:11434/api/version
   ```

### Ollama 無法啟動

```bash
# 查看 Ollama 日誌
ollama list

# 重新下載模型
ollama pull mistral
```

### SurrealDB 出現「Address already in use」

```bash
docker compose down
docker compose up -d
```

---

## 為什麼要用外部 Ollama?

| 做法 | Ollama 跑在 Docker 內 | Ollama 跑在外部 |
|----------|-----------------|-----------------|
| **資源隔離** | 分開 | 與主機共用 |
| **GPU 存取** | 需要設定 Docker GPU | 原生 GPU 存取 |
| **模型管理** | 透過 `docker exec` | 直接用終端機 |
| **記憶體用量** | 與主機隔離 | 與主機應用程式共用 |

如果你符合以下情況,建議使用**外部 Ollama**:
- 已經安裝並設定好 Ollama
- 想要 GPU 存取,但不想處理 Docker GPU passthrough 的複雜設定
- 偏好直接用命令列管理模型

---

## 更進一步

- **新增更多模型**:執行 `ollama pull <model>`,然後在 Open Notebook 中重新偵測
- **檢查 Ollama 狀態**:`ollama list` 會列出已下載的模型
- **自訂 Ollama**:編輯 `~/.ollama/config.yaml` 進行進階設定

---

**需要協助?** 加入我們的 [Discord 社群](https://discord.gg/37XJPXfz2w)
