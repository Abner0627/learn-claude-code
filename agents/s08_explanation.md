# s08_background_tasks.py 執行流程詳解

本章介紹了「非阻塞執行（Non-blocking Execution）」，讓 Agent 能夠在等待長耗時指令的同時繼續工作。當輸入 Prompt 為 `"Run a long test suite (5 mins) and then fix a bug in main.py."` 時，流程如下：

### 第一輪迴圈 (Iteration 1: 發起背景任務)

1.  **使用者發送請求**。
2.  **模型決定並行執行**：
    *   模型呼叫 `background_run`：指令為 `"pytest long_tests/"`。
3.  **主執行緒分流**：
    *   `BackgroundManager` 啟動背景執行緒處理測試。
    *   主 Agent 收到 `task_id`（例如：`a1b2c3`）並立即回覆：`"Test suite started (a1b2c3). I will now fix the bug in main.py."`

---

### 第二、三輪迴圈 (Iteration 2-3: 平行工作)

1.  **執行修復**：模型利用 `edit_file` 修補 `main.py`。
2.  **測試持續中**：背景執行緒中的測試仍在跑，且主 Agent 並未因此被阻塞。

---

### 第四輪迴圈 (Iteration 4: 任務回報)

1.  **測試完成**：背景任務執行完畢，結果被放入 `notification_queue`。
2.  **自動注入結果**：在模型下一次呼叫前，系統自動將結果以 `user` 角色訊息的形式注入 Context。
    *   範例：`"<background-results> [bg:a1b2c3] completed: Tests failed at test_auth.py </background-results>"`。
3.  **處理失敗**：模型看到測試失敗，決定停止當前修復工作，先處理測試回報的 Bug。

### 核心總結
本章展示了 **「發射後不管（Fire and forget）」** 的工作模式。背景任務與主對話循環異步進行，並透過下一次對話前的「結果灌入」機制實現非阻塞的感知，這對於編譯、部署或大型測試等場景極具價值。

---

## 常見問題 Q&A
