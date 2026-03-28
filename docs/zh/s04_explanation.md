# s04_subagent.py 執行流程詳解

本章介紹了「上下文隔離（Context Isolation）」，透過產生子 Agent 來處理特定子任務，避免主 Agent 的對話歷史過於龐大。

### 第一輪迴圈 (Iteration 1: 委派任務)

1.  **使用者請求**：`"Analyze the structure of this project and summarize each file."`
2.  **模型決定委派**：主 Agent 決定由專員處理各個檔案的分析。
3.  **發出 `task` 請求**：
    ```json
    {
        "type": "tool_use",
        "name": "task",
        "input": {"prompt": "List all files in src/ and summarize their purpose."}
    }
    ```

---

### 子 Agent 執行 (Sub-agent execution)

1.  **建立全新 Context**：`run_subagent` 被呼叫，啟動一個**空的 `sub_messages` 列表**。
2.  **子 Agent 獨立執行**：
    *   子 Agent 接收到 `"List all files..."`。
    *   子 Agent 呼叫 `bash` 指令並讀取檔案內容。
    *   子 Agent 在自己的小循環中完成了多輪對話。
3.  **生成總結**：子 Agent 完成任務，回傳一段文字：`"src/ contains 3 files: main.py, utils.py..."`。

---

### 主 Agent 接收回報 (Iteration 2)

1.  **獲取總結**：主 Agent 收到 `task` 的執行結果（即子 Agent 的總結文字）。
2.  **繼續任務**：主 Agent 根據子 Agent 的總結，決定下一步是要進行修改還是直接回覆使用者。

### 核心總結
子 Agent 的關鍵在於 **「對話歷史的斷裂（Context Discarding）」**。子 Agent 執行過程中的中間步驟（如 `bash` 的細節輸出）通通會被丟棄，只有主 Agent 真正需要的「結論」會被保留下來，極大地節省了 Context 資源。

---

## 關鍵設計洞察

### 子代理的工具集限制（禁止遞迴）

子代理擁有除 `task` 以外的所有基礎工具。這是一個重要的安全設計：若子代理也能呼叫 `task`，就可能產生遞迴生成（子代理又生成子代理），導致無法控制的 API 費用和深度。

```python
PARENT_TOOLS = CHILD_TOOLS + [task_tool]   # 只有父代理有 task 工具
CHILD_TOOLS  = [bash, read_file, write_file, edit_file, todo]
```

### 30 步安全上限

子代理的循環有硬性上限：

```python
for _ in range(30):  # 最多執行 30 輪工具呼叫
    ...
```

這防止了子代理在異常情況下無限循環消耗 API。若任務在 30 步內未完成，循環會強制退出，並回傳當時最後一輪的文字輸出（可能是部分結果）。

### 只有最終文字被回傳

子代理執行過程中可能進行了 30 多次工具呼叫，但主代理收到的只是最後一條 `text` 區塊：

```python
return "".join(
    b.text for b in response.content if hasattr(b, "text")
) or "(no summary)"
```

整個 `sub_messages` 列表在 `run_subagent()` 函式結束後會被 Python 的垃圾回收機制自動釋放。主代理的 `messages` 中不留下任何子代理的執行細節。

### 子代理與任務委派（s04）vs 持久隊員（s09）的邊界

子代理是**無狀態的**：每次呼叫 `task` 工具都會建立一個全新的 `sub_messages = []`，沒有跨呼叫的記憶。這讓它非常適合「一次性查詢」，但不適合需要多輪協商的複雜任務（那是 s09 持久隊員的場景）。

---

## 常見問題 Q&A

### 1. 在 Claude Code 與 Gemini CLI 中，實際上是如何使用 sub agent 的？

