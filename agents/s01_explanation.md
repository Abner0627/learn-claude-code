# s01_agent_loop.py 執行流程詳解

當輸入 Prompt 為 `"Create a file called hello.py that prints 'Hello, World!'"` 時，程式的執行流程如下：

### 第一輪迴圈 (Iteration 1)

1.  **發送 Prompt**：`client.messages.create` 被呼叫，帶入使用者的問題。
2.  **模型回應 (LLM Response)**：
    *   模型判斷需要執行指令，回傳 `stop_reason == "tool_use"`。
    *   `response.content` 包含一個工具區塊：
        ```python
        {
            "type": "tool_use",
            "id": "toolu_123",
            "name": "bash",
            "input": {"command": "echo "print('Hello, World!')" > hello.py"}
        }
        ```
3.  **紀錄對話**：將此 `assistant` 的工具呼叫存入 `messages` (即 `history`)。
4.  **執行工具**：
    *   `run_bash` 被觸發，在系統中建立 `hello.py`。
    *   `output` 為 `"(no output)"`。
5.  **回報結果**：
    *   `results` 列表填入 `tool_result`。
    *   執行 `messages.append({"role": "user", "content": results})`。*
    *   **注意：這裡角色是 `user`，因為是系統回饋給模型執行結果。**

---

### 第二輪迴圈 (Iteration 2)

1.  **再次呼叫模型**：這一次 `messages` 包含了原始問題、工具呼叫請求、以及工具執行結果。
2.  **模型確認結果**：
    *   模型看到工具回報成功，判斷任務已完成。
    *   回傳 `stop_reason == "end_turn"`。
    *   `response.content` 為純文字區塊：
        ```python
        {
            "type": "text",
            "text": "I have created the hello.py file for you."
        }
        ```
3.  **紀錄對話**：將最終回答存入 `history`。
4.  **退出迴圈**：`if response.stop_reason != "tool_use": return` 成立，結束 `agent_loop`。

---

### 最終輸出 (Main Process)

在 `__main__` 區塊中，程式會抓取 `history` 最後一筆資料，並印出模型的文字回覆：
> `I have created the hello.py file for you.`

### 核心總結
這個過程體現了 **Agent Loop** 的核心：模型透過「**思考 -> 提議執行指令 -> 看到執行結果 -> 再次思考**」的循環來完成任務。其中 `role: "user"` 的設計是為了讓模型知道這是來自環境的回饋。

---

## 常見問題 Q&A

### 1. 程式碼中的 `\033[33m` 是什麼意思？
這是 **ANSI 轉義序列 (ANSI Escape Codes)**，用於控制終端機的文字格式。
*   `\033[33m`：開啟**黃色**文字模式。
*   `\033[0m`：**重設**格式，恢復終端機預設顏色。
*   中間的 `$ {block.input['command']}` 則是模型準備執行的指令，加上顏色是為了在終端機中清楚區分「模型的話」與「正在執行的指令」。

### 2. 為什麼系統給模型的回饋（工具執行結果）角色是 `user`？
這是基於 **「模型 vs 外界」** 的二元邏輯。
在 Anthropic 與多數 AI 模型 API 的規範中：
*   **`assistant`**：僅代表「模型自己」說出的話或發出的請求。
*   **`user`**：代表「模型之外的一切」。這包含**真人使用者的輸入**以及**系統/環境的執行結果**。

如果不將工具結果標記為 `user` 而標記為 `assistant`，模型會認為那是它自己說過的話，導致它無法區分「指令請求」與「執行反饋」，進而造成邏輯混亂。

### 3. 總結來說，除了模型自己的響應外，其餘都被視為 `user` 嗎？
只要是輸入給模型的資訊（無論來自人類或電腦程式），在對話協議中通通被視為 `user` 角色，這維持了清晰的「請求-回應」循環。

### 4. 隨著任務累積，`messages` 變龐大是否會拖垮模型表現？
隨著 Agent 執行更多步驟，Context 會不斷累積，主要影響有三點：
1.  **成本增加**：API 費用是按 Token 計算的。每次循環都要重新發送完整的歷史紀錄，費用會隨步數呈線性增長。
2.  **延遲變長**：模型處理更長的輸入需要更久的時間，導致反應變慢。
3.  **注意力下降**：雖然現代模型（如 Claude 3.5 Sonnet）支援極大的 Context Window，但極長的文件仍可能導致模型出現「注意力稀釋」，漏掉細節或變得不穩定。

**解決方案：** 在生產級的 Agent 中，通常會加入「Context 管理」機制，例如：
*   **摘要化**：將較早的對話壓縮成一段摘要。
*   **滾動窗口**：只保留最近的 N 輪對話。
*   **快取技術 (Prompt Caching)**：重複利用之前的輸入內容以降低成本與延遲（如 Anthropic 的快取功能）。
*   可參考目錄中的 `s06_context_compact.py` 學習如何實作。
