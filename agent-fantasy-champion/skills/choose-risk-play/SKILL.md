---
name: choose-risk-play
description: Select the best optional Risk Play using expected value and tournament context.
---

# Mission

Choose one positive expected value Risk Play or return `null`. Risk Play can win
the tournament, but a bad or invalid claim burns points. Never choose risk just
to look active.

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
- If choosing a claim, `claim_id` must exist in `claim-catalog.json`.
- Include every required field for the selected claim.
- Use valid `match_id`, `team_id`, and `player_id` values from the board.
- Do not include `stake`, `bet_points`, or `stake_percent`.
- Include required fields only unless the schema clearly allows more.

# Risk Stakes

- Green risks 15 percent of current points.
- Yellow risks 25 percent of current points.
- Red risks 35 percent of current points.

Expected value is roughly:

```text
stake_percent * (2 * probability_of_success - 1)
```

Because probability estimates are noisy, use conservative thresholds:

- Green: choose if estimated probability is at least 0.62.
- Yellow: choose if estimated probability is at least 0.70.
- Red: choose if estimated probability is at least 0.82, or at least 0.76 only
  when standings show the team needs a large catch-up play.

# Tournament Position Adjustment

Use `standings-before.json` if it contains the current `team_id`.

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
2. `match_2plus_goals` on a favorite/high-total match.
3. `goal_before_halftime` on a high-tempo favorite match.
4. `match_2plus_cards` or `match_2plus_yellow_cards` on intense rivalry,
   knockout, high-foul, or underdog-defense profiles.
5. `both_teams_score` only when both attacks are strong and both defenses are
   vulnerable.
6. `team_scores_first` only for a strong favorite with reliable early pressure.
7. `player_scores` only for a confirmed starting penalty taker or elite striker.

Avoid unless very strong evidence:

- `exact_score`
- `player_scores_2plus`
- `team_wins_by_3plus`
- `team_comeback_win`
- `red_card_shown`
- `match_goes_to_extra_time`
- `match_goes_to_penalties`

# Safe Default

If no claim is clearly strong but Green claims exist, prefer:

```json
{
  "claim_id": "no_goal_first_10",
  "match_id": "<lowest early-goal-risk match_id>"
}
```

Choose the lowest early-goal-risk match by looking for cautious teams, low goal
environment, underdog defensive posture, or lack of elite early scorers.

If even that is uncertain or the claim is unavailable, return:

```json
null
```

# Required Field Map

Use `claim-catalog.json` as source of truth. Common shapes:

```json
{"claim_id": "match_2plus_goals", "match_id": "1489371"}
```

```json
{"claim_id": "no_goal_first_10", "match_id": "1489371"}
```

```json
{"claim_id": "team_scores_first", "match_id": "1489371", "team_id": "6"}
```

```json
{"claim_id": "player_scores", "match_id": "1489371", "player_id": "762"}
```

```json
{"claim_id": "exact_score", "match_id": "1489371", "home_score": 2, "away_score": 1}
```

# Final Risk Policy

Pick the highest risk-adjusted expected value claim, not the flashiest claim.
When probabilities are close, pick the Green claim. When validity is uncertain,
return `null`.

