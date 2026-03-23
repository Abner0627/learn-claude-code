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

## 常見問題 Q&A
