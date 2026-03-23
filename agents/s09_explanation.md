# s09_agent_teams.py 執行流程詳解

本章展示了 Agent 團隊協作。透過文件系統作為信箱 (Mailbox)，讓多個模型常駐於系統中異步工作。當 Lead Agent 收到 `"Ask Alice to fix my code and Bob to review it."` 時，流程如下：

### 第一輪迴圈 (Iteration 1: 分派工作)

1.  **Lead 發號施令**：
    *   呼叫 `spawn_teammate` 啟動 `Alice` 與 `Bob`。
    *   這會為 `Alice` 與 `Bob` 各自開啟一個獨立的執行緒循環。
2.  **發送訊息**：
    *   Lead 呼叫 `send_message("Alice", "Fix logic in auth.py")`。
    *   這將訊息寫入 `.team/inbox/Alice.jsonl`。

---

### Alice 執行緒工作 (Alice's Loop)

1.  **排空信箱 (Draining Inbox)**：
    *   `Alice` 定期讀取 `Alice.jsonl`。
    *   訊息被載入 `Alice` 的對話歷史中。
2.  **開始工作**：
    *   `Alice` 呼叫 `edit_file` 修改 `auth.py`。
3.  **主動回報**：
    *   完成後，`Alice` 呼叫 `send_message("Lead", "Fixed. Asking Bob for review.")`。
    *   同時發訊息給 `Bob`：`send_message("Bob", "Review auth.py")`。

---

### Bob 執行緒工作 (Bob's Loop)

1.  **接收與執行**：
    *   `Bob` 從信箱收到 `Alice` 的通知。
    *   呼叫 `read_file` 進行程式碼審查。
2.  **給出回報**：`Bob` 向 `Lead` 回報審查結果。

### 核心總結
本章展示了 **「信箱通訊模式（Mailbox Pattern）」** 的力量。不同於子 Agent，團隊成員是常駐且擁有自主性的。透過文件系統解耦（Decoupling），我們實現了一個能跨執行緒、且易於觀測與除錯的異步 Agent 協作架構。

---

## 常見問題 Q&A
