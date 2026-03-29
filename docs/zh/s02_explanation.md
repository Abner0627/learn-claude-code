# s02_tool_use.py 執行流程詳解

## 問題與解決方案摘要

### 問題

只有 `bash` 時，所有操作都走 Shell。`cat` 截斷行為不可預測，`sed` 遇到特殊字元就崩潰，每次 `bash` 呼叫都是不受約束的安全面。專用工具（`read_file`、`write_file`）可以在工具層面做路徑沙箱。核心洞察：新增工具**不需要修改迴圈**。

### 解決方案

透過 Tool Dispatch Map 將工具名稱**對映到處理函式**，使用 `TOOL_HANDLERS` 字典。每個工具對應一個 Handler，並透過 `safe_path()` 實現路徑沙箱以防止路徑逃逸。**新增工具 = 新增 Handler + 新增 Schema，迴圈永遠不變。**

---

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

## 關鍵設計洞察

### 加工具 = 加 Handler + 加 Schema，循環永遠不變

s02 最重要的設計原則是：**擴充工具能力不需要修改 Agent Loop**。每次新增工具只需要兩個步驟：

1. **新增 Schema**：在 `TOOLS` 列表中加入工具的 JSON Schema 定義，讓模型知道這個工具的名稱、描述與參數格式。
2. **新增 Handler**：在 `TOOL_HANDLERS` 字典中加入對應的 Python 函式。

```python
# 加一個工具就這樣：
TOOLS.append({"name": "search_file", "description": "...", "input_schema": {...}})
TOOL_HANDLERS["search_file"] = lambda **kw: run_search(kw["pattern"], kw["path"])
# 循環本身一行都不需要改。
```

### Dispatch Map 取代 if/elif 鏈

s01 中只有一個工具（bash），工具執行是硬編碼的。s02 引入了字典查找（dispatch map）：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

相比 `if block.name == "bash": ... elif block.name == "read_file": ...` 的 if/elif 鏈，dispatch map 的優勢在於：新增工具不需要修改任何現有的條件判斷邏輯。

### `safe_path()` 的沙盒機制

專用工具的核心安全優勢在於 `safe_path()` 函式：

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()   # 將相對路徑轉為絕對路徑
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

`.resolve()` 會展開所有的 `../`、符號連結（symlinks）和相對路徑，確保路徑比較不被繞過。若模型嘗試存取 `../etc/passwd` 這類路徑，`is_relative_to(WORKDIR)` 會回傳 `False` 並直接拋出例外。

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