# s03_todo_write.py 執行流程詳解

## 問題與解決方案摘要

### 問題

多步驟任務中，模型容易遺失進度——重複執行已完成的步驟、跳過必要步驟、或偏離原定目標。對話越長問題越嚴重：工具結果不斷填滿上下文，系統提示的影響力逐漸被稀釋。一個 10 步的重構任務可能做完 1–3 步後就開始即興發揮，因為第 4–10 步已被擠出注意力範圍之外。

### 解決方案

使用 `TodoManager` 儲存帶有狀態的項目，並**限制同一時間只能有一個 `in_progress` 項目**。搭配 Nag Reminder 機制：當模型連續 3 輪以上未呼叫 `todo` 工具時，**自動注入提醒訊息**。「同時只能有一個 in_progress」強制執行順序聚焦，Nag Reminder 則製造問責壓力，確保模型不會遺忘進度追蹤。

---

本章介紹了規劃（Planning）機制，透過讓模型主動操作一個結構化的任務列表（TodoManager），並在它忘記更新時進行提醒。

### 第一輪迴圈 (Iteration 1: 分解任務)

1.  **使用者請求**：`"Refactor my code: rename 'data' to 'results' and add comments."`
2.  **模型建立計畫**：模型判斷這是一個多步驟任務，首先呼叫 `todo` 工具。
    *   `todo` 呼叫包含兩個項目：`#1: Rename variables`, `#2: Add comments`。
    *   狀態設為 `#1: in_progress`。
3.  **回報清單**：`TodoManager` 渲染出目前清單：`[>] #1: Rename... [ ] #2: Add...`。

---

### 第二、三輪迴圈 (Iteration 2-3: 執行與遺忘)

1.  **執行任務**：模型開始使用 `edit_file` 工具進行重新命名。
2.  **略過計畫**：在後續幾次工具執行中，模型完成了程式碼修改，但可能因為過於專注於細節，**忘記**呼叫 `todo` 工具來標記進度。
3.  **Nag Reminder 介入**：
    *   `rounds_since_todo` 到達臨界值（3次）。
    *   程式碼在下一次對話中強行插入 `"<reminder>Update your todos.</reminder>"`。

---

### 第四輪迴圈 (Iteration 4: 修正進度)

1.  **模型警覺**：看到提醒後，模型意識到需要同步進度。
2.  **更新計畫**：模型呼叫 `todo`，將 `#1` 設為 `completed`，將 `#2` 設為 `in_progress`。
3.  **繼續工作**：清單被重新渲染：`[x] #1: Rename... [>] #2: Add...`。

### 核心總結
本章的核心在於 **「規劃與執行間的強迫聯繫」**。透過 Nag Reminder 機制，我們解決了模型在執行長任務時容易「迷失」或「規劃與行動脫鉤」的問題。

---

## 常見問題 Q&A

### 1. 幫 Agent 加入 to-do tool 後與直接在 prompt 中規劃流程，兩者的差異在哪？

主要的差異在於**狀態的顯性化 (Explicit State)** 與 **閉環反饋 (Closed-loop Feedback)**：
*   **顯性與結構化**：在 Prompt 中規劃只是模型的一段「自言自語」，屬於非結構化文字，模型很容易在後續對話中忽略它。`todo` 工具則將規劃轉化為具體的數據結構（JSON），並由外部的 `TodoManager` 持久化，這讓模型每一輪都能看到一個整潔、帶狀態（pending/in_progress/completed）的清單。
*   **同步監控**：s03 引入了 `Nag Reminder` 機制。如果模型只顧著執行工具（如不斷 `bash` 或 `edit_file`）而超過三輪沒更新 `todo`，系統會主動介入提醒。這種「強制對齊」確保了模型的行動始終圍繞著既定目標，避免了「跑題」或「迷失在細節中」。

### 2. 承上，他們優缺點分別為何，哪個又最能節省 token？

**優缺點對比：**