在實際的生產環境中，Sub-agent 的運作模式如下：
*   **Claude Code (Anthropic)**：當主 Agent 判定任務需要進行大量「低價值、高噪音」的操作（例如掃描整個專案目錄、檢查數十個檔案的內容）時，它會呼叫一個 `task` 工具。這個工具在後端會實例化一個**全新的對話 Loop**。這個子代理擁有相同的工具集（read/write/bash），但其 `messages` 歷史是空的。子代理完成任務後，會輸出一段「最終報告」，這段報告會作為 `tool_result` 回傳給主 Agent。
*   **Gemini CLI (Google AI)**：邏輯類似，但常利用 **「模型分級」**。主 Agent (Gemini 1.5 Pro) 可能會委派給子代理 (Gemini 1.5 Flash) 去做瑣碎的資訊提取工作。這樣既節省了主 Agent 的 Context 空間，也降低了 API 延遲與成本。

### 2. 若由使用者新建一個 session 完成 sub task 後，再將內容讓該 session 的 agent 擷取生成 summary.md 後，最後給原 session 的 agent 讀取。這樣跟 sub agent 有什麼差異？優缺點如何？

這種「手動搬運」模式本質上是 **「人工模擬的 Context Caching」**。

**與自動化 Sub-agent 的差異：**
*   **控制流**：Sub-agent 是由模型「自主」決定的；手動模式則是由人類「決定」何時該切換 Context。
*   **整合度**：Sub-agent 的結果直接回到對話流中，能立即觸發下一步行動；手動模式需要人類進行內容的 Copy-Paste，且 `summary.md` 的質量取決於你如何引導第二個 Session 的 Agent。

**優缺點對比：**

| 特性 | 自動化 Sub-agent (s04 模式) | 手動 Session + `summary.md` |
| :--- | :--- | :--- |
| **上下文管理** | **自動隔離**。子代理完成後，其內部的數十次工具往返會被銷毀，不污染主代理的記憶。 | **手動壓縮**。透過 `summary.md` 將大量細節壓縮成精華，隔離效果最徹底。 |
| **效率** | **高**。一氣呵成，主代理能動態調整子代理的 Prompt。 | **低**。需要人工切換視窗、搬運內容，且難以處理「遞迴式」的複雜任務。 |
| **Token 成本** | **中**。子代理的所有步驟仍需支付 Token 費用。 | **低**。使用者可以只將最重要的部分餵回主 Session，避免無謂消耗。 |
| **專注度** | **優**。子代理沒有主代理的包袱，能更專注於當下的細節。 | **極優**。全新的對話環境（System Prompt）能提供最強的約束力。 |

### 3. 承第 1 點，也就是說在實際生產環境中，它們都有類似的內建功能嗎？

是的。在現代的 Agent 框架中（如 Anthropic 的 Claude Code 或 Google 的 Gemini CLI），子代理（Sub-agent）委派不僅是內建功能，更是其處理複雜工程任務的**核心基石**。

**具體的內建方式如下：**

*   **作為「特化工具」存在**：在生產環境中，子代理通常被封裝成一組專門的工具。例如：
    *   **`codebase_investigator`**：這是一個專門負責「讀取與理解程式碼庫」的子代理。當主代理需要了解某個複雜的類別繼承關係時，它會呼叫此工具。
    *   **`generalist`**：這是一個處理「批次性、重複性」任務的子代理。當主代理需要為 50 個檔案添加 License Header 時，它會委派給 `generalist`。
*   **基礎設施的支持**：這些內建工具背後有一套完整的「代理循環 (Agent Loop)」基礎設施。當主代理呼叫這些工具時，系統會自動：
    1.  **實例化 (Instantiate)**：建立一個新的、擁有特定 System Prompt 的 LLM 實例。
    2.  **資源隔離 (Sandbox)**：給予子代理受限或特定的工具存取權。
    3.  **自動總結 (Auto-summarization)**：在子代理完成任務後，強制其輸出 Summary，並將該總結作為工具結果回傳。
*   **動態決定委派**：在生產環境中，模型會根據任務的「模糊度」或「規模」動態決定是否需要啟動子代理。例如，簡單的修改會直接執行，而「分析整個專案的安全性」則會自動觸發子代理委派。

**總結**：你看到的 `s04` 代碼是這些強大 CLI 工具的**簡化原型**。在實際產品中，這套機制被用來解決單一對話長度限制、模型注意力分散以及處理海量程式碼檔案等核心痛點。
