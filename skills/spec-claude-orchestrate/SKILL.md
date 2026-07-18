---
name: spec-claude-orchestrate
description: >-
  Run spec-driven development entirely inside Claude Code, autonomously from propose to commit —
  Claude is both the global commander (propose, every review gate, runtime verification, archive,
  selective commit; no approval waits, stops only on real blockers) and the executor (small
  changes on the main thread; big multi-phase changes via ONE executor subagent per change,
  continued with SendMessage). Use this WHENEVER taking a feature through the spec pipeline or
  continuing one mid-flight in a repo with spectra or openspec, and the user wants Claude to
  handle it all. Typical asks: 走 spec 流程做某個功能、你全權處理這個 change 一路做到 commit、用
  spectra/openspec 開一個 change 自己實作自己驗、繼續上一個 change、archive 並 commit。Do NOT
  use for one-off questions, plain edits, CLI/config troubleshooting, 一般 code review、潤飾文字、
  跑測試, or anything that doesn't touch a spec.
compatibility: Requires a spec tool (spectra CLI or openspec) initialized in the repo.
---

# Spec-Claude Orchestrate

把 spec-driven 開發完整跑在 **Claude Code 單引擎**內：你（Claude 主線）當**全局指揮**，持有 change 的全局狀態與**所有決策權**；實作依規模走**主線直做**或**executor subagent**（混合制）。

## 核心原則

**依工作性質分工**——判準：需要**全局視野或下判斷**的，主線親自來；把**既定指示落地執行**的，依規模分流（步驟 4）。

- **決策／思考／驗收 → 主線親自做**（或另開唯讀 subagent 輔助）：`propose`、`discuss`、`ask`、`explore`、`verify`、`audit`、`analyze`、`drift`、`archive` 把關、`commit`
- **實作／執行 → 混合制**：小 change 主線直做；大 change 交 executor subagent，主線驗收

**Grounding 準則**：任何判斷盡量依**實際程式碼**判讀、或查**官方文件／教學**求證——不臆測、不亂猜。**驗收同理**：看實際行為的實跑證據，**不把「測試綠燈／TDD 過」當通過**（測試多半是執行者自己寫的，同一個誤解會同時進實作與測試）。唯有真的 ground 不了（落入下方**自治停止條件**）才停下來問使用者，其餘一律自主往下。

**角色紀律**：即使執行者也是 Claude，「指揮」與「執行」的判斷不混用——review gate 一律回到主線以實際程式碼下判斷，**不因為是自己（或自家 subagent）寫的就放水**。

## 自治與停止條件

觸發本 skill 即視為對 propose → apply → review → archive → selective commit 的**預先授權**：proposal、apply、archive、commit 一律**不等使用者核准**，預設從 propose 自動執行到 commit。過程持續回報簡短進度，但**進度回報不是核准請求**。

只在下列情況停下來等使用者輸入：

1. 需求歧義重大，從 artifacts、實際程式碼或最小驗證都無法安全定案；
2. 多個既有 change 同時符合要求，語境不足以安全選擇；
3. 執行需要新權限、外部 credential、scope 擴張，或會動到 scope 外資料；
4. blocker 經修正嘗試後仍無法排除；
5. 使用者明確要求「只做 proposal」「先停在 proposal」或排除 archive／commit。

## 步驟 1 — 偵測 spec 工具（spectra 優先、openspec fallback）

開工前先判斷用哪一套，因為它決定 review gate 怎麼做：

1. `spectra` CLI 可用（`command -v spectra`）**且**專案有 `.spectra.yaml` → **spectra 模式**（可用 `spectra-verify`/`spectra-audit`/`spectra-drift` 驗收）
2. 否則專案有 `openspec/` → **openspec 模式**（review gate 依程式碼手動審）
3. 兩者皆無 → 停下，請使用者先初始化 spec 環境

> 註：文中 `spectra-propose`/`spectra-discuss`/`spectra-verify` 等是 `spectra init/update --tools claude` 產生在**專案** `.claude/skills/` 的 project-level skills，不是 CLI 子命令。若專案裡沒有，先跑 `spectra update` 補齊；補不了就依該階段目的手動操作（CLI 有 `validate`/`analyze`/`drift` 可輔助）。

