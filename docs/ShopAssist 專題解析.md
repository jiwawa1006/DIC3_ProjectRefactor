根據《期末專題提案作業規範》的要求標準，以及《A2Z outline example.pdf》的 26 字母研究論文與提案架構，針對專案文件《ShopAssist\_Proposal\_Document.pdf》與簡報《ShopAssist\_Proposal\_PPT.pptx》進行系統性解析 1-5：

### 一、 依據《期末專題提案作業規範》核心維度檢視符合度

#### 1\. 核心定位符合度（AI Agent \+ 領域知識 \+ 數據分析 \+ 真實世界應用）

* **超越單一 Model 與單純 Chatbot**：專題規範明確要求專案必須具備特定專業能力的 Domain AI Agent，而非單純的模型訓練或 Chatbot 1, 5。  
* **ShopAssist 的定位與實踐** 3, 6-8：  
* **AI Agent**：以 Planner & Reasoning Agent（模組 ③）為控制中心，依據 Algorithm 1 進行五種意圖路由分流（SENSITIVE, REFUSE, HANDOFF, CLARIFY, RETRIEVE），並協調檢索、證據檢查與模擬工單建立機制 6, 7, 9-11。  
* **Domain Knowledge（領域知識）**：知識庫退換貨條款嚴格對照台灣《消費者保護法》第 19 條（七日無理由解除權）、第 19-2 條（15日內退款）、《通訊交易解除權合理例外情事適用準則》第 2 條（七種例外）、消保處函釋（「一經拆封不得退換」條款無效、個人衛生用品要件），並結合 PCI DSS 金融防護與個資法規 3, 9, 12-18。  
* **Data Analytics（數據分析）**：建構兩道證據閘門（生成前「分數閘門」：最高 dense 相似度 \\\\(\\ge 0.60\\\\) 或 \\\\(0.55 \\le \\text{dense} \< 0.60 \\land \\text{coverage} \\ge 0.4\\\\)；生成後「充分性閘門」：can\_answer 自評）與生成後「依據檢查」（引用有效性 \+ 阿拉伯數字/天數比對）8, 19-21。評估面規劃 78 題 13 項指標進行無評審模型干預的數據分析 19, 22-24。  
* **Real-World Application（真實世界應用）**：針對台灣網購售後服務常見爭議（淘寶售後佔客服聊天 56.5%）與企業須為 AI 客服編造政策負賠償責任（如 Moffatt v. Air Canada 判例判賠 812.02 加幣）之真實產業痛點 13, 25-28。

#### 2\. Abstract（摘要）與 Introduction（緒論）結構對照

* **Abstract 六段式（A–F）對照** 12, 19, 25, 29-31：  
* **A. Attention Getter**：引用資策會 MIC (87% 國人使用 AI 客服、36% 轉介真人失敗)、Salesforce 2025 年報告、淘寶對話數據及 Moffatt v. Air Canada 法院判例引題 4, 25, 28。  
* **B. But (挑戰)**：現有 RAG 無法完全消除幻覺（CRAG 準確率上限 44%）、強模型在 context 不足時反而更少拒答，且商用生成式搜尋引用支持度僅 51.5% 12, 29, 32。  
* **C. Cure (解藥)**：提出 ShopAssist——只依商店知識庫回答、證據不足拒答、缺少資訊追問、超出權限轉真人 19, 29。  
* **D. Development (方法)**：採用 13 條意圖規則 \+ BM25 與 dense (bge-small-zh-v1.5) \+ RRF 混合檢索 \+ 兩道證據閘門 \+ 依據檢查 9, 10, 19-21, 33。  
* **E. Experiments (評估)**：建構 78 題繁中電商評估集，對比 LLM-only、Basic RAG 與 ShopAssist 三種變體，並進行檢索與元件消融實驗 19, 23, 30, 34。  
* **F. Findings (預期產出)**：嚴格遵循規範「Proposal 階段僅列 Expected Outcomes，不虛構數據」30, 31, 35，預期降低不安全回答率，提高拒答/轉接召回率，並量化「降低不安全回答率所付出的錯誤拒答率代價」30, 35。  
* **Introduction 論證精神**：回答「Why is this project worth doing?」26, 36，條列售後問答責任歸屬、消保法底線等客觀證據，並指出既有研究缺口（UAEval4RAG、MTRAG-UN 缺乏轉接，τ-bench 不查依據）29, 37-40。

