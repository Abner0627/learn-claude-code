# s06_context_compact.py 執行流程詳解

## 問題與解決方案摘要

### 問題

上下文窗口是有限的。讀取一個 1000 行的檔案就消耗約 4000 token；讀取 30 個檔案、執行 20 條指令，輕易就能突破 100k token。不壓縮，Agent 根本無法在大型專案中持續運作。

### 解決方案

三層壓縮，激進程度遞增：
- **第一層（micro_compact）**：每次 LLM 呼叫前，將超過 3 輪的舊 `tool_result` 替換為 `[Previous: used {tool_name}]`
- **第二層（auto_compact）**：token 超過 50,000 時，儲存完整對話至 `.transcripts/`，讓 LLM 生成摘要，並重置 `messages`
- **第三層（compact tool）**：模型主動呼叫 `compact` 工具，實現手動壓縮

---

本章介紹了「對話壓縮（Context Compression）」技術，透過三層管線確保 Agent 具備長程任務的處理能力。

### 微壓縮階段 (Micro Compact - 每一輪)

1.  **清空舊內容**：在每一輪對話前，程式會掃描 `messages`。
2.  **替換標籤**：對於非最近 3 次的工具結果，若內容過長，會被替換為 `[Previous: used bash]`。
    *   這保留了「模型曾做過這件事」的歷史記憶，但大幅降低了 Token 消耗。

---

### 自動壓縮階段 (Auto Compact - 門檻觸發)

1.  **Token 估算**：如果對話歷史的 Token 估算值超過 `THRESHOLD` (如 50,000)。
2.  **發起摘要請求**：
    *   程式自動存檔完整歷史至 `.transcripts/`。
    *   向模型發出一個隱含請求：`"Summarize this conversation for continuity..."`。
3.  **重設對話**：
    *   將所有舊訊息移除。
    *   替換為單一的總結訊息，並附上存檔路徑。

---

### 手動壓縮階段 (Manual Compact - 模型觸發)

1.  **模型自我判斷**：當模型認為任務已經切換，或之前的對話細節不再重要時。
2.  **主動呼叫 `compact`**：模型發出工具呼叫請求壓縮對話。
3.  **執行壓縮邏輯**：觸發與自動壓縮相同的「存檔 + 摘要」流程。

### 核心總結
本章實作了 **「策略性遺忘」**。透過摘要化與歷史歸檔，Agent 既能維持對大局的理解，又能將注意力集中在當下的工作，實現了理論上的「無限對話」。

---

## 三層壓縮機制詳解

### Layer 1：微壓縮 (Micro Compact) 的設計原則

`KEEP_RECENT` 參數（預設為 3）決定保留最近幾次工具結果的完整內容。超過此數量的舊結果會被替換為形如 `[Previous: used bash]` 的佔位符。

這樣設計的關鍵在於：**模型需要記得「做過什麼」，但不需要記得「完整輸出是什麼」**。只要知道「三輪前呼叫了 bash」，模型就能正常推進，而不必在每次 LLM 呼叫時重讀數千 token 的命令輸出。

### Layer 2：自動壓縮 (Auto Compact) 的觸發與恢復

自動壓縮的觸發條件是 Token 估算值超過 `THRESHOLD`（如 50,000）。其核心流程分為三步：

1. **封存完整歷史**：將目前所有的 `messages` 寫入 `.transcripts/transcript_{timestamp}.jsonl`。即使後續上下文被清空，完整記錄仍保存在磁碟上，可供人工審查或事後分析。
2. **呼叫模型做摘要**：傳送獨立的摘要請求給 LLM，要求其從完整歷史中提煉出「繼續工作所需的關鍵資訊」。
3. **重設 messages**：將對話歷史縮減為兩條訊息：摘要（role: user）+ 模型確認繼續（role: assistant）。

### Layer 3：手動壓縮 (Compact Tool) 的適用場景

手動壓縮由模型主動呼叫，適用於以下情況：
- **任務切換**：之前大量讀取的程式碼內容與新任務無關。
- **注意力重置**：模型判斷上下文「太雜」，主動要求清場後重新聚焦。

### 三層壓縮的觸發時機比較

| 壓縮層 | 觸發時機 | 保留粒度 | 資料丟失風險 |
| :--- | :--- | :--- | :--- |
| Micro Compact | 每一輪自動觸發 | 最近 N 次工具輸出的完整內容 | **極低** |
| Auto Compact | Token 超過閾值時自動觸發 | 一段 LLM 摘要文字 | **低**（Transcript 封存） |
| Manual Compact | 模型主動呼叫 `compact` | 同 Auto Compact | **低** |

> **重要原則**：壓縮並非刪除，而是「移出活躍上下文」。完整歷史始終存活於 `.transcripts/` 中，資訊沒有真正丟失。

### 循環整合三層的程式碼結構

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                      # 每輪靜默執行 Layer 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)     # Token 超標觸發 Layer 2
        response = client.messages.create(...)
        # ... 工具執行 ...
        if manual_compact_requested:
            messages[:] = auto_compact(messages)     # 模型主動觸發 Layer 3
```

`messages[:]` 是 Python 的原地修改語法，確保呼叫者持有的列表引用也被更新。

---

## 常見問題 Q&A
