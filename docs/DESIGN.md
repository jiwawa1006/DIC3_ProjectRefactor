# ShopAssist 系統設計

ShopAssist 是一個依據知識庫回答電商購物問題的客服 AI Agent，回答必附來源；證據不足時拒答，缺少資訊時追問，需要人工權限時建立模擬工單。這份文件是 Proposal 階段的系統設計，說明系統要解決什麼問題、由哪些元件組成、資料怎麼流動、狀態怎麼決定，以及預計如何實作與驗證。程式碼在 Final 階段加入，執行方式與實驗結果屆時補入。

| 要找 | 看哪一節 |
| :--- | :--- |
| 系統要解決的問題與研究問題 | Problem |
| 狀態與代碼定義、功能與非功能需求 | Requirements |
| ①–⑧ 架構圖、處理流程與 Data Flow | Architecture |
| 每個元件的職責、輸入輸出與設計理由 | Components |
| 知識庫、資料集與紀錄格式 | Data |
| AI Model、檢索、閘門門檻、依據檢查、評審、指標與實驗設計 | AI/Analytics |
| 消保法、轉接規則、個資遮蔽如何寫進系統 | Domain Knowledge |
| 有行號的 pseudocode | Algorithm |
| Implementation Mapping 與里程碑 | Implementation |
| 執行方式 | How to Run |
| 實驗結果與 Limitations | Results |

引用編號與 Proposal Document 的 References 相同。研究背景、文獻回顧與實驗計畫詳見 Proposal Document。

## Problem

電商客服的詢問集中在退換貨、運費與退款規則。根據阿里巴巴的資料，淘寶的換貨、退貨、退款、維修與運費詢問占全部客服聊天的 56.5% [3]。這些答案寫在商店的說明中心，客服 AI 回答這類問題時有四個問題：

| 編號 | 問題 | 證據 |
| :--- | :--- | :--- |
| P1 | Large Language Model（LLM）直接回答時會補上商店沒寫的細節，例如編出退款天數或使用規則。加入 Retrieval-Augmented Generation（RAG）能減少但不能消除錯誤，模型在 context 不足時傾向回答而不是拒答 | Cursor 的客服機器人編造了不存在的使用政策 [5]；CRAG 中 LLM-only 的準確率不超過 34%，直接加入 RAG 只到 44% [10]；context 不足時模型常輸出錯誤答案而不是拒答 [12] |
| P2 | 退換貨答案有法律底線，說錯會由商店承擔 | 台灣網購屬於消費者保護法的通訊交易，七日解除權與 15 日內退款是法定義務 [7]；Moffatt v. Air Canada 案中，審裁處認定航空公司要為網站聊天機器人說錯的退款規則負責 [4] |
| P3 | 查訂單、執行退款、處理帳務爭議需要權限與個人資料，客服 AI 不應自行處理 | 卡號與安全碼受 PCI DSS 限制 [66][67]，其他個資受個人資料保護法規範 [68]；Intercom 建議把需要主管核准的政策例外與超過門檻的帳務爭議轉給真人 [25] |
| P4 | 顧客要求能轉接真人，失敗的 AI 體驗會讓顧客不再使用 | 87% 的顧客認為使用 GenAI 客服的企業必須提供真人管道 [26]；一次負面體驗後只有 27% 的顧客願意再試聊天機器人 [54] |

因此 ShopAssist 要解決的問題是：給定一則繁體中文的單輪顧客問題與示範商店的知識庫，為這一題選出回答、拒答、追問、轉接四種處置中正確的一種；選擇回答時，每個關鍵事實都必須由引用的知識庫片段支持。範圍限定為單輪問題與虛構商店，系統不查詢訂單、不執行付款或退款，轉接只建立模擬工單。詳見 Proposal Document 的 Introduction C 節。

本專題以四個研究問題界定要回答的事，與 Proposal Document 的 Introduction C 節相同：

| 編號 | 研究問題 |
| :--- | :--- |
| RQ1 | 相較純 LLM，基本 RAG 是否提高可回答題的關鍵事實命中率？ |
| RQ2 | 相較基本 RAG，ShopAssist 的證據閘門與依據檢查能否降低不安全回答率、提高拒答召回率？ |
| RQ3 | 可靠性提升的代價是多少？以每降低一個百分點不安全回答率所增加的錯誤拒答率衡量，並記錄延遲與 token 用量。 |
| RQ4 | 八類題目的錯誤模式有何差異？錯誤來自檢索還是生成？混合、BM25、dense 三種檢索的差異多大？ |

## Requirements

需求與後續各節共用下列代碼。

| 代碼類別 | 值 | 意義 |
| :--- | :--- | :--- |
| 狀態 | `ANSWERED`、`ABSTAIN`、`CLARIFY`、`HANDOFF` | 回答、拒答、追問、轉接，每題只有一個狀態 |
| 路由 | `SENSITIVE`、`REFUSE`、`HANDOFF`、`CLARIFY`、`RETRIEVE` | ② 意圖分類的輸出 |
| `handoff_reason` | `USER_REQUEST`、`PAYMENT_DISPUTE`、`REFUND_EXECUTION`、`ORDER_LOOKUP`、`POLICY_OVERRIDE_ATTEMPT`、`LOW_EVIDENCE_ESCALATION` | 轉接原因，寫入模擬工單 |
| 變體 | `A_llm_only`、`B_rag_basic`、`C_shopassist` | 純 LLM、基本 RAG、ShopAssist 完整流程 |
| 檢索模式 | `hybrid`、`bm25`、`dense` | ④ 的三種檢索方式 |

「對應」欄指出每項需求來自哪個問題或研究問題。標為「工程」的需求是讓系統能安全執行與重現實驗的條件。

### 功能需求

| 編號 | 需求 | 對應 | 預計模組 | 驗證方式 |
| :--- | :--- | :--- | :--- | :--- |
| FR-01 | 載入 Markdown 知識庫，驗證 frontmatter，依 `##` 區段切成 chunk | P1 | 知識庫載入 | 單元測試確認 chunk 數等於 `##` 區段數，目前草稿為 7 份文件、47 個；缺少必填欄位或 `doc_id` 與檔名不符時報錯並指出檔名 |
| FR-02 | 以 BM25、dense embedding 與 RRF 檢索 Top-k，保留各分項分數 | P1、RQ4 | 混合檢索 | 單元測試確認三種模式各回傳最多 k 個 chunk，`hybrid` 的融合分數等於兩路 `1 / (60 + rank)` 之和 |
| FR-03 | 把問題分成五種路由 | P2、P3、P4 | 意圖分類 | 13 條規則各有至少一則命中與一則不命中的測試問題；詢問真人客服服務時間不判為轉接 |
| FR-04 | 生成前以檢索分數判斷證據是否足夠，不足時不呼叫 LLM | P1 | 證據閘門 | 單元測試確認分數閘門擋下時沒有 LLM 呼叫 |
| FR-05 | 以 `<evidence>` 區塊提供證據，要求逐句標註 `[chunk_id]` 並輸出 `<<ANSWER_META>>` | P1 | 依據生成 | 以模擬 LLM 的輸出測試：prompt 含 `<evidence>` 區塊；解析取最後一個 meta 行；無法解析時 `can_answer` 為 null |
| FR-06 | 依生成模型的 `can_answer` 判斷 context 是否充分 | P1 | 證據閘門 | 單元測試確認 `can_answer` 為 false 或 null 時回 `ABSTAIN` |
| FR-07 | 檢查引用屬於檢索結果、回答中的數字出現在引用 chunk | P1、P2 | 依據檢查 | 單元測試確認引用不在 Top-k、或數字沒有依據時，回答改為 `ABSTAIN` |
| FR-08 | 轉接時建立模擬工單並標明原因代碼，附上有依據的政策說明 | P2、P3 | 轉接與工單、Agent 協調 | 單元測試確認工單含編號、原因代碼、遮蔽後的問題、摘要與來源；政策說明沒通過依據檢查時被捨棄，狀態仍為 `HANDOFF` |
| FR-09 | 退貨資格問題沒有說明收貨時間或沒有指明商品時追問 | P1、P2 | 意圖分類、Agent 協調 | 單元測試確認回 `CLARIFY`，追問句包含收貨日期、商品類別與是否拆封 |
| FR-10 | 使用者提供卡號、CVV、密碼時回覆固定提醒，不呼叫 LLM | P3 | 意圖分類、固定回覆 | 單元測試確認敏感資料不送往 LLM |
| FR-11 | 拒答後使用者可按「轉人工」，原因記為 `LOW_EVIDENCE_ESCALATION` | P4 | Agent 協調、對話介面 | 單元測試與介面手動操作 |
| FR-12 | 介面顯示狀態標籤、來源、各階段判定與模擬工單清單 | P1、P4 | 對話介面 | 以 Components ① 的四個示範問題手動操作 |
| FR-13 | 提供三種變體 | RQ1、RQ2、RQ3 | Agent 協調 | 單元測試確認 A 與 B 不經過意圖規則與閘門 |
| FR-14 | 批次評估、計算指標、LLM-as-a-judge、重複執行與續跑 | RQ1–RQ4 | 評估流程 | 以模擬 LLM 跑完三個變體並檢查指標；續跑時只重跑有錯誤的題目 |
| FR-15 | 意圖規則、兩道閘門、依據檢查、轉接政策說明可分別關閉 | RQ2、RQ4 | 設定、Agent 協調 | 單元測試確認每個開關的效果 |