| 維度 | 專用 `todo` 工具 | 直接在 Prompt 中規劃 |
| :--- | :--- | :--- |
| **精準度** | **高**。強制模型分解步驟並逐一確認。 | **中**。初始規劃可能很完美，但執行時易偏移。 |
| **可監測性** | **優**。人類開發者可透過 UI 或日誌直接掌握進度。 | **差**。需閱讀長篇對話才能理解模型目前的進度。 |
| **容錯率** | **高**。即使模型一時迷失，Reminder 也能將其拉回。 | **低**。一旦計畫被遺忘或被上下文淹沒，任務極易失敗。 |
| **靈活性** | **中**。受限於定義好的清單格式與狀態。 | **極高**。模型可隨意調整敘述方式與深度。 |

**Token 消耗分析：**
*   **短期/簡單任務**：**直接在 Prompt 規劃最節省 Token**。因為它不需要額外的 Tool Definition，也不需要為了更新狀態而額外增加 Tool Use 的往返輪次。
*   **長期/複雜任務**：**使用 `todo` 工具長遠來看更省 Token**。雖然每一輪更新 `todo` 會增加幾百個 Token 的消耗，但它能顯著降低模型「迷失方向」或「陷入死循環」的機率。一旦模型迷失（例如重複執行錯誤的指令），所造成的 Token 浪費將遠超 `todo` 工具的維護成本。

### 3. 綜合 s02 與 s03，具體來說要怎麼將這些工具加入給 Agent？請分別以 Claude Code 與 Gemini CLI 為例。

以下針對 `read_file`、`write_file` 與 `todo_list` 三個概念，展示在兩大主流 SDK 中的具體實作方式。

#### A. Claude Code 實作模式 (Anthropic SDK)
Claude 透過 `tools` 參數接收工具定義，並在對話中返回 `tool_use` 區塊。

1.  **宣告工具清單 (JSON Schema)**：
    ```python
    TOOLS = [
        {
            "name": "read_file",
            "description": "讀取指定路徑的檔案內容",
            "input_schema": {
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"]
            }
        },
        {
            "name": "write_file",
            "description": "將內容寫入指定檔案",
            "input_schema": {
                "type": "object",
                "properties": {"path": {"type": "string"}, "content": {"type": "string"}},
                "required": ["path", "content"]
            }
        },
        {
            "name": "todo_list",
            "description": "更新並追蹤任務清單狀態",
            "input_schema": {
                "type": "object",
                "properties": {
                    "items": {
                        "type": "array", 
                        "items": {
                            "type": "object",
                            "properties": {
                                "id": {"type": "string"},
                                "text": {"type": "string"},
                                "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}
                            }
                        }
                    }
                },
                "required": ["items"]
            }
        }
    ]
    ```

2.  **執行循環與派發 (Loop & Dispatch)**：
    ```python
    response = client.messages.create(model=MODEL, tools=TOOLS, messages=history)
    
    if response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                # 呼叫對應的本地 Handler (如 s02/s03 實作的函數)
                output = call_local_handler(block.name, block.input)
                # 封裝成 tool_result 傳回給 Claude
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)
                })
        history.append({"role": "user", "content": tool_results})
    ```

---

#### B. Gemini CLI 實作模式 (Google AI SDK)
Gemini 支持直接將 Python 函數作為工具，並自動生成 Schema。

1.  **定義 Python 函數工具**：
    ```python
    def read_file(path: str) -> str:
        """讀取指定路徑的檔案內容。"""
        return Path(path).read_text()

    def write_file(path: str, content: str) -> str:
        """將內容寫入指定檔案。"""
        Path(path).write_text(content)
        return "Success"

    def todo_list(items: list) -> str:
        """更新並追蹤任務清單狀態，items 包含 id, text, status。"""
        return my_todo_manager.update(items)

    # 註冊至模型
    model = genai.GenerativeModel(
        model_name='gemini-1.5-pro',
        tools=[read_file, write_file, todo_list]
    )
    ```

