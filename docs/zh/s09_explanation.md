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

## 信箱通訊架構詳解

### 持久隊員 vs 一次性子代理 (s04)

| 維度 | s04 子代理 (Subagent) | s09 持久隊員 (Teammate) |
| :--- | :--- | :--- |
| **生命週期** | 生成 → 完成 → 消亡（一次性） | spawn → WORKING → IDLE → WORKING → ... → SHUTDOWN |
| **固定身份** | 無 | 有名字（name）、角色（role）、狀態 |
| **跨呼叫記憶** | 無 | 有（在同一線程的 `messages` 中持續累積） |
| **通訊能力** | 無（只回傳摘要給父代理） | 可主動發訊息給任何隊員 |
| **適用任務** | 一次性資訊查詢、短暫委派 | 需要多輪協商或長程持續工作的任務 |

### JSONL 信箱的追加-排空 (Append-Drain) 模式

`.team/inbox/alice.jsonl` 是一個追加式日誌檔案，其讀寫操作在不同線程中進行：

- **`send()`**：任何隊員呼叫此方法，訊息會被以 JSON 格式追加（`append`）到目標的 `.jsonl` 檔案末尾。這是非阻塞操作，寫入後立即返回。
- **`read_inbox()`**：讀取所有行後，立即將檔案內容清空（`write_text("")`），確保每條訊息只被處理一次。

這種設計的優點在於：寫入（`send`）與讀取（`read_inbox`）發生在不同線程中，且都只需要基本的檔案操作，不需要複雜的跨線程鎖機制。

### 訊息結構

每條訊息的 JSON 結構如下：

```json
{
  "type": "message",
  "from": "alice",
  "content": "Fixed auth.py. Please review.",
  "timestamp": 1730000000.0
}
```

`type` 欄位決定接收方如何處理這條訊息。預設為 `message`（普通訊息），s10 中會引入 `shutdown_request`、`plan_request` 等協議類型。

### 隊員的 `_teammate_loop` 信箱整合

每個隊員在每次 LLM 呼叫前讀取信箱，確保最新訊息被注入上下文。若信箱不為空，訊息以 `<inbox>` 標籤包裹注入：

```python
inbox = BUS.read_inbox(name)
if inbox != "[]":
    messages.append({"role": "user",
        "content": f"<inbox>{inbox}</inbox>"})
    messages.append({"role": "assistant",
        "content": "Noted inbox messages."})
```

### 團隊名冊（`config.json`）的結構

```json
{
  "members": [
    {"name": "alice", "role": "coder", "status": "working"},
    {"name": "bob",   "role": "tester", "status": "idle"}
  ]
}
```

`status` 欄位（`working` / `idle` / `shutdown`）讓 Lead 可以隨時掌握整個團隊的工作狀態，也是 s10 協議握手與 s11 自主認領的基礎。

---

## 常見問題 Q&A