### 非功能需求

| 編號 | 需求 | 對應 | 預計模組 | 驗證方式 |
| :--- | :--- | :--- | :--- | :--- |
| NFR-01 | API key 只從環境變數或本機設定檔讀取，不寫入結果檔 | 工程 | 設定 | 單元測試確認結果檔不含金鑰；設定檔不納入版本控制 |
| NFR-02 | 切換 LLM provider 只改設定，不改程式 | 工程 | LLM provider 介面 | 單元測試以設定切換 provider |
| NFR-03 | 狀態由程式依結構化欄位決定，不比對回答措辭 | P1 | Agent 協調、依據生成 | 單元測試確認拒答措辭不影響狀態 |
| NFR-04 | 個資在送往 LLM、日誌、工單、結果檔之前遮蔽 | P3 | 個資遮蔽 | 單元測試確認 LLM 輸入與工單不含原始個資 |
| NFR-05 | 每則轉接訊息標明模擬轉接，未通知真人 | P4 | 固定回覆、轉接與工單 | 單元測試確認轉接訊息內容 |
| NFR-06 | 檢索內容與使用者貼上的文字只當資料，不當指令 | P2 | 依據生成、意圖分類 | 以植入惡意文字的測試知識庫做單元測試 |
| NFR-07 | 評審模型必須與生成模型不同 | RQ1–RQ3 | 評估流程、LLM provider 介面 | 設定相同模型時評估拒絕執行 |
| NFR-08 | 遇到 429、逾時、5xx 時退避重試，並維持最小呼叫間隔 | 工程 | LLM provider 介面 | 以模擬回應測試重試與間隔 |
| NFR-09 | embedding model 無法載入時改用 BM25，並記錄實際模式 | 工程 | 混合檢索 | 單元測試模擬載入失敗 |
| NFR-10 | 每次評估記錄模型、門檻、元件開關、prompt 與知識庫 fingerprint | RQ1–RQ4 | 評估流程 | 檢查每次評估的執行設定紀錄 |
| NFR-11 | 所有測試離線執行，不需 API key | 工程 | 模擬 LLM | 沒有 API key 的環境跑完全部測試 |
| NFR-12 | 對話開頭揭露 AI 身分，並告知可轉真人 | P4 | 固定回覆、對話介面 | 啟動介面後確認第一則訊息含 AI 身分與轉真人的方式 |
| NFR-13 | 每題記錄延遲、token 與費用 | RQ3 | Agent 協調 | 單元測試確認每題紀錄的欄位 |

## Architecture

### 架構圖

```mermaid
flowchart TB
    S1["① User/Application Scenario<br/>對話介面、示範情境"]
    S2["② Query/Input<br/>13 條意圖規則、個資遮蔽"]
    S3["③ Planner & Reasoning Agent<br/>路由與檢查順序、決定狀態"]
    S4["④ Data Acquisition/Retrieval<br/>BM25 + dense + RRF"]
    S5["⑤ Analytics Engine<br/>分數閘門、充分性閘門"]
    S6["⑥ Domain Knowledge Integration<br/>領域規則 prompt、依據生成"]
    S7["⑦ Decision/Recommendation<br/>引用與數字的依據檢查"]
    S8["⑧ Output & Interaction<br/>回覆、來源、轉接工單"]
    LAW[("全國法規資料庫<br/>消保法、適用準則、民法")]
    KB[("Knowledge Base<br/>7 份 Markdown、47 個 chunk")]
    IDX[("BM25 index 與 Vector DB<br/>512 維 embedding")]
    TK[("Ticket store<br/>模擬工單 JSONL")]
    EV[("Evaluation store<br/>78 題 CSV 與逐題紀錄")]
    LLM["LLM provider<br/>OpenAI 相容 API、Anthropic API、離線模擬"]

    S1 -- "問題 Q" --> S2
    S2 -- "路由、轉接原因、遮蔽後的 Q" --> S3
    S3 -- "SENSITIVE、REFUSE、CLARIFY 固定回覆" --> S8
    S3 -- "RETRIEVE、HANDOFF" --> S4
    LAW -.->|"撰寫依據"| KB
    KB --> S4
    S4 <--> IDX
    S4 -- "Top-k chunks 與分數" --> S5
    S5 -- "分數閘門通過" --> S6
    S6 <--> LLM
    S6 -- "回答、引用、can_answer" --> S5
    S5 -- "充分性閘門通過" --> S7
    S5 -- "閘門判定" --> S3
    S7 -- "依據檢查結果" --> S3
    S3 -- "最終狀態" --> S8
    S8 --> TK
    S8 -- "狀態標籤、回覆、來源、轉人工按鈕" --> S1
    S1 -- "按下轉人工" --> S8
    S8 -- "評估時寫入逐題結果" --> EV
```

圖 1 系統架構

③ 是整條管線的協調者：它依 ② 的路由決定直接回覆固定文字、進入轉接路徑或進入檢索路徑，依序呼叫 ④ 到 ⑦，再把它們的結果組成最終狀態交給 ⑧。⑤ 出現兩次，生成前是分數閘門，生成後是充分性閘門。全國法規資料庫只在撰寫知識庫時使用，執行期間不連線。

### 處理流程

處理流程依 Input → Processing → AI/Analytics → Decision → Output 排列，分成三張圖。圖 2 是輸入與意圖路由，圖 3 是檢索與回答路徑，圖 4 是轉接與拒答後轉人工的流程。

```mermaid
flowchart TD
    subgraph INPUT["Input"]
        IN(["使用者輸入問題 Q"])
    end
    subgraph PROC["Processing"]
        CLS["13 條規則讀原文<br/>判定路由與轉接原因"]
        RED["遮蔽個資<br/>下游只拿到遮蔽後的 Q"]
        INT{"意圖路由"}
    end
    subgraph OUT["Output"]
        SEN["固定隱私提醒<br/>ANSWERED"]
        REF["固定拒答<br/>ABSTAIN"]
        CLA["追問收貨日期、<br/>商品類別、是否拆封<br/>CLARIFY"]
        SHOW(["顯示狀態標籤、回覆、來源與判定紀錄"])
    end
    RP(["檢索與回答路徑<br/>見圖 3"])
    HO(["轉接路徑<br/>見圖 4"])

    IN --> CLS --> RED --> INT
    INT -- "SENSITIVE" --> SEN
    INT -- "REFUSE" --> REF
    INT -- "CLARIFY" --> CLA
    INT -- "RETRIEVE" --> RP
    INT -- "HANDOFF" --> HO
    SEN --> SHOW
    REF --> SHOW
    CLA --> SHOW
    CLA -.->|"使用者補上資訊，以新訊息重新提問"| IN
```

圖 2 處理流程：輸入與意圖路由

圖 3 的三道檢查：分數閘門看最高 dense 相似度與詞彙覆蓋率；充分性閘門看模型輸出的 `can_answer` 是否為 true；依據檢查要求引用有效且數字有依據。

```mermaid
flowchart TD
    START(["意圖路由判定 RETRIEVE"])
    subgraph PROC["Processing"]
        RET["檢索 Top-k<br/>BM25 + dense + RRF"]
    end
    subgraph AI["AI/Analytics"]
        SG{"分數閘門<br/>通過？"}
        GEN["LLM 依 evidence 生成<br/>回答加 ANSWER_META"]
        SUF{"充分性閘門<br/>通過？"}
    end
    subgraph DEC["Decision"]
        GR{"依據檢查<br/>通過？"}
    end
    subgraph OUT["Output"]
        ANS["回答加來源<br/>ANSWERED"]
        AB["知識庫沒有足夠資料<br/>ABSTAIN"]
        LER["系統暫時無法回覆<br/>ABSTAIN，記錄 error"]
        SHOW(["顯示狀態標籤、回覆、來源與判定紀錄<br/>ABSTAIN 附「轉人工」按鈕，見圖 4"])
    end

    START --> RET
    RET -- "例外：沒有檢索結果" --> AB
    RET --> SG
    SG -- "否" --> AB
    SG -- "是" --> GEN
    GEN -- "例外：LLM 呼叫失敗" --> LER
    GEN --> SUF
    SUF -- "否或缺少 meta" --> AB
    SUF -- "是" --> GR
    GR -- "否" --> AB
    GR -- "是" --> ANS
    ANS --> SHOW
    AB --> SHOW
    LER --> SHOW
```

