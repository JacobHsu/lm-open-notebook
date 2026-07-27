> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](quick-start.md)

# 快速上手 - 開發環境

5 分鐘讓 Open Notebook 在本機跑起來。

## 前置需求

- **Python 3.11+**
- **Git**
- **uv**(套件管理工具)- 用 `curl -LsSf https://astral.sh/uv/install.sh | sh` 安裝
- **Docker**(選用,用於 SurrealDB)

## 1. Clone 儲存庫(2 分鐘)

```bash
# 先在 GitHub 上 Fork 這個儲存庫,再 clone 你的 fork
git clone https://github.com/YOUR_USERNAME/open-notebook.git
cd open-notebook

# 新增 upstream remote 以便更新
git remote add upstream https://github.com/lfnovo/open-notebook.git
```

## 2. 安裝依賴(2 分鐘)

```bash
# 安裝 Python 依賴
uv sync

# 確認 uv 可以正常運作
uv --version
```

## 3. 啟動服務(1 分鐘)

在不同的終端機視窗中執行:

```bash
# 終端機 1:啟動 SurrealDB(資料庫)
make database
# 或者:docker run -d --name surrealdb -p 127.0.0.1:8000:8000 surrealdb/surrealdb:v2 start --user root --pass password memory

# 終端機 2:啟動 API(後端,連接埠 5055)
make api
# 或者:uv run --env-file .env uvicorn api.main:app --host 0.0.0.0 --port 5055

# 終端機 3:啟動前端(UI,連接埠 3000)
cd frontend && npm run dev
```

## 4. 確認一切正常(立即可查)

- **API 健康檢查**:http://localhost:5055/health → 應回傳 `{"status": "ok"}`
- **API 文件**:http://localhost:5055/docs → 互動式 API 文件
- **前端**:http://localhost:3000 → Open Notebook UI

**三個都正常顯示?** ✅ 你已經準備好開始開發了!

---

## 下一步

- **第一個 Issue?** 挑一個 [good first issue](https://github.com/lfnovo/open-notebook/issues?q=label%3A%22good+first+issue%22)
- **想了解程式碼?** 閱讀[架構概觀](architecture.zh.md)
- **要開始改程式碼?** 依照[貢獻指南](contributing.zh.md)
- **需要設定細節?** 參考[開發環境設定](development-setup.zh.md)

---

## 疑難排解

### 「Port 5055 already in use」
```bash
# 找出誰在佔用這個連接埠
lsof -i :5055

# 改用其他連接埠
uv run uvicorn api.main:app --port 5056
```

### 「Can't connect to SurrealDB」
```bash
# 確認 SurrealDB 是否正在執行
docker ps | grep surrealdb

# 重新啟動它
make database
```

### 「Python version is too old」
```bash
# 確認你的 Python 版本
python --version  # 應該是 3.11+

# 指定使用 Python 3.11
uv sync --python 3.11
```

### 「npm: command not found」
```bash
# 從 https://nodejs.org/ 安裝 Node.js
# 接著安裝前端依賴
cd frontend && npm install
```

---

## 常用開發指令

```bash
# 執行測試
uv run pytest

# 格式化程式碼
make ruff

# 型別檢查
make lint

# 執行完整堆疊
make start-all

# 查看 API 文件
open http://localhost:5055/docs
```

---

需要更多協助?參考[開發環境設定](development-setup.zh.md)了解詳情,或加入我們的 [Discord](https://discord.gg/37XJPXfz2w)。
