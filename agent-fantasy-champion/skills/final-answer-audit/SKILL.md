---
name: final-answer-audit
description: Validate the final daily submission JSON using a manual checklist before returning it.
---

# Mission

Before final output, audit the candidate answer using read-only reasoning. The
final response must be one JSON object and nothing else.

# Required Final Shape

The final object must contain:

- `team_id`: the exact team ID supplied by the platform request.
- `matchday_id`: the exact ID from `/workspace/game-board/matchday.json`.
- `fantasy_xi`: exactly 11 unique player ID strings.
- `risk_play`: either `null` or one valid claim object.
- `strategy`: one short explanation sentence.

# Team ID

Use the exact team ID supplied by the platform request. Do not use
`example-team`, `cris-team`, or any sample value unless that is the actual team
ID for the current platform request.

# Manual Audit Checklist

Verify all of this before returning:

- The answer follows `/workspace/output-format/daily-submission.schema.json`.
- The current `/workspace/rules/fantasy-xi.md` and
  `/workspace/rules/risk-play.md` were applied.
- `/workspace/rules/bracket-play.md` was considered only for active bracket
  requirements.
- The example in `/workspace/output-format/examples/daily-submission.json` was
  used only as a shape guide, not as a source of sample IDs.
- Top-level object has `team_id`, `matchday_id`, `fantasy_xi`, `risk_play`, and
  `strategy`.
- No extra top-level fields are present unless the schema explicitly requires
  them.
- `team_id` is non-empty and matches the platform request.
- `matchday_id` equals the value in `/workspace/game-board/matchday.json`.
- `fantasy_xi` has exactly 11 entries.
- Every `fantasy_xi` entry is a string.
- All 11 `fantasy_xi` values are unique.
- Every selected ID exists in `/workspace/game-board/players.json`.
- Every selected player is eligible for the current `matchday_id`.
- Formation is legal:
  - 1 GK
  - 3 to 5 DEF
  - 3 to 5 MID
  - 1 to 3 FWD
- If `risk_play` is `null`, that is valid.
- If `risk_play` is not `null`, `claim_id` exists in `/workspace/game-board/claim-catalog.json`.
- If `risk_play` is not `null`, every required field from the selected claim is
  present.
- Every risk `match_id`, `team_id`, and `player_id` exists in the board files.
- Risk play does not include `stake`, `bet_points`, or `stake_percent`.
- Bracket fields are absent unless the current schema and bracket board require
  bracket play for this run.
- Final answer has no Markdown, no comments, no explanations outside the JSON
  object, and no trailing notes.
- `strategy` is concise and mentions the selection logic and risk posture.

# Strategy Sentence

The `strategy` value should briefly explain why the lineup should score well:

- formation shape
- starter and scoring-rule logic
- main team or attacking stack
- risk posture

Keep it short. Do not include private reasoning or long analysis.

# Last Practical Check

If the final XI contains mostly famous names, verify they are likely starters.
If the final XI contains many defenders, verify clean-sheet scoring and matchup
quality justify it. If the Risk Play contradicts the lineup correlation, change
the Risk Play or return `null`.

# Risk Object Review

Use `/workspace/game-board/claim-catalog.json` as the only source of truth for
required risk fields.

Examples of valid shapes described in prose:

- A match-level Green claim needs `claim_id` and `match_id`.
- A team-level claim needs `claim_id`, `match_id`, and `team_id`.
- A player scoring claim needs `claim_id`, `match_id`, and `player_id`.
- An exact score claim needs `claim_id`, `match_id`, `home_score`, and
  `away_score`.

# Final Output Rule

Return exactly one JSON object. Do not wrap it in a Markdown fence. Do not add
any explanation before or after the object.