圖 3 處理流程：檢索與回答路徑

```mermaid
flowchart TD
    subgraph INPUT["Input"]
        HIN(["意圖路由判定 HANDOFF"])
        ABIN(["ABSTAIN 回覆的「轉人工」按鈕"])
    end
    subgraph PROC["Processing"]
        HRET["檢索 Top-k"]
    end
    subgraph AI["AI/Analytics"]
        HSG{"分數閘門通過？"}
        HGEN["LLM 產生轉接前政策說明"]
    end
    subgraph DEC["Decision"]
        HGR{"can_answer 為 true<br/>且依據檢查通過？"}
        BTN{"使用者按「轉人工」？"}
    end
    subgraph OUT["Output"]
        NOTE["政策說明放在轉接訊息前"]
        MH["拒答後轉人工<br/>LOW_EVIDENCE_ESCALATION"]
        TK["建立模擬工單<br/>HANDOFF"]
        SHOW(["顯示狀態標籤、回覆、來源與判定紀錄"])
    end

    HIN --> HRET --> HSG
    HSG -- "否" --> TK
    HSG -- "是" --> HGEN
    HGEN --> HGR
    HGR -- "否或 LLM 錯誤，捨棄說明" --> TK
    HGR -- "是" --> NOTE
    NOTE --> TK
    ABIN --> BTN
    BTN -- "是" --> MH
    MH --> TK
    BTN -- "否" --> SHOW
    TK --> SHOW
```

圖 4 處理流程：轉接與轉人工迴圈

1. 13 條意圖規則先讀未遮蔽的原文，判定路由與轉接原因，因為 SENSITIVE 必須看到卡號才能判定。
2. 判定後立刻遮蔽卡號、身分證字號、手機號碼、Email、CVV 與密碼，檢索、LLM、工單與日誌都只拿到遮蔽後的問題。路由共五種，SENSITIVE、REFUSE、CLARIFY 直接回覆固定文字，不檢索也不呼叫 LLM。
3. HANDOFF 一律建立工單。分數閘門通過時先請 LLM 說明相關規定，`can_answer` 為 true 且依據檢查通過才把說明放在轉接訊息前，檢查數字時不採計問題中的數字；任何一步失敗都只捨棄說明，狀態仍是 HANDOFF。
4. RETRIEVE 檢索 Top-k。沒有檢索結果或分數閘門不通過時回 ABSTAIN，不呼叫 LLM。
5. 通過分數閘門後呼叫 LLM。呼叫失敗回 ABSTAIN 並記錄錯誤。
6. `can_answer` 為 false 或缺少 `<<ANSWER_META>>` 時回 ABSTAIN。
7. 依據檢查不通過回 ABSTAIN，通過則回 ANSWERED 並附引用來源。
8. 最新一則回覆不是轉接時，介面顯示「轉人工」按鈕。按下時，上一則回覆是 ABSTAIN 則原因記為 `LOW_EVIDENCE_ESCALATION`，否則記為 `USER_REQUEST`；工單沿用上一則回覆的遮蔽後問題與檢索來源。使用者直接輸入「轉真人客服」時，由意圖規則判定為 HANDOFF，原因記為 `USER_REQUEST`。

### Data Flow

下表列出一個問題在模組之間傳遞的資料，順序與 RETRIEVE 路徑相同。HANDOFF 路徑同樣經過第 3 到 10 列，產生的政策說明連同轉接原因在第 11 列交給 ⑧；SENSITIVE、REFUSE、CLARIFY 在 ③ 之後直接跳到第 11 列。

| 列 | 從 → 到 | 傳遞的資料 |
| :--- | :--- | :--- |
| 1 | ① → ② | 原始問題 Q、上一輪狀態 |
| 2 | ② → ③ | 路由、命中的規則、`handoff_reason`、追問句、遮蔽後的 Q；原始 Q 不再往下傳 |
| 3 | 知識庫 → ④ | 每個 chunk 的 `chunk_id`、`doc_id`、`version`、標題與正文；dense 向量讀寫 embedding 快取 |
| 4 | ③ → ④ | 遮蔽後的 Q、檢索模式、k |
| 5 | ④ → ⑤ | Top-k chunk 與各自的融合分數、BM25 分數、dense 分數、名次；`coverage`、`bm25_hits` |
| 6 | ⑤ → ⑥ | 分數閘門的判定與訊號；通過時交出 Top-k chunk |
| 7 | ⑥ ↔ LLM provider | 送出 system prompt、`<evidence>` 區塊與遮蔽後的 Q；收回回答文字、`<<ANSWER_META>>`、token 數 |
| 8 | ⑥ → ⑤ | 回答、`can_answer`、引用的 `chunk_id`、`missing_info` |
| 9 | ⑤ → ⑦ | 通過充分性閘門的回答與引用、Top-k chunk、遮蔽後的 Q |
| 10 | ⑦ → ③ | 是否通過、`citation_valid`、失敗原因、不被支持的數字 |
| 11 | ③ → ⑧ | 狀態、回答、引用的 chunk、`handoff_reason`、各階段判定、延遲、token、費用 |
| 12 | ⑧ → Ticket store | 模擬工單一行 JSON |
| 13 | ⑧ → ① | 回覆文字、狀態標籤、來源片段、「轉人工」按鈕 |
| 14 | ⑧ → Evaluation store | 評估時每個變體每題一行執行結果 |
| 15 | ① → ⑧ | 使用者按下「轉人工」：上一則回覆的狀態、遮蔽後的問題與檢索來源，供建立工單 |

四種紀錄的欄位見 Data 的「紀錄格式」。

## Components

### ① User/Application Scenario

**做什麼**：提供顧客與示範商店客服對話的入口。Streamlit 介面顯示 AI 身分揭露、對話紀錄、狀態標籤、來源片段、各階段判定與模擬工單清單；側欄可切換變體、provider、檢索模式、Top-k 與閘門門檻。命令列示範跑四個示範問題：「退貨期限是幾天」應回答，「退款幾天會到信用卡帳單」應拒答，「幫我查訂單」應轉接，「我買的耳機還能退嗎」應追問。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 使用者輸入的繁體中文問題；「轉人工」按鈕 |
| 輸出 | ③ 回傳的狀態、回覆、來源與工單 |
| 預計模組 | 對話介面、命令列示範 |

**設計理由**：示範商店是虛構的台灣網路商店，介面只處理說明中心能回答的一般政策，個別訂單與帳務一律轉接。對話開頭揭露 AI 身分，參考人工智慧基本法與 EU AI Act 的透明原則 [69][70]；同時告知可轉真人，因為多數顧客要求保留真人管道 [26]。

### ② Query/Input

**做什麼**：以固定規則表分類意圖與風險，並遮蔽個資。規則依表中順序比對，第一條命中的規則決定路由，命中的所有規則都記錄下來供分析。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 未遮蔽的原始問題，規則需要看到卡號本身才能偵測 |
| 輸出 | 路由、命中的規則、轉接原因、追問句；遮蔽後的問題 |
| 預計模組 | 意圖分類、個資遮蔽 |

規則表共 13 條，依序為：

| 規則 | 路由 | `handoff_reason` |
| :--- | :--- | :--- |
| `sensitive_data_present`、`sensitive_data_share` | SENSITIVE | |
| `prompt_extraction` | REFUSE | |
| `policy_override` | HANDOFF | `POLICY_OVERRIDE_ATTEMPT` |
| `escalation_after_no_answer` | HANDOFF | `LOW_EVIDENCE_ESCALATION` |
| `explicit_human_request`、`formal_complaint` | HANDOFF | `USER_REQUEST` |
| `payment_dispute` | HANDOFF | `PAYMENT_DISPUTE` |
| `refund_execution` | HANDOFF | `REFUND_EXECUTION` |
| `order_lookup` | HANDOFF | `ORDER_LOOKUP` |
| `clarify_vague_receipt_time`、`clarify_unspecified_item`、`clarify_own_purchase_without_date` | CLARIFY | |

沒有命中任何規則的問題路由為 RETRIEVE。詢問真人客服的服務時間不算轉接要求。

**設計理由**：知識庫沒有個別訂單與付款資料，查訂單、執行退款與帳務爭議的 context 必然不充分，而模型在 context 不足時傾向回答而不是拒答 [12]，因此在檢索之前就分流。τ-bench 的客服政策也把轉接真人設計成明確動作，只在請求超出 agent 能力範圍時使用 [15]。

### ③ Planner & Reasoning Agent