兩套共用 openspec 資料格式，但 **CLI 狀態不互通**：**同一個 change 從頭到尾固定用同一套工具**。

> 註：`spectra` CLI 沒有指定目錄的 flag，所有 `spectra` 命令都要在 **spec 專案根 cwd** 執行。

## 步驟 2 — spec 分流

- **不需動 spec 的小事** → 直接做（研究 → 實作 → commit），**不進 pipeline**。
- **需要動 spec** → 進入下面的 change 編排（步驟 3 起）。

## 步驟 3 — propose（主線親自；定案後不等核准直接往下）

1. **只在必要時釐清**：需求有影響方向的重大歧義（自治停止條件 1）才向使用者提問（spectra 模式可用 `spectra-discuss`）；否則以最合理解讀**自行定案**，把採用的假設與 out-of-scope 寫進 proposal。
2. **產出 artifacts**：產 proposal／specs／design／tasks（spectra 模式 `spectra-propose`，產完會 `spectra park`），並跑該模式的 validate／analyze 收斂到無 Critical／Warning。
3. **回報即前進**：顯示 change 名稱、proposal 摘要與 tasks——這是**進度回報，不是核准請求**，回報後**立即進入 apply**，不等使用者回覆。spectra 模式由主線立刻 unpark 接續；只有使用者明確要求 proposal-only 才停在 parked。

propose 是全局決策，**不外包給 executor**。

## 步驟 4 — 執行（混合制：依規模分流）

**分流判準**（heuristic，非硬規則）：

- **主線直做**：單一任務、動少數檔案（約 ≤5 個）、預期一輪完成、且主線 context 還輕。省 token、使用者可即時盯。
- **開 executor subagent**：tasks.md 有多任務／多階段、預期多輪 review-修改、或主線 context 已重（大實作別擠在 context 後段做）。

**executor 模式的操作**：

- **change 第一次派工**：用 Agent tool 開一隻 **general-purpose** executor。指示帶上：change 名稱、本輪 tasks 範圍、spec artifacts 路徑、輸出契約（做了什麼／改了哪些檔／自測結果／遇到的問題；自測結果要附實際執行的命令與輸出，不只測試綠燈）。**專案可在本機運行時，executor 要實際啟動專案走過本輪關鍵路徑再回報完成，unit test 綠燈不足以宣告完成；客觀無法運行則回報具體阻礙＋最接近的替代驗證＋尚未驗證的行為**（詳見步驟 5 的實跑完成契約）。
- **同一 change 後續階段／要求修改**：用 **SendMessage 續同一隻** executor，context 完整保留。
- **一個 change = 一隻 executor**。executor 的回報只是輸入，驗收永遠在主線（步驟 5）。
- executor 在背景跑時主線**別動同一批檔案**（避免互相踩）；不同 change 可各自一隻 executor 並行，SendMessage 指名續不會混。
- 派工指示**依複雜度伸縮**：簡單任務精簡帶過；複雜任務給明確 task ＋ 輸出契約 ＋ verification／grounding 要求。
- **model／effort 不必指定**：executor 未 override 時**沿用主線 session 的模型與 effort**；想省 token 可在派工時降級 `model`（sonnet／haiku）。（agent 定義檔若綁固定 `model:` 會優先於繼承；`~/.claude/agents/` 現況統一 `model: inherit`。）
- 主線做到一半發現比預期大 → 可中途升級：把目前進度寫回 tasks.md，再開 executor 接手。

**executor 換手與跨 session**：subagent 不能跨 session 續（session 結束就消失）。**耐久狀態一律以 spec artifacts 為準**——要求 executor **每完成一個任務就同步勾 tasks.md**（spectra 模式優先用 `spectra task done`，會一併記錄 touched files），並善用 `spectra park`／`in-progress` 標記。新 session 續 change、executor 中途掛掉、或同一隻續太多輪出現 context 疲勞 → 一律開**新** executor、靠 artifacts ground，不靠記憶。

## 步驟 5 — 每階段 review gate（主線親自）

executor（或主線自己）完成一輪後，由**主線**驗收：

