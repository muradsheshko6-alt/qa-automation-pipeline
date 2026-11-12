# QA Automation Pipeline (Claude Code plugin)

A five-stage multi-agent pipeline, plus a standing external-code-review loop, for turning a live web app into a maintained, CI-ready Playwright/pytest automation suite with full traceability back to a test-case management tool.

This isn't a single script or prompt — it's a small team of specialized Claude Code subagents, each scoped to one stage, orchestrated by a skill that knows when to dispatch which one and where a human needs to look before the pipeline continues.

```
Site Mapper  -->  [human review]  -->  Suite Designer  -->  [human review]  -->  Automation Triage  -->  Automation Engineer  -->  Description Polish  -->  (standing) External Review Loop
```

| Stage | What it does | Who does it |
|---|---|---|
| 1. Site Mapper | Explores the target app end-to-end via live browser automation, produces a Modules -> Features map | `site-mapper` subagent (bundled) |
| 2. Suite Designer | Turns the map into structured test cases inside a test-case management tool | Your tool's own AI generator (reference: Coverly) or hand-authored |
| 3. Automation Triage | Reviews every case against real site constraints, tags what's realistically automatable | `automation-triage` subagent (bundled) |
| 4. Automation Engineer | Builds the real Playwright + pytest suite, API-first, Page Object Model, dynamic test data, parallel-safe | `automation-engineer` subagent (bundled) |
| 5. Description Polish | End-of-batch cleanup pass: condenses each case's accumulated Triage/Engineer notes into a short, structured description instead of a run-on wall of text | Whoever's orchestrating (no dedicated subagent needed — it's bookkeeping, not building) |
| Standing | An external reviewer (reference: Codex) periodically reviews the automation project's own code; findings get verified and fixed | `automation-engineer` again, dispatched against verified findings |

Full orchestration logic, stage-by-stage instructions, and the credential-handling discipline that ties it all together live in [`skills/qa-pipeline/SKILL.md`](skills/qa-pipeline/SKILL.md).

## Why four stages instead of one

Each stage is a genuinely different kind of judgment call:
- **Mapping** a site is reconnaissance — breadth and accuracy matter more than speed.
- **Deciding what to automate** is a risk/value call grounded in what mapping actually found, not assumptions about "typical" apps.
- **Building automation** is engineering — architecture, reliability, parallel safety, CI-readiness.
- **Reviewing it later** benefits from a second, independent set of eyes that wasn't in the room when the code was written.

Collapsing these into one pass tends to lose the human checkpoints that catch problems while they're still cheap to fix — an incomplete site map, or a generated test case that overclaims what it actually verifies, otherwise propagates silently through every later stage.

## What's in this repo

```
.claude-plugin/
  plugin.json
  marketplace.json
agents/
  site-mapper.md           # Stage 1
  automation-triage.md      # Stage 3
  automation-engineer.md    # Stage 4
skills/
  qa-pipeline/
    SKILL.md                # orchestration logic + the review-loop convention
```

## Installing

As a Claude Code plugin, from inside Claude Code:

```
/plugin marketplace add muradsheshko6-alt/qa-automation-pipeline
/plugin install qa-automation-pipeline@qa-automation-pipeline
```

Or copy `agents/*.md` into your project's `.claude/agents/` and `skills/qa-pipeline/` into your project's `.claude/skills/`. Claude Code picks up the agents and the skill automatically.

## Prerequisites

- A browser-automation MCP server (the reference implementation uses Playwright MCP) for the Site Mapper and Automation Engineer stages.
- A test-case management tool with an MCP server, or at least an API you can wrap one around (the reference implementation uses Coverly; `automation-triage.md` and `automation-engineer.md` both note exactly which tool calls to swap if you're using something else — TestRail, Zephyr, or a custom tracker all work the same way).
- Optionally, an external code-review tool configured to periodically review the automation project and write findings to a known path in the repo (the reference implementation uses Codex) — this isn't required to build the suite, but it's what keeps it honest as it grows.

## Adapting this to your own project

The three bundled agents are already written generically (no app name, no specific site baked in) — they take the target URL, environment, and scope as task input each time they're dispatched. The only thing you'll likely want to adapt per project is swapping `mcp__coverly__*` tool references for your own test-case management tool's MCP tool names, which is called out explicitly in both `automation-triage.md` and `automation-engineer.md`.

## A note on credentials

This pipeline is built around one firm rule: real credentials (test account passwords, API tokens) only ever get written into a project's own gitignored `.env`, by the orchestrating session directly — never by a subagent, and never into the test-case management tool's case text or any other third-party-visible surface. See `SKILL.md` for the full reasoning; it's worth reading before you adapt this for a project with real stakes.

## License

MIT
