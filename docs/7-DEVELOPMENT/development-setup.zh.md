> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](development-setup.md)

# 本機開發環境設定

本指南會帶你完成 Open Notebook 的本機開發環境設定,依照以下步驟即可在你的機器上跑起完整堆疊。

## 前置需求

開始之前,請確認已安裝以下工具:

- **Python 3.11+** - 用 `python --version` 檢查
- **uv**(建議)或 **pip** - 安裝方式見:https://github.com/astral-sh/uv
- **SurrealDB** - 透過 Docker 或執行檔安裝(見下方)
- **Docker**(選用)- 用於容器化資料庫
- **Node.js 18+**(選用)- 用於前端開發
- **Git** - 版本控制

## 步驟 1:Clone 與初始設定

```bash
# Clone 儲存庫
git clone https://github.com/lfnovo/open-notebook.git
cd open-notebook

# 新增 upstream remote,方便讓你的 fork 保持更新
git remote add upstream https://github.com/lfnovo/open-notebook.git
```

## 步驟 2:安裝 Python 依賴

```bash
# 使用 uv(建議)
uv sync

# 或使用 pip
pip install -e .
```

## 步驟 3:環境變數

在專案根目錄建立 `.env` 檔案並填入你的設定:

```bash
# 從範例複製
cp .env.example .env
```

編輯 `.env` 設定內容:

```bash
# 資料庫
SURREAL_URL=ws://localhost:8000/rpc
SURREAL_USER=root
SURREAL_PASSWORD=password
SURREAL_NAMESPACE=open_notebook
SURREAL_DATABASE=development

# 憑證加密(儲存 API 金鑰時必須設定)
OPEN_NOTEBOOK_ENCRYPTION_KEY=my-dev-secret-key

# 應用程式
OPEN_NOTEBOOK_PASSWORD=  # 選用的密碼保護
DEBUG=true
LOG_LEVEL=DEBUG
```

### AI 供應商設定

啟動 API 與前端之後,透過 Settings UI 設定你的 AI 供應商:

1. 開啟 **http://localhost:3000** → **Settings** → **API Keys**
2. 點選 **Add Credential** → 選擇你的供應商
3. 輸入你的 API 金鑰(從供應商的後台取得)
4. 點選 **Save**,再點選 **Test Connection**
5. 點選 **Discover Models** → **Register Models**

常見供應商:
- **OpenAI** - https://platform.openai.com/api-keys
- **Anthropic (Claude)** - https://console.anthropic.com/
- **Google** - https://ai.google.dev/
- **Groq** - https://console.groq.com/

本機開發時,你也可以使用:
- **Ollama** - 免 API 金鑰,在本機執行(見下方「本機 Ollama」)

> **注意:** API 金鑰的環境變數(例如 `OPENAI_API_KEY`)已被淘汰,請改用 Settings UI 管理憑證。

## 步驟 4:啟動 SurrealDB

### 方案 A:使用 Docker(建議)

```bash
# 以記憶體模式啟動 SurrealDB(連接埠只發布在 localhost——
# 因為資料庫使用預設帳密,絕對不要發布在 0.0.0.0)
docker run -d --name surrealdb -p 127.0.0.1:8000:8000 \
  surrealdb/surrealdb:v2 start \
  --user root --pass password \
  memory

# 或使用持久化儲存
docker run -d --name surrealdb -p 127.0.0.1:8000:8000 \
  -v surrealdb_data:/data \
  surrealdb/surrealdb:v2 start \
  --user root --pass password \
  file:/data/surreal.db
```

### 方案 B:使用 Make

```bash
make database
```

### 方案 C:使用 Docker Compose

```bash
docker compose up -d surrealdb
```

### 確認 SurrealDB 正常運作

```bash
# 應該會顯示伺服器資訊
curl http://localhost:8000/
```

## 步驟 5:執行資料庫遷移

啟動 API 時,資料庫遷移會自動執行。第一次啟動時會套用所有尚未執行的遷移。

如果要手動驗證遷移:

```bash
# API 啟動時會自動執行遷移
uv run python -m api.main
```

檢查日誌,你應該會看到類似這樣的訊息:
```
Running migration 001_initial_schema
Running migration 002_add_vectors
...
Migrations completed successfully
```

## 步驟 6:啟動 API 伺服器

在新的終端機視窗中執行:

```bash
# 終端機 2:啟動 API(連接埠 5055)
uv run --env-file .env uvicorn api.main:app --host 0.0.0.0 --port 5055

# 或使用捷徑指令
make api
```

你應該會看到:
```
INFO:     Application startup complete
INFO:     Uvicorn running on http://0.0.0.0:5055
```

### 確認 API 正常運作