#### 3\. Related Work 與 Deep Research 三大問題

* **AI 技術與領域知識平衡（1/2 AI \+ 1/2 Domain）** 36, 41：  
* **AI 面**：包含客服 RAG (LinkedIn KG)、Groundedness 評估 (FActScore, ALCE, RAGAS)、拒答機制 (AbstentionBench, Sufficient Context)、Prompt Injection 防禦 (BIPIA, StruQ) 與 Agent 轉接 (τ-bench) 32, 42-45。  
* **Domain 面**：包含消保法第 19 條、19-2 條、合理例外準則、消保處函釋、PCI DSS FAQ 1085/1319、個資法、資策會 MIC 調查、Gartner 報告與 Air Canada 判例 16, 27, 45-48。  
* **Article Summary Table & Research Gap**：提供包含 26 筆文獻/法規之對照表 45，導出 Research Gap：**文獻中沒有研究在同一組題目上同時量測引用支持度、拒答、追問與轉接，亦缺乏以繁體中文電商政策及台灣消保法為基準的端到端評估** 29, 40。  
* **Deep Research 三大核心問題對照** 49, 50：  
* **Q1. 5 個核心思維模式**：1.客服承諾即企業承諾 2.消保法是底線 3.知識管理是基礎 4.Reliability over reach 5.衡量正確解決而非擋下多少 51-55。  
* **Q2. 3 個重要專業爭論**：爭論一（AI 取代 vs. 輔助真人）、爭論二（RAG 能否消弭幻覺）、爭論三（盡量回答 vs. 積極拒答與轉接）55-59。  
* **Q3. 10 個深度理解問題**：涵蓋 Why (相關仍錯答)、What-if (新舊版本政策衝突)、Trade-off (提高檢索門檻)、Failure Case (訂單查詢誘騙)、Evidence (引用支持證明)、Comparison (BM25 vs 向量)、Design Decision (不建 KG/不微調)、Real-world Scenario (商品損毀退款)、Evaluation (deflection 迷思) 與 Security (Indirect Prompt Injection 防禦) 60-67。

#### 4\. Proposed Scheme（系統架構與技術表示）

* **八大模組架構（完整對應規範 ①～⑧）** 7, 68, 69：  
* **① User/Application Scenario**：Streamlit 對話介面，揭露 AI 身分並提供「轉人工」按鈕 9, 70-72。  
* **② Query/Input Processing**：13 條規則意圖分流 (SENSITIVE, REFUSE, HANDOFF, CLARIFY, RETRIEVE)，並執行 PCI DSS 與個資法遮蔽 9, 18, 72。  
* **③ Planner & Reasoning Agent**：系統核心，依 Algorithm 1 決定最終 4 種處置狀態（ANSWERED, ABSTAIN, CLARIFY, HANDOFF），確保行為可追溯與可稽核 10, 11, 72。  
* **④ Data Acquisition/Retrieval**：7 份文件、47 個 chunk，採用 BM25 \+ bge-small-zh-v1.5 \+ RRF 混合檢索 Top-5 10, 20, 72。  
* **⑤ Analytics Engine**：兩道證據閘門（生成前「分數閘門」與生成後「充分性閘門」）8, 20, 33, 72。  
* **⑥ Domain Knowledge Integration**：消保法 Prompt 注入、boundary awareness 防範 Prompt Injection 21, 33, 72。  
* **⑦ Decision/Recommendation**：依據檢查（引用有效性與數字比對）17, 21, 72, 73。  
* **⑧ Output & Interaction**：生成最終回覆、轉接前政策說明，並建立帶有 6 種原因代碼之 JSONL 模擬工單 17, 72, 73。  
* **流程圖與虛擬碼** 11, 74-76：  
* **Flowchart**：提供圖 2 (意圖分流)、圖 3 (檢索與回答)、圖 4 (轉接與轉人工迴圈)，清楚標明 Decision, Exception, Loop 與 Human Interaction 節點 74, 75, 77, 78。  
* **Pseudocode**：提供帶有具體步驟與行號的演算法 Algorithm 1: ShopAssist Agent (01–23 行)，明確標註模組與消融實驗開關 11, 79-81。

