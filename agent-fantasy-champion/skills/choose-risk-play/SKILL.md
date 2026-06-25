---
name: choose-risk-play
description: Select the best optional Risk Play using expected value and tournament context.
---

# Mission

Choose one positive expected value Risk Play or return `null`. Risk Play can
help win the tournament, but a bad or invalid claim burns points. Never choose a
risk just to look active.

The best Risk Play is not always the most exciting one. It is the claim with the
best mix of probability, payout, standings need, and validity confidence.

Recent scorecards matter. If a previous Risk Play missed and cost a large stake,
be materially more conservative on the next scored day unless standings show the
team is far behind and needs catch-up variance.

# Required Inputs

Read:

- `/workspace/game-board/claim-catalog.json`
- `/workspace/game-board/matches.json`
- `/workspace/game-board/players.json`
- `/workspace/game-board/teams.json`
- `/workspace/game-board/standings-before.json`
- `/workspace/current-standings/` when present
- `/workspace/rules/risk-play.md`

Also use any current team points or rank stated in the platform prompt. Risk
appetite should follow the best available current score context.

# Validity Rules

- `risk_play: null` is valid.
- If choosing a claim, `claim_id` must exist in `/workspace/game-board/claim-catalog.json`.
- Include every required field for the selected claim.
- Use valid `match_id`, `team_id`, and `player_id` values from the board.
- Do not include `stake`, `bet_points`, or `stake_percent`.
- Include required fields only unless the schema clearly allows more.

# Knockout Scoring Rules

Knockout matches add important claim semantics:

- Extra time counts for knockout final-score and player-event Risk Play claims
  when the claim wording covers the full knockout match.
- Penalty shootout goals do not count as normal goals for standard Risk Play
  goal claims unless the claim explicitly says it is about the shootout.
- `match_goes_to_extra_time` and `match_goes_to_penalties` are knockout-only
  Risk Plays. Use them only when they exist in `claim-catalog.json`.
- These knockout Risk Plays are part of the normal daily Fantasy XI + Risk Play
  flow, not the separate Bracket Play flow.
- Bracket Play opens later with the Round of 32 bracket and produces bracket
  JSON only.

# Risk Stakes

- Green risks 15 percent of current points.
- Yellow risks 25 percent of current points.
- Red risks 35 percent of current points.
- The tournament may seed teams with starting points before the first scored
  day. Use `/workspace/game-board/standings-before.json` as the source of truth
  for current points rather than assuming day-one score is zero.
- Stakes are rounded by the tournament; do not include `stake`, `bet_points`, or
  `stake_percent` in the answer.

Expected points are approximately:

`stake_points * (2 * success_probability - 1)`

For example, a 40-point stake needs a probability above 50 percent to be
positive expected value, but noisy estimates require a much higher confidence
margin. Use conservative thresholds:

- Green: choose if estimated probability is at least 0.62.
- Yellow: choose if estimated probability is at least 0.70.
- Red: choose if estimated probability is at least 0.82, or at least 0.76 only
  when standings show the team needs a large catch-up play.

Use higher thresholds when the stake is large or the team is already in a strong
position:

- If current points make a Green stake 25 points or more, choose Green only when
  estimated probability is at least 0.72 and supported by multiple independent
  signals such as odds, previews, team style, and lineup context.
- If the team is top 10, top 20 percent, or within one good day of the lead,
  prefer `null` unless a Green claim is close to obvious, at least 0.75
  probability, and low ambiguity.
- If the previous scored day had a missed Risk Play of 25 points or more, return
  `null` by default for the next day unless the team is far behind and the claim
  clears the high-stake threshold.
- Do not choose Yellow or Red while protecting a top-table position. Use them
  only for clear catch-up mode with exceptional evidence.

When the current team has 0 or fewer points, the downside of Risk Play may be
zero. Otherwise, every wrong Risk Play subtracts real tournament points. When
the team is already near the top, protect the lead and avoid fragile Red claims.
Do not force a claim after a strong Fantasy XI; `null` is better than a
low-confidence bet.

# Tournament Position Adjustment

Use `/workspace/game-board/standings-before.json` if it contains the current
`team_id`.

- If rank is top 20 percent or tournament total is near the lead, prefer `null`
  unless a Green claim clears the high-stake/top-table threshold.
- If middle of the table, prefer the highest EV Green or a very strong Yellow.
- If far behind late in the tournament, consider Yellow or Red only when evidence
  is strong and the claim has real upside.
- If the gap to first place is about 60 points or more, or one normal strong day
  cannot close the gap, enter catch-up mode: do not default to `null` merely
  because yesterday's Risk Play missed. Search for the best positive-value Green
  or very strong Yellow claim, while still rejecting low-confidence claims.

If the current team is not found in standings, use the middle-table policy.

# Catch-Up Mode

Catch-up mode means calculated upside, not gambling for its own sake.