```bash
# 檢查健康檢查端點
curl http://localhost:5055/health

# 查看 API 文件
open http://localhost:5055/docs
```

## 步驟 7:啟動前端(選用)

如果你要開發前端,在另一個終端機啟動 Next.js:

```bash
# 終端機 3:啟動 Next.js 前端(連接埠 3000)
cd frontend
npm install  # 只需在第一次執行
npm run dev
```

你應該會看到:
```
> next dev
  ▲ Next.js 16.x
  - Local:        http://localhost:3000
```

### 開啟前端

在瀏覽器開啟:http://localhost:3000

## 驗證清單

設定完成後,確認一切正常運作:

- [ ] **SurrealDB**:`curl http://localhost:8000/` 有回傳內容
- [ ] **API**:`curl http://localhost:5055/health` 回傳 `{"status": "ok"}`
- [ ] **API 文件**:`open http://localhost:5055/docs` 可以正常開啟
- [ ] **資料庫**:API 日誌顯示遷移已完成
- [ ] **前端**(選用):`http://localhost:3000` 可以載入

## 開發工作流程:什麼時候該用哪種?

| 工作流程 | 使用時機 | 速度 | 與正式環境的相似度 |
|----------|----------|-------|-------------------|
| **本機服務**(`make start-all`) | 日常開發,最快的迭代速度 | ⚡⚡⚡ 快 | 中 |
| **Docker Compose**(`make dev`) | 測試容器化設定 | ⚡⚡ 中 | 高 |
| **本機 Docker 建置**(`make docker-build-local`) | 測試 Dockerfile 的變更 | ⚡ 慢 | 非常高 |
| **多平台建置**(`make docker-push`) | 發佈正式版本(見[發佈流程](../../.github/RELEASE_PROCESS.md)) | 🐌 非常慢 | 完全一致 |

本機服務具備熱重載、可直接查看日誌且容易除錯;Docker Compose(透過 `make dev` 使用 `examples/docker-compose-dev.yml`、透過 `make full` 使用 `examples/docker-compose-full-local.yml`)則更接近正式環境。在 PR 中改動任何 Docker 相關內容之前,請先執行 `make docker-build-local`。

## 一起啟動服務

### 快速啟動所有服務

```bash
make start-all    # SurrealDB + API + worker + 前端
make status       # 查看目前執行中的服務
make stop-all     # 停止所有服務
```

### 個別終端機(建議用於開發)

**終端機 1 - 資料庫:**
```bash
make database
```

**終端機 2 - API:**
```bash
make api
```

**終端機 3 - 背景 worker**(podcast、embedding、來源處理都需要):
```bash
make worker-start
```

**終端機 4 - 前端:**
```bash
cd frontend && npm run dev
```

### 效能建議

1. 日常工作請使用 `make start-all`,而不是 Docker
2. 讓 SurrealDB 在多次工作階段之間持續執行(`make database`)
3. 只有在測試 Dockerfile 變更時才使用 `make docker-build-local`
4. 除非要準備發佈,否則不要跑多平台建置
5. 遇到奇怪的狀況時清一下快取:`make clean-cache`、`docker system prune -a`

## 開發工具設定

### Pre-commit Hooks(選用但建議設定)

Pre-commit hooks 會在每次提交前自動執行設定好的檢查,
與 CI 的檢查項目一致,讓本機提交失敗的原因跟 PR 會失敗的原因相同。
設定檔 `.pre-commit-config.yaml` 串接了以下工具:

| 工具 | 檢查內容 | 對應的 CI 項目 |
|------|----------------|---------------|
| **ruff**(lint) | Python lint 規則(`E`、`F`、`I`) | `ruff check .` |
| **ruff**(format) | Python 格式化(行長 88) | 尚未納入 CI 檢查 |
| **mypy** | Python 型別正確性 | `python -m mypy .` |
| **pre-commit-hooks** | 大型檔案、合併衝突、YAML/TOML 語法、行尾空白、檔案結尾換行 | — |

Pre-commit 已經包含在專案的開發依賴中,安裝好 hooks 之後,
每次 `git commit` 時就會自動執行:

```bash
uv run pre-commit install
```

**手動執行:**

```bash
# 檢查所有檔案(在變更 hook 設定後很好用)
uv run pre-commit run --all-files

# 只執行指定的 hook
uv run pre-commit run ruff --all-files
```

**暫時跳過 hooks:**

```bash
# 跳過此次提交的所有 hooks
git commit --no-verify

# 跳過特定 hook(例如較慢的 mypy)
SKIP=mypy git commit
```

**更新 hook 版本:**

```bash
uv run pre-commit autoupdate
```

請讓 `.pre-commit-config.yaml` 裡的 `rev:` 版本,與 `pyproject.toml` 中
`[dependency-groups] dev` 所列的版本保持一致。

