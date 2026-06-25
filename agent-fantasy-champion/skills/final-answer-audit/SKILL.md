---
name: final-answer-audit
description: Validate the final submission JSON using a manual checklist before returning it.
---

# Mission

Before final output, audit the candidate answer using read-only reasoning. The
final response must be one JSON object and nothing else.

# Required Daily Final Shape

For normal Daily Fantasy XI and Risk Play runs, the final object must contain:

- `team_id`: the exact team ID supplied by the platform request.
- `matchday_id`: the exact ID from `/workspace/game-board/matchday.json`.
- `fantasy_xi`: exactly 11 unique player ID strings.
- `risk_play`: either `null` or one valid claim object.
- `strategy`: one short explanation sentence.

# Required Bracket Final Shape

For the one-time bracket-only run, the final object must follow the active
Bracket Play schema exactly. Read `/workspace/rules/bracket-play.md` and the
current output schema before auditing.

During a bracket-only run:

- Include the full bracket fields required by the schema.
- Required top-level fields are `team_id`, `bracket_id`, `picks`, and
  `champion_team_id`; `strategy` is optional.
- Use only valid team IDs, match IDs, round IDs, and winner IDs from the
  workspace.
- Treat `phase: "bracket"` in `/workspace/game-board/matchday.json` as Bracket
  Play activation.
- Include picks for the full knockout tree.
- Ensure `champion_team_id` exactly matches the final winner selected in the
  bracket.
- Do not include `fantasy_xi` or `risk_play` unless the active bracket schema
  explicitly requires them.
- If a `strategy` field is allowed, keep it to one sentence about champion path,
  favorite/upset balance, and knockout risk posture.

# Team ID

Use the exact team ID supplied by the platform request. Do not use
`example-team`, `cris-team`, or any sample value unless that is the actual team
ID for the current platform request.

# Manual Audit Checklist

Verify all of this before returning:

- The answer follows `/workspace/output-format/daily-submission.schema.json`.
- The current `/workspace/rules/fantasy-xi.md` and
  `/workspace/rules/risk-play.md` were applied.
- The runtime `/workspace` files were treated as the source of truth for the
  Daily Answer Contract, not any stale remembered scoring or sample IDs.
- `/workspace/rules/bracket-play.md` was considered for active bracket
  requirements, especially the one-time bracket-only run.
- The example in `/workspace/output-format/examples/daily-submission.json` was
  used only as a shape guide, not as a source of sample IDs.
- No extra top-level fields are present unless the schema explicitly requires
  them.
- `team_id` is non-empty and matches the platform request.
- For a normal daily run, top-level object has `team_id`, `matchday_id`,
  `fantasy_xi`, `risk_play`, and `strategy`.
- For a normal daily run, `matchday_id` equals the value in
  `/workspace/game-board/matchday.json`.
- For a normal daily run, `fantasy_xi` has exactly 11 entries.
- For a normal daily run, every `fantasy_xi` entry is a string.
- For a normal daily run, all 11 `fantasy_xi` values are unique.
- For a normal daily run, every selected ID exists in
  `/workspace/game-board/players.json`.
- For a normal daily run, every selected player is eligible for the current
  `matchday_id`.
- For a normal daily run, formation is legal:
  - 1 GK
  - 3 to 5 DEF
  - 3 to 5 MID
  - 1 to 3 FWD
- For a normal daily run, if `risk_play` is `null`, that is valid.
- For a normal daily run, if `risk_play` is not `null`, `claim_id` exists in
  `/workspace/game-board/claim-catalog.json`.
- For a normal daily run, if `risk_play` is not `null`, every required field
  from the selected claim is present.
- For a normal daily run, every risk `match_id`, `team_id`, and `player_id`
  exists in the board files.
- For a normal daily run, Risk Play does not include `stake`, `bet_points`, or
  `stake_percent`.
- For knockout daily runs, extra time was counted for knockout final-score and
  player-event Risk Play claims when claim wording supports it.
- For knockout daily runs, penalty shootout goals were not counted as normal
  Fantasy XI goals or standard Risk Play goals.
- Knockout-only claims such as `match_goes_to_extra_time` and
  `match_goes_to_penalties` came from `claim-catalog.json` and were treated as
  Risk Play, not Bracket Play.
- Bracket fields are absent for normal daily runs.
- For a bracket-only run, Fantasy XI and Risk Play fields are absent unless the
  active schema explicitly requires them.
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
If the final XI is dominated by one favorite team, verify it is not a lazy
favorite-stack bias before accepting it.

