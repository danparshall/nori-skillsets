# Researcher — A Nori Skillset for Research Workflows

**A research-first Nori Skillset for data science, academic projects, and exploratory work where sessions don't always end with a pull request.**

Most coding agent configurations assume every session ships a feature. Research sessions often ship *understanding*. This Skillset preserves engineering discipline (TDD, systematic debugging, code review) while making the workflow flexible enough for exploration, long-running research lines, and multi-session continuity.

## Who is this for?

- Data scientists running multi-week analysis projects
- Academic researchers managing literature, hypotheses, and experiments
- Anyone using Claude Code for exploratory work where the output is insight, not just code

If every session ends with a PR and your branches live for hours, the [amol/SWE Skillset](https://noriskillsets.dev/) is probably a better fit. If your branches live for weeks and you need the agent to remember what happened last Tuesday — read on.

## Installation

```bash
# Install the nori-skillsets CLI if you haven't already
npm install -g nori-skillsets

# Initialize nori in your project (if not already done)
nori-skillsets init

# Download and switch to the researcher Skillset
nori-skillsets download researcher
nori-skillsets switch researcher
```

## First session

On your first session with the researcher profile, the agent will:

1. **Ask for your background** — name, domain, how you work. Saved once and reused in all future sessions so the agent can tailor its responses.
2. **Check your repo structure** — if the research documentation scaffold is missing, it offers to create it.
3. **Ask which branch to work on** — research lines have names and history; the agent joins an existing line rather than auto-creating a new one.
4. **Propose a session name** — e.g., `20260314_axis_stability_analysis`. This becomes the filename for the conversation summary.

After that, it adapts to whatever the session needs.

## What makes this different from a SWE Skillset

| Concern | SWE approach | Researcher approach |
|---------|-------------|-------------------|
| Session output | Always a PR | Understanding, plan, OR code — depends on the session |
| Pipeline | Fixed: research -> plan -> TDD -> docs -> PR | Conditional: adapt to what the session needs |
| Cross-session memory | None | Research log + conversation summaries + finish-convo skill |
| Worktree lifecycle | Short-lived, auto-created | Long-lived, explicitly chosen, with `data/` symlinks |
| Documentation | Code docs only | Code docs + literature tracking + research artifacts |
| Epistemic posture | Ship the feature | Accumulate evidence; all findings provisional |
| Branch management | Implied merge at session end | Never merge without explicit request |
| Intellectual honesty | "Push back on bad ideas" | Full anti-sycophancy framework with sycophancy-check command |

## How session continuity works

The core problem: every Claude Code session starts with zero memory. The researcher profile solves this with a documentation stack that the agent reads at the start of each session and writes to at the end.

### Pre-flight reads (every session)

The agent reads these files before doing anything else:

1. **STATUS.md** — branch inventory, current branch status, recent session log
2. **README.md** — what this repo does and why
3. **docs/active/\<branch\>/RESEARCH_LOG.md** — session history and trajectory for this research line

### Session wrap-up

At the end of each session, the `finish-convo` skill:
- Saves a structured conversation summary to `docs/active/<branch>/convos/`
- Updates RESEARCH_LOG.md with a session entry
- Updates STATUS.md with current status
- Commits and pushes for backup
- Does NOT create PRs or merge anything

You can also checkpoint mid-session with the `update-docs` skill — same as finish-convo but without committing, so the session continues.

## Documentation structure

The profile scaffolds this structure in your repo (via `init-research-repo` or on first session):

```
your-repo/
  CLAUDE.md                              # Agent instructions for this repo
  README.md                              # What this repo does and why
  STATUS.md                              # Branch inventory + current status
  PAPER_INDEX.md                         # One-line summary per paper (entry point)
  PAPER_SUMMARIES.md                     # Detailed findings per paper
  papers/                                # Raw PDFs
  papers/text/                           # Extracted text (searchable by agent)
  docs/
    active/
      <branch-name>/
        RESEARCH_LOG.md                  # Session index for this research line
        convos/                          # One summary per session (YYYYMMDD_topic.md)
        plans/                           # Implementation plans (link back to convos)
        results/                         # Analysis outputs with provenance
    historical/                          # Archived research lines (never deleted)
```

**Each file has exactly one job.** Information lives in one place — PAPER_INDEX.md is for finding papers, PAPER_SUMMARIES.md is for reading about them, RESEARCH_LOG.md is for understanding the trajectory of a research line.

When a research line is complete, `git mv docs/active/<branch> docs/historical/<branch>` and add an entry to STATUS.md. Historical docs are never deleted — they're recoverable when you need to revisit prior reasoning, but not loaded into session context by default.

## Skills included

### Research-specific skills (new)

| Skill | What it does |
|-------|-------------|
| **finish-convo** | End-of-session: saves convo summary, updates log and status, commits and pushes |
| **update-docs** | Mid-session checkpoint: same as finish-convo but session continues |
| **add-paper** | Full paper integration: download PDF, extract text, index in PAPER_INDEX.md, write summary in PAPER_SUMMARIES.md |
| **audit-docs** | Checks docs/active/ for consistency — orphaned files, missing links, unindexed convos. Prompts for fixes, never auto-corrects. |
| **audit-papers** | Checks papers/ structure — every PDF has text extraction, every paper indexed and summarized |
| **clean-worktrees** | Safe worktree consolidation with data preservation and merge safety checks |
| **init-research-repo** | Scaffolds the full documentation structure in a new or existing repo |

### Modified from SWE profile

| Skill | What changed |
|-------|-------------|
| **use-worktree** | Adds `data/` symlink to main worktree (prevents data loss on cleanup). Adds research doc scaffolding. |
| **write-a-plan** | Requires link to originating conversation. Adds confidence level, "what could change" section, and open questions. |

### Carried from SWE profile (unchanged)

These work well for research too:

- **test-driven-development** — when writing code, write the test first. Research code is prone to subtle bugs.
- **systematic-debugging** / **root-cause-tracing** — prevents "just add a try/except."
- **testing-anti-patterns** — "never test mock behavior" matters even more when mocking a data source can silently hide garbage output.
- **brainstorming** — structured idea refinement before implementation.
- **handle-large-tasks** — context window management for long sessions.
- **finishing-a-development-branch** — available when you do want to create a PR.
- **receiving-code-review** / **creating-skills** / **updating-noridocs** / **building-ui-ux** / **webapp-testing** / **maintaining-decision-docs**

### Subagents

Eight specialized subagents for delegation:

- **nori-codebase-locator** / **nori-codebase-analyzer** / **nori-codebase-pattern-finder** — codebase exploration
- **nori-code-reviewer** — post-change review
- **nori-web-search-researcher** — literature discovery and web research
- **nori-change-documenter** / **nori-initial-documenter** — documentation generation
- **docs** — general documentation agent

## Intellectual honesty and anti-sycophancy

The researcher profile frames the agent as a collaborative research partner, not a service. Key behaviors:

- **Leads with concerns at full strength.** If something might not work, it says so before exploring fixes — not "great idea, with one small caveat."
- **Treats findings as provisional.** Prior documentation represents trajectory, not truth. When you say "the data showed X, let's pivot," the agent trusts you and helps explore the new direction.
- **Sycophancy check.** Say "sycophancy check" and the agent reviews its own reasoning vs. what it told you, flagging where it pulled punches.
- **Pushes back.** Flags bad ideas, unreasonable expectations, and mistakes. Does not say "you are absolutely right" — ever.

## Adapting this Skillset

The GUIDE.md in this directory explains the design rationale — why each change was made relative to the SWE profile, what problems each piece solves, and what to keep vs. modify if you're building your own research-oriented Skillset.

## Resources

- **Nori Skillsets Registry**: [noriskillsets.dev](https://noriskillsets.dev/)
- **Nori Documentation**: [noriskillsets.dev/docs](https://noriskillsets.dev/docs/building-a-skillset)
- **GitHub**: [github.com/tilework-tech/nori-skillsets](https://github.com/tilework-tech/nori-skillsets)
