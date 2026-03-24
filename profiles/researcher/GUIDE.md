# Adapting Nori for Research Workflows

**Audience:** Nori users familiar with the `amol` (SWE) profile who want to use Claude Code for research — data science, academic projects, exploratory analysis, or any work where sessions don't always end with a pull request.

**TL;DR:** The amol profile assumes every session ships a feature. Research sessions often ship *understanding*. The changes below preserve amol's engineering discipline while making the workflow flexible enough for exploration, long-running research lines, and multi-session continuity.

---

## What the amol profile assumes (and where it breaks)

The amol CLAUDE.md prescribes a fixed pipeline:

```
research → plan → approval → TDD → docs → PR
```

Every step is mandatory. Every session ends with `finishing-a-development-branch`. This works well for parallel feature development — you get consistent, reviewable output from every agent session.

It breaks for research because:

1. **Not every session produces code.** Some sessions are pure exploration: reading papers, analyzing output, discussing hypotheses, running one-off experiments. Forcing a plan-then-TDD sequence on "let's look at the distribution of R-axis ratings" wastes time and produces artificial artifacts.

2. **Research lines span many sessions.** A feature branch lives for hours or days. A research line lives for weeks. Amol has no mechanism for session-to-session memory — each agent starts fresh with zero context about what previous sessions discovered.

3. **Findings are provisional.** In SWE, you converge toward a spec and ship it. In research, today's best hypothesis gets revised by tomorrow's data. The agent needs to treat prior documentation as trajectory, not truth.

4. **Data outlives worktrees.** Research generates large intermediate artifacts (API responses, checkpoints, parsed results) in gitignored directories. Amol's worktree skill doesn't account for these — cleaning up a worktree permanently deletes them.

## The changes, in order of impact

### 1. Make the pipeline conditional

This is the single most important change. Replace amol's rigid sequence with adaptive behavior based on what the session actually needs.

In your profile's CLAUDE.md, replace the fixed pipeline block with something like:

```markdown
- Do the work. This is a research-first profile — no forced pipeline.
  Adapt to what the session needs:
  - **Research/exploration:** Read papers, analyze data, discuss hypotheses,
    run experiments. No forced plan or TDD.
  - **Implementation:** If the user asks to implement something, use TDD.
    Read and follow `{{skills_dir}}/test-driven-development/SKILL.md`.
  - **Planning:** If a conversation produces something ready to implement,
    use `{{skills_dir}}/write-a-plan/SKILL.md` to create a plan doc.
```

The TDD skill, systematic debugging, and testing anti-patterns all stay available. You just stop forcing them on sessions that don't involve writing production code.

### 2. Add session continuity

This is the highest-ROI addition for multi-session work. You need three things:

**a) A research log per branch.** Create `docs/active/<branch-name>/RESEARCH_LOG.md`. Each session appends an entry (newest first) with: date, what was explored, what was found, what's still open. The agent reads this at session start to understand the trajectory of the research line.

**b) Conversation summaries.** Store structured summaries in `docs/active/<branch-name>/convos/`, prefixed with dates (e.g., `20260314_axis_stability_analysis.md`). These are more detailed than log entries — they capture the reasoning, evidence, and decision points from a session.

**c) A finish-convo skill.** Replace amol's automatic PR creation at session end with a skill that:
- Saves a convo summary to the branch's `convos/` directory
- Appends a session entry to RESEARCH_LOG.md
- Commits and pushes for backup
- Does NOT create PRs or merge anything

The key insight: in research, the *accumulated context* is the product, not the code diff. The finish-convo skill preserves that context for the next session.

**d) Convo naming.** Have the agent propose a session name at the start (format: `YYYYMMDD_topic_description`). This becomes the filename for the conversation summary. Small thing, but it makes the `convos/` directory navigable.

### 3. Add branch/worktree discipline for research

Amol auto-creates a worktree whenever you're on main. Research needs a more deliberate approach:

- **Ask which branch to work on** instead of auto-creating. Research lines have names and history; the agent should join an existing line, not create a new one by default.
- **Add `data/` symlinks** to the worktree creation skill. If you have gitignored directories (checkpoints, raw API responses, intermediate data), symlink them to the main worktree so they survive cleanup. This prevents real data loss — learned the hard way.
- **Never merge without explicit request.** Research branches stay alive for weeks. Multiple agents may be working in different worktrees simultaneously. Add a hard rule: the agent pushes for backup, but never merges.
- **Always `git fetch` before reporting branch status.** In multi-agent setups, other sessions push independently. Local refs go stale fast.

### 4. Add epistemic framing

Add a "Research Context" section to your profile CLAUDE.md:

```markdown
# Research Context

This profile is for research work. Findings in docs are provisional —
evidence accumulates gradually, and today's best understanding may shift
tomorrow.

**When the user says "the data showed X, let's pivot," TRUST THEM** —
they have seen results you haven't. Your job is to help explore the new
direction, not defend old hypotheses.

Do NOT treat any prior doc as settled truth. Read RESEARCH_LOG.md to
understand the trajectory of thinking, not just the latest conclusion.
```

Without this, agents tend to treat their own prior documentation as authoritative and resist pivots. In research, the data leads; the docs follow.