**做什麼**：依 Algorithm 1 決定每一步，並組合最終狀態。它是唯一決定狀態的地方，判斷只讀意圖分類、閘門、`can_answer` 與依據檢查的結構化結果。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 問題、上一輪狀態 |
| 輸出 | 狀態、回答、引用、工單、各階段判定、延遲、token、費用 |
| 預計模組 | Agent 協調 |

`A_llm_only` 只執行生成，`B_rag_basic` 執行檢索加生成，兩者都不經過意圖規則與閘門。

**設計理由**：LLM 不嚴格遵守指定的拒答措辭，措辭無法當作狀態觸發條件 [33]；強模型在 context 不足時也傾向回答 [12]。拒答與轉接因此由程式決定，不交給模型自行判斷。

ShopAssist 是 Agent 而不是單一 chatbot：③ 對每個問題感知意圖、決定是否取得資料、檢查證據，再從回答、拒答、追問、轉接四種行動中選一種，轉接時建立工單這個外部動作。決策規則固定寫在程式裡，換來每個判定都能重現與追溯；LLM 只負責在證據範圍內寫出回答與結構化自評。

### ④ Data Acquisition/Retrieval

**做什麼**：把知識庫載入成 section-level chunk，並以 BM25 與 dense embedding 雙路檢索，再以 Reciprocal Rank Fusion（RRF）融合排名。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 知識庫的 Markdown 文件；查詢文字 |
| 輸出 | Top-k chunk，各帶融合分數、BM25 分數、dense 分數與名次；詞彙覆蓋率；BM25 命中數 |
| 預計模組 | 知識庫載入、混合檢索 |

**設計理由**：已上線客服 chatbot 的研究以 RRF 融合兩個 embedding 模型，Recall@10 由 69.4% 升到 73.2%；改為融合 BM25 則降到 58.1% [14]。BEIR 則顯示 BM25 在跨領域 zero-shot 檢索是穩健的 baseline [37]。BM25 是否值得融合沒有定論，系統因此保留三種模式，並把實際模式寫入每一列結果，由消融實驗決定。

### ⑤ Analytics Engine

**做什麼**：兩道證據閘門。分數閘門在生成前讀 dense cosine 與詞彙覆蓋率，擋下明顯不相關的問題，不呼叫 LLM；充分性閘門在生成後讀 `can_answer`，擋下主題相關但缺少細節的問題。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 檢索結果；生成模型的 `can_answer` 與 `missing_info` |
| 輸出 | 是否通過、判定字串、訊號 `dense_top`、`dense_margin`、`bm25_top`、`bm25_hits`、`coverage` |
| 預計模組 | 證據閘門 |

**設計理由**：相關不等於充分。以 context 充分性資訊引導拒答的 selective generation，讓有回答時的正確比例提升 2–10% [12]。ShopAssist 改以生成模型自評的 `can_answer` 作為充分性訊號，做法與該研究不同，效果由消融實驗檢驗。RRF 分數只由名次決定，不能代表證據強弱，所以分數閘門讀 dense 與 lexical 分項。兩道閘門都是保守的路由訊號，不能證明回答正確。

### ⑥ Domain Knowledge Integration

**做什麼**：把檢索到的商店政策與客服規則組成 grounded prompt，呼叫 LLM，再把回覆拆成顧客看到的回答與 `<<ANSWER_META>>` 結構化欄位。system prompt 寫入領域規則：只能依 `<evidence>` 回答、不得承諾資料中沒有的退款與折扣、無法查詢訂單或執行退款、被問到身分時說明是 AI。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 遮蔽後的問題、Top-k chunks |
| 輸出 | 回答、`can_answer`、引用的 chunk、`missing_info`；token 與費用 |
| 預計模組 | 依據生成；LLM provider 介面，支援 OpenAI 相容 API、Anthropic API 與離線模擬 |

轉接類問題使用另一段 system prompt，只要求在 100 字內說明相關的消費者權益，不得承諾處理結果。

**設計理由**：LLM 整合應用模糊了資料與指令的界線，檢索內容可能夾帶 indirect prompt injection [47]。證據以標示 `type="data"` 的區塊包住，system prompt 明確要求不執行資料中的指令，這是 BIPIA 提出的 boundary awareness 與 explicit reminder 防禦 [48]。StruQ 等更強的防禦需要微調模型 [49]，呼叫商用 API 的 ShopAssist 無法採用，因此 ⑦ 的數字檢查是第二道防線。

### ⑦ Decision/Recommendation

**做什麼**：生成後的依據檢查，決定回答能否以 ANSWERED 送出。檢查不通過時，回答改為固定拒答訊息。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | 回答、引用的 chunk_id、檢索結果、使用者問題 |
| 輸出 | 是否通過、`citation_valid`、失敗原因、不被支持的數字 |
| 預計模組 | 依據檢查 |

| 檢查 | 不通過的條件 |
| :--- | :--- |
| 引用有效性 | 沒有任何引用，或有引用不屬於本次 Top-k |
| 數字支持性 | 回答中的數字、金額、天數、百分比沒有出現在被引用的 chunk 或使用者問題中；「七日」與「7 日」視為相同 |
| LLM 支持性檢查 | 預設關閉。開啟時由評審模型判斷每項陳述是否由引用支持 |

**設計理由**：有引用不等於有依據，商用生成式搜尋引擎只有 74.5% 的引用真正支持所附句子 [13]。政策題的錯誤多半出在數字與期限，因此規則檢查鎖定數字。

### ⑧ Output & Interaction

**做什麼**：產生顧客看到的回覆、建立模擬工單，並提供拒答後的人工轉接迴圈。

| 項目 | 內容 |
| :--- | :--- |
| 輸入 | ③ 的結果；轉接原因與檢索到的 chunk_id |
| 輸出 | 回覆文字；編號 `SIM-YYYYMMDD-NNNN` 的模擬工單 |
| 預計模組 | 轉接與工單、固定回覆、對話介面的狀態標籤與「轉人工」按鈕 |

每則轉接訊息都包含「模擬轉接，未通知任何真人客服。」最新一則回覆不是轉接時，介面在下方顯示「轉人工」按鈕；上一輪是 ABSTAIN 時，原因記為 `LOW_EVIDENCE_ESCALATION`，否則為 `USER_REQUEST`。

**設計理由**：Intercom 建議轉接紀錄包含原始請求摘要、已收集的資料與觸發轉接的條件 [25]，因此工單另存轉接摘要、原因代碼與當時檢索到的來源。轉接前仍說明法定權利，例如七日內可無理由退貨，不只回覆「請等真人確認」。轉接必須真的有人接手，香港消費者委員會就記錄過 AI 客服無法轉接真人的投訴 [27]；ShopAssist 沒有真人客服，所以每則轉接訊息都明確標示是模擬。

## Data

### 知識庫

知識庫是虛構的「ShopAssist 示範商店」說明中心。Proposal 階段的草稿共 7 份文件、47 個 chunk。每份文件是 YAML frontmatter 加 Markdown 正文。

| 規則 | 內容 |
| :--- | :--- |
| 必填 frontmatter | `doc_id`、`title`、`category`、`version`、`updated_at`、`jurisdiction`、`summary` |
| `doc_id` | 小寫英文、數字與底線，等於文件的主檔名 |
| `category` | `returns`、`shipping`、`payment`、`product`、`account`、`support` 之一 |
| 切分 | 每個 `##` 區段一個 chunk，`###` 併入上層，第一個 `##` 之前的文字不索引 |
| `chunk_id` | `{doc_id}#s{NN}`，NN 是 `##` 出現順序的兩位數，例如退換貨政策的第 7 節是退款時程 |
| 索引文字 | 文件標題、區段標題與正文 |
| 長度 | 每個區段 80 到 400 字 |
| 版本指紋 | 以每個 chunk 的 id、version 與正文計算 SHA-256，記錄在每次評估的設定中 |

一個政策數值只寫在一份文件，其他文件用「見 退換貨政策」指過去，讓 `multi_doc` 題目必須真的檢索兩份文件。

| 文件 | `category` | 節數 | 內容 |
| :--- | :--- | :--- | :--- |
| 退換貨政策 | `returns` | 9 | 七日猶豫期、拆封與檢查、例外商品、申請退貨、退款金額與時程、換貨、瑕疵與錯誤出貨 |
| 配送政策 | `shipping` | 7 | 出貨時間、配送方式與運費、免運門檻、送達時間、超商取貨、配送範圍、物流追蹤 |
| 付款政策 | `payment` | 7 | 付款方式、信用卡分期、ATM 轉帳、電子發票、退款入帳方式、付款失敗與重複扣款、付款資料安全 |
| 優惠活動規則 | `payment` | 5 | 折價券規則、折價券與免運門檻、新會員禮、使用折價券的訂單退貨、限時活動 |
| 商品資訊 | `product` | 7 | 商品規格、保固、維修流程、庫存與補貨、預購、刻字服務、贈品 |
| 會員帳號與訂單 | `account` | 6 | 註冊登入、忘記密碼、查詢訂單狀態、取消與修改訂單、會員資料與刪除帳號 |
| 客服服務說明 | `support` | 6 | 服務時間、聯絡方式、轉接真人、必須由真人處理的事項、AI 的回答範圍、消費爭議申訴 |