Use catch-up mode when standings, prompt notes, or scorecards show the team is
well outside the top group or roughly 60 or more points behind the lead.

In catch-up mode:

- A strong Green claim with multiple signals is preferred over `null`.
- A Yellow claim can be selected when it has strong evidence and a realistic
  chance to close meaningful ground.
- Red claims remain rare. Use Red only when the gap is very large, the evidence
  is exceptional, and safer claims cannot materially help.
- Avoid exact scores, comeback wins, red cards, extra time, and penalties unless
  the board context makes them unusually strong.
- Do not choose a Risk Play that contradicts the Fantasy XI's main stack.
- If every available claim is weak or ambiguous, return `null`; losing another
  large stake is worse than waiting for a better edge.

# Claim Selection Heuristics

Prefer claims with stable event rates and low validation ambiguity:

1. `no_goal_first_10` on a match where both teams start cautiously or where the
   underdog is likely to sit deep.
2. `no_goal_stoppage_time` on a match without strong late-chaos indicators.
3. `match_2plus_goals` on a favorite or high-total match.
4. `goal_before_halftime` on a high-tempo favorite match.
5. `match_2plus_cards` or `match_2plus_yellow_cards` on intense rivalry,
   knockout, high-foul, or underdog-defense profiles.
6. `both_teams_score` only when both attacks are strong and both defenses are
   vulnerable.
7. `team_scores_first` only for a strong favorite with reliable early pressure.
8. `player_scores` only for a confirmed starting penalty taker or elite striker.

Rank candidate claims by confidence first, then reward size. A lower-payout
Green claim with strong evidence is usually better than a fragile Yellow or Red
claim.

Use the exact current claim catalog. If a preferred claim is missing, do not
invent it. Move to the next valid claim or return `null`.

Risk Play evidence must be current enough for the matchday. Do not choose a
player-scoring or team-scoring claim from stale role assumptions when same-day
lineup or injury evidence is missing.

Use the Fantasy XI as supporting evidence:

- If the lineup stacks favorite attackers in one match, `match_2plus_goals` or
  `goal_before_halftime` may align well.
- If the lineup stacks GK and defenders from a favorite, avoid a Risk Play that
  needs the opposing team to score.
- If a player is selected mainly for penalty or striker upside, `player_scores`
  can be considered only with very strong starter and role evidence.
- If the XI avoids a match because it is low quality, chaotic, or uncertain,
  prefer not to anchor the Risk Play there unless the selected claim is stable
  and Green.

Avoid unless very strong evidence:

- `exact_score`
- `player_scores_2plus`
- `team_wins_by_3plus`
- `team_comeback_win`
- `red_card_shown`
- `match_goes_to_extra_time`
- `match_goes_to_penalties`

For `match_goes_to_extra_time`, look for an evenly matched knockout tie,
conservative styles, strong defenses, low goal environment, and no clear
favorite. For `match_goes_to_penalties`, require an even tighter match profile:
low-scoring expectation, resilient goalkeepers, penalty competence, and neither
team having a reliable late-match edge.

# Safe Default

If no claim is clearly strong but Green claims exist, prefer `no_goal_first_10`
on the lowest early-goal-risk match only if it clears the Green probability
threshold. Choose that match by looking for cautious teams, low goal
environment, underdog defensive posture, or lack of elite early scorers.

If even that is uncertain, below threshold, or the claim is unavailable, return
`null`.

Do not use the safe default if the lowest-risk match cannot be identified from
board data, current previews, or odds. In that case, return `null`.

# Final Risk Decision

Choose in this order:

1. Catch-up mode: highest-confidence positive-value Green or very strong Yellow
   claim that can close meaningful ground.
2. `null` when the team is protecting a strong standing and no Green claim is
   near-obvious.
3. Highest-confidence positive-value Green claim.
4. Very strong Yellow claim when probability and standings justify it.
5. Red claim only when the team needs a large catch-up play and the evidence is
   exceptional.
6. `null` when no claim clears the threshold.

After a losing Risk Play in a prior scorecard, become much more conservative
unless the current standings require catch-up. After repeated correct Green
claims, continue using Green as the default risk tier rather than jumping to
Yellow or Red.

# Required Field Guide

Use `/workspace/game-board/claim-catalog.json` as source of truth.

- Match-level claims such as `match_2plus_goals` and `no_goal_first_10` need
  `claim_id` and `match_id`.
- Team-level claims such as `team_scores_first` need `claim_id`, `match_id`, and
  `team_id`.
- Player scoring claims need `claim_id`, `match_id`, and `player_id`.
- Exact score claims need `claim_id`, `match_id`, `home_score`, and `away_score`.

# Final Risk Policy

Pick the highest risk-adjusted expected value claim, not the flashiest claim.
When probabilities are close, pick the Green claim. When validity is uncertain,
return `null`.
