---
name: market-intelligence
description: Gather fast public match and player context before optimizing the Fantasy XI and Risk Play.
---

# Mission

Use the runtime board as the source of truth for IDs and eligibility. Use public
internet only as a decision-quality layer: likely starters, injuries,
suspensions, tactical roles, team strength, odds, and goal environment.

If network access is disabled or research is slow, skip public research and rely
on the board-only optimizer in `pick-fantasy-xi`.

# Time Budget

Spend at most 90 seconds on public research. Do not get stuck browsing. The agent
must still submit a valid JSON answer.

Use the time in two passes:

1. Spend about 45 seconds finding official lineups, same-day team news, injuries,
   suspensions, red-card bans, yellow-card accumulation bans, and predicted
   starters for the highest-value matches.
2. Spend about 45 seconds checking odds or reputable previews for favorite
   strength, clean-sheet chance, goal environment, set pieces, penalties, and
   player roles.

If official lineups are available, stop searching and use them. A confirmed
starter beats every projection. A confirmed non-starter, unavailable player, or
player ruled out must not be selected.

# Files To Read First

- `/workspace/game-board/matchday.json`
- `/workspace/game-board/matches.json`
- `/workspace/game-board/players.json`
- `/workspace/game-board/teams.json`
- `/workspace/game-board/standings-before.json`
- `/workspace/game-board/claim-catalog.json`
- `/workspace/rules/fantasy-xi.md`
- `/workspace/rules/risk-play.md`
- `/workspace/rules/bracket-play.md`
- `/workspace/output-format/daily-submission.schema.json`
- `/workspace/output-format/examples/daily-submission.json`
- `/workspace/team/README.md` when present
- every `/workspace/team/**/SKILL.md`

Use the `/workspace` paths exactly as provided by the platform.

# Workspace Contract

Use every provided file for its intended role:

- `matchday.json` provides the matchday ID and lock timing.
- `matches.json` provides match IDs and playing teams.
- `players.json` is the only valid source for Fantasy XI player IDs.
- `teams.json` provides team IDs, names, roster metadata, and team context.
- `standings-before.json` informs risk appetite based on tournament position.
- `claim-catalog.json` is the only valid source for Risk Play claims and
  required fields.
- `fantasy-xi.md` and `risk-play.md` define scoring and validity rules.
- `bracket-play.md` matters when bracket play is open.
- `daily-submission.schema.json` is the final output contract.
- `daily-submission.json` shows the expected answer shape.
- Team README and skill files are the team's strategy instructions.

Do not add bracket picks unless the current workspace and output schema make
bracket play active for this run.

# Research Plan

For each match in `matches.json`, build a compact mental table:

- Favorite, underdog, and whether the match is expected to be high scoring.
- Clean sheet likelihood for each team.
- Likely starting goalkeeper.
- Likely starting defenders for stronger clean sheet teams.
- Main attackers: forwards, advanced midfielders, penalty takers, set-piece
  takers, and players with strong recent goal involvement.
- The best 1 or 2 fantasy candidates from the match, even when neither team is
  the slate favorite.
- Confirmed injuries, suspensions, red-card bans, yellow-card accumulation bans,
  rotation warnings, or players unlikely to start.
- Official lineups if kickoff is close. Official lineups outrank all predictions.

Research priority:

1. Official lineups or official team announcements when available.
2. Current injury, suspension, and squad availability news.
3. Probable lineups from reputable match previews.
4. Betting or consensus market view for favorite, clean sheet, and goal total.
5. Recent form, role, set pieces, penalties, and goal involvement.

Do not spend equal time on every player. First identify the likely best teams and
highest-total matches, then focus research on their starters and primary
attackers.

# Daily News Gate

Before finalizing the XI, run this mental gate for every selected player:

- `green`: official starter, or strong same-day predicted starter with no injury
  concern.
- `yellow`: likely involved but role, minutes, or fitness is uncertain.
- `red`: ruled out, injured, suspended, red-card banned, yellow-card accumulation
  banned, absent, not in squad, expected bench, or mentioned as unlikely to
  start.

