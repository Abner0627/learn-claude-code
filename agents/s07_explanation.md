# s07_task_system.py 執行流程詳解

本章展示了持久化任務管理。透過將任務板放在對話（Context）外部，確保狀態能跨越壓縮與重啟。當輸入 Prompt 為 `"Plan the auth migration."` 時，流程如下：

### 第一輪迴圈 (Iteration 1: 規劃任務)

1.  **使用者發送請求**。
2.  **模型建立任務**：呼叫 `task_create` 建立多個項目。
    *   範例：`#1: Refactor DB`, `#2: Update API`。
3.  **磁碟儲存**：任務被儲存為 `.tasks/task_1.json`, `.tasks/task_2.json`。

---

### 第二輪迴圈 (Iteration 2: 建立依賴)

1.  **設定順序**：模型判斷需要先做 `#1` 再做 `#2`。
2.  **建立 `blockedBy` 關聯**：
    *   模型呼叫 `task_update`：將 `#2` 的 `blockedBy` 設為 `[1]`。
3.  **回報依賴圖**：`task_list` 會顯示：`[ ] #2: Update API (blocked by: [1])`。

---

### 第三輪迴圈 (Iteration 3: 解開依賴)

1.  **執行完成**：模型完成 `#1` 的代碼工作。
2.  **標記完成**：呼叫 `task_update` 將 `#1` 設為 `completed`。
3.  **依賴清除 (Dependency Clearing)**：
    *   `TaskManager` 在底層自動偵測到 `#1` 已完成。
    *   將 `#1` 從 `#2` 的 `blockedBy` 列表移除。
4.  **解鎖下一步**：現在 `task_list` 顯示 `#2` 已變為可執行的狀態。

### 核心總結
本章展示了 **「控制平面（Control Plane）的分離」**。透過將工作流邏輯轉移到外部 JSON 文件，我們建立了比單純 `todo` 更強大的、支援並行與複雜依賴的任務追蹤系統。

---

## 任務圖 (DAG) 詳解

### 三個核心查詢

`TaskManager` 設計的目的是隨時回答三個問題：

1. **什麼現在可以做？** → 找出 `status == "pending"` 且 `blockedBy == []` 的任務
2. **什麼被卡住了？** → 找出 `blockedBy` 不為空的任務，並顯示其前置依賴
3. **什麼做完了？** → 找出 `status == "completed"` 的任務

### `blockedBy` 與 `blocks` 的對稱性

任務間的依賴關係是**雙向記錄**的：

- `blockedBy`：「我的前置任務是誰？」（正向依賴）
- `blocks`：「我完成後誰可以開始？」（反向依賴）

這種冗餘設計讓任務圖可以被快速查詢，無需每次都遍歷所有任務才能找到下游任務。

### 自動依賴解鎖機制

當一個任務被標記為 `completed` 時，`_clear_dependency` 方法會自動掃描所有任務，將該任務的 ID 從其他任務的 `blockedBy` 列表中移除：

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

這意味著：模型**不需要**手動解鎖下游任務。完成任務的瞬間，任務看板的「可執行任務」集合會自動更新。

### s03 TodoManager vs s07 TaskManager

| 維度 | s03 TodoManager（記憶體） | s07 TaskManager（磁碟） |
| :--- | :--- | :--- |
| **存活範圍** | 僅限單次對話 | 跨對話、跨壓縮、跨重啟 |
| **關係表達** | 無（扁平清單） | `blockedBy` + `blocks` 邊 |
| **並行性** | 無法識別 | 顯式表示可並行任務 |
| **狀態精度** | 做完 / 沒做完 | `pending → in_progress → completed` |
| **多代理支援** | 無 | `owner` 欄位標記誰認領了 |
| **壓縮後存活** | 否（記憶體中） | 是（JSON 在磁碟上） |

### 任務圖是後續章節的骨架

從 s07 開始，任務圖成為所有協調機制的核心基礎：
- **s08 後台任務**：後台執行的 Shell 指令可以在完成後更新任務狀態。
- **s09-s10 Agent 團隊**：隊員認領任務時需要讀寫任務圖。
- **s11 自主代理**：空閒隊員掃描任務圖找工作。
- **s12 Worktree 隔離**：每個任務可以綁定一個獨立的 Git Worktree 目錄。

---

## 常見問題 Q&A
