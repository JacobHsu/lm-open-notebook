> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](frontend.md)

# 前端架構

Next.js 應用程式的分層方式,以及資料如何在其中流動。規範性規則(指令、i18n、注意事項)記載於 [`frontend/AGENTS.md`](../../frontend/AGENTS.md);這篇文件則是心智模型(mental model)。

## 分層

```
Pages (src/app/, App Router) → Feature components (src/components/) → Hooks (src/lib/hooks/)
                                                                          ↓
                              Stores (src/lib/stores/) → API modules (src/lib/api/) → Backend
```

- **Pages** —— 路由端點。路由群組 `(auth)` / `(dashboard)` 用來組織路由,而不影響 URL。Pages 會呼叫 hooks 並渲染元件。
- **Components** —— 依功能分資料夾(`source/`、`notebooks/`、`podcasts/` 等),各自擁有頁面層級的狀態(loading、error);`components/ui/` 是無狀態的 Radix UI 包裝元件,以 Tailwind + CVA 設計樣式。
- **Hooks**(`src/lib/hooks/`)—— TanStack Query 的包裝層。Query hooks 回傳 `{ data, isLoading, error, refetch }`;mutation hooks 會使快取失效並顯示 toast。較複雜的 hooks(`useNotebookChat`、`useAsk`)還會加入 session 管理、上下文建構、SSE 串流。
- **Stores**(`src/lib/stores/`)—— 使用 Zustand 管理 auth 與 modal 狀態;`persist` middleware 會同步到 localStorage(auth token 存放在 `auth-storage`)。
- **API modules**(`src/lib/api/`)—— 依命名空間分類的型別化客戶端(`sourcesApi.list()` 等),建立在單一 axios 實例之上,並附加 auth/FormData/401 攔截器。

`app/layout.tsx` 中的 Provider 樹狀結構(由外到內):ErrorBoundary → ThemeProvider → QueryProvider → I18nProvider → ConnectionGuard → Toaster。

## 流程走查:notebook 聊天

1. `notebooks/[id]/page.tsx` 把 `notebookId` 傳給 `ChatColumn`。
2. `useNotebookChat()` 查詢 sessions、管理訊息狀態,並回傳 `{ messages, sendMessage(), setModelOverride() }`。
3. 送出訊息時:`buildContext()` 組合已選取的 sources/notes(token/字元數統計),呼叫 `chatApi.sendMessage()`,並套用**樂觀更新**(訊息先在本地端加入,若失敗則移除)。
4. 回應會更新 TanStack Query 快取;其他地方相關的 source/note mutation 會廣泛地讓快取失效,以便刷新過期的 UI。
5. 若在 session 建立前就設定模型覆寫,會先暫存為 pending 狀態,並在 session 建立時套用。

## 流程走查:檔案上傳

1. `SourceDialog` 收集檔案;`useFileUpload` 建構 FormData——巢狀的 JSON 欄位會被字串化。
2. 客戶端攔截器會刪除 Content-Type 標頭,讓瀏覽器自行設定 multipart boundary。
3. 上傳成功後,`queryClient.invalidateQueries(['sources'])` 會重新抓取列表;`useSourceStatus` 會在 source 處理中的期間每 2 秒輪詢一次。

## 快取策略

Query key 是階層式的(`QUERY_KEYS.sources(notebookId)`),但快取失效的範圍刻意設計得**較廣**(`['sources']` 會涵蓋所有相關項目)——這是精準度與簡單性之間的取捨。經常變動的資料會使用 `refetchOnWindowFocus: true`。

## 驗證(Auth)

Token 是透過實際的 API 呼叫(`/notebooks`)來驗證,而不是解碼 JWT,並在 auth store 中快取 30 秒。回應攔截器會在收到 401 時清除驗證狀態並導向 `/login`。登出僅在客戶端進行。

## 錯誤處理

`getApiErrorMessage()`(`lib/utils/error-handler.ts`)會先嘗試 i18n 對照,再退回後端提供的描述性訊息——而後端的錯誤分類系統本身已經讓這些訊息對使用者友善(參見 [architecture.zh.md](architecture.zh.md))。Mutation 會以 toast 的形式呈現錯誤;應用程式層級的 ErrorBoundary 則會攔截渲染錯誤。
