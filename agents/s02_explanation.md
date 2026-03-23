# s02_tool_use.py 執行流程詳解

本章展示了如何擴充 Agent 的能力，使其能夠透過多種工具（讀取、寫入、編輯、執行指令）與環境互動。當輸入 Prompt 為 `"Read README.md and then create a summary.txt"` 時，流程如下：

### 第一輪迴圈 (Iteration 1)

1.  **發送 Prompt**：`client.messages.create` 帶入使用者的讀取與寫入請求。
2.  **模型回應 (LLM Response)**：
    *   模型判斷需要先「讀取」，回傳 `stop_reason == "tool_use"`。
    *   `response.content` 包含一個 `read_file` 的工具區塊：
        ```json
        {
            "type": "tool_use",
            "name": "read_file",
            "input": {"path": "README.md"}
        }
        ```
3.  **執行工具**：
    *   `TOOL_HANDLERS` 查找 `read_file` 對應的 `run_read` 函式。
    *   讀取檔案內容並回傳給模型。

---

### 第二輪迴圈 (Iteration 2)

1.  **再次思考**：模型接收到 `README.md` 的內容，現在需要執行「寫入」。
2.  **模型回應**：
    *   再次回傳 `tool_use`，這次是 `write_file`：
        ```json
        {
            "type": "tool_use",
            "name": "write_file",
            "input": {"path": "summary.txt", "content": "...(摘要內容)..."}
        }
        ```
3.  **執行工具**：`run_write` 將摘要寫入磁碟。

---

### 第三輪迴圈 (Iteration 3)

1.  **最終確認**：模型看到寫入成功的確認訊息。
2.  **結束任務**：回傳 `stop_reason == "end_turn"`，並給出文字回覆：`"I have summarized README.md into summary.txt."`

### 核心總結
本章引進了 **Tool Dispatch Map** 的概念。Agent 迴圈本身不變，但透過擴充 `TOOLS` 列表與 `TOOL_HANDLERS` 映射表，模型能操作的維度從單一的 Bash 指令擴展到了精細的檔案操作。

---

## 常見問題 Q&A

### 1. 此處每個工具有一個處理函數，這麼做為什麼能使路徑沙箱防止逃逸工作區？

這主要是因為透過**參數化接口 (Parameterized Interface)** 與 **路徑正規化檢查** 的結合：
*   **明確的參數空間**：在 `read_file` 或 `write_file` 工具中，路徑被定義為一個獨立的 `path` 參數。相較於從一段雜亂的 Bash 指令（如 `cat ../../secret.txt`）中解析路徑，直接攔截參數要簡單且可靠得多。
*   **強制檢查機制**：在 `safe_path` 函式中，使用了 `Path(p).resolve()` 將相對路徑轉為絕對路徑，並透過 `is_relative_to(WORKDIR)` 進行檢測。這能有效阻止 `../` 等跳脫符號、符號連結 (Symbolic Links) 或絕對路徑的攻擊。
*   **Bash 的侷限性**：若只給模型一個 `bash` 工具，要防止逃逸就必須撰寫極其複雜的指令過濾器，來對抗各種 Shell 變數、管線與重定向技巧，這在安全性上是非常脆弱的。

### 2. 該方式與直接調用 bash 哪個比較節省 token？兩者有什麼優缺點？

**Token 效率分析：**
*   **基礎成本**：直接調用 Bash 只需要定義一個工具，Tool Definition 的 Token 較少；專用工具越多，每輪對話的基礎 Token 負擔越高。
*   **執行成本**：在「執行操作」時，專用工具通常更省 Token。例如，模型進行「檔案精確替換」時，調用 `edit_file` 只需要給出 `old_text` 與 `new_text`；若用 Bash，則需撰寫複雜的 `sed`、`awk` 或臨時 Python 腳本，且出錯率高，容易導致對話輪次增加（多消耗 Token）。

**優缺點對比：**

| 維度 | 專用工具處理函數 (如 read/write) | 直接調用 Bash |
| :--- | :--- | :--- |
| **安全性** | **極高**。可精準實作路徑沙箱與權限控制。 | **極低**。極難防範指令注入、刪庫或路徑逃逸。 |
| **可靠性** | **高**。不依賴作業系統環境（如 BSD vs GNU 指令差異）。 | **中**。受限於 Shell 環境、安裝的套件與權限。 |
| **靈活性** | **低**。只能執行預定義好的邏輯。 | **極高**。理論上能完成任何系統支持的操作。 |
| **開發難度** | 較高。需手動實作與測試每個 Handler 的安全性。 | 低。只需將指令丟給子進程。 |
| **觀察性** | **優**。日誌能清楚顯示「讀取了哪個檔案」，而非「執行了某段腳本」。 | **差**。難以從複雜的指令字串中分析模型意圖。 |

**總結建議**：為了安全性與穩定性，建議將「**高頻且具風險**」的操作（如檔案讀寫、搜索）封裝成專用工具，而將「低風險或高度靈活」的需求交給受限的 Bash。
