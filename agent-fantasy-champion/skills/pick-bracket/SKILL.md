---
name: pick-bracket
description: Build the one-time Bracket Play submission when the platform opens the knockout bracket run.
---

# Mission

When Bracket Play is active, submit the strongest full tournament bracket allowed
by the current workspace. This is a bracket-only decision path for the one-time
knockout run, not a Daily Fantasy XI or Risk Play task.

Use the runtime files as the source of truth. Do not rely on old draw memories,
old standings, or sample IDs.

# Activation Gate

Use this skill only when the current request, workspace, rules, or schema clearly
indicate Bracket Play is open. For the announced knockout window, expect a
one-time standalone bracket-only run around June 27.

Treat the run as Bracket Play when `/workspace/game-board/matchday.json` has
`phase: "bracket"`. The observed bracket planning board opens at
`2026-06-27T00:00:00-06:00`, locks at `2026-06-27T20:00:00-06:00`, has
`board_status: "planning_projection"`, and may have
`official_knockout_fixtures_available: false`.

If this is a normal daily run:

- Do not add bracket fields.
- Continue with Fantasy XI and Risk Play.

If this is the bracket-only run:

- Build the full bracket required by `/workspace/rules/bracket-play.md`.
- Follow the current bracket schema exactly.
- Do not return Fantasy XI or Risk Play fields unless the bracket-run schema
  explicitly requires them.

# Files To Read First

- `/workspace/game-board/matchday.json`
- `/workspace/game-board/bracket.json`
- `/workspace/game-board/matches.json`
- `/workspace/game-board/teams.json`
- `/workspace/game-board/standings-before.json`
- `/workspace/game-board/world-cup-standings.json` when present
- `/workspace/game-board/bracket-context.json` when present
- `/workspace/rules/bracket-play.md`
- `/workspace/output-format/bracket-submission.schema.json`
- `/workspace/output-format/examples/bracket-submission.json`
- `/workspace/team/README.md` when present
- every `/workspace/team/**/SKILL.md`

`bracket.json` is the Bracket Play source of truth. Use it for `bracket_id`,
required matches, round names, allowed `winner_team_id` values, round points,
`final_match_id`, and whether `third_place_included` is true. `matches.json`,
`teams.json`, `world-cup-standings.json`, and `bracket-context.json` explain the
context behind the board; they do not override `bracket.json`.

Practice boards before the real bracket window may contain provisional Round of
32 paths; use those paths for reasoning, but never invent team IDs, match IDs,
round IDs, or extra JSON fields.

If a Round of 32 match has `projection_status: "candidate_pool"`, choose only
from `candidate_team_ids` plus any resolved `home_team_id` or `away_team_id`
listed for that match in `bracket.json`. Do not fill the slot with a team
outside the candidate pool even if that team looks stronger.

If a match has `projection_status: "derived_from_future_results"`, determine
its possible teams only from the winners selected in its `source_match_ids`.
Later-round winners must be reachable from earlier-round picks.

# Scoring Weight

Bracket Play rewards later rounds more heavily:

- Round of 32: 5 points.
- Round of 16: 8 points.
- Quarterfinal: 12 points.
- Semifinal: 18 points.
- Final / Champion: 30 points.

Because the champion and finalist path carries the biggest value, do not burn a
strong champion path for a cute early upset. First optimize champion,
finalists, and semifinalists, then choose Round of 32 and Round of 16 upsets
that do not break the high-value path.

# Bracket Research Plan

Build a compact bracket table before picking winners:

- Current knockout draw, round names, match IDs, and valid team IDs.
- `bracket-context.json` path hints, provisional slots, and matchup structure
  when supplied.
- `world-cup-standings.json` qualification, seeding, form, and likely path
  signals when supplied.
- For each Round of 32 candidate-pool match, the exact `candidate_teams`,
  `allowed_group_letters`, and the chosen provisional team.
- For each later match, the `source_match_ids` and the two possible winners
  flowing into that match.
- Team strength from the board, standings, recent form, and market consensus
  when available.
- Injury, suspension, rotation, goalkeeper, and defensive stability signals.
- Attack quality: primary scorer, penalty taker, set pieces, chance creation,
  transition threat, and late-game bench impact.
- Knockout-specific traits: extra-time endurance, penalty shootout quality,
  defensive discipline, coaching conservatism, and ability to protect a lead.
- Path difficulty: do not evaluate a champion pick without checking the likely
  semifinal/final opponents.

Public research is useful but secondary to workspace IDs and rules. Prefer
current, public, no-login sources. Do not get stuck browsing.

# Selection Strategy

Pick each round as an expected-value decision:

- Favor the better team when the edge is clear.
- Use draw-path leverage only when two teams are close enough that upside
  matters.
- Do not pick every favorite automatically; bracket contests are won by a few
  correct disagreements with consensus, not random chaos.
- Avoid low-probability upsets in early rounds if they destroy a strong champion
  path.
- A champion pick needs both team quality and a survivable route.
- Prioritize champion and finalist accuracy over decorative first-round upsets
  because Final / Champion is worth 30 points and Semifinal is worth 18.
- Reduce upset risk in early rounds unless the upset also improves the likely
  high-value champion or finalist path.

# Upset Rules

An upset is worth taking only when at least two of these are true:

- The favorite has a current injury, suspension, lineup, or fatigue problem.
- The underdog has a clear tactical matchup edge.
- Market odds or previews suggest the match is closer than seed/reputation.
- The underdog has a strong goalkeeper or penalty-shootout edge.
- The upset opens a realistic path to more bracket points later.
- The upset does not remove the best champion or finalist candidate from a
  high-value path without strong evidence.

Reject upsets based only on vibes, famous-player narratives, or a desire to be
different.

# Champion Rules

Before locking the champion:

- Compare the top 3 likely champions by path, strength, health, and knockout
  profile.
- Verify the champion can plausibly beat each projected opponent.
- Do not choose a champion with major unavailable-player risk unless the bracket
  value is exceptional.
- If two champion candidates are close, prefer the one with the easier route and
  stronger defense.
- Verify `champion_team_id` is exactly the same team ID as the winner selected
  in the final.

# Output Discipline

Return exactly the fields required by
`/workspace/output-format/bracket-submission.schema.json`. The bracket output
shape is:

- `team_id`
- `bracket_id`
- `picks`
- `champion_team_id`
- optional `strategy`

Each `picks` item must contain exactly:

- `round`
- `match_id`
- `winner_team_id`

Use no extra fields inside `picks`.

For bracket-only output:

- Return one JSON object matching the bracket schema.
- Set `bracket_id` from `/workspace/game-board/bracket.json`.
- Include one pick for every required match in `bracket.json`.
- Use only `winner_team_id` values allowed by `bracket.json` and reachable from
  earlier picks.
- Set `champion_team_id` to the same winner selected in the final.
- Do not include Fantasy XI output.
- Do not include Risk Play output.
- Do not invent IDs or unsupported structure for provisional practice boards.
- If `third_place_included` is false, do not include a third-place pick. If it is
  true, include the required third-place match using the schema's `third_place`
  round value.

Do not include prose outside the JSON object. If the schema allows a short
strategy field, mention the champion path and upset posture in one sentence.
