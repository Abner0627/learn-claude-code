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

## 後台任務機制詳解

### 守護線程 (Daemon Thread) 的特性

`BackgroundManager` 使用 `daemon=True` 建立後台線程。這意味著：
- 當主程式退出時，所有守護線程會被**自動強制終止**，不會阻礙程式關閉。
- 若使用普通線程（非守護），主程式結束後仍需等待所有線程完成才能退出，可能造成程式「卡住」。

### 執行緒安全的通知佇列

後台線程完成任務後，透過 `threading.Lock()` 安全地將結果推入通知佇列：

```python
with self._lock:
    self._notification_queue.append({
        "task_id": task_id, "result": output[:500]
    })
```

主循環在每次 LLM 呼叫前呼叫 `drain_notifications()` 清空佇列，確保結果只被處理一次，且不存在競態條件（race condition）。注意結果會被截斷至 500 字元，避免一次性注入過多 token。

### 結果注入的時機與訊息格式

後台任務的結果**不會立即打斷**正在進行的 LLM 呼叫，而是在**下一輪循環開始前**批次注入。注入後，會追加一條 `assistant` 確認訊息，讓對話流保持正確的 user/assistant 交替格式：

```
[user]      <background-results>
              [bg:a1b2] Tests failed at test_auth.py
            </background-results>
[assistant] Noted background results.
[user]      (使用者的下一個問題或模型的下一步工具呼叫)
```

### 300 秒超時保護

每個後台子進程都有 300 秒（5 分鐘）的超時限制。超時後，子進程會收到 `SIGKILL` 信號，並向通知佇列報告 `"Error: Timeout (300s)"`，讓主 Agent 得知該任務已失敗。

### 適用場景分析

| 適合後台執行 | 不適合後台執行 |
| :--- | :--- |
| `npm install`、`pip install` | 需要立即結果的查詢 |
| `pytest`、`cargo build` | 依賴上一步結果的序列操作 |
| `docker build`、`git clone` | 需要 stdin 互動的指令 |
| 部署腳本、大型壓縮任務 | 需要精確控制執行順序的工作 |

### 循環本身仍是單線程

s08 只是讓**子進程 I/O**並行化，主 Agent Loop 本身仍然是單線程的 `while True`。這意味著：
- 不存在多個 LLM 呼叫同時進行的情況。
- 通知佇列的排空發生在明確的、可預測的時間點。
- 比起完全非同步的架構，這種設計更容易除錯與理解。

---

## 常見問題 Q&A
