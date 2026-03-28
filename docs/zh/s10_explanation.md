# s10_team_protocols.py 執行流程詳解

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
