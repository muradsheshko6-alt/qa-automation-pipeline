---
name: qa-pipeline
description: Orchestrates a full QA automation pipeline for a web app -- mapping its features, generating test cases in a test-case management tool, triaging what's realistically automatable, building a real Playwright/pytest suite, and running a standing external-code-review loop (e.g. Codex) against that suite. Use this whenever the user wants to set up automated testing for a web app from scratch, wants to add a new batch of test coverage to an existing automation project, wants to wire in a code-review loop for an automation project, or mentions building/extending a QA automation pipeline, a test-case management tool (Coverly, TestRail, Zephyr, etc.), or turning manual test cases into automated ones. Also trigger when the user shares an external reviewer's findings about an automation project and wants them verified and acted on.
---

# QA Automation Pipeline

This skill orchestrates four build stages plus one standing maintenance loop. It does not do the work itself — it tells you which specialized subagent to dispatch, in what order, with what human checkpoints in between, and how to run the review loop that keeps the resulting suite honest over time.

The stages exist because each one is a genuinely different kind of judgment call, and collapsing them loses something real: mapping a site is reconnaissance, deciding what to automate is a risk/value call, building automation is engineering, and reviewing it later is a different set of eyes. Keep them separate even under time pressure.

```
Site Mapper  -->  [human review]  -->  Suite Designer  -->  [human review]  -->  Automation Triage  -->  Automation Engineer  -->  (standing) External Review Loop
```

## Stage 1 — Site Mapper

Dispatch the `site-mapper` agent (bundled with this skill) with the target URL and any environment-specific constraints (how to get past OTP/email verification, which account types exist, etc.). It explores the whole site live and produces a `modules_features.json` plus a human-readable report.

**Always stop for human review after this stage.** The map is the foundation everything else is built on; a gap here propagates through all four remaining stages silently. Don't let an agent chain straight from mapping into case generation without a human looking at the map first.

## Stage 2 — Suite Designer

This is deliberately *not* a custom subagent in this skill. The reference implementation uses Coverly's own built-in AI case generator: you feed it the Stage 1 map (or point it at the site), and it drafts structured test cases directly into a suite there. If your test-case management tool has an equivalent AI generator, use that instead; if it doesn't, cases can be authored by hand from the map — the pipeline doesn't care how the suite gets populated, only that it ends up as real cases in a tool the next two stages can read from and write back to.

**Always stop for human review after this stage too.** Generated cases need a human to confirm they actually reflect what Stage 1 found, especially around anything flagged as unclear/blocked.

## Stage 3 — Automation Triage

Dispatch the `automation-triage` agent against the suite from Stage 2. It reads every case, decides `to_automate` vs `manual` against the real constraints Stage 1 found (not assumptions), and tags each case accordingly with a short rationale where the call isn't obvious.

If you (the human) already know a blocker found in Stage 1 is being fixed, tell the triage agent explicitly — it's instructed to bias toward tagging `to_automate` when you've said so, rather than parking things as manual out of excess caution. Read its triage report before moving on; it will flag genuinely ambiguous cases for you rather than guessing.

## Stage 4 — Automation Engineer

Dispatch the `automation-engineer` agent against the `to_automate`-tagged cases. It scaffolds or extends one real Playwright + pytest project — Page Object Model, dynamic Faker-based test data, parallel-safe by construction, CI-ready from the first commit — implements each case, runs it for real, and writes the result back to the test-case management tool (`automated`, or left `to_automate` with a clear reason if it genuinely couldn't be done as scoped).

A few things worth over-explaining because they matter more than they look:

- **Never let it mark something `automated` without having actually run it and watched it pass.** A test that compiles but was never executed against the real app is worse than no test — it reports false confidence.
- **A case that's genuinely blocked should come back blocked, not faked.** If (for example) a password-reset-expiry test can't be verified because the token's real TTL makes waiting impractical and there's no way to shorten or force-expire it, the right outcome is an honest, documented block routed back to triage/dev-team — not a test that doesn't actually test what it claims to.
- **Bugs found along the way are a feature of this process, not a distraction from it.** Driving the real app end-to-end surfaces real regressions (a stale locator, a config default pointing at a dropdown option that no longer exists, a UI overlay that can't be dismissed) that a human clicking through manually might never hit with enough repetition to notice. Fix what's safely fixable in the test layer; flag anything that's an actual product bug to the people who own that code.

## Standing: the external code-review loop

Once a project exists, it keeps evolving — new cases, new fixtures, new page objects — and a second, independent set of eyes catches things the Automation Engineer agent, working alone, tends to miss: subtle self-referential bugs (a test fixture that quietly asserts against itself), weak success assertions, secrets that would leak into a build artifact, CI that silently swallows failures.

Set up an external code-review tool (the reference setup here uses Codex, but any capable external reviewer works) to periodically review the automation project's source and write its findings to a known path in the repo (e.g. `codex_outputs/QA_AUTOMATION_REVIEW.md`). Then, every time a new review lands:

1. **Read it fully before acting on any of it.** Treat it as a second opinion, not an instruction set to blindly execute.
2. **Verify a sample of its most serious findings yourself** by reading the actual code it's pointing at — an external reviewer without live execution access can misjudge severity or miss context (e.g. flagging something as unconfirmed because it can't see the real directory layout). Confirm the real ones before spending engineering effort on them.
3. **Let the human decide remediation scope** — don't default to "fix everything" or "fix nothing" on your own judgment; ask, unless the user has already told you their standing preference.
4. **Dispatch the Automation Engineer agent to apply approved fixes**, highest severity first, with the original finding's file/line reference and the verified root cause in its brief — not just "fix what Codex said," which forces it to re-derive context you already have.
5. **Re-verify for real after the fixes land** (lint clean, full suite still green) before considering the review closed.

## A rule worth being strict about: credential handling

Across every stage, real credentials (test account passwords, API tokens, app passwords) must only ever end up in one place: a project's own gitignored `.env` file, written directly by you (the orchestrating session), never by a subagent and never into the test-case management tool's case text, suite descriptions, or any other third-party-visible surface. Subagents reference credentials only by environment-variable name (mirroring whatever pattern the project already uses for its first stable credential, e.g. `STABLE_LOGIN_EMAIL`/`STABLE_LOGIN_PASSWORD`) — they read `.env`, they don't write real values into it or see them echoed back in their own output.

This isn't excess caution for its own sake: subagent transcripts, tool outputs, and anything synced to an external test-case tool are a much wider blast radius than one gitignored file on disk, and credentials have a way of ending up quoted in exactly the places you don't want them once a convention isn't held firmly from the start.

## Picking up an existing project

This pipeline is not just for greenfield builds. For an existing automation project:
- A new batch of test coverage still goes through Stages 2-4 in order — new cases get designed/authored, triaged, then built — even if Stage 1's map already exists and doesn't need re-running.
- Before adding new cases for something you suspect might already be covered, check the test-case management tool first (search/list existing cases in the relevant suite). Building a case that duplicates existing coverage, or worse, building automation for a case whose description overclaims what it actually tests (a real failure mode: a case that says it covers two boundary conditions but whose steps only exercise one), wastes effort and leaves a false sense of coverage. When you find a case like that, narrow its description to what it actually verifies and open a new case for the gap, rather than silently leaving the overclaim in place.
- The external review loop (above) runs continuously regardless of whether you're in an active build phase — treat a new review landing as its own trigger, not something that waits for the next planned work session.
