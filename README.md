# 1DD architecture-doc skills for Devin

Two skills that make Devin produce the same single-file HTML architecture pages as the 1DD artifacts (Event Map, Grid Flow): same fonts, colours, arrows, hover-to-trace, and the same "verified against code, File:line or it didn't happen" discipline.

| Skill | Produces | Invoke |
|---|---|---|
| `event-map` | producers → exchanges/topics → consumers map with curved arrows, trace-on-hover, pinned detail panel, generated per-module table, findings | `@skills:event-map whole repo` |
| `arch-doc` | module walkthrough: reality check, lanes overview with arrows, call chains with anchors, breakpoint tables, gotchas, mermaid state machines, docs-vs-code drift | `@skills:arch-doc M7 hub flow` |

Each `SKILL.md` is self-contained: the full HTML template is embedded at the bottom, so pasting just the `.md` is enough.

## Install — pick one

**A. Commit to the repo (recommended).** Unzip at the repo root so you get:

```
.agents/skills/event-map/SKILL.md
.agents/skills/event-map/templates/event-map.html
.agents/skills/arch-doc/SKILL.md
.agents/skills/arch-doc/templates/arch-doc.html
```

Devin scans `.agents/skills/*/SKILL.md` when it clones the repo. It will also pick these up from `.claude/skills/` if you'd rather keep them next to Claude Code's.

**B. Paste into Devin's web app.** Customize → Skills → new skill → paste the whole `event-map.SKILL.md` (frontmatter included). Repeat for `arch-doc.SKILL.md`.

**C. One-off session.** Rename to `event-map.devin.md` and drag it in when starting a session. The body already uses the playbook headings (Procedure, Specifications, Advice, Forbidden actions).

## Example prompts

- `@skills:event-map whole repo` — rebuild the 1DD Event Map from current `main`.
- `@skills:event-map M5 dispatch only, docs/architecture/m5-events.html`
- `@skills:arch-doc M9 airline consolidation, end to end, for a debugger`
- `@skills:arch-doc shipment state machine — every transition with its guard and anchor`

## What Devin should hand back

A PR with one `.html` file under `docs/architecture/`, a full-page screenshot and a trace screenshot in the description, and a one-line-per-finding summary in chat.