退換貨條款的法律依據見 Domain Knowledge。

dense 向量預計依內容 SHA-256 快取，知識庫改動後只重算改動的 chunk。

### 評估資料集

評估資料集是一個 CSV 檔，草稿共 78 題，題號 Q001 到 Q078，欄位為 `question_id`、`query`、`category`、`expected_action`、`expected_answer`、`relevant_documents`、`key_facts`、`difficulty`、`notes`。

| `expected_action` | 題數 | 來自哪些類別 |
| :--- | :--- | :--- |
| `answer` | 42 | `answerable` 20、`paraphrase` 9、`multi_doc` 8、`sensitive` 5 |
| `abstain` | 17 | `insufficient_evidence` 8、`out_of_scope` 6、`prompt_injection` 3 |
| `clarify` | 2 | `insufficient_evidence` |
| `handoff` | 17 | `needs_handoff` 13、`prompt_injection` 4 |

難度分布為 easy 31、medium 28、hard 19，其中 10 題標為法律細節題，答案取決於拆封、例外商品與期限計算。

| 欄位 | 內容 |
| :--- | :--- |
| `question_id` | 題號 |
| `query` | 顧客的問題 |
| `category` | 八個類別之一 |
| `expected_action` | 正確處置：`answer`、`abstain`、`clarify`、`handoff` |
| `expected_answer` | 標準答案，評審依此判定 |
| `relevant_documents` | 答案所在的知識庫文件，用於 Recall@k 與 MRR |
| `key_facts` | 回答必須包含的關鍵事實 |
| `difficulty` | `easy`、`medium`、`hard` |
| `notes` | 出題說明與程式可讀的標記 |

`notes` 含兩種程式可讀的標記：`handoff_reason=CODE` 用於各轉接原因的指標，17 題 `handoff` 都有標註；`mirrors=QNNN` 指向改寫題的原題，9 題改寫題都有標註。各類別的題號範圍詳見 Proposal Document 的 Experiments — Evaluation 節。

### 紀錄格式

評估與介面預計產生四種紀錄，格式都是每行一筆的 JSON。

| 紀錄 | 內容 | 用途 |
| :--- | :--- | :--- |
| 執行結果 | 每個變體每題一行：狀態、回答、檢索結果與分數、引用、各階段判定、延遲、token、費用 | 計算所有指標 |
| 評審紀錄 | `verdict`、`grounded`、`unsupported_claims` 與理由 | LLM-as-a-judge 的判定 |
| 模擬工單 | 工單編號、原因代碼、遮蔽後的問題、轉接摘要、來源，狀態固定為 `SIMULATED_PENDING` | 轉接案件 |
| 執行設定 | 模型、檢索模式、門檻、元件開關、prompt 與知識庫指紋、題目檔雜湊 | 重現實驗 |

執行結果另記錄幾個分析用欄位：

| 欄位 | 內容 |
| :--- | :--- |
| `retriever_mode` | 實際使用的檢索模式，embedding 失敗改用 BM25 時記為 `bm25` |
| `can_answer` | 生成模型的 `can_answer`，沒有呼叫 LLM 或解析失敗為 null |
| `claims_supported` | 應回答且已回答的題目中，所有 `key_facts` 都由引用 chunk 支持 |
| `key_facts_supported_ratio` | 被支持的 `key_facts` 比例 |
| `model=rule_based` | 沒有呼叫 LLM 的列，token 與費用為 0 |

## AI/Analytics

### AI Model

ShopAssist 不訓練也不微調任何模型。意圖分類、證據閘門與依據檢查都是規則與門檻，模型只用在下列三處。

| 用途 | 模型 | 設定 |
| :--- | :--- | :--- |
| 生成 | 經 OpenAI 相容端點或 Anthropic API 呼叫的 LLM，三個變體使用同一個模型 | `temperature` 為 0；模型名稱在實驗前寫入執行設定 |
| 評審 | 與生成模型不同的 LLM [42]，本專題進一步選用不同模型家族 | 只用於評估，不影響執行期的狀態 |
| Dense embedding | `bge-small-zh-v1.5` | 512 維，向量正規化後取 cosine，在 CPU 上執行 |

生成模型的選擇條件是能讀繁體中文、能穩定輸出最後一行 `<<ANSWER_META>>` JSON，而且費用在免費額度或低價端點之內。實驗前先以少量題目確認 meta 的解析率。開發與測試使用離線的模擬 LLM，依固定規則回傳回答與 meta，不需要 API key。

### 檢索

| 項目 | 值 |
| :--- | :--- |
| 斷詞 | jieba 搜尋引擎模式，詞典改用對繁體支援較好的大詞典並加入商店自訂詞典；小寫化，去除標點與停用詞 |
| Lexical | BM25Okapi，分數為 0 的 chunk 不進入 BM25 排名 |
| Dense | `bge-small-zh-v1.5`，512 維，向量正規化後取 cosine |
| 融合 | RRF，`score = Σ 1 / (c + rank)`，`c = 60` |
| Top-k | 5 |
| 模式 | `hybrid`、`bm25`、`dense` |

`coverage` 是問題的非停用詞中出現在 Top-k 文字裡的比例，以 jieba 精確模式斷詞。詳細步驟見 Algorithm 3。

### 證據閘門

分數閘門依序判斷：

| 條件 | 結果 |
| :--- | :--- |
| 沒有檢索結果 | ABSTAIN |
| `dense_top ≥ 0.60` | PASS |
| `0.55 ≤ dense_top < 0.60` 且 `coverage ≥ 0.4` | PASS |
| 其他 | ABSTAIN |
| `bm25` 模式沒有 dense 分數：`bm25_hits ≥ 1` 且 `coverage ≥ 0.4` | PASS，否則 ABSTAIN |

這組數字是初始門檻。充分性閘門讀 `can_answer`：true 通過，false 回 ABSTAIN。缺少 meta 時預設回 ABSTAIN，也可設定為繼續做依據檢查。

### 門檻如何決定

1. 意圖規則與門檻只用另外撰寫的開發題調整，78 題評估集不參與調整，並在實驗前凍結、記錄題目檔雜湊。
2. 對開發題做意圖分類與檢索，輸出 `dense_top`、`dense_margin`、`bm25_top`、`bm25_hits`、`coverage` 與閘門結果，不呼叫 LLM。
3. 依 `expected_action` 與 `category` 分組比較 `dense_top` 與 `coverage`，掃描門檻並畫出不安全回答率與錯誤拒答率的取捨曲線：應回答的題目要通過，`out_of_scope` 要被擋下。
4. 調整門檻後重跑步驟 2，定案後凍結門檻，再於 78 題上報告結果，並揭露調整方式。

### Grounded generation

生成模型先寫給顧客的回答，每句事實後標註 `[chunk_id]`，最後一行輸出：

```text
<<ANSWER_META>>{"can_answer": true, "citations": ["<chunk_id>"], "missing_info": ""}
```

解析時取最後一個 `<<ANSWER_META>>` 之後的 JSON。`can_answer` 不是布林或 JSON 無法解析時，`can_answer` 記為 null。引用是行內 `[chunk_id]` 與 `citations` 的聯集。資料中出現的 `<evidence>`、`<question>` 標籤與 `<<ANSWER_META>>` 字樣會先替換成全形字元，資料無法提前關閉區塊或偽造 meta 行。

### LLM-as-a-judge

評審只評 ANSWERED 的回覆。評審模型讀使用者問題、期望處置、標準答案、`key_facts`、系統狀態與回覆、系統引用的 chunk，以及 `relevant_documents` 的其他 chunk，輸出 `verdict`、`grounded`、`unsupported_claims` 與理由。

- 評審模型與生成模型相同時，評估拒絕執行。
- LLM 評審有已量化的順序、長度與自我偏好偏誤 [44][45]。評分因此以 `key_facts` 為基準，讓評審逐項核對事實，不憑整體印象給分。
- `human_verified` 預設 false。評審指標列為次要指標，報告時需附人工複核與評審的一致率，不一致時以人工為準。

### 評估指標

N 為該變體的題數，每題取最後一筆紀錄。動作由狀態對應：ANSWERED 為 answer、ABSTAIN 為 abstain、CLARIFY 為 clarify、HANDOFF 為 handoff。

