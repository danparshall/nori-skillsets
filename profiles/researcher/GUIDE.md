# Adapting Nori for Research Workflows

**Audience:** Nori users familiar with the `amol` (SWE) profile who want to use Claude Code for research — data science, academic projects, exploratory analysis, or any work where sessions don't always end with a pull request.

**TL;DR:** The amol profile assumes every session ships a feature. Research sessions often ship *understanding*. This profile preserves amol's engineering discipline while making the workflow flexible enough for exploration, long-running research lines, and multi-session continuity.

---

## What the amol profile assumes (and where it breaks)

The amol CLAUDE.md prescribes a fixed pipeline:

```
research → plan → approval → TDD → docs → PR
```

Every step is mandatory. Every session ends with `finishing-a-development-branch`. This works well for parallel feature development — you get consistent, reviewable output from every agent session.

It breaks for research because:

1. **Not every session produces code.** Some sessions are pure exploration: reading papers, analyzing output, discussing hypotheses, running one-off experiments. Forcing a plan-then-TDD sequence wastes time and produces artificial artifacts.

2. **Research lines span many sessions.** A feature branch lives for hours or days. A research line lives for weeks. Amol has no mechanism for session-to-session memory — each agent starts fresh with zero context about what previous sessions discovered.

3. **Findings are provisional.** In SWE, you converge toward a spec and ship it. In research, today's best hypothesis gets revised by tomorrow's data. The agent needs to treat prior documentation as trajectory, not truth.

4. **Data outlives worktrees.** Research generates large intermediate artifacts (API responses, checkpoints, parsed results) in gitignored directories. Amol's worktree skill doesn't account for these — cleaning up a worktree permanently deletes them.

## The CLAUDE.md diff

The managed block is the core difference. Here's what changes:

| # | Amol | Researcher | Why |
|---|------|-----------|-----|
| 1 | Auto-create worktree on main | Ask which branch; only create if needed | Research lines have names and history |
| 2 | Forced research → plan → TDD sequence | Conditional: adapt to session type | Not every session produces code |
| 3 | No session naming | Propose convo name at start | Makes convos/ directory navigable |
| 4 | No prior context read | Pre-flight: read STATUS.md, README.md, RESEARCH_LOG.md | Session-to-session memory |
| 5 | Ends with finishing-a-development-branch | Ends with finish-convo (or update-docs mid-session) | Save context, don't force a PR |
| 6 | No merge guardrails | "NEVER merge without explicit request" | Multi-agent safety, long-lived branches |
| 7 | No personal onboarding | Checks for user background, solicits on first session | Tailors responses to researcher's domain |
| 8 | No repo structure check | Verifies research doc structure, offers to scaffold | Catches missing infrastructure early |
| 9 | No date awareness | Runs `date` command at session start | Prevents confusing planned vs. completed work |

## The documentation stack

The researcher profile expects a specific documentation structure. Each file has a defined role — information lives in exactly one place.

### Repo-level files (stable across branches)