### 二、 依據《A2Z SOP Outline》26 字母結構深度解析

字母,A2Z SOP 定義 2,"ShopAssist 具體落實與內容映射 11, 12, 19, 24, 25, 29, 30, 39, 45"  
A,Attention Getter：動機與引題,"引用資策會 MIC (87% 國人使用 AI 客服、36% 轉介真人失敗)、Salesforce 2025、淘寶 56.5% 售後對話及 Moffatt v. Air Canada 判賠 812.02 加幣 4, 25, 26, 28。"  
B,But：挑戰與現有不足,"指出現有 RAG 仍有 \>17% 幻覺率、CRAG 上 RAG 準確率僅 44%、強模型 context 不足時反而更少拒答、商用搜尋引用支持度僅 51.5% 12, 28, 29, 32。"  
C,Cure：提出的創新解藥,"提出 ShopAssist AI Agent：只依商店知識庫回答、證據不足拒答、缺少資訊追問、超出權限轉真人 4, 19, 29。"  
D,Development：方法設計依據,"整合 13 條意圖規則、BM25 \+ dense (bge-small-zh-v1.5) \+ RRF 混合檢索、雙重證據閘門與數字依據檢查 9, 10, 19-21, 33。"  
E,Experiments：評估規劃,"建構 78 題繁中電商評估集（涵蓋 8 大類別），對比 LLM-only、Basic RAG 與 ShopAssist 完整流程，重複至少 3 次 19, 23, 30, 34, 82。"  
F,Findings：預期成果（無虛構）,"嚴格遵守規範不虛構數據，預期降低不安全回答率、提高拒答/轉接召回率，並量化錯誤拒答率增長之代價 30, 31, 35。"  
G,General：領域大背景現況,"台灣 2025 年零售業網路銷售額達 6,716 億元，網路購物屬於消保法通訊交易範疇 13, 46。"  
H,However：深層技術與應用瓶頸,"客服答錯政策責任由商家承擔，且退貨規則涉及消保法第 19 條與合理例外準則，規則複雜且具模糊空間 12, 13, 25, 26。"  
I,In literature：文獻發展梳理,"從客服 RAG (LinkedIn KG)、Groundedness (FActScore, ALCE)、拒答 (AbstentionBench) 梳理至 agent 轉接 (τ-bench) 32, 42-45。"  
J,Judgement：各方法優劣點評析,"指出客服 RAG 只評檢索或輔助人員；拒答研究無轉接；τ-bench 有轉接卻不查依據；現有研究無端到端繁中法規評估 29, 37-40。"  
K,Keypoint：設計精神與核心貢獻,"處置由程式決定；引用有效與主張被支持分開報告；拒答/追問/轉接為一等行為；知識庫對照消保法撰寫 39, 83。"  
L,List organization：文章結構導覽,"明確說明章節安排：Abstract、Introduction、Related Work、Deep Research、Proposed Scheme、References 3, 39, 41。"  
M,Methods：既有文獻方法分類,"劃分為 AI 技術（RAG 檢索、Groundedness、拒答與 Prompt Injection 防禦）與 Domain 知識（消保法規、PCI DSS、客服營運與責任案例）兩大類 16, 27, 32, 41-48。"  
N,New proposed：本專案創新技術,"「程式強控處置狀態 \+ 消保法對照知識庫 \+ 雙重證據閘門 \+ 數值與引用依據核驗」之端到端電商 Agent 6, 19, 21, 29, 39。"  
O,Organize：文獻比較與歸納表,"提供包含 26 筆文獻/法規之對照表 (Article Summary Table)，並收斂導出 Research Gap 40, 45, 84-86。"  
PA,Aim：研究目標陳述,"打造「Reliability over reach」、具備來源依據檢查與人工轉接機制的電商購物客服 AI Agent 3, 6, 53。"  
PB,Based on：依託基礎與背景,"依託消保法第 19 條、適用準則第 2 條、消保處函釋、PCI DSS 安全規範與 RRF 排名融合演算法 3, 10, 13-16, 27。"  
PC,Cure / Cause：機制有效的原因,"透過程式規則與兩道閘門過濾無效檢索與缺乏自評，取代 LLM 自由處置決策，消除政策編造 6, 19, 20, 29。"  
PD,Design：系統架構設計,"典範轉移：從「由 LLM 自由決定是否拒答/轉接」轉變為「程式強控處置狀態 (ANSWERED, ABSTAIN, CLARIFY, HANDOFF) \+ LLM 限域生成」6, 10。"  
PE,Paper Element：模組元件清單,"嚴格拆解 ①～⑧ 八大模組（UI、Input Processing、Planner Agent、Retrieval、Analytics Engine、Domain Integration、Decision Check、Output & Ticket）7, 68, 71, 72。"  
PF,Formulation：功能與數學形式,"定義分數閘門公式（\\\\(\\text{dense\\\_top} \\ge 0.60\\\\) 或 \\\\(0.55 \\le \\text{dense} \< 0.60 \\land \\text{coverage} \\ge 0.4\\\\)）、RRF 名次融合公式與去標識化規則 9, 10, 20, 21。"  
PG,Graph：架構圖與流程圖,"提供完整系統架構圖 (圖 1\) 與 3 張跨階段 Flowchart (圖 2 意圖分流、圖 3 檢索回答、圖 4 轉接與轉人工迴圈) 68, 74, 75, 77。"  
PH,How：如何達成設計目標,"透過 13 條優先意圖規則、BM25 \+ dense 混合檢索、雙重證據閘門及數字與引用核驗達成 9, 10, 20, 21, 33, 72, 73。"  
PI,Implementation：虛擬碼落實,"提供帶行號的 Algorithm 1: ShopAssist Agent（01–23 行），明確標註模組與消融實驗開關 11, 79-81。"  
PJ,Jump to an example：情境範例展示,"提供 4 類經典範例場景（政策查詢→回答、細節未寫→拒答、缺少資訊→追問、訂單/退款權限→轉接模擬工單）34, 70。"  
PK,Key points contribution：核心貢獻總結,"條列 5 大貢獻：程式決定處置 Agent、消保法知識庫、78 題繁中評估集、無評審模型評估流程、對照與消融實驗 39, 83。"  
PL,Later simulation：後續模擬評估規劃,"規劃 78 題測試集、3 個變體（A\_llm\_only, B\_rag\_basic, C\_shopassist）各自重複 3 次之實驗與 95% paired bootstrap 檢定 19, 30, 34, 35, 87。"  
PM,Math proof / Logic：邏輯檢驗,"嚴謹推導不安全回答率與錯誤拒答率之對立取捨關係、Truthfulness 扣分邏輯及 paired bootstrap 信賴區間檢定邏輯 22, 24, 35, 87。"  
Q,Quality & Quantity：指標與評估量,"處置正確率、不安全回答率、錯誤拒答率、拒答/轉接 P\&R、Recall@k、MRR、引用有效率、關鍵事實支持率、改寫一致率、延遲與 token 量 22, 24, 87。"  
R,Related method to compare：比較基準,"選定 A\_llm\_only (純 LLM) 與 B\_rag\_basic (基本 RAG) 為 Baseline 比較基準 30, 34。"  
S,Simulation Setup：實驗環境設定,"7 份文件/47 個 chunk 知識庫、78 題 8 類評估集，於 CPU \+ Python 3 (jieba, BAAI/bge-small-zh-v1.5) 與 API 運作 10, 23, 82, 88, 89。"  
T,Tuning parameters：調優參數,"檢索 Top-k (\\\\(k=5\\\\))、分數閘門門檻 (\\\\(\\tau\_{\\text{dense}}=0.60, 0.55\\\\), \\\\(\\tau\_{\\text{coverage}}=0.4\\\\))、RRF 常數 (\\\\(k=60\\\\)) 10, 20。"  
U,Useful results：預期有用產出,"產生處置狀態標籤、來源引用、追問指引、6 種原因代碼之 JSONL 模擬工單及無評審模型的客觀評估報告 9, 17, 24, 73。"  
V,Verify：有效性驗證,"驗證 C\_shopassist 是否在不安全回答率顯著低於 Basic RAG 的同時，維持合理的錯誤拒答率代價 (H1, H2, H3 假設驗證) 35, 41。"  
W,What proposed：回顧總結,"總結本專案為解決電商 AI 客服編造政策責任與轉接失敗問題之 ShopAssist 領域代理人 6, 25, 29。"  
X,eXcel：專精情境與優勢,"專精於「台灣網購退換貨法規（消保法 §19）」與「可靠度優先於涵蓋度（Reliability over reach）」之客服場景 13-16, 53。"  
Y,Yields：預期最終交付物,"交付物包含 Proposal Document、PPT、GitHub Repo、DESIGN.md、78 題評估集與完整實驗對照數據 39, 83, 90, 91。"  
Z,Zen：反思與未來展望,"反思單輪評估之局限，未來展望規劃多輪對話動態轉接驗證、LLM 處置決策變體對比與真實客服人機協同實測 22, 35, 92。"

