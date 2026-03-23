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

## 常見問題 Q&A