Never keep a `red` player. Replace a `yellow` player when a similar legal
`green` player exists at the same position or when the formation can legally
shift to a stronger `green` candidate.

Treat same-day words like "ruled out", "will miss", "sidelined", "injury
setback", "not in the squad", "suspended", "red-card ban", "yellow-card
accumulation", "bench", "rotation", and "fitness doubt" as downgrade signals.
If those words appear in a current source, the player needs newer official
evidence to stay in the XI.

# Starter Certainty

The most common way to lose Fantasy XI points is selecting players who do not
start or do not play enough minutes. Make starter certainty explicit:

- Lock in official starters when available.
- Prefer reliable probable starters over bigger names with uncertain minutes.
- Downgrade players returning from injury, rotation candidates, or players
  mentioned as bench options.
- If two candidates have similar upside, choose the one with clearer minutes.
- Do not select a famous player with an injury or bench signal just because the
  board contains strong historical stats.

# Famous Name Trap

Do not pick a famous player only because they are famous. They still need to be
eligible, likely to play, and aligned with the current scoring rules. A less
famous set-piece taker, penalty taker, attacking fullback, or nailed starter can
beat a famous rotation player.

# Public Research Output

Keep a compact mental table with these columns for each match:

- match id
- favorite and underdog
- estimated goal environment
- estimated clean-sheet chance for each team
- likely starting GK for each team
- high-confidence DEF starters on clean-sheet teams
- high-upside MID/FWD starters
- penalties, set pieces, and primary creators
- top 1 or 2 challenger candidates for the final XI
- avoid list with the reason

Use that table to improve the board-only XI. Do not put this table in the final
answer.

# Good Public Sources

Prefer sources that are current and specific:

- Official team or tournament lineup posts.
- Reputable live score or lineup pages.
- Club/national team match previews with probable lineups.
- Betting odds or consensus previews for favorite, clean sheet, and goal total
  signals.
- Recent player/team news for injuries, suspension, card bans, and role changes.

Use only public pages that do not require sign-in. Do not assume a page is
current unless its date clearly applies to the matchday.

# Convert Research Into Signals

Use these approximate values when scoring players:

- `start_probability`
  - 1.00 official starter.
  - 0.85 to 0.95 strong predicted starter.
  - 0.60 to 0.75 likely involved but uncertain.
  - 0.25 to 0.50 bench, rotation, or unclear role.
  - 0.00 injured, suspended, unavailable, or not in squad.
- `team_strength`
  - 1.00 heavy favorite.
  - 0.75 favorite.
  - 0.55 slight favorite or even.
  - 0.35 underdog.
  - 0.15 heavy underdog.
- `goal_environment`
  - 1.00 match likely to have 3+ goals.
  - 0.70 match likely to have 2+ goals.
  - 0.50 normal.
  - 0.30 low-scoring profile.
- `clean_sheet_probability`
  - 0.60 strong clean sheet chance.
  - 0.40 reasonable clean sheet chance.
  - 0.25 normal.
  - 0.10 weak clean sheet chance.
- `attacking_role`
  - 1.00 penalty taker, talisman, central forward, or elite creator.
  - 0.75 regular scorer/creator or set-piece taker.
  - 0.50 advanced role with some goal involvement.
  - 0.25 defensive or low-output role.

# Match-Level Signals

For each match, keep these signals in mind:

- Favorite strength: how likely one team is to control the match.
- Goal total: whether attackers or clean-sheet defenders deserve more weight.
- Lineup certainty: how reliable the starting information is.
- Tactical mismatch: whether one team has a clear attacking edge.
- Card intensity: whether the match profile supports card-based Risk Plays.

Use these signals to help `pick-fantasy-xi` and `choose-risk-play` make a
coherent decision rather than two unrelated choices.

# Conflict Rules

- Board IDs and eligibility always win over public names. If a researched player
  is not in `players.json`, do not select them.
- If public sources disagree, prefer official lineup, then most recent
  team-specific preview, then consensus.
- If a player has great reputation but low start probability, downgrade them.
- If a player has weak public data but strong board prior stats and likely starts,
  keep them in contention.
- If public research is stale or unclear, rely on the current board, rules, and
  conservative validity.
