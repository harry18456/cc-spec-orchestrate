English | [繁體中文](README.zh-TW.md)

# spec-orchestrate

Run spec-driven development **entirely inside Claude Code** — Claude acts as both the **global commander** and the **executor**.
You give instructions in natural language; the skill orchestrates the rest: detect the spec tool → propose (discuss and settle with you) → hybrid execution (small changes done directly on the main thread, big ones delegated to an executor subagent) → per-stage review → archive → commit.

Currently ships one skill: **`spec-claude-orchestrate`**.

## Prerequisites

1. The project has a **spectra** or **openspec** environment initialized.
2. Open Claude Code at the **spec project root** (the spectra CLI resolves the cwd).
3. Nothing else — **zero dependencies**: no external CLI, no network; it all runs inside Claude Code.

## Installation

```
/plugin marketplace add harry18456/cc
/plugin install spec-orchestrate@harry18456
```

Restart Claude Code to take effect.

> ⚠️ If you previously installed the skill by manually copying it into `~/.claude/skills/spec-claude-orchestrate/` (or a project's `.claude/skills/`), remove the old folder before installing the plugin, so the same skill doesn't load twice.

### Use locally right away (no push, no marketplace)

Junction the repo into the skills directory and the next session auto-loads it as `spec-orchestrate@skills-dir`:

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\spec-orchestrate" -Target "D:\side_project\cc-spec-orchestrate"
```

> ⚠️ Before switching to a proper marketplace install later, remove this junction first (`Remove-Item $env:USERPROFILE\.claude\skills\spec-orchestrate`) to avoid loading the same-named skill twice.

## How to trigger

- **Automatic**: in a repo with spectra/openspec, just state the intent, e.g.
  - "Build a CSV-export feature through the spec pipeline, all the way to commit"
  - "Continue the previous change; verify it yourself and archive"
- **Manual**: say "run this with spec-claude-orchestrate", or type `/spec-orchestrate:spec-claude-orchestrate`.
- **Want a checkpoint**: say "just propose first, show me" or "don't commit yet" and Claude stops there — **the default is fully autonomous, it won't pause for your approval**.

> It won't misfire on: general code review, wording polish, running tests, typo fixes, config troubleshooting — anything that doesn't touch a spec.

## How it works: two nested loops

```
┌─ Outer: you ↔ Claude (commander) ─────────────────────────┐
│  you give an instruction →                                 │
│      ┌─ Inner: commander ↔ executor (auto) ─────────┐      │
│      │ small change: the main thread implements it  │      │
│      │ big change: one executor subagent implements │      │
│      │   → main-thread review → pass → next stage   │      │
│      │   → issues → SendMessage the executor to fix │      │
│      └──────────────────────────────────────────────┘      │
│  ← Claude only comes back to you at key points             │
└─────────────────────────────────────────────────────────────┘
```

**The inner loop runs on its own — you don't have to babysit every stage. The outer loop is autonomous too** — propose→apply→review→archive→commit runs end to end without pausing for approval; Claude only comes back at the points below.

## Your role: autonomous by default, you step in only at key points

Triggering the skill **pre-authorizes** the whole pipeline; Claude stops for you only when:

| Stop point | When |
|------------|------|
| Major requirement ambiguity | A direction-changing ambiguity it can't ground from code/artifacts — otherwise it settles on the most reasonable reading itself |
| Major decision / stuck | A blocker it can't clear, or something needing new permissions / external credentials / out-of-scope data |
| You call a halt | You said "just propose" or "don't commit yet" — it stops there |

Everything else runs without approval — the proposal summary and each stage's result are **progress reports, not requests for your sign-off**.

## Hybrid execution: who writes the code?

| Situation | Executor | Why |
|-----------|----------|-----|
| Single task, few files, done in one round | **Main thread directly** | Saves tokens; you can watch in real time |
| Multi-task/multi-stage, several review rounds expected, main context already heavy | **Executor subagent** | Keeps the main context clean; review stays independent |

- One change = one executor (started with the Agent tool, follow-ups continue the same one via SendMessage).
- Review gates always come back to the main thread — **no free pass just because it wrote the code itself** (main-thread-implemented changes get a clean-context subagent review first).

## Example walkthrough (one change end-to-end, fully autonomous)

```
You:    Use the spec pipeline for "export search results as CSV", all the way to commit
Claude: [detects spectra mode; no major ambiguity → settles it itself, assumptions written into the proposal]
        [produces proposal/specs/tasks + validate/analyze] Here's the plan summary, starting apply now ← a report, not a prompt to approve
        [tasks have 3 stages → spawns an executor subagent] → report
        → [main-thread review + actually runs the project through the key path] → pass
        → SendMessage for the next stage → (only comes back if stuck / major ambiguity) → …
        → runs the key path for real → archive → selective commit
Claude: Done — committed (hash + runtime-verification evidence); stopped to ask you at X points only
```

> If a requirement has a direction-changing ambiguity, Claude asks you once before proposing (one question at a time), settles it, then proceeds on its own — no per-stage approval after that.

## spectra vs openspec (auto-detected)

- On start the skill detects: `spectra` CLI + `.spectra.yaml` present → **spectra mode** (review gates can use `spectra-verify`/`spectra-audit`/`spectra-drift`, i.e. the project-level skills `spectra init/update` generates under the project's `.claude/skills/`); otherwise → **openspec mode** (review gates are manual, judged against the actual code).
- Both share the openspec data format, but **CLI state is not interchangeable**: one change sticks to one tool from start to finish.

## ⚠️ Notes

- **Across sessions**: executor subagents die with the session; **durable state lives in the spec artifacts** (tick tasks.md as you go, `spectra park`/`in-progress` markers). A new session continuing a change spawns a fresh executor grounded on the artifacts.
- **One change, one executor**: all follow-up instructions for the same change go to the same executor via SendMessage; don't spawn another.
- **Apply runtime-completion contract (hard condition)**: if the project can run locally it **must actually be started and driven through the key path** before apply counts as done (CLI executed, API endpoints hit, UI e2e) — green tests / TDD / lint / build are only supporting evidence, and **runnable-but-not-run does not count as done and cannot be archived**. When it genuinely can't be run (missing credential/hardware/service), Claude states the reason, the closest substitute, and what stays unverified. Runtime issues go back to the same executor to fix and re-verify.
- **The spectra CLI resolves the cwd**: `spectra` has no flag to point at a directory, so every `spectra` command must run at the spec project root (Claude's file-editing tools use absolute paths and aren't affected).

## This plugin's components

Currently uses **skills** only (`skills/spec-claude-orchestrate/`, auto-discovered by Claude Code — no plugin.json declaration needed). `commands/`, `agents/`, `hooks/`, `output-styles/`, `bin/` are reserved skeletons; future extensions just drop files in.

## Related files

- **Skill body**: [`skills/spec-claude-orchestrate/SKILL.md`](skills/spec-claude-orchestrate/SKILL.md)
