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

# Required Inputs

Read:

- `/workspace/game-board/claim-catalog.json`
- `/workspace/game-board/matches.json`
- `/workspace/game-board/players.json`
- `/workspace/game-board/teams.json`
- `/workspace/game-board/standings-before.json`
- `/workspace/rules/risk-play.md`

# Validity Rules

- `risk_play: null` is valid.
- If choosing a claim, `claim_id` must exist in `/workspace/game-board/claim-catalog.json`.
- Include every required field for the selected claim.
- Use valid `match_id`, `team_id`, and `player_id` values from the board.
- Do not include `stake`, `bet_points`, or `stake_percent`.
- Include required fields only unless the schema clearly allows more.

# Risk Stakes

- Green risks 15 percent of current points.
- Yellow risks 25 percent of current points.
- Red risks 35 percent of current points.

Expected value is roughly stake percent multiplied by two times success
probability minus one. Because probability estimates are noisy, use conservative
thresholds:

- Green: choose if estimated probability is at least 0.62.
- Yellow: choose if estimated probability is at least 0.70.
- Red: choose if estimated probability is at least 0.82, or at least 0.76 only
  when standings show the team needs a large catch-up play.

When the current team has little or no accumulated score, the downside of Risk
Play may be smaller, so a strong Green or Yellow claim can be worthwhile. When
the team is already near the top, protect the lead and avoid fragile Red claims.

# Tournament Position Adjustment

Use `/workspace/game-board/standings-before.json` if it contains the current
`team_id`.

- If rank is top 20 percent or tournament total is near the lead, prefer Green or
  `null`.
- If middle of the table, prefer the highest EV Green or a very strong Yellow.
- If far behind late in the tournament, consider Yellow or Red only when evidence
  is strong and the claim has real upside.

If the current team is not found in standings, use the middle-table policy.

# Claim Selection Heuristics

Prefer claims with stable event rates and low validation ambiguity:

1. `no_goal_first_10` on a match where both teams start cautiously or where the
   underdog is likely to sit deep.
2. `match_2plus_goals` on a favorite or high-total match.
3. `goal_before_halftime` on a high-tempo favorite match.
4. `match_2plus_cards` or `match_2plus_yellow_cards` on intense rivalry,
   knockout, high-foul, or underdog-defense profiles.
5. `both_teams_score` only when both attacks are strong and both defenses are
   vulnerable.
6. `team_scores_first` only for a strong favorite with reliable early pressure.
7. `player_scores` only for a confirmed starting penalty taker or elite striker.

Rank candidate claims by confidence first, then reward size. A lower-payout
Green claim with strong evidence is usually better than a fragile Yellow or Red
claim.

Use the Fantasy XI as supporting evidence:

- If the lineup stacks favorite attackers in one match, `match_2plus_goals` or
  `goal_before_halftime` may align well.
- If the lineup stacks GK and defenders from a favorite, avoid a Risk Play that
  needs the opposing team to score.
- If a player is selected mainly for penalty or striker upside, `player_scores`
  can be considered only with very strong starter and role evidence.

Avoid unless very strong evidence:

- `exact_score`
- `player_scores_2plus`
- `team_wins_by_3plus`
- `team_comeback_win`
- `red_card_shown`
- `match_goes_to_extra_time`
- `match_goes_to_penalties`

# Safe Default

If no claim is clearly strong but Green claims exist, prefer `no_goal_first_10`
on the lowest early-goal-risk match. Choose that match by looking for cautious
teams, low goal environment, underdog defensive posture, or lack of elite early
scorers.

If even that is uncertain or the claim is unavailable, return `null`.

# Final Risk Decision

Choose in this order:

1. Highest-confidence positive-value Green claim.
2. Very strong Yellow claim when probability and standings justify it.
3. Red claim only when the team needs a large catch-up play and the evidence is
   exceptional.
4. `null` when no claim clears the threshold.

After a losing Risk Play in a prior scorecard, become more conservative unless
the current standings require catch-up. After repeated correct Green claims,
continue using Green as the default risk tier rather than jumping to Red.

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
