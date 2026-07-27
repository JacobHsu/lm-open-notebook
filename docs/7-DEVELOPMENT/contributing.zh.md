> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](contributing.md)

# 貢獻 Open Notebook

感謝你有興趣貢獻 Open Notebook!我們歡迎任何技術水平的開發者參與貢獻。本指南將協助你了解我們的貢獻工作流程,以及什麼樣的貢獻算是好的貢獻。

## 🚦 想法用 Discussions,工作用 Issues

Open Notebook 將**探索**與**執行**分開:

- **功能請求、想法、行為變更、產品/設計/架構提案,以及貢獻提案,一律從 [GitHub Discussions](https://github.com/lfnovo/open-notebook/discussions/new?category=ideas) 開始。**社群在這裡探討問題,維護者在這裡做出產品或設計決策。
- **可重現的錯誤從 [GitHub Issues](https://github.com/lfnovo/open-notebook/issues/new/choose) 開始。**
- **實作從已核准的 Issue 開始。**當一個想法夠明確且被接受後,維護者會從該 Discussion 建立 Issue,界定範圍並指派,之後才開始撰寫程式碼。

也就是說,非小型的貢獻會走以下兩條路徑之一:

```text
想法或功能 → Discussion → 決策 → 已核准的 Issue → 程式碼 → PR
可重現的錯誤             → 已分流的 Issue → 程式碼 → PR
```

**可以略過以上兩者、直接開 PR 的情況:**
- 錯字、失效連結,以及小型的文件澄清
- 小型且明顯的錯誤修正——只有幾行、答案顯而易見、不涉及設計決策
- 翻譯修正或補齊缺漏的 i18n 鍵值

**一定要先有 Discussion 的情況:**
- 任何規模的新功能
- 架構或結構性變更
- Breaking changes(破壞性變更)
- 產品、UX 或行為變更
- 任何「怎麼做」有一種以上合理答案的事情

**已經在沒有事先討論或核准 Issue 的情況下寫了不小的東西?**別把它丟掉:把 PR 標記為**草稿(draft)**。針對功能或設計提案開一個 Discussion,或針對可重現的錯誤開一個 Issue,並從 PR 連結過去。維護者會協助安排後續流程。

**為什麼要有這個流程?**
- 避免重複工作
- 確保解法符合我們的架構與設計原則
- 在動手寫程式碼前先取得回饋,節省你的時間
- 協助維護者掌控專案方向

> ⚠️ **沒有已核准 Issue 的非小型 Pull Request 可能會被關閉**,即使程式碼寫得很好也一樣。Discussion 是探索想法的地方;Issue 則是專案對執行這件事的承諾。

## 行為準則

參與本專案即表示你同意遵守我們的[行為準則](/CODE_OF_CONDUCT.md)。請保持尊重、建設性與合作的態度。

## 我能如何貢獻?

### 回報錯誤

1. **搜尋既有 Issue**——確認這個錯誤是否已經被回報過
2. **建立錯誤報告**——使用[錯誤報告範本](https://github.com/lfnovo/open-notebook/issues/new?template=bug_report.yml)
3. **提供詳細資訊**,包含:
   - 重現步驟
   - 預期行為與實際行為
   - 日誌、截圖或錯誤訊息
   - 你的環境(作業系統、Docker 版本、Open Notebook 版本)
4. **表明是否想自己修**——如果你有興趣,勾選「I would like to work on this」

### 建議功能

1. **搜尋既有的 Discussions 與 Issues**——確認這個問題是否已經在被討論或處理中
2. **開一個 Idea Discussion**——使用 [Ideas 表單](https://github.com/lfnovo/open-notebook/discussions/new?category=ideas)
3. **從問題與成果談起**——說明你想達成什麼、現在有什麼困難、成功的樣子是什麼
4. **有用的話,附上可能的方向**——歡迎附上實作構想與參考資料,但非必要
5. **加入探討**——協助回答問題、評估取捨,或測試原型
6. **等待「畢業」後再動手寫程式碼**——一旦被接受,該提案會變成一個或多個可被指派的已核准 Issue

### 貢獻程式碼(Pull Requests)

**重要:對於非小型的工作,請在動手前先從已核准且已指派的 Issue 開始。想法要先經過 Discussions 才能走到這一步;可重現的錯誤則要先經過 Issue 分流。**

一旦你的 Issue 被指派:

1. **Fork 這個儲存庫**,並從 `main` 建立你的分支
2. **理解我們的願景與原則**——閱讀 [VISION.md](../../VISION.md)(產品是什麼、要往哪裡走)以及 [design-principles.md](design-principles.zh.md)(工程實踐)
3. **遵循我們的架構**——參考架構文件以了解專案結構
4. **寫出高品質的程式碼**——遵循 [code-standards.md](code-standards.zh.md) 中列出的標準
5. **測試你的變更**——測試指引請見 [testing.md](testing.zh.md)
6. **更新文件**——如果你變更了功能,請更新相關文件
7. **建立你的 PR**:
   - 參照 Issue 編號(例如「Fixes #123」)
   - 說明變更了什麼、為什麼變更
   - UI 變更請附上截圖
   - 讓 PR 保持聚焦——一個 PR 只處理一個 Issue

### 什麼樣的貢獻才是好的貢獻?

✅ **我們喜歡以下這樣的 PR:**
- 解決 Issue 中描述的實際問題
- 遵循我們的架構與程式碼標準
- 包含測試與文件
- 範圍清楚(專注於一件事)
- 有清楚的 commit 訊息

❌ **我們可能會關閉以下這樣的 PR:**
- 非小型變更卻沒有對應的已核准 Issue(小型且明顯的修正不在此限——參見上方的工作流程)
- 沒有經過討論就引入 Breaking changes
- 與我們的架構願景相衝突
- 缺少測試或文件
- 試圖同時解決多個不相關的問題

### AI 輔助與代理人產出的 PR

有相當比例的貢獻——包括我們自己的貢獻在內——是用編碼代理人(Claude Code、Cursor、Copilot 等)寫成的。這是受歡迎的。工具本身不會改變契約;**操作者依然是作者**:

1. **PR 由你負責。**你必須讀過、理解並能夠解釋這份 diff 裡的每一行。在審查中,「這是代理人寫的」永遠不是一個答案。
2. **先討論再承諾;先有已核准的 Issue 再實作。**代理人讓產出大量未經請求的 PR 變得很廉價——這類 PR 無論程式碼品質如何,都會像其他未經指派的 PR 一樣被關閉。小型且明顯的修正不在此限。對於較大的工作,請用 Discussion 來形塑想法,或用 Issue 回報可重現的錯誤,然後等待一個已核准的工作項目。
3. **測試必須是真的跑過。**貼上真實的輸出結果。代理人「宣稱」測試通過並不算測試證據。
4. **讓你的代理人參考正確的內容。**這個儲存庫提供了 `AGENTS.md` 檔案(根目錄、`open_notebook/`、`frontend/`)說明規範性規則,以及 [change-playbooks.md](change-playbooks.zh.md) 提供常見變更的逐步操作指南——讀過這些文件的代理人產出的 PR 會更快通過審查。
5. **保持範圍聚焦。**代理人往往會順手「改善」周邊的程式碼。不相關的重構應該放到獨立的 issue/PR。

是否揭露使用了 AI 輔助,我們表示歡迎但非強制——結果的責任才是重點,而無論如何那都是你的責任。

## Git Commit 訊息

- 使用現在式(「Add feature」而非「Added feature」)
- 使用祈使語氣(「Move cursor to...」而非「Moves cursor to...」)
- 第一行限制在 72 個字元以內
- 在第一行之後,盡量多參照 Issue 與 Pull Request

## 開發工作流程

### 分支策略

我們採用**功能分支工作流程**:

1. **主分支**:`main` - 可用於正式環境的程式碼
2. **功能分支**:`feature/description` - 新功能
3. **錯誤修正**:`fix/description` - 錯誤修正
4. **文件**:`docs/description` - 文件更新

### 進行變更

1. **建立功能分支**:
```bash
git checkout -b feature/amazing-new-feature
```

2. **依照我們的程式碼標準進行變更**

3. **測試你的變更**:
```bash
# 執行測試
uv run pytest

# 執行 lint
uv run ruff check .

# 執行格式化
uv run ruff format .
```

4. **Commit 你的變更**:
```bash
git add .
git commit -m "feat: add amazing new feature"
```

5. **Push 並建立 PR**:
```bash
git push origin feature/amazing-new-feature
# 接著在 GitHub 上建立 Pull Request
```

### 保持你的 Fork 更新

```bash
# 抓取 upstream 的變更
git fetch upstream

# 切換到 main 並合併
git checkout main
git merge upstream/main

# Push 到你的 fork
git push origin main
```

## Pull Request 流程

建立 Pull Request 時:

1. **連結你的 Issue**——在 PR 描述中參照 Issue 編號
2. **描述你的變更**——說明變更了什麼、為什麼變更
3. **提供測試證據**——截圖、測試結果或日誌
4. **檢查 PR 範本**——確認你已完成所有必填欄位
5. **等待審查**——維護者將在一週內審查你的 PR

### PR 審查期望

- 程式碼審查的回饋是針對程式碼,不是針對人
- 對建議與替代方案抱持開放態度
- 以清楚且尊重的態度回應審查意見
- 若回饋不清楚,請提出問題

## 目前的優先領域

我們目前積極尋求以下領域的貢獻:

1. **前端強化**——協助改善 Next.js/React UI,加入即時更新與更好的使用者體驗
2. **測試**——擴大所有元件的測試涵蓋率
3. **效能**——非同步處理改善與快取
4. **文件**——API 範例與使用者指南
5. **整合**——新的內容來源與 AI 供應商

## 尋求協助

### 社群支援

- **Discord**:[加入我們的 Discord 伺服器](https://discord.gg/37XJPXfz2w)取得即時協助
- **GitHub Discussions**:用於提問、發想、功能建議、產品方向、設計與架構討論
- **GitHub Issues**:用於可重現的錯誤回報與已核准的工作項目

### 文件參考

- [VISION.md](../../VISION.md) - 產品定位與目前方向
- [設計原則](design-principles.zh.md) - 工程實踐與反模式
- [決策紀錄](decisions/README.md) - 事情為什麼會是這樣
- [程式碼標準](code-standards.zh.md) - 依語言區分的程式碼指引
- [測試指南](testing.zh.md) - 如何撰寫測試
- [開發環境設定](development-setup.zh.md) - 開始本機開發

## 表彰

我們透過以下方式表彰貢獻:

- 在發佈紀錄中列出 **GitHub 名單**
- 在 Discord 中的**社群表彰**
- 專案分析中的**貢獻統計**
- 對活躍貢獻者的**維護者候選考量**

---

感謝你為 Open Notebook 做出貢獻!你的貢獻讓研究對每個人來說都更容易取得、更保有隱私。

若對本指南或一般貢獻事宜有任何疑問,歡迎在 [Discord](https://discord.gg/37XJPXfz2w) 聯絡我們,或開一個 GitHub Discussion。
