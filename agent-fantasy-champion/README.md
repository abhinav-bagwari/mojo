# Agent Fantasy Champion

Championship goal: maximize expected tournament points while keeping every
submission valid. The agent should behave like a matchday fantasy analyst:
read the current board, understand the rule text, gather fast public lineup and
odds context when available, choose a legal high-upside Fantasy XI, choose only a
positive-value Risk Play, handle the one-time Bracket Play run when active, and
return one clean JSON object.

Applying the skills in this order:

1. `skills/market-intelligence/SKILL.md`
2. `skills/pick-fantasy-xi/SKILL.md`
3. `skills/choose-risk-play/SKILL.md`
4. `skills/pick-bracket/SKILL.md` only when Bracket Play is active
5. `skills/final-answer-audit/SKILL.md`

For a bracket-only run, use `market-intelligence`, then `pick-bracket`, then
`final-answer-audit`. Skip Fantasy XI and Risk Play unless the active schema
explicitly asks for them.

Use the full platform workspace contract before deciding:

- Matchday, matches, players, teams, standings, and claim catalog from
  `/workspace/game-board/`.
- For Bracket Play, the official bracket board from
  `/workspace/game-board/bracket.json`; this is the source of truth for
  `bracket_id`, required `match_id` values, allowed winners, round names, and
  whether a third-place pick is included.
- Bracket context and World Cup standings from `/workspace/game-board/` when
  supplied, especially `bracket-context.json` and `world-cup-standings.json`.
- Fantasy XI, Risk Play, and Bracket Play rules from `/workspace/rules/`.
- Required schema and example answer from `/workspace/output-format/`.
- Team README and every team skill from `/workspace/team/`.

Treat those runtime `/workspace` files as the Daily Answer Contract source of
truth. Do not rely on stale remembered scoring, old examples, or public player
names that are not present in the official board.

Only include fields supported by the current schema. Bracket Play has two modes:

- Daily Fantasy XI and Risk Play continue as usual for normal daily runs.
- On the one-time bracket-only run, submit the full tournament bracket and do
  not include Fantasy XI or Risk Play unless that run's schema explicitly
  requires them.

Knockout scoring reminders:

- Extra time counts for knockout final-score and player-event Risk Play claims.
- Penalty shootout goals do not count as normal goals for Fantasy XI or standard
  Risk Play claims.
- Knockout-only Risk Plays such as `match_goes_to_extra_time` and
  `match_goes_to_penalties` are normal daily Risk Play choices when present in
  `claim-catalog.json`; they are not Bracket Play.
- Bracket Play is separate and opens later with the Round of 32 bracket.

Bracket Play scoring makes late rounds matter most:

- Round of 32: 5 points.
- Round of 16: 8 points.
- Quarterfinal: 12 points.
- Semifinal: 18 points.
- Final / Champion: 30 points.

For bracket practice before the real window, boards may contain provisional
Round of 32 paths. Use `world-cup-standings.json` and `bracket-context.json` to
reason about likely paths, but never invent IDs or unsupported JSON structure.
When `matchday.json` has `phase: "bracket"`, treat the run as Bracket Play.
When `official_knockout_fixtures_available` is false or `board_status` is
`planning_projection`, treat every candidate-pool slot as provisional and use
only official IDs already present in the workspace.

Bracket output shape is:

- `team_id`
- `bracket_id`
- `picks`: one object per required bracket match, each with `round`, `match_id`,
  and `winner_team_id`
- `champion_team_id`
- optional `strategy`

Winning priorities:

1. Validity first, because an invalid lineup loses the day.
2. Current availability second, because a non-playing star scores zero.
3. Minutes third, because starters who reach 60 minutes create a 4-point floor.
4. Scoring-rule fit fourth, using the current `fantasy-xi.md` and
   `risk-play.md`.
5. Upside fifth, through goals, assists, set pieces, penalties, clean sheets,
   saves, and attacking roles.
6. Risk Play last, based on expected value and leaderboard situation: protect a
   lead, but take calculated positive-value risk when chasing a large gap.
7. Bracket Play when active, because knockout-season predictions can create a
   long-range tournament edge.

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
- Before accepting a concentrated lineup, compare the weakest selected players
  against the best candidates from every match on the slate.
- Do not let one favorite team consume the whole XI. On a multi-match slate,
  default to at most 5 players from one team, allow 6 only with direct
  expected-points wins, and reject 7 or more unless the board is genuinely thin.
- Treat the best likely-starting striker, penalty taker, or team talisman from
  every match as a must-review slate-breaker before accepting extra defenders or
  low-upside midfielders from a favorite stack.
- When standings show Mojo is far behind the lead, prioritize slate-breaking
  attackers and high-confidence upside over low-variance safety picks.
- If the public-research step finds official lineups, treat them as decisive.

Daily improvement loop:

- Treat scorecards and run logs as feedback about what the agent missed.
- If XI points lag, improve starter detection, scoring-rule weighting, and
  formation choice.
- If Risk Play loses points, raise the probability threshold or return `null`
  more often.
- If Bracket Play is active, build the full bracket from current teams, draw,
  rules, injuries, strength, knockout path, and upside scenarios instead of
  copying favorites blindly.
- If validation fails, simplify immediately and protect validity before adding
  more strategy.
