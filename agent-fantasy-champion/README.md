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
2. Current availability second, because a non-playing star scores zero.
3. Minutes third, because starters who reach 60 minutes create the daily floor.
4. Scoring-rule fit fourth, using the current `fantasy-xi.md` and
   `risk-play.md`.
5. Upside fifth, through goals, assists, set pieces, penalties, clean sheets,
   saves, and attacking roles.
6. Risk Play last, based on expected value and leaderboard situation.

Daily operating principle:

- Build a legal board-only XI first.
- Improve it with current public lineup, injury, odds, and role evidence when
  available.
- Before final output, re-check every selected player for same-day availability.
- Remove anyone described as ruled out, injured, suspended, red-card banned,
  yellow-card accumulation banned, unavailable, absent from the squad, or likely
  benched unless there is newer official evidence.
- Prefer one or two strong clean-sheet stacks, but do not let defensive stacking
  crowd out clearly superior starting attackers and advanced midfielders.
- If the public-research step finds official lineups, treat them as decisive.

Daily improvement loop:

- Treat scorecards and run logs as feedback about what the agent missed.
- If XI points lag, improve starter detection, scoring-rule weighting, and
  formation choice.
- If Risk Play loses points, raise the probability threshold or return `null`
  more often.
- If validation fails, simplify immediately and protect validity before adding
  more strategy.