| File | Role | When to read |
|------|------|-------------|
| **CLAUDE.md** | Agent instructions for this repo. How to work here, not what we're building. | Every session start |
| **README.md** | What this repo does and why. Updated when something merges to main. Stable between merges. | Every session start |
| **STATUS.md** | Where everything is. Complete branch inventory (active and archived), detailed status for the current branch, recent session log. | Every session start, every branch switch |
| **PAPER_INDEX.md** | One-sentence summary of each paper in papers/. Entry point for literature lookup. | When you need a paper on a topic |
| **PAPER_SUMMARIES.md** | Key conclusions per paper with numerical findings. Too long for every session — use after the index points you somewhere. | On demand |
| **papers/** | Raw PDFs of source literature. | On demand |
| **papers/text/** | Extracted text from PDFs, for searching and discussing fine details. | On demand, when summaries aren't enough |

### Branch-level files (per research line)

| File | Role | When to read |
|------|------|-------------|
| **docs/active/\<branch\>/RESEARCH_LOG.md** | The index for this branch. Which convos tied to which plans, session history, trajectory of thinking. Newest entries first. | Every session start |
| **docs/active/\<branch\>/convos/** | Conversation summaries. One per session, named `YYYYMMDD_topic.md`. | When you need to understand why a decision was made |
| **docs/active/\<branch\>/plans/** | Implementation plans. Each MUST point back to the originating convo. | When implementing something |
| **docs/active/\<branch\>/results/** | Analysis outputs, figures, data summaries. Each links back to the convo that produced it. | On demand |

### Lifecycle: active → historical

When a research line is complete and its branch is merged:
1. `git mv docs/active/<branch> docs/historical/<branch>`
2. Add an entry to the "Archived Research Lines" table in STATUS.md (branch name, date, one-line summary of what was learned)

Historical docs are **never deleted** — always recoverable when you need to revisit prior reasoning. But they're not loaded into session context by default. STATUS.md tells agents what's in historical/ and why, so they know it exists without reading it.

## The skill changes

### New skills

| Skill | Purpose |
|-------|---------|
| **update-docs** | The core operation. Checkpoints research progress mid-session: creates/updates convo summary, saves results with provenance links, updates RESEARCH_LOG.md and STATUS.md. Does NOT commit or push — the session continues. |
| **finish-convo** | Thin wrapper: runs update-docs, then commits and pushes. This is the session boundary marker. |
| **add-paper** | Full paper integration pipeline: download PDF, extract text to papers/text/, add one-liner to PAPER_INDEX.md, read the paper and write a full summary in PAPER_SUMMARIES.md. Ensures every paper is searchable and indexed. |
| **audit-docs** | Checks docs/active/ for consistency: convos indexed in RESEARCH_LOG? Plans linked to convos? Results have provenance? Prompts the user for discrepancies — never auto-fixes (other sessions may be active). |
| **clean-worktrees** | Safely consolidates accumulated worktrees with data preservation, merge safety checks, and research doc archiving. |

### Modified from amol

| Skill | What changed |
|-------|-------------|
| **use-worktree** (was using-git-worktrees) | Adds `data/` symlink to main worktree, preventing data loss on cleanup. Adds research doc scaffolding (RESEARCH_LOG.md + convos/ + plans/ + results/). |
| **write-a-plan** (was writing-plans) | Checks for convo file before creating plan — runs update-docs to create one if missing. Adds epistemic provenance: originating conversation, confidence level, "what could change" footer. |
| **audit-papers** (was auditing-paper-summaries) | Adds structural check: every PDF has text extraction, every paper indexed in PAPER_INDEX.md and summarized in PAPER_SUMMARIES.md. Prompts user for fixes. |

### The dependency chain

```
update-docs (core operation)
  ├── creates/updates convo in convos/
  ├── saves results to results/ with provenance links
  ├── updates RESEARCH_LOG.md
  └── appends one-liner to STATUS.md

finish-convo = update-docs + commit + push

write-a-plan
  └── checks for convo → runs update-docs if missing
      → then writes plan with convo link

audit-docs (checks consistency)
  └── convos indexed? plans linked? results have provenance?
      → prompts user, never auto-fixes

add-paper (integrates a new paper)
  └── download PDF → extract text → add to PAPER_INDEX
      → read paper → add to PAPER_SUMMARIES → stage

audit-papers (checks completeness)
  └── PDFs have text? papers indexed? summaries accurate?
      → prompts user for structural fixes
```

### Kept from amol (unchanged)

These work well for research too:

- **TDD** — when you are writing code, write the test first. Research code is especially prone to subtle bugs.
- **Systematic debugging and root-cause tracing** — prevents the "just add a try/except" instinct.
- **Testing anti-patterns** — "never test mock behavior" is even more important in research, where mocking a data source can silently hide that your pipeline produces garbage on real data.
- **TodoWrite enforcement** — feels bureaucratic but prevents the agent from losing its place in long sessions.
- **Subagents** — codebase analyzer, pattern finder, code reviewer, web search researcher all work for research repos.

## Collaborative framing and intellectual honesty

The researcher profile opens with "You are a collaborative research partner" before any constraints. This matters — the agent needs to understand its role before it learns the rules. The anti-sycophancy instructions that follow are stronger than amol's because the stakes are different in research:

- **The agent is a collaborator, not a service.** Its job is to help explore ideas, draw connections, challenge weak reasoning, and build on what's working. Framing it this way before the constraints gives them purpose.
- **False confidence is costly.** Validating a flawed argument or supplying plausible-sounding but shaky evidence can waste weeks of work built on a bad foundation.
- **Softened objections are dangerous.** The most likely failure mode is the agent turning a real concern into a polite caveat. The profile instructs: lead with concerns at full strength, then separately assess if they're fixable.
- **Sycophancy check command.** Users can say "sycophancy check" and the agent will review its own chain-of-thought vs. what it conveyed, flagging where it pulled punches.

## Getting started

### Option A: Switch to the researcher profile

If the researcher profile is available in the registry:

```bash
nori-skillsets install researcher
nori-skillsets switch-skillset researcher
```

### Option B: Install from a local copy

Copy the `profiles/researcher/` directory to `~/.nori/profiles/researcher/`, then:

```bash
nori-skillsets switch-skillset researcher
```

### First session

On your first session with the researcher profile, it will:
1. Ask for your background (name, domain, how you think) — saved above the managed block for all future sessions
2. Check if your repo has the research doc structure — offer to scaffold if missing
3. Ask which branch to work on
4. Propose a convo name for the session

After that, it adapts to whatever the session needs.

## Summary

| Concern | Amol approach | Researcher approach |
|---------|--------------|-------------------|
| Session output | Always a PR | Understanding, plan, OR code — depends on the session |
| Pipeline | Fixed: research → plan → TDD → docs → PR | Conditional: adapt to what the session needs |
| Cross-session memory | None | RESEARCH_LOG.md + convo summaries + update-docs/finish-convo |
| Mid-session save | None | update-docs: checkpoint progress without ending the session |
| Worktree lifecycle | Short-lived, auto-created | Long-lived, explicitly chosen, with data symlinks |
| Documentation | Code docs only | Code docs + literature tracking + research artifacts + audit skills |
| Provenance | Plans are specs | Plans link to convos; results link to convos; convos indexed in log |
| Epistemic posture | Ship the feature | Accumulate evidence; all findings provisional |
| Branch management | Implied merge at session end | Never merge without explicit request |
| Intellectual honesty | "Push back on bad ideas" | Full anti-sycophancy framework with sycophancy-check command |
| Date awareness | None | Runs `date` at session start to prevent temporal confusion |
