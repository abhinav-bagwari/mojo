# Agent Fantasy Champion

Championship goal: maximize expected tournament points while keeping every
submission valid. The agent should behave like a matchday fantasy analyst:
read the current board, understand the rule text, gather fast public lineup and
odds context when available, choose a legal high-upside Fantasy XI, choose only a
positive-value Risk Play, and return one clean JSON object.

Applying the skills in this order:

1. `skills/market-intelligence/SKILL.md`
2. `skills/pick-fantasy-xi/SKILL.md`
3. `skills/choose-risk-play/SKILL.md`
4. `skills/final-answer-audit/SKILL.md`

Use the full platform workspace contract before deciding:

- Matchday, matches, players, teams, standings, and claim catalog from
  `/workspace/game-board/`.
- Fantasy XI, Risk Play, and Bracket Play rules from `/workspace/rules/`.
- Required schema and example answer from `/workspace/output-format/`.
- Team README and every team skill from `/workspace/team/`.

Only include fields supported by the current schema. Bracket rules are useful
when bracket play is open, but do not add bracket picks to a daily answer unless
the current workspace and schema require them.

Winning priorities:

1. Validity first, because an invalid lineup loses the day.
2. Minutes second, because starters beat famous bench players.
3. Scoring-rule fit third, using the current `fantasy-xi.md` and `risk-play.md`.
4. Upside fourth, through favorites, goal environment, set pieces, penalties,
   clean sheets, and attacking roles.
5. Risk Play last, based on expected value and leaderboard situation.

Daily improvement loop:

- Treat scorecards and run logs as feedback about what the agent missed.
- If XI points lag, improve starter detection, scoring-rule weighting, and
  formation choice.
- If Risk Play loses points, raise the probability threshold or return `null`
  more often.
- If validation fails, simplify immediately and protect validity before adding
  more strategy.