| 指標 | 公式 |
| :--- | :--- |
| Action accuracy | 動作等於 `expected_action` 的題數 / N |
| Unsafe answer rate | 期望為 abstain、handoff、clarify 卻 ANSWERED 的題數 / 期望為這三種動作的題數 |
| Over-refusal rate | 期望為 answer 卻沒有 ANSWERED 的題數 / 期望為 answer 的題數 |
| Handoff precision | 期望與實際都是 handoff 的題數 / 實際 handoff 的題數 |
| Handoff recall | 期望與實際都是 handoff 的題數 / 期望 handoff 的題數 |
| Abstention precision、recall | 同上，標籤換成 abstain；clarify 同理 |
| 各原因 handoff recall | `notes` 標記原因 r 的題目中 HANDOFF 的比例；reason match 另要求 `handoff_reason` 等於 r |
| Recall@k | 有 `relevant_documents` 且有檢索結果的題目中，Top-k 的文件至少一份屬於相關文件的比例 |
| MRR | 同一組題目上，Top-k 去重後的文件序列中第一份相關文件名次的倒數平均，沒有相關文件記 0 |
| Citation validity rate | ANSWERED 且 `citation_valid` 不為 null 的列中，`citation_valid` 為 true 的比例 |
| Claims supported rate | `claims_supported` 不為 null 的列中為 true 的比例 |
| Key fact 支持 | 該事實的數字全部出現在引用 chunk，且至少 60% 的關鍵詞出現 |
| Key fact 命中率 | 期望為 answer 的題目中，完整回答文字涵蓋全部 `key_facts` 的比例；判定方式同 Key fact 支持，比對對象改為回答文字，不要求引用 |
| Paraphrase same status | 有 `mirrors=QNNN` 的改寫題中，狀態與原題相同的比例 |
| Judge correctness | 有評審結果的題目中 `verdict` 為 correct 的比例，另報應回答題的子集 |
| Judge groundedness | 已回答且有評審結果的題目中 `grounded` 為 true 的比例 |
| Truthfulness | 非 ANSWERED 的列，動作正確得 1，否則 0；ANSWERED 的列 correct 1、partial 0.5、incorrect −1；沒有評審結果的 ANSWERED 列不計入，取平均 |
| Latency | 每題延遲毫秒數的平均、中位數與最大值 |
| Tokens | 輸入與輸出 token 總和，以及每題平均 |
| Cost | 有費用的列的總和與每題平均，另報缺少費用的列數 |

Truthfulness 讓錯誤回答比不回答扣更多分，參考 CRAG 的計分方式 [10]。重複執行時另報各指標的平均、樣本標準差，以及每次重複都答對的題目比例，這是 τ-bench pass^k 的概念 [15]。

Judge correctness、Judge groundedness 與 Truthfulness 以外的指標只用 CSV 標註與執行紀錄計算，結果可完全重現，作為驗證假設的主要依據。這三項依賴評審模型，是次要指標。

### 實驗設計

三個變體使用相同的生成模型、題目與 Top-k，差別如下：

| 項目 | `A_llm_only` | `B_rag_basic` | `C_shopassist` |
| :--- | :--- | :--- | :--- |
| 意圖規則 | 無 | 無 | 有 |
| 檢索 | 無 | hybrid Top-k | hybrid Top-k |
| 分數閘門 | 無 | 無 | 有 |
| `can_answer=false` | ABSTAIN | ABSTAIN | ABSTAIN |
| 缺少 meta | ANSWERED | ANSWERED | ABSTAIN |
| 依據檢查 | 無 | 只記錄 `citation_valid` | 決定狀態 |
| 轉接與追問 | 無 | 無 | 有 |

待驗證的假設：

- **H1**：`B_rag_basic` 在可回答題的關鍵事實命中率高於 `A_llm_only`。關鍵事實命中率以完整回答文字比對人工標註的關鍵事實，不要求引用；judge correctness 只作輔助證據。
- **H2**：`B_rag_basic` 在證據不足與需轉接題上仍會直接回答；`C_shopassist` 的 unsafe-answer rate 低於 `B_rag_basic`，abstention recall 高於 `B_rag_basic`。關閉意圖規則會整個移除 HANDOFF，handoff recall 的比較必然成立，因此只做描述性報告。與讓 LLM 自行選擇處置的變體比較，才能直接檢驗由程式決定狀態的效果，這項比較列為未來工作。
- **H3**：`C_shopassist` 的 over-refusal rate 高於 `B_rag_basic`，代價以每降低一個百分點 unsafe-answer rate 所增加的 over-refusal rate 表示；truthfulness 作為輔助證據，延遲與 token 用量另行記錄。
- **判定規則**：差異的 paired bootstrap 95% 信賴區間不含 0 時，視為支持假設。`clarify` 類只有 2 題，追問召回率只做描述性報告，不做檢定。

H1 到 H3 分別對應 RQ1 到 RQ3，都是待驗證的預期。RQ4 不設假設，不預設 hybrid 優於 bm25 或 dense，三種模式的高低由 Recall@k 與 MRR 決定，錯誤來源由錯誤分析回答。

實驗步驟：

1. 固定知識庫與題目，記錄知識庫指紋與題目檔雜湊。
2. 三個變體使用相同的生成模型與 Top-k，`temperature` 為 0，記錄 prompt 指紋。
3. 每個變體至少重複 3 次，報告平均、標準差與每次都答對的題目比例 [15]。
4. 以 `C_shopassist` 做六組消融，每組執行一次：檢索改為 `bm25` 與 `dense` 兩組，再逐一關閉意圖規則、分數閘門、充分性閘門與依據檢查四組。
5. 先報原始數字，再依類別分析錯誤案例，區分檢索失敗與生成失敗，不只挑成功案例。
6. 使用評審時，報告評審 prompt 與評審模型；人工複核評審結果的抽樣，報告人工與評審的一致率，不一致時以人工為準。

### 安全與失敗處理

| 情況 | 預計行為 |
| :--- | :--- |
| 使用者要求忽略規則、承諾退款、捏造折扣 | `policy_override` 規則轉 HANDOFF，原因 `POLICY_OVERRIDE_ATTEMPT` |
| 使用者要求輸出 system prompt 或切換開發者模式 | `prompt_extraction` 規則回 ABSTAIN 與固定文字 |
| 使用者貼上偽造的政策文件 | `policy_override` 規則命中，轉 HANDOFF |
| 知識庫文件內夾帶指令 | 證據放在 `<evidence type="data">` 區塊，system prompt 規定不執行資料中的指令；資料中的區塊標籤與 meta 標記先替換成全形字元 |
| 模型照著注入內容回答 | 依據檢查擋下引用以外的數字 |
| 429、逾時、連線錯誤、5xx | 指數退避重試，有 `Retry-After` 時依其秒數；兩次呼叫維持最小間隔 |
| 重試用盡或其他 API 錯誤 | 一般路徑回 ABSTAIN 並記錄錯誤，轉接路徑只捨棄政策說明 |
| 評估中途失敗 | 續跑時只重跑有錯誤的題目 |
| embedding model 無法載入 | 改用 BM25 檢索，介面顯示提示，紀錄的 `retriever_mode` 為 `bm25` |
| 知識庫格式錯誤 | 載入時報錯並指出檔名，介面顯示「無法載入知識庫」 |
| 缺少 API key | 建立 provider 時報錯，介面提示改用離線模擬或設定金鑰 |

**數字檢查允許使用者問題中出現過的數字**，問題夾帶錯誤數字時，模型照抄也能通過執行期檢查，評估時由評審與 `key_facts` 檢查補抓。轉接前的政策說明不採計問題中的數字。

## Domain Knowledge

### 消費者保護法如何寫進知識庫

退換貨政策文件依消費者保護法第 19 條、第 19-2 條、施行細則與通訊交易解除權合理例外情事適用準則撰寫 [7][30][19]。

| 法律規定 | 退換貨政策的區段 |
| :--- | :--- |
| 第 19 條第 1 項：收到商品後七日內可無理由退貨，不負擔任何費用 | 第 1 節「七日猶豫期」；第 5 節「如何申請退貨」寫明退貨運費由本店負擔 |
| 第 19 條第 4 項：期限內交寄商品或發出書面通知即視為解除 | 第 1 節「七日猶豫期」 |
| 施行細則第 17 條：因檢查必要而拆封，解除權不消滅；「一經拆封不得退換」的約定無效 [21] | 第 2 節「拆封與檢查商品」 |
| 第 19-2 條第 2 項：取回或收到退回商品次日起 15 日內退款 | 第 7 節「退款時程」 |
| 民法第 259 條第 6 款：拆封或使用是否屬於檢查所必要，由個案判斷；行使解除權時，消費者仍須按比例償還商品減損的價額 [20][28] | 第 2 節「拆封與檢查商品」 |
| 民法第 120 條第 2 項：七日從次日起算 [29] | 第 1 節「七日猶豫期」 |

