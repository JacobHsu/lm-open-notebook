> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](prompts.md)

# Prompt 工程

Prompt 是如何組織的,以及它們所使用的模式。所有 Prompt 都是 `prompts/` 底下的 Jinja2 樣板,透過 [ai-prompter](https://github.com/lfnovo/ai-prompter) 函式庫渲染——Prompt 工程的邏輯放在樣板裡,而不是 Python 程式碼中。

## 版面配置與渲染

樣板依工作流程分組——`ask/`、`chat/`、`source_chat/`、`podcast/`——並以不含副檔名的路徑來參照:

```python
from ai_prompter import Prompter
prompt = Prompter(prompt_template="ask/entry", parser=parser).render(data=state)
```

機制性規則(路徑語法、`data=` 鍵值比對、parser 注入、不支援繼承、快取→須重啟)記載於 [`open_notebook/AGENTS.md`](../../open_notebook/AGENTS.md)。這篇文件涵蓋的是*設計模式*本身。

## 模式:多階段鏈(ask 工作流程)

ask 流程是由 `graphs/ask.py` 編排的三個樣板組成:

```
entry.jinja          user question → JSON search strategy(PydanticOutputParser)
   ↓
query_process.jinja  one search term + retrieved results → sub-answer(parallel, one per search)
   ↓
final_answer.jinja   all sub-answers → synthesized final response with citations
```

各階段的邊界讓每個 Prompt 只需專注做好一件事,而 JSON 格式的策略輸出讓後續的 fan-out 具有確定性。

## 模式:條件式變數注入

樣板透過 Jinja 條件式接受選用變數,讓同一份樣板能服務多種不同的上下文形狀(podcast outline 可處理清單或字串形式的上下文;source_chat 會視情況注入選用的 notebook/insight 資料):

```jinja
{% if notebook %}
# PROJECT INFORMATION
{{ notebook }}
{% endif %}
```

要留意寬鬆的真值判斷(`{% if var %}` 對空字串/空清單會判定為 false)以及 for 迴圈的假設(把字串當成清單傳入時,會逐字元疊代)。

## 模式:重複強調引用規則

負責產生回應的樣板(ask、chat)會反覆陳述引用規則——`[source:id]`、`[note:id]`、`[insight:id]`、「不可捏造文件 ID」——而且**會重複多次並附上範例**。LLM 若缺乏這樣的提醒就容易產生幻覺引用;重複陳述加上範例能明顯降低這個問題。修改這些樣板時請保留這種重複。

## 模式:格式指示委派

樣板會透過 `{{ format_instructions }}` 這個插槽,交由呼叫端的 OutputParser 填入內容。輸出格式的演進因此可以在 Python(Pydantic 模型)中完成,不必動到樣板。若樣板缺少這個佔位符,parser 會被靜默忽略——新增結構化輸出時務必檢查是否有這個佔位符。

## 模式:擴充思考的分離(podcast)

Podcast 樣板會指示具備思考能力的模型把推理過程放在 `<think>` 標籤內,並在標籤之後才輸出 JSON;下游的 `clean_thinking_content()` 會移除這些標籤。如果新增的樣板需要從具備思考能力的模型取得結構化輸出,請加入相同的指示區塊。
