---
name: council
description: >-
  Alias slash command (/council) for the llm-council skill. Only invoke when the user explicitly types /council; for automatic semantic trigger matching, use llm-council.
---

# Council Alias (`/council`)

This skill is an alias shortcut for `/llm-council`.

When invoked, immediately read and execute the full procedure in the `llm-council` skill (`../llm-council/SKILL.md` relative to this skill directory, or `~/.gemini/config/skills/llm-council/SKILL.md`), treating everything the user wrote after `/council` as the question/topic for the council.