# Availability And Upside Audit

Before returning the JSON, do one final pass over the selected XI:

- Remove any player with current ruled-out, injured, suspended, red-card ban,
  yellow-card accumulation ban, unavailable, absent, official-substitute-only,
  expected-bench, or strong fitness-doubt evidence.
- Make sure every selected player is either an official starter, a strong
  predicted starter, or the best board-only minutes option at that position.
- If official lineups are unavailable and a credible probable XI exists, any
  selected teammate missing from that probable XI must have clear replacement
  evidence or a stronger expected-points case than predicted starters elsewhere.
- Compare the lowest-upside selected DEF against the best omitted MID/FWD.
- Compare the lowest-upside selected MID against the best omitted attacker,
  set-piece taker, penalty taker, or primary creator.
- If the XI is concentrated in only two teams or skips a match entirely, verify
  the best 1 or 2 candidates from every skipped match lost a direct expected
  points comparison to the weakest selected players.
- If one team has 6 selected players, verify the 6th player clearly beats every
  omitted match's best likely-starting attacker, creator, set-piece taker, or
  penalty taker.
- If one team has 7 or more selected players on a multi-match slate, rebuild the
  XI unless the board is genuinely thin and lacks legal likely starters
  elsewhere.
- Never return an all-one-team XI on a multi-match slate.
- Verify no eligible likely-starting slate-breaker attacker was omitted without
  beating a direct comparison against the weakest selected DEF, MID, and FWD.
- If a clean-sheet stack uses GK plus 3 defenders from the same team, verify that
  the clean-sheet case is stronger than the omitted attacking alternatives.
- If public research was unavailable, say only in the `strategy` sentence that
  the selection used board starts/minutes and conservative risk. Do not add any
  explanation outside the JSON.

# Risk Object Review

Use `/workspace/game-board/claim-catalog.json` as the only source of truth for
required risk fields.

Before returning a non-null Risk Play, verify that current points, rank, recent
Risk Play results, and estimated stake justify the downside. If the team is near
the top or the Green stake is large, `risk_play: null` is preferred unless the
claim is close to obvious.

If standings show the team is far behind the lead, verify the Risk Play posture
matches catch-up mode: prefer a strong positive-value Green or very strong
Yellow claim over passive `null`, but still reject weak or contradictory claims.

Examples of valid shapes described in prose:

- A match-level Green claim needs `claim_id` and `match_id`.
- A team-level claim needs `claim_id`, `match_id`, and `team_id`.
- A player scoring claim needs `claim_id`, `match_id`, and `player_id`.
- An exact score claim needs `claim_id`, `match_id`, `home_score`, and
  `away_score`.

# Bracket Object Review

For a bracket-only run, audit the bracket before returning:

- Every predicted winner exists in the current team board.
- Every referenced match, round, slot, or path ID exists in the active bracket
  board or schema.
- `bracket_id` equals `/workspace/game-board/bracket.json` `bracket_id`.
- `picks` has one item for every required match in `bracket.json`.
- Every `picks` item has exactly `round`, `match_id`, and `winner_team_id`.
- No `picks` item has extra fields.
- Every `round` value is one of `round_of_32`, `round_of_16`, `quarterfinal`,
  `semifinal`, `third_place`, or `final`.
- Every `winner_team_id` is allowed by `bracket.json` for that match or is
  reachable from prior winners for derived later-round matches.
- If `official_knockout_fixtures_available` is false or `board_status` is
  `planning_projection`, unresolved Round of 32 teams came only from the
  relevant match's `candidate_team_ids`.
- The winner of each later round is reachable from the winners selected in prior
  rounds.
- Every match with `projection_status: "derived_from_future_results"` uses only
  winners flowing from its `source_match_ids`.
- Champion, finalists, and semifinalists form a coherent path.
- `champion_team_id` matches the selected final winner.
- Round weighting was respected: Round of 32 5, Round of 16 8, Quarterfinal 12,
  Semifinal 18, Final / Champion 30.
- `world-cup-standings.json` and `bracket-context.json` were used when present,
  especially for provisional practice boards.
- Upsets are justified by current evidence, not added randomly.
- The champion pick has a plausible route through the projected bracket.
- No stale sample IDs, placeholder teams, invented IDs, unsupported fields, or
  old draw assumptions are present.
- A third-place pick is included only if `bracket.json` has
  `third_place_included: true`; otherwise no third-place pick is present.

# Final Output Rule

Return exactly one JSON object. Do not wrap it in a Markdown fence. Do not add
any explanation before or after the object.
