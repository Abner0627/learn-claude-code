# s10_team_protocols.py 執行流程詳解

## 問題與解決方案摘要

### 問題

s09 中隊員能工作、能通訊，但**缺少結構化協調**：關機、計畫審批等操作都需要握手機制——領導發起請求，隊員批准（完成後退出）或拒絕（繼續工作）。兩種協議結構相同：一方發送帶有唯一 ID 的請求，另一方引用同一 ID 回應。

### 解決方案

透過 `request_id` 關聯模式實現結構化協議。關機與計畫審批兩種協議共用同一個 FSM 狀態機：`[pending]` → `[approved]` 或 `[rejected]`。領導生成 `request_id` 並透過收件匣發起請求，隊員以 `approve` 或 `reject` 回應並引用同一 `request_id`。一個 FSM，兩種用途，可擴展至任何需要握手確認的場景。

---

本章介紹了結構化協作協議。透過 `request_id` 關聯模式，讓模型之間可以進行可靠的「握手」確認。當 Lead 決定關閉隊員 `Alice` 時，流程如下：

### 第一階段 (發起請求)

1.  **Lead 決定終止**：
    *   Lead 呼叫 `shutdown_request("Alice")`。
    *   系統自動生成 `request_id`（例如：`req_456`）。
2.  **存入追蹤器 (Tracker)**：
    *   `shutdown_requests["req_456"]` 狀態設為 `pending`。
3.  **發送協議訊息**：
    *   將帶有 `request_id` 的關機請求訊息寫入 `Alice.jsonl`。

---

### 第二階段 (接收與決策)

1.  **Alice 接收協議**：
    *   `Alice` 讀取信箱，看到類型為 `shutdown_request` 的協議訊息。
2.  **回覆協議**：
    *   `Alice` 呼叫 `shutdown_response` 工具。
    *   帶入相同的 `request_id`（`req_456`）並設 `approve: true`。
3.  **回傳確認**：確認訊息被發送回 Lead。

---

### 第三階段 (執行與結案)

1.  **Lead 匹配回覆**：
    *   Lead 端的 `MessageBus` 收到回覆。
    *   透過 `request_id` 匹配，將 `shutdown_requests["req_456"]` 標記為 `approved`。
2.  **安全退場**：`Alice` 的執行緒在完成最後一次工具回傳後，偵測到已同意關機，正式停止循環。

### 核心總結
本章展示了 **「狀態關聯協議」** 的實作。透過 `request_id` 模式，我們可以建立起可靠的對話握手機制，解決了非同步通訊中「訊息順序錯亂」或「回覆對象不明」的問題。

---

## 協議機制詳解

### 兩種協議的結構對照

s10 實作了兩種請求-回應協議，結構完全相同，只是用途不同：

| 面向 | 關機協議 (Shutdown Protocol) | 計劃審批協議 (Plan Approval Protocol) |
| :--- | :--- | :--- |
| **發起方** | Lead（主動要求隊員關機） | Teammate（提交計劃待審批） |
| **接收方** | Teammate | Lead |
| **批准效果** | Teammate 完成收尾後退出循環 | Teammate 開始執行計劃 |
| **拒絕效果** | Teammate 繼續工作 | Teammate 修改後重新提交 |
| **追蹤器** | `shutdown_requests` | `plan_requests` |

### FSM 狀態機

每個請求都有一個有限狀態機（FSM），只有兩種終態：

```
[pending] --approve--> [approved]
[pending] --reject---> [rejected]
```

發起方可以隨時透過 `request_id` 查詢請求目前處於哪個狀態。

### 為什麼需要 `request_id`？

在非同步多代理通訊中，訊息可能出現亂序或重疊。若沒有 `request_id`：
- Lead 發出多個關機請求後收到回覆，無法判斷哪個回覆對應哪個請求。
- 隊員可能收到多個計劃審批請求，回覆時也無法明確引用。

`request_id`（也稱為 Correlation ID）是分散式系統中的標準手法，確保請求與回覆之間的一對一配對關係。