### 程式碼品質指令

```bash
# Lint Python 程式碼(自動修正)
make ruff
# 或:ruff check . --fix

# 型別檢查 Python 程式碼
make lint
# 或:uv run python -m mypy .

# 執行測試
uv run pytest

# 執行測試並產生涵蓋率報告
uv run pytest --cov=open_notebook
```

## 常見開發工作

### 執行測試

```bash
# 執行所有測試
uv run pytest

# 執行特定測試檔
uv run pytest tests/test_notebooks.py

# 執行測試並產生涵蓋率報告
uv run pytest --cov=open_notebook --cov-report=html
```

### 建立功能分支

```bash
# 建立並切換到新分支
git checkout -b feature/my-feature

# 進行變更後提交
git add .
git commit -m "feat: add my feature"

# 推送到你的 fork
git push origin feature/my-feature
```

### 從 Upstream 更新

```bash
# 取得最新變更
git fetch upstream

# Rebase 你的分支
git rebase upstream/main

# 推送更新後的分支
git push origin feature/my-feature -f
```

## 疑難排解

### SurrealDB 出現「Connection refused」

**問題**:API 無法連線到 SurrealDB

**解決方法**:
1. 確認 SurrealDB 是否正在執行:`docker ps | grep surrealdb`
2. 確認 `.env` 中的 URL:應該是 `ws://localhost:8000/rpc`
3. 重新啟動 SurrealDB:`docker stop surrealdb && docker rm surrealdb`
4. 接著重新啟動:`docker run -d --name surrealdb -p 127.0.0.1:8000:8000 surrealdb/surrealdb:v2 start --user root --pass password memory`

### 「Address already in use」

**問題**:連接埠 5055 或 3000 已被佔用

**解決方法**:
```bash
# 找出佔用連接埠的行程
lsof -i :5055  # 檢查連接埠 5055

# 終止該行程(macOS/Linux)
kill -9 <PID>

# 或改用其他連接埠
uvicorn api.main:app --port 5056
```

### 找不到模組的錯誤

**問題**:執行 API 時出現匯入錯誤

**解決方法**:
```bash
# 重新安裝依賴
uv sync

# 或使用 pip
pip install -e .
```

### 資料庫遷移失敗

**問題**:API 因為遷移錯誤而無法啟動

**解決方法**:
1. 確認 SurrealDB 是否正在執行:`curl http://localhost:8000/`
2. 確認 `.env` 中的憑證與你的 SurrealDB 設定相符
3. 查看日誌中具體的遷移錯誤訊息:`make api 2>&1 | grep -i migration`
4. 確認資料庫存在:到 http://localhost:8000/ 檢查 SurrealDB 主控台

### 遷移沒有套用

**問題**:資料庫結構看起來是舊的

**解決方法**:
1. 重新啟動 API——遷移會在啟動時執行:`make api`
2. 確認日誌顯示「Migrations completed successfully」
3. 確認 `/migrations/` 資料夾存在且裡面有檔案
4. 確認 SurrealDB 可寫入,且不是處於唯讀模式

## 選用:本機 Ollama 設定

如果要用本機 AI 模型測試:

```bash
# 從 https://ollama.ai 安裝 Ollama

# 拉取模型(例如 Mistral 7B)
ollama pull mistral
```

接著透過 Settings UI 設定:
1. 前往 **Settings** → **API Keys** → **Add Credential** → **Ollama**
2. 輸入 Base URL:`http://localhost:11434`
3. 點選 **Save**,再點選 **Test Connection**
4. 點選 **Discover Models** → **Register Models**

## 選用:Docker 開發環境

在 Docker 中執行整個堆疊:

```bash
# 啟動所有服務
docker compose --profile multi up

# 查看日誌
docker compose logs -f

# 停止服務
docker compose down
```

## 下一步

設定完成之後:

1. **閱讀貢獻指南** - [contributing.zh.md](contributing.zh.md)
2. **探索架構** - 查閱相關文件
3. **找一個 Issue 來處理** - 在 GitHub 上尋找「good first issue」
4. **設定 Pre-commit** - 安裝 git hooks 以確保程式碼品質
5. **加入 Discord** - https://discord.gg/37XJPXfz2w

## 尋求協助

如果卡關了:

- **Discord**:[加入我們的伺服器](https://discord.gg/37XJPXfz2w)取得即時協助
- **GitHub Issues**:查看現有 issue 是否有類似問題
- **GitHub Discussions**:在討論區提問
- **文件**:參考 [code-standards.zh.md](code-standards.zh.md) 與 [testing.zh.md](testing.zh.md)

---

**準備好要貢獻了嗎?** 前往 [contributing.zh.md](contributing.zh.md) 了解貢獻工作流程。
