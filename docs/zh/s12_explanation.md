# s12_worktree_task_isolation.py 執行流程詳解

本章介紹了「物理隔離（Physical Isolation）」。利用 Git Worktree 讓 Agent 可以在並行的、互不干擾的目錄中執行多項任務。當輸入 Prompt 為 `"Work on auth refactor and UI redesign in parallel."` 時，流程如下：

### 第一階段 (建立平行通道)

1.  **使用者請求**：模型需要平行處理兩個大任務。
2.  **分配工作通道 (Lane Allocation)**：
    *   模型呼叫 `worktree_create("auth-lane", task_id=12)`。
    *   Git 在 `.worktrees/auth-lane/` 建立一個全新的簽出目錄。
3.  **重複動作**：為 UI 重構建立另一個通道 `worktree_create("ui-lane", task_id=13)`。

---

### 第二階段 (平行執行)

1.  **隔離執行**：模型利用 `worktree_run` 指令。
    *   `worktree_run("auth-lane", "npm run refactor")`：該指令只在 `auth-lane` 目錄下執行，完全不影響主目錄。
2.  **狀態觀察**：模型呼叫 `worktree_status` 觀察各個通道的檔案變動情況。
3.  **事件記錄**：所有的建立與執行動作都會被自動記錄在 `events.jsonl` 中，供外部審計或 UI 顯示。

---

### 第三階段 (結案與清理)

1.  **任務完成**：在兩個通道中的工作皆完成。
2.  **選擇結案方式**：
    *   模型呼叫 `worktree_remove("auth-lane", complete_task=True)`。
    *   Git 移除該 Worktree，並自動更新 `.tasks/task_12.json` 為 `completed`。
3.  **保留研究**：模型可以呼叫 `worktree_keep("ui-lane")`，在 index 中標記為 `kept`，這表示該通道不會被移除，供之後查閱。

### 核心總結
本章展示了 **「目錄即通道（Directory as a Lane）」** 的實作。透過將任務綁定到獨立的 Worktree，我們解決了多任務環境下的檔案衝突與狀態混亂問題，實現了生產等級的並行任務執行能力。

---

## Worktree 隔離機制詳解

### 控制平面 vs 執行平面

s12 將任務管理分為兩個平面：

| 平面 | 目錄 | 職責 |
| :--- | :--- | :--- |
| **控制平面** | `.tasks/` | 記錄「做什麼」：任務的 ID、狀態、依賴、擁有者 |
| **執行平面** | `.worktrees/` | 記錄「在哪做」：每個任務對應的獨立 Git Worktree 目錄 |

兩個平面透過 `worktree` 欄位雙向綁定：
- `task_1.json` 中有 `"worktree": "auth-refactor"`。
- `.worktrees/index.json` 中有 `"task_id": 1`。

`bind_worktree()` 方法在建立 Worktree 時同時更新兩側的狀態，確保兩個平面始終保持一致。

### Git Worktree 的核心原理

`git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD` 在**同一個 Git 儲存庫**中建立一個額外的工作目錄，並將其指向一個獨立分支。這讓多個任務可以：
- 修改相同的檔案（各自在自己的分支上，互不污染）。
- 隨時可以對各自的改動進行 `git diff` 或 `git commit`。

相比「複製整個專案目錄」，Worktree 幾乎不佔用額外磁碟空間，因為它共享同一個 `.git` 目錄的物件儲存。

### 事件流 (`events.jsonl`) 的完整類型

每個生命週期步驟都會記錄一條事件，便於外部系統（如 CI、監控面板）訂閱和審計：

| 事件類型 | 觸發時機 |
| :--- | :--- |
| `worktree.create.before` | `git worktree add` 執行前 |
| `worktree.create.after` | Worktree 成功建立並完成綁定後 |
| `worktree.create.failed` | 建立失敗（如分支名衝突） |
| `worktree.remove.before` | `git worktree remove` 執行前 |
| `worktree.remove.after` | 成功移除，任務同步標記完成 |
| `worktree.remove.failed` | 移除失敗（如目錄有未提交的改動） |
| `worktree.keep` | Worktree 被標記為保留（不移除） |
| `task.completed` | 任務狀態更新為 `completed` |

### `keep` vs `remove` 的選擇

| 操作 | 行為 | 適用場景 |
| :--- | :--- | :--- |
| `worktree_keep(name)` | 在 `index.json` 標記為 `kept`，保留目錄 | 需要事後人工查閱或手動合併 |
| `worktree_remove(name, complete_task=True)` | 刪除目錄 + 完成任務 + 觸發事件 | 自動化流程中的正常完成 |

### 崩潰恢復機制

若 Agent 程序意外崩潰，可以從磁碟重建現場：

1. 掃描 `.tasks/` 目錄 → 恢復所有任務的狀態。
2. 讀取 `.worktrees/index.json` → 恢復 Worktree 清單。
3. 比對兩者的 `task_id` 綁定關係 → 識別哪些任務在哪個目錄執行中。
4. 讀取 `.worktrees/events.jsonl` → 確認最後一個成功的事件，判斷哪些操作需要重做。

**核心原則**：對話記憶（messages）是易失的；磁碟狀態是持久的。崩潰後重建依靠的是磁碟，而非模型的記憶。

---

## 常見問題 Q&A
