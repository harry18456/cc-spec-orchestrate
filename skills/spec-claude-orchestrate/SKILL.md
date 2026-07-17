---
name: spec-claude-orchestrate
description: >-
  Run spec-driven development entirely inside Claude Code — Claude is both the global commander
  (propose, every review gate, archive, commit) and the executor (small changes on the main
  thread; big multi-phase changes via ONE executor subagent per change, continued with
  SendMessage). Use this WHENEVER taking a feature through the spec pipeline or continuing one
  mid-flight in a repo with spectra or openspec, and the user wants Claude to handle it all.
  Typical asks: 走 spec 流程做某個功能、你全權處理這個 change、用 spectra/openspec 開一個 change
  先 propose 我確認後你自己實作、一個階段一個階段做完你自己驗、繼續上一個 change、archive 並
  commit。Do NOT use for one-off questions, plain edits, CLI/config troubleshooting, 一般 code
  review、潤飾文字、跑測試, or anything that doesn't touch a spec.
compatibility: Requires a spec tool (spectra CLI or openspec) initialized in the repo.
---

# Spec-Claude Orchestrate

把 spec-driven 開發完整跑在 **Claude Code 單引擎**內：你（Claude 主線）當**全局指揮**，持有 change 的全局狀態與**所有決策權**；實作依規模走**主線直做**或**executor subagent**（混合制）。

## 核心原則

**依工作性質分工**——判準：需要**全局視野或下判斷**的，主線親自來；把**既定指示落地執行**的，依規模分流（步驟 4）。

- **決策／思考／驗收 → 主線親自做**（或另開唯讀 subagent 輔助）：`propose`、`discuss`、`ask`、`explore`、`verify`、`audit`、`analyze`、`drift`、`archive` 把關、`commit`
- **實作／執行 → 混合制**：小 change 主線直做；大 change 交 executor subagent，主線驗收

**Grounding 準則**：任何判斷盡量依**實際程式碼**判讀、或查**官方文件／教學**求證——不臆測、不亂猜。**驗收同理**：看實際行為的實跑證據，**不把「測試綠燈／TDD 過」當通過**（測試多半是執行者自己寫的，同一個誤解會同時進實作與測試）。唯有真的無法處理，才停下來請使用者確認。

**角色紀律**：即使執行者也是 Claude，「指揮」與「執行」的判斷不混用——review gate 一律回到主線以實際程式碼下判斷，**不因為是自己（或自家 subagent）寫的就放水**。

## 人類介入點（其餘放手）

使用者的**主要關卡在 propose（步驟 3）**——在那裡把需求**討論清楚、使用者確認後才往下**。之後 apply→archive 的內層 review loop 由主線**自主跑**，只在這些點回報使用者：① 卡住、或判斷不了的重大決策 ② 最後 commit。其餘**不必一階一階等使用者拍板**。

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

## 步驟 3 — propose（主線親自，且**與使用者討論清楚**）

這是整個 change 的**關鍵人類關卡**——後面 apply/archive 會放手自主，所以「要做什麼」一定要在這裡跟使用者談清楚再定案。

1. **先討論清楚**：需求不夠明確就先釐清——提 2-3 個方向／取捨、確認範圍與邊界，**一次問一個**，談到雙方都清楚為止（spectra 模式可用 `spectra-discuss`）。
2. **產出 artifacts**：討論定案後才產 proposal／specs／design／tasks（spectra 模式 `spectra-propose`，產完會 `spectra park`）。
3. **確認 gate**：把 proposal／tasks 給使用者過目，**明確得到「可以」才進入 apply**——不要自己往下衝。

propose 是全局決策，**不外包給 executor**。

## 步驟 4 — 執行（混合制：依規模分流）

**分流判準**（heuristic，非硬規則）：

- **主線直做**：單一任務、動少數檔案（約 ≤5 個）、預期一輪完成、且主線 context 還輕。省 token、使用者可即時盯。
- **開 executor subagent**：tasks.md 有多任務／多階段、預期多輪 review-修改、或主線 context 已重（大實作別擠在 context 後段做）。

**executor 模式的操作**：

- **change 第一次派工**：用 Agent tool 開一隻 **general-purpose** executor。指示帶上：change 名稱、本輪 tasks 範圍、spec artifacts 路徑、輸出契約（做了什麼／改了哪些檔／自測結果／遇到的問題；自測結果要附實際執行的命令與輸出，不只測試綠燈）。
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

判斷後：
- **通過** → 進下一階段（SendMessage 續 executor，或主線繼續）
- **要改** → 給修改指示（SendMessage 同一隻 executor 修；主線模式就自己修）→ 再 review

可選：想要第二意見時，派 **code-reviewer** subagent 獨立審 diff，但**最終判斷仍是主線**。

executor 自覺處理有問題上報時 → 主線做判斷 → 給新指示 SendMessage 續跑。

## 步驟 6 — archive（主線執行、自我把關）

歸檔是機械操作（spectra／openspec 的 archive 指令），主線直接跑；但跑之前先確認**能不能歸檔**：

1. **tasks 全部完成、spec delta 齊全**——spectra 模式可用 `spectra status --change <名稱> --json`／`spectra validate` 機械檢查。
2. **verify／audit 通過**（步驟 5 的 review gate 已收斂）。
3. **實際行為驗證（硬條件）**：測試綠燈／TDD 跑完**不算驗證**——測試多半是執行者自己寫的，同一個誤解會同時出現在實作與測試裡。主線要看到**實跑證據**（跑了什麼命令、觀察到什麼輸出），必要時親自重跑關鍵情境：CLI 就實際執行命令、API 就起服務打 endpoint、UI 就跑 e2e 或實走關鍵流程。真的無法實跑的 change（純重構、文件類）→ 用最接近真實的方式驗（例如 drift 檢查＋既有整合測試），並明講「用什麼驗的、沒驗到什麼」。

不符就回步驟 5 補完。

## 步驟 7 — commit（主線親自）

由主線做最後的 change commit（全局收尾，commit message 用 en-us、conventional commits）。