### 完整的關機握手流程

```
Lead                              Alice
  |                                 |
  |-- shutdown_request(req_id) ---> |   (訊息寫入 Alice.jsonl)
  |                                 |
  |                                 |   (Alice 讀取信箱，看到 shutdown_request)
  |                                 |   (Alice 判斷：目前無關鍵工作，同意)
  |                                 |
  |<-- shutdown_response(req_id) -- |   (approve: true)
  |                                 |
  |  [Lead 查找 req_id，          |
  |   更新狀態為 approved]         |
  |                                 |
  |                            (Alice 偵測到 approved，退出循環)
```

### 計劃門控的安全意義

在高風險場景（如修改生產資料庫結構、大規模刪除檔案），讓隊員「先請示、後行動」是重要的安全機制。Lead 在 `plan_approval_response` 中可以附上 `feedback` 文字，指示隊員調整計劃：

```python
BUS.send("lead", req["from"], feedback,
         "plan_approval_response",
         {"request_id": request_id, "approve": False})
```

### 一個 FSM、兩種用途的可擴展性

關機協議與計劃審批協議的相同結構意味著，同一套 `pending → approved/rejected` 模式可以套用到任何需要握手確認的場景，例如：
- 資源鎖定申請
- 程式碼合併前的審查
- 危險指令的二次確認

---

## 常見問題 Q&A

### 1. s10 只是增加了關機協議，這個設計的實際用處是什麼？

s10 其實新增了**兩種**協議，不只關機：

| 協議 | 發起方 | 用途 |
| :--- | :--- | :--- |
| **Shutdown Protocol** | Lead → Teammate | 優雅終止隊員 |
| **Plan Approval Protocol** | Teammate → Lead | 高風險動作前先請示 |

兩者共用相同的 `request_id` 關聯模式，但解決的問題完全不同。

**關機協議的實際用處：**

沒有這個協議，停止隊員只有兩個選擇：(1) 等它自然跑完 50 輪上限，(2) 強制 kill 執行緒。這兩種都有問題：

*   **等自然結束**：若隊員卡在等待或重複嘗試，會白白消耗 API Token 直到上限。
*   **強制 kill**：隊員可能正在寫到一半的檔案、或尚未發出最後的回報，強制終止會留下不一致的狀態。

關機協議給了隊員**拒絕權**（`approve: false`）：隊員若正在處理關鍵工作（如正在寫入資料庫、尚未完成的程式碼修改），可以回覆「我還沒做完，請稍後再試」，避免資料損毀。這與 Unix 的 `SIGTERM`（請優雅退出）vs `SIGKILL`（強制殺死）的設計哲學完全相同。

**計劃審批協議的實際用處（更重要）：**

這是 s10 中更有實用價值的設計。隊員在執行**高風險操作前**必須先提交計劃等待 Lead 批准：

```python
# 隊員的 System Prompt 明確要求這件事
"Submit plans via plan_approval before major work."
```

實際應用場景：
*   隊員準備刪除一批舊檔案 → 先提交計劃 → Lead 確認範圍正確再批准
*   隊員要修改資料庫 Schema → 先請示 → Lead 可附上 `feedback` 要求調整後重新提交
*   隊員打算 force push 到主分支 → 先請示 → Lead 直接拒絕並指示改開 PR

這在本質上是一個 **Human-in-the-loop（人工介入點）** 的輕量實作，不需要暫停整個系統，只需要 Lead（或背後的人類）回覆一個 approve/reject 即可。

**兩個協議合在一起解決的根本問題：**

s09 的隊員一旦 spawn 就完全自主，Lead 無法在過程中介入或收回控制權。s10 補上了兩個「控制閥」：
1. 任務結束時：`shutdown_request` 確保收尾乾淨。
2. 任務進行中：`plan_approval` 確保高風險決策點有人把關。

這讓系統從「完全自主」進化為「**受控自主**」——隊員自主執行，但在邊界條件上保留人工控制。

