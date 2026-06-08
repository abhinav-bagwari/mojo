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

# Files To Read First

- `/workspace/game-board/matchday.json`
- `/workspace/game-board/matches.json`
- `/workspace/game-board/players.json`
- `/workspace/game-board/teams.json`
- `/workspace/game-board/claim-catalog.json`
- `/workspace/rules/fantasy-xi.md`
- `/workspace/rules/risk-play.md`
- `/workspace/output-format/daily-submission.schema.json`

Use `{CONTAINER_WORKSPACE_DIR}` in shell commands if the prompt provides it.
Otherwise use `/workspace`.

# Research Targets

For each match in `matches.json`, build a compact mental table:

- Favorite, underdog, and whether the match is expected to be high scoring.
- Clean sheet likelihood for each team.
- Likely starting goalkeeper.
- Likely starting defenders for stronger clean sheet teams.
- Main attackers: forwards, advanced midfielders, penalty takers, set-piece
  takers, and players with strong recent goal involvement.
- Confirmed injuries, suspensions, rotation warnings, or players unlikely to
  start.
- Official lineups if kickoff is close. Official lineups outrank all predictions.

# Good Public Sources

Prefer sources that are current and specific:

- Official team or tournament lineup posts.
- Reputable live score or lineup pages.
- Club/national team match previews with probable lineups.
- Betting odds or consensus previews for favorite, clean sheet, and goal total
  signals.
- Recent player/team news for injuries, suspension, and role changes.

Do not use sources that require credentials. Do not assume a page is current
unless its date clearly applies to the matchday.

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

# Conflict Rules

- Board IDs and eligibility always win over public names. If a researched player
  is not in `players.json`, do not select them.
- If public sources disagree, prefer official lineup, then most recent
  team-specific preview, then consensus.
- If a player has great reputation but low start probability, downgrade them.
- If a player has weak public data but strong board prior stats and likely starts,
  keep them in contention.

