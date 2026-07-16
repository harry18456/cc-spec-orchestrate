English | [繁體中文](README.zh-TW.md)

# spec-orchestrate

Run spec-driven development **entirely inside Claude Code** — Claude acts as both the **global commander** and the **executor**.
You give instructions in natural language; the skill orchestrates the rest: detect the spec tool → propose (discuss and settle with you) → hybrid execution (small changes done directly on the main thread, big ones delegated to an executor subagent) → per-stage review → archive → commit.

Currently ships one skill: **`spec-claude-orchestrate`** — the **Claude-only counterpart** of `spec-codex-orchestrate` (a Codex-delegating variant, not included in this plugin): same process philosophy, no Codex subscription required.

## Prerequisites

1. The project has a **spectra** or **openspec** environment initialized.
2. Open Claude Code at the **spec project root** (the spectra CLI resolves the cwd).
3. **No** codex plugin required — that is exactly why this skill exists.

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
  - "Without codex, build a CSV-export feature through the spec pipeline yourself — propose first so I can review"
  - "Continue the previous change; verify it yourself when done and move on if it's fine"
- **Manual**: say "run this with spec-claude-orchestrate", or type `/spec-orchestrate:spec-claude-orchestrate`.

> Versus the codex variant: codex plugin available and you want to delegate → `spec-codex-orchestrate`; unsubscribed / unavailable / you want Claude to own it all → this skill.
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

**The inner loop runs on its own — you don't have to babysit every stage.**

## Your role: settle the proposal, then let go

| When you step in | What you do |
|------------------|-------------|
| **propose (the main gate)** | Discuss with Claude until "what to build" is **clear**; apply starts only after you approve the proposal/tasks |
| Major decisions / stuck | When Claude can't decide, you make the call |
| Final commit | Close out the change |

## Hybrid execution: who writes the code?

| Situation | Executor | Why |
|-----------|----------|-----|
| Single task, few files, done in one round | **Main thread directly** | Saves tokens; you can watch in real time |
| Multi-task/multi-stage, several review rounds expected, main context already heavy | **Executor subagent** | Keeps the main context clean; review stays independent |

- One change = one executor (started with the Agent tool, follow-ups continue the same one via SendMessage).
- Review gates always come back to the main thread — **no free pass just because it wrote the code itself** (main-thread-implemented changes get a clean-context subagent review first).

## Example walkthrough (one change end-to-end)

```
You:    Use the spec pipeline for "export search results as CSV"; discuss it with me first, then do it all yourself
Claude: [detects spectra mode] A few things to confirm: export the current page or everything? Which columns? …
You:    (discussion…) everything, include the header row
Claude: [produces proposal/specs/tasks] Here's the plan — OK?
You:    OK, go
Claude: [tasks have 3 stages → spawns an executor subagent] → report → [main-thread review] → pass
        → SendMessage for the next stage → (only comes back if stuck) → … → archive
Claude: Done — here's the final commit (for you to sign off)
```

## spectra vs openspec (auto-detected)

- On start the skill detects: `spectra` CLI + `.spectra.yaml` present → **spectra mode** (review gates can use `spectra-verify`/`spectra-audit`/`spectra-drift`, i.e. the project-level skills `spectra init/update` generates under the project's `.claude/skills/`); otherwise → **openspec mode** (review gates are manual, judged against the actual code).
- Both share the openspec data format, but **CLI state is not interchangeable**: one change sticks to one tool from start to finish.

## ⚠️ Notes

- **Across sessions**: executor subagents die with the session; **durable state lives in the spec artifacts** (tick tasks.md as you go, `spectra park`/`in-progress` markers). A new session continuing a change spawns a fresh executor grounded on the artifacts.
- **One change, one executor**: all follow-up instructions for the same change go to the same executor via SendMessage; don't spawn another.
- **Hard condition for archive**: green tests / a finished TDD run **doesn't count as verification** — before archiving, Claude reruns the key scenarios for real evidence (CLI actually executed, API endpoints actually hit, UI e2e). Changes that can't be run for real get an explicit statement of how they were verified instead.
- Pitfalls the codex variant has that this one doesn't: file edits don't depend on cwd alignment (Claude tools use absolute paths; spectra CLI commands still run at the project root), and there is no "resume-last continued the wrong thread" problem (SendMessage targets a named executor, so parallel changes don't get mixed up).

## This plugin's components

Currently uses **skills** only (`skills/spec-claude-orchestrate/`, auto-discovered by Claude Code — no plugin.json declaration needed). `commands/`, `agents/`, `hooks/`, `output-styles/`, `bin/` are reserved skeletons; future extensions (e.g. adding the codex-delegating variant) just drop files in.

## Related files

- **Skill body**: [`skills/spec-claude-orchestrate/SKILL.md`](skills/spec-claude-orchestrate/SKILL.md)
- **Codex-delegating variant**: `spec-codex-orchestrate` — a separate skill, not shipped in this plugin (requires the codex plugin and a subscription)