### 5. Write domain-specific audit skills

Amol's skills are generic SWE. If your project has structured knowledge artifacts — paper summaries, experiment logs, decision documents — write skills that audit them. The pattern:

1. Create a `skills/<skill-name>/` directory with a `SKILL.md`
2. Bundle a Python audit script that parses the artifact and checks for format violations, missing fields, and coverage gaps
3. The skill tells the agent when and how to run the script

Examples from practice:
- **Paper summary auditing**: Parses a PAPER_SUMMARIES.md file, checks each entry for required metadata fields (authors, year, key findings), flags entries missing numerical results, and optionally verifies claims against extracted paper text.
- **Decision doc maintenance**: Parses DOCS_INDEX.md and DOCS_SUMMARY.md, finds coverage gaps (files with no index entry), checks for broken cross-references, and validates ADR format.
- **Worktree inventory**: Reports branch status, uncommitted changes, merge status, untracked data files, and sizes across all worktrees. Prevents the "which worktree had my results?" problem.

These compound over time. Each session leaves the knowledge artifacts slightly more correct and complete.

### 6. Modify plan documents for epistemic provenance

Amol's `writing-plans` skill creates plans for "a senior engineer with zero codebase context." That's right — but research plans also need provenance. Replace or extend it with a plan template that adds:

- **Originating conversation** — link to the convo summary that produced this plan
- **Confidence level** — "High" (well-validated) vs. "Exploratory" (hypothesis-driven)
- **What could change** — what evidence would invalidate this plan
- **Open questions** — what the implementing agent should flag if they discover something unexpected

This matters because research plans are often written by one agent session and implemented by another, days later. The implementing agent needs to know how much to trust the plan and when to stop and ask.

## What to keep from amol

Not everything needs changing. These parts of the amol profile work well for research too:

- **TDD skill** — when you are writing code, write the test first. Research code is especially prone to subtle bugs because the "correct" output isn't always obvious. TDD catches regressions when you refactor analysis pipelines.
- **Systematic debugging and root-cause tracing** — research codebases accumulate technical debt fast. These skills prevent the "just add a try/except" instinct.
- **Testing anti-patterns** — the "never test mock behavior" rule is even more important in research, where mocking a data source can silently hide that your pipeline produces garbage on real data.
- **TodoWrite enforcement** — the using-skills meta-skill's insistence on tracking every step via TodoWrite feels bureaucratic but prevents the agent from losing its place in long sessions.
- **Anti-sycophancy tone** — "push back on bad ideas" is load-bearing in research. You want the agent to flag when your hypothesis doesn't match the data, not agree with everything.
- **Subagents for delegation** — the codebase analyzer, pattern finder, and code reviewer subagents work for research repos too. The web search researcher is useful for literature discovery.

## Minimal starter changes

If you want to start small, make these three changes to a copy of the amol profile:

1. **Replace the fixed pipeline** in CLAUDE.md with the conditional block from section 1
2. **Add a finish-convo skill** that saves context instead of creating PRs
3. **Add `data/` symlinks** to your worktree creation skill

Everything else can be added incrementally as your workflow demands it.

## File structure for a research profile

```
~/.nori/profiles/your-name/
  profile.json                          # Copy from amol, change name
  CLAUDE.md                             # Modified: conditional pipeline + research context
  skills/
    # === Kept from amol (unchanged) ===
    using-skills/SKILL.md
    test-driven-development/SKILL.md
    testing-anti-patterns/SKILL.md
    systematic-debugging/SKILL.md
    root-cause-tracing/SKILL.md
    creating-debug-tests-and-iterating/SKILL.md
    handle-large-tasks/SKILL.md
    receiving-code-review/SKILL.md
    creating-skills/SKILL.md
    updating-noridocs/SKILL.md
    brainstorming/SKILL.md

    # === Modified from amol ===
    use-worktree/SKILL.md               # Was using-git-worktrees; add data/ symlinks
    write-a-plan/SKILL.md               # Was writing-plans; add epistemic provenance

    # === New for research ===
    finish-convo/SKILL.md               # Session wrap-up: save context, update log, push
    clean-worktrees/SKILL.md            # Safe worktree consolidation with data preservation
    # ... your domain-specific audit skills ...

  subagents/                            # Keep all 7 from amol
  slashcommands/                        # Keep /nori-init-docs
```

## Summary

| Concern | Amol approach | Research adaptation |
|---------|--------------|-------------------|
| Session output | Always a PR | Understanding, plan, OR code — depends on the session |
| Pipeline | Fixed: research → plan → TDD → docs → PR | Conditional: adapt to what the session needs |
| Cross-session memory | None | RESEARCH_LOG.md + convo summaries + finish-convo skill |
| Worktree lifecycle | Short-lived, auto-created | Long-lived, explicitly chosen, with data symlinks |
| Documentation stance | "Update noridocs" (code docs) | Code docs + domain knowledge artifacts + audit skills |
| Epistemic posture | Ship the feature | Accumulate evidence; all findings provisional |
| Branch management | Implied merge at session end | Never merge without explicit request |
| Plan documents | Implementation spec | Implementation spec + provenance + confidence + open questions |
