[English](README.md) | 繁體中文

# spec-orchestrate

把 spec-driven 開發完整跑在 **Claude Code 單引擎**內的編排 plugin——Claude 同時當**全局指揮**與**執行者**。
你用自然語言下指令,skill 負責編排:偵測 spec 工具 → propose(跟你討論定案)→ 混合制實作(小改主線直做、大改派 executor subagent)→ 每階段 review → archive → commit。

目前內含一個 skill:**`spec-claude-orchestrate`**。

## 前提

1. 專案已初始化 **spectra** 或 **openspec** 環境。
2. 在 **spec 專案根目錄**開 Claude Code(spectra CLI 認 cwd)。
3. 除此之外**零依賴**:不需外部 CLI、不連網,全部跑在 Claude Code 內。

## 安裝

```
/plugin marketplace add harry18456/cc
/plugin install spec-orchestrate@harry18456
```

重開 Claude Code 生效。

> ⚠️ 若你先前把 skill 手動複製到 `~/.claude/skills/spec-claude-orchestrate/`(或專案的 `.claude/skills/`),安裝 plugin 前先移除舊資料夾,避免同名 skill 載入兩次。

### 本機立即使用(免 push、免 marketplace)

把 repo 以 junction 連到 skills 目錄,下個 session 就自動以 `spec-orchestrate@skills-dir` 載入:

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\spec-orchestrate" -Target "D:\side_project\cc-spec-orchestrate"
```

> ⚠️ 之後改從 marketplace 正式安裝前,先移除這個 junction(`Remove-Item $env:USERPROFILE\.claude\skills\spec-orchestrate`),避免同名 skill 重複載入。

## 怎麼觸發

- **自動**:在有 spectra/openspec 的專案,講出意圖即可,例如
  - 「用 spec 流程做一個匯出 CSV 的功能,一路做到 commit」
  - 「繼續上一個 change,自己驗完就 archive」
- **手動**:直接說「用 spec-claude-orchestrate 跑這個」,或輸入 `/spec-orchestrate:spec-claude-orchestrate`。
- **想中途把關**:明講「只做 proposal 先給我看」或「先別 commit」,Claude 就會停在該點——**預設是全程自治、不會停下等你核准**。

> 不會誤觸發的情況:一般 code review、潤飾文字、跑測試、改 typo、config 排錯等「不碰 spec」的事。

## 怎麼運作:兩層 loop

```
┌─ 外層:你 ↔ Claude(指揮)─────────────────────────┐
│  你下指令 →                                         │
│       ┌─ 內層:指揮 ↔ 執行(自動跑)────┐            │
│       │ 小改:主線直做                   │            │
│       │ 大改:executor subagent 實作     │            │
│       │   → 主線 review → 通過續下一階段 │            │
│       │   → 有問題 SendMessage 叫它修    │            │
│       └────────────────────────────────┘            │
│  ← Claude 只在關鍵點回來找你                          │
└──────────────────────────────────────────────────────┘
```

**內層自動跑,你不必一階一階盯。外層預設也自治**——propose→apply→review→archive→commit 一路自己跑,不會停下等你核准,只在下面這些點回來找你。

## 你的角色:預設全程自治,你只在關鍵點介入

觸發 skill 就等於**預先授權**整條 pipeline;Claude 只在這些情況停下等你:

| 停下的點 | 什麼時候 |
|---------|---------|
| 需求重大歧義 | 影響方向的歧義,從程式碼/artifacts 也 ground 不了時才問你;否則自行合理定案 |
| 重大決策 / 卡住 | blocker 排不掉、或需要新權限/外部 credential/動到 scope 外資料 |
| 你主動喊停 | 你明講「只做 proposal」「先別 commit」時停在該點 |

其餘一律不等核准——proposal 摘要、每階段結果都只是**進度回報**,不是要你按「同意」。

## 混合制:誰動手實作?

| 情況 | 執行者 | 為什麼 |
|------|--------|--------|
| 單一任務、少數檔案、一輪可完成 | **主線直做** | 省 token、你能即時盯 |
| 多任務/多階段、預期多輪修改、主線 context 已重 | **executor subagent** | 主線 context 保持乾淨、review 有獨立性 |

- 一個 change = 一隻 executor(Agent tool 起手、後續用 SendMessage 續同一隻)。
- review gate 一律回主線判斷——**不因為是自己寫的就放水**(主線直做的 change 會先另開乾淨 context 的 subagent 審過再判斷)。

## 使用範例(一個 change 從頭到尾、全程自治)

```
你:用 spec 流程做「搜尋結果匯出 CSV」,一路做到 commit
Claude:[偵測 spectra 模式;需求無重大歧義 → 自行合理定案,假設寫進 proposal]
        [產出 proposal/specs/tasks + validate/analyze] 這是計畫摘要,直接開始實作 ← 進度回報,不等你點頭
        [tasks 有 3 階段 → 開 executor subagent 實作] → 回報
        → [主線 review + 實跑專案走過關鍵路徑驗證] → 通過
        → SendMessage 續下一階段 →(只有卡住/重大歧義才回報你)→ …
        → 實跑走過關鍵路徑 → archive → selective commit
Claude:完成,已 commit(hash + 實跑驗證證據);過程只在 X 處停下問你
```

> 若需求有影響方向的重大歧義,Claude 會在 propose 前先問你一句(一次一個),談清楚再自行往下;之後不再逐階等你。

## spectra vs openspec(自動切換)

- skill 啟動時偵測:有 `spectra` CLI ＋ `.spectra.yaml` → **spectra 模式**(review gate 可用 `spectra-verify`/`spectra-audit`/`spectra-drift`,即 `spectra init/update` 產生在專案 `.claude/skills/` 的 skills);否則 → **openspec 模式**(review gate 由 Claude 依程式碼手動審)。
- 兩者共用 openspec 資料格式,但 **CLI 狀態不互通**:同一個 change 固定用同一套工具跑到底。

## ⚠️ 注意事項

- **跨 session**:executor subagent 隨 session 結束消失,**耐久狀態以 spec artifacts 為準**(tasks.md 即時勾、`spectra park`/`in-progress` 標記)。新 session 續 change 會開新 executor、靠 artifacts 接手。
- **一個 change 一隻 executor**:同 change 的修改指示都用 SendMessage 續同一隻,不要另開。
- **apply 實跑完成契約(硬條件)**:專案能在本機跑就**必須實際啟動走過關鍵路徑**才算 apply 完成(CLI 實跑、API 打 endpoint、UI e2e)——測試綠燈/TDD/lint/build 只是輔助證據,**能跑卻沒跑不得宣告完成、也不得 archive**;客觀無法跑(缺 credential/硬體/服務)則明講原因、替代驗證與沒驗到什麼。runtime 發現問題會交回同一隻 executor 修完重驗。
- **spectra CLI 認 cwd**:`spectra` 沒有指定目錄的 flag,所有 `spectra` 命令都要在 spec 專案根執行(Claude 的檔案編輯工具走絕對路徑,不受此限)。

## 這個 plugin 的組件

目前只用了 **skills**(`skills/spec-claude-orchestrate/`,Claude Code 自動發現、不必在 plugin.json 宣告)。`commands/`、`agents/`、`hooks/`、`output-styles/`、`bin/` 為預留骨架,日後擴充直接放檔案即可。

## 相關檔案

- **skill 本體**:[`skills/spec-claude-orchestrate/SKILL.md`](skills/spec-claude-orchestrate/SKILL.md)