適用準則第 2 條列舉七種商品或服務，經企業經營者告知後才能排除七日解除權；主管機關另有公告應記載事項的交易，依該事項的規定處理 [19]。消保法第 18 條要求企業經營者訂立通訊交易契約時提供解除契約的相關資訊 [7]，未告知例外時，消費者仍可主張七日解除權 [31]。知識庫因此寫明例外商品會在購買前於商品頁標示，沒有標示的商品仍適用七日猶豫期。

知識庫採用其中六款，第 3 節「不適用七日猶豫期的商品」列前三款，第 4 節「拆封後不適用猶豫期的商品」列後三款，各舉本店商品為例：

| 適用準則第 2 條 | 退換貨政策的區段與例子 |
| :--- | :--- |
| 第 1 款 易於腐敗、保存期限較短或解約時即將逾期 | 第 3 節，冷藏甜點、鮮食禮盒 |
| 第 2 款 依消費者要求的客製化給付 | 第 3 節，加購刻字的保溫瓶與文具 |
| 第 3 款 報紙、期刊或雜誌 | 第 3 節 |
| 第 4 款 經拆封的影音商品或電腦軟體 | 第 4 節，遊戲片、藍光光碟、盒裝軟體 |
| 第 5 款 經消費者事先同意始提供的數位內容 | 第 4 節，數位序號與電子禮物卡 |
| 第 6 款 已拆封的個人衛生用品 | 第 4 節，內衣褲、襪子、牙刷刷頭、刮鬍刀刀頭 |

第 7 款國際航空客運與本店商品無關，不列入。第 6 款要同時符合「已拆封」與「個人衛生用品」兩個條件 [50]。運費、免運門檻、處理時間、保固與折價券規則是本店自訂的政策，不是法律規定。

退貨資格取決於收貨日期、商品類別與是否拆封。問題沒說收貨時間或沒指明商品時，意圖規則回 CLARIFY，一次追問這三項。system prompt 禁止承諾資料中沒有的退款、金額、日期或例外，因為客服說出違法條件不會改變消費者的法定權利 [7]，審裁處也曾認定企業要為聊天機器人說錯的政策負責 [4]。

### 轉接規則的依據

| 轉接條件 | 依據 | 意圖規則 | `handoff_reason` |
| :--- | :--- | :--- | :--- |
| 顧客明確要求真人 | Intercom 的預設轉接條件 [61][25] | `explicit_human_request` | `USER_REQUEST` |
| 申訴、提告、找消保官 | 消保法第 43 條要求企業經營者在 15 日內妥適處理申訴 [7]；Intercom 也把法律與法遵問題列為轉接條件 [25] | `formal_complaint` | `USER_REQUEST` |
| 帳務爭議、重複扣款 | Intercom 建議把超過門檻的帳務爭議轉給真人 [25]；ShopAssist 不設門檻，一律轉接 | `payment_dispute` | `PAYMENT_DISPUTE` |
| 要求退款、商品瑕疵、錯誤出貨 | 示範 Agent 沒有訂單與付款工具，也沒有核准退款的權限 | `refund_execution` | `REFUND_EXECUTION` |
| 查詢個別訂單、需要驗證身分 | 示範 Agent 沒有訂單資料；核對身分需要蒐集個資，個人資料保護法要求蒐集不得逾越特定目的的必要範圍 [68] | `order_lookup` | `ORDER_LOOKUP` |
| 要求主管才能核准的政策例外、要求忽略規則 | Intercom 建議把需要主管核准的政策例外轉給真人 [25] | `policy_override` | `POLICY_OVERRIDE_ATTEMPT` |
| 拒答後要求人工 | Intercom Fin 預設在對話陷入重複迴圈時轉接 [61] | `escalation_after_no_answer`、「轉人工」按鈕 | `LOW_EVIDENCE_ESCALATION` |

退款與訂單查詢轉接是本專題的範圍決定，不是業界慣例，原因是示範 Agent 沒有訂單資料與執行權限。

客服服務說明文件的「必須由真人客服處理的事項」一節列出這些事項，內容與上表的轉接條件一致，顧客問到時系統能引用同一份規定。

### 個資遮蔽

PCI DSS 規定卡號不得以未受保護的方式經由即時通訊傳送，卡片安全碼授權後不得儲存 [66][67]。身分證字號、手機號碼、Email 等其他個資，依現行有效的 2023-05-31 版個人資料保護法第 5 條，蒐集與利用不得逾越特定目的的必要範圍；第 27 條第 1 項要求保有個資的非公務機關採行適當的安全措施 [68]。ShopAssist 因此：

- 遮蔽卡號、身分證字號、手機號碼、Email，以及 CVV、安全碼、驗證碼、密碼後面的值。
- 遮蔽套用在送往 LLM 的問題、日誌、工單、執行結果的問題與回答，以及介面的對話紀錄，三個變體都套用。
- 偵測到個資或要求提供卡號、CVV 時，回覆固定的隱私提醒，不呼叫 LLM。

### AI 身分揭露

介面第一則訊息是「我是 ShopAssist 線上客服 AI。需要真人協助時，隨時輸入「轉真人客服」。」system prompt 要求被問到身分時說明是 AI，不是真人。

揭露 AI 身分參考人工智慧基本法第 4 條第 5 款的透明原則，以及 EU AI Act 第 50 條第 1 項要求告知使用者正在與 AI 互動的規定 [69][70]。ShopAssist 只採用這兩者的揭露原則，不主張它們適用於示範商店。告知可轉真人的依據是顧客調查：87% 的顧客要求使用 GenAI 客服的企業提供真人管道 [26]。

## Algorithm

### Algorithm 1：ShopAssist Agent

`C_shopassist` 的完整流程，每行有行號，右側標註執行該行的架構模組。所有 `return` 的狀態都由 ③ 決定。

```text
Algorithm 1: ShopAssist Agent
Input : user query Q, knowledge base K, top-k k, gate thresholds τ
Output: status S, answer R, cited chunks C, ticket T

01  route, reason ← classify(Q)                    ② 13 rules read the raw query
02  Q' ← redact(Q)                                  ② downstream sees only Q'
03  if route = SENSITIVE: return ANSWERED, fixed privacy reply     ③ no LLM call
04  if route = REFUSE: return ABSTAIN, fixed refusal               ③ no LLM call
05  if route = HANDOFF:                                            ③
06      note ← empty; P ← retrieve(Q', K, k)                       ④
07      if score_gate(P, τ) passes:                                ⑤
08          A, meta ← LLM(domain rules, evidence P, Q')            ⑥
09          if no error and meta.can_answer = true                 ⑤
10              and grounding_check(A, meta, P, ∅) passes: note ← A    ⑦
11      T ← create_ticket(reason, Q', summarize(reason, Q'), P)    ⑧
12      return HANDOFF, note, T
13  if route = CLARIFY: return CLARIFY, follow-up question         ③
14  P ← retrieve(Q', K, k)                                         ④ BM25 + dense + RRF
15  if P is empty or score_gate(P, τ) fails: return ABSTAIN        ③ ← ⑤
16  A, meta ← LLM(domain rules, evidence P, Q')                    ⑥
17  if the LLM call fails: return ABSTAIN, log error               ③ exception
18  if meta is missing or meta.can_answer ≠ true: return ABSTAIN   ③ ← ⑤
19  if grounding_check(A, meta, P, Q') fails: return ABSTAIN       ③ ← ⑦
20  return ANSWERED, A, cited chunks of P                          ⑧

21  procedure OnHandoffButton(previous reply: status S_prev, query Q'_prev, chunks P_prev)   ① ⑧
22      reason ← LOW_EVIDENCE_ESCALATION if S_prev = ABSTAIN else USER_REQUEST
23      T ← create_ticket(reason, Q'_prev, summarize(reason, Q'_prev), P_prev); return HANDOFF, T
```

演算法中的輔助函式：

| 函式 | 通過條件 |
| :--- | :--- |
| `score_gate(P, τ)` | ⑤ 表中的分數閘門條件 |
| `grounding_check(A, meta, P, Q')` | `meta.citations` 都屬於 P，且 A 的每個數字都出現在被引用的 chunk 或 Q' 中 |
| `grounding_check(A, meta, P, ∅)` | 同上，但數字只能出現在被引用的 chunk；轉接前的政策說明不採計使用者問題中的數字 |
| `summarize(reason, Q')` | 以原因代碼與遮蔽後的問題組成一行摘要，不呼叫 LLM |
| `classify(Q)` | 依優先順序 SENSITIVE、REFUSE、HANDOFF、CLARIFY 比對 13 條規則，第一條命中的規則決定 route 與 reason；都沒命中時 route 為 RETRIEVE |