- **spectra 模式**：另開一隻 Explore subagent 執行 `spectra-verify`／`spectra-audit`／`spectra-drift`（report-only；派工 prompt 明講：讀專案 `.claude/skills/` 對應 skill 照做，或跑 CLI `validate`／`analyze`／`drift`），再依其報告 ＋ 實際程式碼下最終判斷。
- **openspec 模式**：直接依實際程式碼／diff 判讀（grounding 準則）。
- **主線直做的 change**：review 前至少另開一隻乾淨視角 subagent（Explore 或 code-reviewer）審過再判斷，降低自我審查偏誤。

> 乾淨視角一律用**全新 context** 的 subagent，勿用 fork 型 subagent——fork 會繼承主線 context，失去獨立審查的意義。

### Apply 實跑完成契約（硬條件）

驗收不只看 review 報告，還要看**實跑證據**——這是「apply 是否完成」的硬門檻，不是延到 archive 才做：

- 專案**具備可用的本機啟動／執行方式、且不需新權限或破壞性外部操作**時，**必須實際啟動專案並走過本輪 change 的關鍵使用路徑**：CLI 就實跑命令看輸入／輸出／失敗路徑、API 就起服務打關鍵 endpoint、UI 就跑 e2e 或實走主要流程看可見狀態。
- **只有實際啟動服務／CLI／UI 觀察到的行為才算 runtime evidence**；unit test、mock test、typecheck、lint、build 成功、reviewer 報告都**只能當輔助證據**。
- **能運行卻沒運行 → 不得宣告本輪 apply 完成、不得進 archive**。
- 因缺 external service／credential／硬體／平台能力／安全環境而**客觀無法運行**時，記錄**具體原因＋最接近的替代驗證＋尚未驗證的行為**，不得謊稱已完整驗證。純重構／文件類同理：用最接近真實行為的整合測試／build／drift 檢查，並明講沒覆蓋到什麼。
- runtime 驗證**發現問題** → 把證據與重現步驟送回**同一隻 executor**修正，修好後重跑本 review gate。

判斷後：
- **通過** → 進下一階段（SendMessage 續 executor，或主線繼續）
- **要改** → 給修改指示（SendMessage 同一隻 executor 修；主線模式就自己修）→ 再 review

可選：想要第二意見時，派 **code-reviewer** subagent 獨立審 diff，但**最終判斷仍是主線**。

executor 自覺處理有問題上報時 → 主線做判斷 → 給新指示 SendMessage 續跑。

## 步驟 6 — archive（主線執行、自我把關、不等核准）

歸檔是機械操作（spectra／openspec 的 archive 指令），主線**自主直接跑**、不等使用者核准；但跑之前先確認同時滿足**能不能歸檔**的硬條件：

1. **tasks 全部完成、spec delta 齊全**——spectra 模式可用 `spectra status --change <名稱> --json`／`spectra validate` 機械檢查。
2. **verify／audit／drift 通過**（步驟 5 的 review gate 已收斂，無未處理的 correctness／security finding）。
3. **實跑證據已在 apply 取得**（步驟 5 的實跑完成契約）：專案可運行時已實際啟動並走過關鍵 scenarios；客觀不可運行時已記錄原因、替代驗證與 residual gap。**測試綠燈／TDD 跑完不等於驗證**——不得用它取代實跑。
4. **scope 外的 dirty 變更已明確排除**（保留使用者既有、與本 change 無關的改動）。

不符就回步驟 5 補完。spectra 模式直接 `spectra archive "<名稱>"`，**不得**用 `--mark-tasks-complete`／`--no-validate` 或以警告繞過硬條件。

## 步驟 7 — commit（主線親自、selective staging、不等核准）

觸發本 skill 已授權 selective commit，主線做最後的 change commit：

1. 重跑 `git status --short`／`git diff`，對照 spectra touched tracking 與開工基線，列出**本 change 實際動到的檔案**。
2. **逐檔 `git add <path>`**，排除所有 scope 外 dirty files；**禁止 `git add .`／`git add -A`**。
3. commit message 用 **en-us、conventional commits**，如實描述變更。
4. 顯示檔案清單與 message 後**直接提交，不等核准**；只有使用者明確排除 commit 時才保留未提交。跑 `git log --oneline -1` 回報 hash 與驗證證據。

commit 是全局收尾，**不外包給 executor**。
