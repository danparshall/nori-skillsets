---
name: Finish-Convo
description: Use at the end of a research session to save conversation context, update the research log, and optionally create handoff docs for implementation
---

<required>
*CRITICAL* Add the following steps to your Todo list using TodoWrite:

1. Determine the current branch name and convo name (established at session start).

2. Save conversation summary to `docs/active/<branch-name>/convos/<convo-name>.md`:

```markdown
# [Convo Name]

**Date:** YYYY-MM-DD
**Branch:** branch-name
**Participants:** Dan + Claude

## Summary
2-3 paragraphs of what was discussed and explored this session.

## Topics Explored
- Bullet points of what was investigated

## Provisional Findings
- What we learned or observed (these are provisional, not conclusions)

## Decisions Made
- Any concrete decisions about next steps or approach
- Link to plan docs if any were created

## Open Questions
- Things we didn't resolve
- Hypotheses that need testing
```

3. Append session entry to `docs/active/<branch-name>/RESEARCH_LOG.md`:

```markdown
## Session: YYYY-MM-DD — [convo-name]
### Topics Explored
- Brief bullet points (can reference the full convo file for detail)

### Provisional Findings
- Key takeaways from this session

### Next Steps
- What to try next session
```

Place the new entry at the TOP of the log (below the header), so the most recent session is first.

4. Update STATUS.md with a one-line session summary.

- Add a line under a "Recent Sessions" section (or create it if it doesn't exist)
- Format: `- YYYY-MM-DD: [branch] explored X, found Y`
- Try to keep entries in date order (newest first), but don't stress if ordering isn't perfect
- Do NOT rewrite STATUS.md conclusions — just append the one-liner

5. If the session produced something ready to implement:

- Ask the user: "This session produced [X] — should I create a plan doc for implementation?"
- If yes: read and follow the `write-a-plan` skill, saving to `docs/active/<branch-name>/plans/`
- The plan MUST reference the originating convo file

6. Stage and commit all changed files:

```bash
git add docs/active/<branch-name>/ STATUS.md
git commit -m "convo: <convo-name> — <one-line summary>"
```

- Add specific files, NOT `git add .` or `git add -A`
- If other files were changed during the session (code, data, etc.), include those too

7. Push to remote for backup:

```bash
git push -u origin <branch-name>
```

Research branches can live for weeks — don't let unpushed work accumulate. Push after every commit.

8. Do NOT:
- Create a PR (research branches stay open until user explicitly asks to merge)
- Merge into main (NEVER without explicit request)
- Run the finish-branch pipeline
- Update all docs as if conclusions are final
- Rewrite STATUS.md with authoritative conclusions
</required>

# Common Mistakes

**Writing convo summaries that sound like settled conclusions**
- Problem: Future agents read "we determined X" and treat it as ground truth
- Fix: Use language like "we explored X and the initial evidence suggests Y"

**Forgetting to link plans to conversations**
- Problem: Implementation plans lose their provenance, become unquestioned specs
- Fix: Every plan doc must have an "Originating conversation" field in its header

**Overwriting STATUS.md**
- Problem: A one-session finding replaces months of accumulated context
- Fix: ONLY append a one-liner. Never rewrite existing STATUS.md content during finish-convo.
