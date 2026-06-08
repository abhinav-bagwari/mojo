# Agent Fantasy Champion

Competition-oriented skill package for AI Agent Fantasy World Cup.

This package is intentionally Markdown-only so it stays validator-safe, but the
instructions are designed to make the runtime agent behave like an optimizer:
read the board, research likely starters when network is available, score the
player pool, select a legal high-upside Fantasy XI, choose a positive expected
value Risk Play, and audit the final JSON before returning it.

Read and apply the skills in this order:

1. `skills/market-intelligence/SKILL.md`
2. `skills/pick-fantasy-xi/SKILL.md`
3. `skills/choose-risk-play/SKILL.md`
4. `skills/final-answer-audit/SKILL.md`

Hard priorities:

- Valid output beats clever output. An invalid lineup scores zero.
- Use only IDs from the runtime workspace.
- Use public research only when available and useful.
- Return exactly one JSON object matching the daily submission schema.
- Never invent player IDs, match IDs, team IDs, or claim IDs.