### 三、 綜合評審亮點與精進建議

1. **結構完備度極高**：  
2. 在提案文件與 PPT 中，均嚴格遵守了 A–F 六段式摘要 4, 12, 19, 25, 29, 30、八大模組架構圖與三張流程圖 68, 71, 74, 75, 77、帶行號演算法 Algorithm 1 11, 80，以及 Deep Research（5 個思維 \+ 3 大爭論 \+ 10 個深度問題）49, 51-67，整體架構與評分規範完全對齊 5, 31, 36, 50, 69, 76, 90, 91, 93, 94。  
3. **學術誠實度（Academic Integrity）**：  
4. 在 Abstract 與 Expected Outcomes 中清楚註明 Proposal 階段僅陳述待驗證的預期成果（Expected Outcomes）30, 35，未捏造任何虛假的實驗數據，完全符合提案階段的學術規範 31。  
5. **後續實作（W1–W8）重點精進建議**：  
6. **門檻動態校準（Threshold Calibration）**：實作階段應使用獨立於 78 題測試集以外的開發題（Development set）校準分數閘門門檻 (\\\\(\\tau\_{\\text{dense}}, \\tau\_{\\text{coverage}}\\\\)) 33, 95，避免產生過擬合現象。  
7. **DESIGN.md 同步與維護**：依照規範第六節之工程交接標準 91, 93，應將 Algorithm 1\~3、JSONL 工單格式、API 規格及變體設定完整寫入 GitHub 的 DESIGN.md 中 6, 11, 88。  
8. **繁體中文斷詞典優化**：jieba 應明確加載 dict.txt.big 繁體大詞典並補充示範商店自訂詞典 82, 95，確保 BM25 詞彙檢索能精準命中繁體中文電商術語（如「猶豫期」、「鑑賞期」）10, 64。

💡 需要針對 ShopAssist 提案中的哪一部分（例如八大模組資料流、78 題評估集細節或演算法偽程式碼）進一步產出專題簡報備忘錄或執行規劃報告嗎？  
