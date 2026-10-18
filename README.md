# SAC — SAP Analytics Cloud Skills for Claude

Claude skills for SAP Analytics Cloud development, maintained by Cognivion.

---

## Skills

### Data Action (`/sac-data-action`)
Build, debug, and verify SAC Data Actions — allocations, formula steps, copy steps, multi-action sequences.

**Install URL:**
```
https://raw.githubusercontent.com/anilkumarkn100/sac/master/skills/SKILL-data-action.md
```

Commands: `/da-help` · `/da-context` · `/da-build` · `/da-verify` · `/da-allocate`

---

### SAC Scripting (`/sac-scripting`)
Build scripted forms and UIs in SAC Analytical Applications — CRUD forms, dropdowns, popups, master data maintenance. Covers the full scripting SDK including unsupported JavaScript constructs.

**Install URL:**
```
https://raw.githubusercontent.com/anilkumarkn100/sac/master/skills/SKILL-sac-scripting.md
```

Commands: `/script-help` · `/script-form` · `/script-widget` · `/script-debug`

---

## How to Install a Skill

1. Open Claude (claude.ai or Claude Code)
2. Go to **Settings → Skills**
3. Add skill from URL → paste the install URL above
4. Save

Skills fetch from this repo on each load — updates are picked up automatically.

---

## Repository Structure

```
skills/
  SKILL-data-action.md      ← Data Action skill (self-contained)
  SKILL-sac-scripting.md    ← SAC Scripting skill (self-contained)
references/                 ← Supporting knowledge base for Data Action skill
patterns/                   ← Known issue patterns and gotchas
```

---

Maintainer: Cognivion SAC Practice
