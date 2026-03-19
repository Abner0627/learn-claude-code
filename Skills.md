# 構建 Claude Code 的經驗: 我們如何使用 Skills (譯)

作者: Thariq Shihipar
原文: Lessons from Building Claude Code: How We Use Skills

**譯者導讀**
本文作者 Thariq Shihipar (@trq212) 是 Anthropic 的 Claude Code 團隊工程師，也是 Skills 功能的核心推動者之一。在加入 Anthropic 之前，他在 MIT Media Lab 讀研期間聯合創建了開源學術發布平台 PubPub，後來參加了 Y Combinator (W20 批次)。他在 X 上經常分享 Claude Code 的一手使用經驗和新功能動態。

這篇文章的價值在於: 它是來自 Anthropic 內部團隊的實戰總結。Anthropic 內部活躍使用的 Skills 已經有幾百個，文中的分類體系和編寫技巧都是從這些真實的內部實踐中提煉出來的。

如果你已經在用 Claude Code 但還沒認真做過 Skills，這篇文章能幫你建立一個系統化的思路: 做什麼類型的 Skills、怎麼寫、怎麼在團隊裡推廣。

---

Skills 已經成為 Claude Code 中使用最廣泛的擴展點 (extension points) 之一。它們靈活、容易製作，分發起來也很簡單。

但也正因為太靈活，你很難知道怎樣用才最好。什麼類型的 Skills 值得做？寫出好 Skill 的秘訣是什麼？什麼時候該把它們分享給別人？

我們在 Anthropic 內部大量使用 Claude Code 的 Skills (技能擴展)，目前活躍使用的已經有幾百個。以下就是我們在用 Skills 加速開發過程中總結出的經驗。

## 什麼是 Skills？

如果你還不了解 Skills，建議先看看我們的文檔或最新的 Skilljar 上關於 Agent Skills 的課程，本文假設你已經對 Skills 有了基本的了解。

我們經常聽到一個誤解，認為 Skills「只不過是 markdown 文件」。但 Skills 最有意思的地方恰恰在於它們不只是文本文件——它們是文件夾，可以包含腳本、資源文件、數據等等，智能體可以發現、探索和使用這些內容。

在 Claude Code 中，Skills 還擁有豐富的配置選項，包括註冊動態鉤子 (hooks)。

我們發現，Claude Code 中最有意思的那些 Skills，往往就是創造性地利用了這些配置選項和文件夾結構。


## Skills 的類型

在梳理了我們所有的 Skills 之後，我們注意到它們大致可以歸為幾個反覆出現的類別。最好的 Skills 清晰地落在某一個類別裡；讓人困惑的 Skills 往往橫跨了好幾個。這不是一份終極清單，但如果你想檢查團隊裡是否還缺了什麼類型的 Skills，這是一個很好的思路。