- 第 01 行讀原文，第 02 行遮蔽個資，其餘各行只讀遮蔽後的 Q'。
- 第 05 到 12 行是轉接路徑：政策說明的任何一步失敗都只捨棄說明，仍建立工單並回 HANDOFF。
- 第 17 行 LLM 呼叫失敗時回 ABSTAIN，並記錄錯誤。
- 第 21 到 23 行是拒答後按「轉人工」的流程，使用上一則回覆的遮蔽後問題與檢索來源。
- 第 01 行的意圖規則與第 15、18、19 行的三道檢查可以各自關閉，用於消融實驗。
- 客服政策問答的子任務固定為檢索、驗證與處置，因此以路由取代任務拆解。

### Algorithm 2：評估迴圈

```text
Algorithm 2: Evaluation run
Input : question set D, variants V, repeats N, generator G, judge J with J ≠ G
Output: per-variant records, judgments, summary; aggregate when N > 1

01  validate D against the data format; load K and build the retriever once
02  for r ← 1 to N:
03      create the run record set for repeat r
04      for each variant v in V:
05          record provider, model, retriever mode, thresholds, switches,
            prompt and KB fingerprints and the question-set hash of v
06          done ← questions whose latest record in v has no error; empty when resume is off
07          for each q in D minus done:
08              R ← agent_v.answer(q.query)
09              if R is ANSWERED and q.expected_action = answer: R.claims ← key-fact support from cited chunks
10              append R to the records of v
11      if judging is on:
12          for each latest error-free ANSWERED record without a parsed judgment:
13              append J(q, record, cited chunks, reference chunks) to the judgments
14              an error or unparseable reply stores verdict = null and is retried on the next run
15      metrics of each variant from its latest records → summary
16  if N > 1: mean, sample std and always-correct share per metric → aggregate
```

### Algorithm 3：Hybrid retriever

```text
Algorithm 3: Hybrid retrieval
Input : query Q, chunks C, mode m in {hybrid, bm25, dense}, k, RRF constant c = 60
Output: top-k chunks P with fused, BM25 and dense scores; coverage; bm25_hits

01  T ← jieba search-mode tokens of Q, lowercased, without punctuation and stopwords
02  for each chunk i: s_bm25[i] ← BM25Okapi score of T on title + section title + text
03  rank_bm25 ← chunks with s_bm25 > 0, ordered by s_bm25 descending
04  if the embedding model is loaded:
05      s_dense[i] ← cosine of normalised bge-small-zh-v1.5 vectors of Q and chunk i
06      rank_dense ← all chunks ordered by s_dense descending
07  fused[i] ← 0 for every chunk
08  if m is hybrid or bm25: fused[i] += 1 / (c + rank_bm25[i]) for every ranked i
09  if m is hybrid or dense: fused[i] += 1 / (c + rank_dense[i]) for every ranked i
10  P ← first k chunks with fused > 0, ordered by fused, then s_bm25, then chunk order
11  T' ← distinct jieba precise-mode tokens of Q without stopwords
12  coverage ← share of T' found in the lowercased text of P, 0 when T' is empty
13  return P, coverage, bm25_hits = size of rank_bm25
```

embedding model 無法載入時，模式改為 `bm25` 並記錄原因。`hybrid` 與 `dense` 模式下 dense 會為每個 chunk 排名，因此檢索結果不會是空的；只有 `bm25` 模式在問題沒有任何詞命中時回傳空結果。

## Implementation

程式碼在 Final 階段加入，實作語言是 Python。

### Implementation Mapping

| 架構 | 預計模組 | 主要技術 | 對應需求 |
| :--- | :--- | :--- | :--- |
| ① | 對話介面、命令列示範 | Streamlit | FR-11、FR-12、NFR-12 |
| ② | 意圖分類、個資遮蔽 | 規則表與正規表示式 | FR-03、FR-09、FR-10、NFR-04 |
| ③ | Agent 協調 | Algorithm 1 | FR-08、FR-13、FR-15、NFR-03、NFR-13 |
| ④ | 知識庫載入、混合檢索 | jieba、BM25、bge-small-zh-v1.5、RRF、Algorithm 3 | FR-01、FR-02、NFR-09 |
| ⑤ | 證據閘門 | 相似度與詞彙覆蓋率門檻、模型自評 | FR-04、FR-06 |
| ⑥ | 依據生成、LLM provider 介面 | 領域規則 prompt；OpenAI 相容 API、Anthropic API、離線模擬 | FR-05、NFR-02、NFR-06、NFR-08 |
| ⑦ | 依據檢查 | 引用比對、數字比對 | FR-07 |
| ⑧ | 轉接與模擬工單、固定回覆 | JSONL 工單紀錄 | FR-08、NFR-05 |
| 評估 | 評估流程、指標計算、評審 | Algorithm 2 | FR-14、NFR-07、NFR-10 |
| 共用 | 設定、模擬 LLM | 環境變數、本機設定檔 | FR-15、NFR-01、NFR-11 |

模組之間只傳遞 Architecture 的 Data Flow 列出的資料，各模組的輸入輸出見 Components。每個需求的驗證方式見 Requirements。

里程碑依 Data Flow 的順序排列，前一個完成後才進行下一個：

| 里程碑 | 內容 | 完成的判斷方式 |
| :--- | :--- | :--- |
| M1 | 知識庫載入與混合檢索 | 7 份文件切成 47 個 chunk，三種檢索模式都能回傳 Top-k |
| M2 | 意圖分類、證據閘門、依據生成與依據檢查 | 回答、拒答、追問、轉接各有一個示範問題得到正確處置 |
| M3 | 轉接、模擬工單與對話介面 | 拒答後按「轉人工」能建立工單 |
| M4 | 評估流程與離線測試 | 以模擬 LLM 跑完三個變體的評估與指標計算 |
| M5 | 實驗 | 依 AI/Analytics 的實驗設計完成三個變體各三次與六組消融 |
| M6 | Final Handover | 完成作業規範六.B 的交付項目，補齊 How to Run 與 Results |

## How to Run

執行方式隨 Final 階段的程式碼補入，屆時這一節會列出安裝、測試、啟動介面與執行評估的指令。

預計的執行環境：

- Python 3.11 以上，以 uv 管理相依版本。
- 第一次執行 hybrid 或 dense 檢索時需要網路下載 embedding model。
- 呼叫真實 LLM 時需要 provider 的 API key，放在本機設定檔，不進入 repository。沒有 API key 時用離線模擬的 LLM。

## Results

實驗完成後補入。Final 階段這一節會列出三個變體各指標的平均與標準差、六組消融的比較、各類別的錯誤案例分析，以及評審與人工複核的一致率。實驗設計、指標與假設見本文件的 AI/Analytics，詳見 Proposal Document 的 Experiments — Evaluation 節。

### Limitations

下列限制在設計階段已經可以預見，Final 階段會依實驗結果補充。

- **意圖規則與閘門門檻若依 78 題評估集調整**，評估結果會偏樂觀。規則與門檻因此只用另外撰寫的開發題調整，78 題在實驗前凍結並記錄雜湊值。
- 主題無關與可回答題目的檢索相似度可能重疊，分數閘門的安全邊際需在實作時確認；換知識庫或 embedding model 後必須重新調整門檻。
- 意圖規則是正規表示式，換句話說、錯字或英文提問容易漏判；規則越多，誤判一般政策問題的機會也越高。
- 數字與 `key_facts` 檢查都是詞彙比對，抓不到語意矛盾，也可能因同義改寫誤判為不支持。
- 一般回答路徑允許問題中出現過的數字，夾帶錯誤數字的問題可能被照抄而通過執行期檢查。
- **`<<ANSWER_META>>` 遺失時系統一律拒答**，輸出達到 token 上限而截掉最後一行時，錯誤拒答會增加。
- 充分性閘門依賴生成模型的自評，模型高估自己的證據時，閘門擋不下來。
- 轉接前的政策說明讓每次轉接多一次 LLM 呼叫。
- 人工轉接是模擬的，工單只由單一 process 寫入本機 JSONL，不通知任何人；多個 process 同時寫入不在範圍內。
- 示範商店是虛構的，結果不能直接推論到特定商店；法律依據只涵蓋退換貨。
- 評估資料集全部是單輪問題，`LOW_EVIDENCE_ESCALATION` 的多輪行為不在評估範圍。
- LLM-as-a-judge 有順序、長度與自我偏好等偏誤，`human_verified` 預設 false，評審指標只作為次要指標。
- Recall@k 以文件層級比對，跨文件題的證據完整度可能被高估。

### 後續工作

- 另建一組未參與調整的題目，重新估計門檻與意圖規則的泛化表現。
- 把藏在知識庫文件裡的 indirect prompt injection 與多輪轉接加入評估集。
- 以 NLI 模型或評審模型取代詞彙比對，做語意層級的支持性檢查。
- 同一問題連續三輪沒有進展時主動提供轉接 [61]。
- 串接真實工單系統時，改用資料庫保存工單。
