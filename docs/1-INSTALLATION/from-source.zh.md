> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](from-source.md)

# 原始碼安裝

Clone 儲存庫並在本機執行。**適用於開發者與貢獻者。**

## 前置需求

- **Python 3.11+** - [下載](https://www.python.org/)
- **Node.js 18+** - [下載](https://nodejs.org/)
- **Git** - [下載](https://git-scm.com/)
- **Docker**(用於 SurrealDB)- [下載](https://docker.com/)
- **uv**(Python 套件管理工具)- `curl -LsSf https://astral.sh/uv/install.sh | sh`
- OpenAI 或類似服務的 API 金鑰(或使用免費的 Ollama)

## 快速設定(10 分鐘)

### 1. Clone 儲存庫

```bash
git clone https://github.com/lfnovo/open-notebook.git
cd open-notebook

# 如果你是 fork 過來的:
git clone https://github.com/YOUR_USERNAME/open-notebook.git
cd open-notebook
git remote add upstream https://github.com/lfnovo/open-notebook.git
```

### 2. 安裝 Python 依賴

```bash
uv sync
uv pip install python-magic
```

#### 2.1 替代方案:Conda 設定(選用)

如果你偏好用 **Conda** 管理環境,可以用以下步驟取代標準的 `uv sync`:

```bash
# 建立並啟用環境
conda create -n open-notebook python=3.11 -y
conda activate open-notebook

# 在 conda 內安裝 uv,以維持與 Makefile 的相容性
conda install -c conda-forge uv nodejs -y

# 同步依賴
uv sync
```

> **注意**:在你的 Conda 環境內安裝 `uv`,能確保 `make start-all`、`make api` 這類指令能持續正常運作。

### 3. 啟動 SurrealDB

```bash
# 終端機 1
make database
# 或者:docker compose up surrealdb
```

### 4. 設定環境變數

```bash
cp .env.example .env
# 編輯 .env 並設定:
# OPEN_NOTEBOOK_ENCRYPTION_KEY=my-secret-key
```

啟動應用程式之後,透過瀏覽器裡的 **Manage → Models** 介面設定 AI 供應商。

### 5. 啟動 API

```bash
# 終端機 2
make api
# 或者:uv run --env-file .env uvicorn api.main:app --host 0.0.0.0 --port 5055
```

### 6. 啟動 Worker

Source 與 note 的處理(內容擷取、嵌入、洞察分析)是以背景工作(background jobs)的形式派送,
由**獨立的 worker** 程序消化。沒有它,每個 source 都會永遠卡在 `Source processing status: CommandStatus.NEW`。

```bash
# 終端機 3
make worker
# 或者:uv run --env-file .env surreal-commands-worker --import-modules commands
```

> `make start-all` 會同時啟動 Database + API + Worker + Frontend;上面的步驟是逐一啟動,
> 讓你能看到每個程序各自的 log。

### 7. 啟動前端

```bash
# 終端機 4
cd frontend && npm install && npm run dev
```

### 8. 存取

- **前端**:http://localhost:3000
- **API 文件**:http://localhost:5055/docs
- **資料庫**:http://localhost:8000

### 9. 設定 AI 供應商

1. 開啟 http://localhost:3000
2. 前往 **Manage** → **Models**
3. 點選 **Add Credential** → 選擇你的供應商 → 貼上 API 金鑰
4. 點選 **Save**,再點選 **Test Connection**
5. 點選 **Discover Models** → **Register Models**

---

## 開發工作流程

### 程式碼品質

```bash
# 格式化並檢查 Python
make ruff
# 或者:ruff check . --fix

# 型別檢查
make lint
# 或者:uv run python -m mypy .
```

### 執行測試

```bash
uv run pytest tests/
```

### 常用指令

```bash
# 啟動全部服務
make start-all

# 查看 API 文件
open http://localhost:5055/docs

# 檢查資料庫遷移
# (API 啟動時會自動執行)

# 清理
make clean
```

---

## 疑難排解

### Python 版本太舊

```bash
python --version  # 檢查版本
uv sync --python 3.11  # 指定使用特定版本
```

### npm: command not found

從 https://nodejs.org/ 安裝 Node.js

### 資料庫連線錯誤

```bash
docker ps  # 檢查 SurrealDB 是否正在執行
docker logs surrealdb  # 查看 log
```

### 連接埠 5055 已被佔用

```bash
# 改用其他連接埠
uv run uvicorn api.main:app --port 5056
```

---

## 下一步

1. 閱讀[開發指南](../7-DEVELOPMENT/quick-start.zh.md)
2. 參見[架構概觀](../7-DEVELOPMENT/architecture.zh.md)
3. 查看[貢獻指南](../7-DEVELOPMENT/contributing.zh.md)

---

## 需要協助?

- **Discord**:[社群](https://discord.gg/37XJPXfz2w)
- **Issues**:[GitHub Issues](https://github.com/lfnovo/open-notebook/issues)