![1](https://pbs.twimg.com/media/HDo5BLyXEAA8McR?format=jpg&name=large)

### 1. 庫與 API 參考
**幫助你正確使用某個庫、命令行工具或 SDK 的 Skills**。它們既可以針對內部庫，也可以針對 Claude Code 偶爾會犯錯的常用庫。這類 Skills 通常會包含一個參考代碼片段的文件夾，以及一份 Claude 在寫代碼時需要避免的踩坑點 (gotchas) 列表。

示例: 
* `billing-lib` — 你的內部計費庫: 邊界情況、容易踩的坑 (footguns) 等
* `internal-platform-cli` — 內部 CLI 工具的每個子命令及其使用場景示例
* `frontend-design` — 讓 Claude 更好地理解你的設計系統

### 2. 產品驗證
**描述如何測試或驗證代碼是否正常工作的 Skills**。通常會搭配 Playwright、tmux 等外部工具來完成驗證。

驗證類 Skills 對於確保 Claude 輸出的正確性非常有用。值得安排一個工程師花上一周時間專門打磨你的驗證 Skills。

可以考慮一些技巧，比如讓 Claude 錄製輸出過程的視頻，這樣你就能看到它到底測試了什麼；或者在每一步強制執行程式化的狀態斷言。這些通常通過在 Skill 中包含各種腳本來實現。

示例: 
* `signup-flow-driver` — 在無頭瀏覽器中跑完註冊→郵件驗證→引導流程，每一步都可以插入狀態斷言的鉤子
* `checkout-verifier` — 用 Stripe 測試卡驅動結帳 UI，驗證發票最終是否到了正確的狀態
* `tmux-cli-driver` — 針對需要 TTY 的交互式命令行測試

### 3. 數據獲取與分析
**連接你的數據和監控體系的 Skills**。這類 Skills 可能會包含帶有憑證的數據獲取庫、特定的儀表盤 ID 等，以及常用工作流和数据获取方式的说明。

示例: 
* `funnel-query` — 「要看註冊→激活→付費的轉化，需要關聯哪些事件？」，再加上真正存放規範 user_id 的那張表
* `cohort-compare` — 對比兩個用戶群的留存或轉化率，標記統計顯著的差異，鏈接到分群定義
* `grafana` — 數據源 UID、集群名稱、問題→儀表盤對照表

### 4. 業務流程與團隊自動化
**把重複性工作流自動化為一條命令的 Skills**。這類 Skills 通常指令比較簡單，但可能會依賴其他 Skills 或 MCP（Model Context Protocol，模型上下文協定）。對於這類 Skills，把之前的執行結果保存在日誌檔中，有助於模型保持一致性並反思之前的執行情況。

示例: 
* `standup-post` — 匯總任務追蹤器、GitHub 活動和 Slack 消息→生成格式化的站會匯報，只報變化部分 (delta-only)
* `create-<ticket-system>-ticket` — 強制執行 schema 加上創建後的工作流
* `weekly-recap` — 已合併的 PR + 已關閉的工單 + 部署記錄→格式化的周報

### 5. 代碼腳手架與模板
**為代碼庫中的特定功能生成框架樣板代碼 (boilerplate) 的 Skills**。你可以把這些 Skills 和腳本組合使用。當你的腳手架（scaffolding）有自然語言需求、無法純靠代碼覆蓋時，這類 Skills 特別有用。

示例: 
* `new-<framework>-workflow` — 用你的註解搭建新的服務/工作流/處理器
* `new-migration` — 你的數據庫遷移文件模板加上常見踩坑點
* `create-app` — 新建內部應用，預配好認證、日誌和部署配置

### 6. 代碼質量與審查
**在團隊內部執行代碼質量標準並輔助代碼審查的 Skills**。可以包含確定性的腳本或工具來保證最大的可靠性。你可能希望把這些 Skills 作為鉤子的一部分自動運行，或者放在 GitHub Action 中執行。

示例: 
* `adversarial-review` — 生成一個全新視角的子智慧體來挑刺，實施修復，反復反覆運算直到發現的問題退化為吹毛求疵 [注：子智慧體（subagent）是指 Claude Code 在執行任務時啟動的另一個獨立 Claude 實例。這裡的做法是讓一個「沒見過這段代碼」的新實例來做代碼審查，避免原實例的思維慣性。]
* `code-style` — 強制執行代碼風格，特別是那些 Claude 默認做不好的風格
* `testing-practices` — 關於如何寫測試以及測試什麼的指導

### 7. CI/CD 與部署
**幫你拉取、推送和部署代碼的 Skills**。這類 Skills 可能會引用其他 Skills 來收集資料。

示例: 
* `babysit-pr` — 監控一個 PR→重試不穩定的 CI→解決合併衝突→啟用自動合併
* `deploy-<service>` — 構建→冒煙測試→漸進式流量切換並對比錯誤率→指標惡化時自動回滾
* `cherry-pick-prod` — 隔離的工作樹 (worktree)→cherry-pick→解決衝突→用模板創建 PR

### 8. 運維手冊
**接收一個現象（比如一條 Slack 消息、一條告警或者一個錯誤特徵），引導你走完多工具排查流程，最後生成結構化報告的 Skills**。

示例: 
* `<service>-debugging` — 把現象對應到工具→查詢模式，覆蓋你流量最大的服務
* `oncall-runner` — 拉取告警→檢查常見嫌疑→格式化輸出排查結論
* `log-correlator` — 給定請求 ID，從所有可能經過的系統中拉取匹配日誌

### 9. 基礎設施運維
**執行日常維護和運維操作的 Skills**。其中一些涉及破壞性操作，需要安全護欄。這些 Skills 讓工程師在執行關鍵操作時更容易遵循最佳實踐。

示例: 
* `<resource>-orphans` — 找到孤立資源→發到 Slack→等待觀察→用戶確認→級聯清理
* `dependency-management` — 你所在組織的依賴審批工作流
* `cost-investigation` — 檢查存儲或帶寬費用為何上漲

![2](https://pbs.twimg.com/media/HDo5Dl9XsAAWY7o?format=jpg&name=large)

## 編寫 Skills 的技巧

確定了要做什麼 Skill 之後，怎麼寫呢？以下是我們總結的一些最佳實踐和技巧。

我們最近還發佈了 Skill Creator，讓在 Claude Code 中創建 Skills 變得更加簡單。

### 不要說顯而易見的事
Claude Code 對你的代碼庫已經非常瞭解，Claude 本身對程式設計也很在行，包括很多默認的觀點。如果你發佈的 Skill 主要是提供知識，那就把重點放在能打破 Claude 常規思維模式的資訊上。

frontend design 這個 Skill 就是一個很好的例子——它是 Anthropic 的一位工程師通過與用戶反復反覆運算、改進 Claude 的設計品味而構建的，專門避免那些典型的套路，比如 Inter 字體和紫色漸變。


### 建一個踩坑點章節
![3](https://pbs.twimg.com/media/HDo5GDjbEAMAlZj?format=jpg&name=large)

任何 Skill 中信息量最大的部分就是踩坑點章節。這些章節應該根據 Claude 在使用你的 Skill 時遇到的常見失敗點逐步積累起來。理想情況下，你會持續更新 Skill 來記錄這些踩坑點。

### 利用文件系統與漸進式披露
![4](https://pbs.twimg.com/media/HDo5JN1WQAAPmnG?format=jpg&name=large)

就像前面說的，Skill 是一個資料夾，不只是一個 markdown 檔。你應該把整個檔案系統當作上下文工程（Context Engineering）和漸進式披露（progressive disclosure）的工具。告訴 Claude 你的 Skill 裡有哪些檔，它會在合適的時候去讀取它們。[注：上下文工程（Context Engineering）是 2025 年由 Andrej Karpathy 等人提出並廣泛傳播的概念，指的是精心設計和管理輸入給大語言模型的上下文資訊，以最大化模型的輸出品質。漸進式披露（progressive disclosure）借用了 UI 設計中的概念，意思是不一次性把所有資訊塞給模型，而是讓它在需要時再去讀取，從而節省上下文視窗空間。]

最簡單的漸進式披露形式是指向其他 markdown 檔讓 Claude 使用。例如，你可以把詳細的函數簽名和用法示例拆分到 `references/api.md` 裡。

另一個例子：如果你的最終輸出是一個 markdown 檔，你可以在 `assets/` 中放一個範本檔供複製使用。

你可以有參考資料、腳本、示例等資料夾，幫助 Claude 更高效地工作。


### 不要把 Claude 限制得太死
Claude 通常會努力遵循你的指令，而由於 Skills 的複用性很強，你需要注意不要把指令寫得太具體。給 Claude 它需要的資訊，但留給它適應具體情況的靈活性。例如：

![5](https://pbs.twimg.com/media/HDo5Lo1W8AArlpo?format=jpg&name=large)

### 考慮好初始設置
![6](https://pbs.twimg.com/media/HDo5OrObEAAENZA?format=jpg&name=large)

有些 Skills 可能需要使用者提供上下文來完成初始設置。例如，如果你做了一個把站會內容發到 Slack 的 Skill，你可能希望 Claude 先問用戶要發到哪個 Slack 頻道。

一個好的做法是把這些設置資訊存在 Skill 目錄下的 `config.json` 文件裡，就像上面的例子那樣。如果配置還沒設置好，智慧體就會向使用者詢問相關資訊。

如果你希望智慧體向使用者展示結構化的多選題，可以讓 Claude 使用 AskUserQuestion 工具。


### description 字段是給模型看的
當 Claude Code 啟動一個會話時，它會構建一份所有可用 Skills 及其描述的清單。Claude 通過掃描這份清單來判斷“這個請求有沒有對應的 Skill？”所以 description 欄位不是摘要——它描述的是何時該觸發這個 Skill。[注：這條建議經常被忽略。很多人寫 description 時會寫“這個 Skill 做什麼”，但 Claude 需要的是“什麼情況下該用這個 Skill”。好的 description 讀起來更像 if-then 條件，而不是功能說明。]

![7](https://pbs.twimg.com/media/HDo5Rspa0AApYXU?format=jpg&name=large)

### 記憶與數據存儲
![8](https://pbs.twimg.com/media/HDo5UrhbEAAqlvQ?format=jpg&name=large)

有些 Skills 可以通過在內部存儲資料來實現某種形式的記憶。你可以用最簡單的方式——一個隻追加寫入的文本日誌檔或 JSON 檔，也可以用更複雜的方式——比如 SQLite 資料庫。

例如，一個 standup-post Skill 可以保留一份 `standups.log`，記錄它寫過的每一條站會彙報。這樣下次運行時，Claude 會讀取自己的歷史記錄，就能知道從昨天到現在發生了什麼變化。

存在 Skill 目錄下的資料可能會在升級 Skill 時被刪除，所以你應該把資料存在一個穩定的資料夾中。目前我們提供了 `${CLAUDE_PLUGIN_DATA}` 作為每個外掛程式的穩定資料存儲目錄。

### 存儲腳本與生成代碼
你能給 Claude 的最強大的工具之一就是代碼。給 Claude 提供腳本和庫，讓它把精力花在組合編排上——決定下一步做什麼，而不是重新構造樣板代碼。

例如，在你的資料科學 Skill 中，你可以放一組從事件源獲取資料的函式程式庫。為了讓 Claude 做更複雜的分析，你可以提供一組輔助函數，像這樣：

![9](https://pbs.twimg.com/media/HDo5XLwXcAA-ocz?format=jpg&name=large)

Claude 就可以即時生成腳本來組合這些功能，完成更高級的分析——比如回答“週二發生了什麼？”這樣的問題。

![10](https://pbs.twimg.com/media/HDo5ZkUW0AAMmfP?format=jpg&name=large)

### 按需鉤子 (On Demand Hooks)

Skills 可以包含只在該 Skill 被調用時才啟動的鉤子（On Demand Hooks），並且在整個會話期間保持生效。這適合那些比較主觀、你不想一直運行但有時候極其有用的鉤子。

例如：
* `/careful` — 通過 PreToolUse 匹配器攔截 Bash 中的 `rm -rf`、`DROP TABLE`、`force-push`、kubectl delete。你只在知道自己在操作生產環境時才需要這個——要是一直開著會讓你抓狂【注：PreToolUse 是 Claude Code 的鉤子（hook）機制之一，會在 Claude 每次調用工具之前觸發。你可以在這個鉤子裡檢查 Claude 即將執行的命令，如果命中危險操作就阻止執行。這裡 `/careful` 是一個按需啟動的 Skill，只有用戶主動調用時才會註冊這個鉤子。】
* `/freeze` — 阻止對特定目錄之外的任何 Edit/Write 操作。在調試時特別有用：“我想加日誌但老是不小心'修'了不相關的代碼”

Skills 最大的好處之一就是你可以把它們分享給團隊的其他人。

你可以通過兩種方式分享 Skills：

* 把 Skills 提交到你的代碼倉庫中（放在 `./.claude/skills` 下）
* 做成外掛程式，搭建一個 Claude Code 外掛程式市場（Plugin Marketplace），讓用戶可以上傳和安裝外掛程式（詳見文檔）

對於在較少代碼倉庫上協作的小團隊，把 Skills 提交到倉庫中就夠用了。但每個提交進去的 Skill 都會給模型的上下文增加一點負擔。隨著規模擴大，內部外掛程式市場可以讓你分發 Skills，同時讓團隊成員自己決定安裝哪些。

### 管理外掛程式市場
怎麼決定哪些 Skills 放進外掛程式市場？大家怎麼提交？

我們沒有一個專門的中心團隊來決定這些事；我們更傾向于讓最有用的 Skills 自然湧現出來。如果你有一個想讓大家試試的 Skill，你可以把它上傳到 GitHub 的一個沙箱資料夾裡，然後在 Slack 或其他論壇裡推薦給大家。

當一個 Skill 獲得了足夠的關注（由 Skill 的作者自己判斷），就可以提交 PR 把它移到外掛程式市場中。

需要提醒的是，創建品質差或重複的 Skills 很容易，所以在正式發佈之前確保有某種審核機制很重要。

### 組合 Skills
你可能希望 Skills 之間互相依賴。例如，你可能有一個檔上傳 Skill 用來上傳檔，以及一個 CSV 生成 Skill 用來生成 CSV 並上傳。這種依賴管理目前在外掛程式市場或 Skills 中還不支援，但你可以直接按名字引用其他 Skills，只要對方已安裝，模型就會調用它們。

### 衡量 Skills 的效果
為了瞭解一個 Skill 的表現，我們使用了一個 PreToolUse 鉤子來在公司內部記錄 Skill 的使用情況（示例代碼在這裡）。這樣我們就能發現哪些 Skills 很受歡迎，或者哪些觸發頻率低於預期。

Skills 是 AI 智慧體（AI Agent）極其強大且靈活的工具，但這一切還處於早期階段，我們都在摸索怎樣用好它們。

與其把這篇文章當作權威指南，不如把它看作我們實踐中驗證過有效的一堆實用技巧合集。理解 Skills 最好的方式就是動手開始做、不斷試驗、看看什麼對你管用。我們大多數 Skills 一開始就是幾行文字加一個踩坑點，後來因為大家不斷補充 Claude 遇到的新邊界情況，才慢慢變好的。