2.  **處理 Function Calling**：
    ```python
    chat = model.start_chat(enable_automatic_function_calling=True)
    response = chat.send_message("請讀取 README.md 並列出待辦事項")
    # 若 enable_automatic_function_calling=True，SDK 會自動執行函數並回傳結果
    # 否則需手動攔截 response.candidates[0].content.parts 中的 function_call
    ```

#### 總結：關鍵差異
*   **Claude** 需要開發者**手動維護** JSON Schema 與 Tool Result 的封裝，適合需要極致控制（如自定義回傳格式）的場景。
*   **Gemini** 提供了更**自動化**的體驗，直接將代碼函數映射為模型能力，開發效率較高，但對函數的 Docstring 撰寫品質有較強依賴。

### 4. 工具 (Tools) 與 技能 (Skills) 有什麼差別？我是否能直接用 Skill 取代 Tool？

簡單來說，**工具是「手腳」，技能是「大腦中的 SOP」**。

**兩者的本質區別：**

| 維度 | 工具 (Tools) | 技能 (Skills) |
| :--- | :--- | :--- |
| **定義** | **能力的硬實作**。如 `read_file`, `bash`。 | **知識的軟注入**。如 `SKILL.md` 中的 SOP。 |
| **形式** | JSON Schema + Python Handler。 | Markdown 指引或任務範例。 |
| **作用** | 讓 Agent 獲得**與外部環境交互**的能力。 | 讓 Agent 獲得**處理特定領域任務**的知識。 |
| **加載時機** | 啟動 Agent 時預先宣告。 | **按需加載 (On-demand)**，需要時模型主動調用。 |

**為什麼 Skill 不能取代 Tool？**
*   **Skill 沒有執行力**：如果你刪掉 `read_file` 工具，只給一個「讀取檔案技能」，模型雖然知道「應該去讀檔案」，但它卻沒有任何 API 可以呼叫來獲取檔案內容。
*   **依賴關係**：Skill 必須**建立在 Tool 之上**。一個高品質的 Skill 通常會教導模型如何「組合呼叫」現有的 Tools。例如，一個 `docker-expert` 技能會指引模型：「第一步：呼叫 `bash` 檢查 Docker 狀態；第二步：呼叫 `write_file` 建立 Dockerfile」。

**什麼時候該用 Skill？**
當你發現某個任務非常複雜，需要多個步驟且模型容易出錯時（例如：從 PDF 提取數據、進行複雜的架構重構），你不應該去寫一個萬能的「大工具」，而應該寫一個 **Skill (SOP)** 來教導模型如何靈活運用現有的 `read`、`write`、`bash` 工具。

**結論**：Tool 提供「底層能力」，Skill 提供「高層邏輯」。兩者相輔相成，而非替代關係。

### 5. 在實際的生產環境 (Claude code 與 Gemini CLI) 是否有類似 read_file、write_file 及 todo_list 的內建功能？

是的，這些功能在生產級的 Agent 中不僅存在，而且更加強大且安全：

*   **讀寫工具 (Read/Write Tools)**：
    *   **Claude Code**: 提供了 `readFile`、`writeFile`、`editFile`、`grep` 與 `ls` 等工具。其中 `editFile` 通常使用精確的文字區塊替換，而非全檔重寫，以節省 Token。
    *   **Gemini CLI**: 內建了 `read_file`、`write_file`、`replace` (用於精確編輯)、`grep_search` 與 `glob` 等工具。
*   **任務規劃工具 (Todo/Planning Tools)**：
    *   **Claude Code**: 具備「規劃 (Planning)」模式，模型會使用 `plan` 工具或內部機制來記錄長程目標與當前進度。
    *   **Gemini CLI**: 強制執行 **Research -> Strategy -> Execution** 流程。在執行階段，模型會先提出計畫 (Plan)，並在每個子任務完成後進行驗證 (Validate)。

**總結**：s03 展示的 `todo_list` 是生產環境中「規劃機制」的簡化教學版。在真實場景中，這些規劃通常會與 UI 整合（例如 CLI 中的進度條或勾選清單），讓人類使用者能即時掌握 Agent 的大腦運作狀態。