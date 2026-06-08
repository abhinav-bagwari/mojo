---
name: pick-fantasy-xi
description: Optimize a legal 11-player Fantasy XI from the provided player board using read-only reasoning.
---

# Mission

Pick the highest expected value legal Fantasy XI while never violating the schema or roster rules. Treat this skill as read-only reasoning over the provided board files.

# Non-Negotiable Validity Rules

- Use exactly 11 unique `player_id` strings from `/workspace/game-board/players.json`.
- Every selected player must be eligible for the current `matchday_id`.
- Every selected player should belong to a team playing in `/workspace/game-board/matches.json`.
- Formation must be legal:
  - exactly 1 GK
  - 3 to 5 DEF
  - 3 to 5 MID
  - 1 to 3 FWD
- Never output names in `fantasy_xi`; output only IDs.
- If uncertain, choose a valid 4-4-2.

# Optimization Philosophy

The tournament has no budget, captain, bench, or substitution step. Expected points should dominate. Stack strong teams when justified. Do not diversify just to look balanced.

Priority order:

1. Validity and likely minutes.
2. Exact scoring-rule fit from `/workspace/rules/fantasy-xi.md`.
3. Official or strongly predicted starters.
4. High-upside attackers from favorites.
5. Goalkeepers and defenders with clean sheet potential.
6. Midfielders with set pieces, penalties, advanced role, or prior goals/assists.
7. Prior World Cup or board stats when public data is missing.
8. Avoid injuries, suspensions, bench-only players, and high card risk.

# Read The Scoring Rules First

Before ranking players, read `/workspace/rules/fantasy-xi.md` and adjust the weights below to the actual point rules. Examples:

- If goals are heavily rewarded, raise forwards, penalty takers, and advanced midfielders.
- If assists and chance creation matter, raise set-piece takers and creators.
- If clean sheets matter, raise favored goalkeepers and defenders.
- If saves matter, some underdog goalkeepers can be useful, but only if the clean-sheet gap is not decisive.
- If cards or goals conceded are punished, downgrade high-card players and weak defensive stacks.
- If minutes or starts are implied by scoring, prefer safer starters over uncertain stars.

# Board-Only Scoring Procedure

If public research is unavailable, use this manual scoring procedure as the baseline. Improve it with lineup and odds evidence only when the evidence is strong.

## Step 1: Build The Candidate Pool

- Read the current `matchday_id` from `/workspace/game-board/matchday.json`.
- Read current match team IDs from `/workspace/game-board/matches.json`.
- Keep only players from `/workspace/game-board/players.json` who:
  - contain the current `matchday_id` in `eligible_matchday_ids`
  - belong to one of the teams playing that matchday
  - have position `GK`, `DEF`, `MID`, or `FWD`

## Step 2: Estimate Team Strength

For each playing team, estimate relative team strength from available board data:

- Add credit for squad players with prior goals.
- Add credit for prior assists.
- Add credit for prior minutes and prior starts.
- Add a small bonus when the team roster has `final_squad_shape` true.
- Compare all playing teams and map them roughly into this range:
  - 1.00 heavy favorite or strongest board profile
  - 0.75 favorite
  - 0.55 even or moderate
  - 0.35 underdog
  - 0.15 heavy underdog

If public odds or reliable previews are available, use them to improve this team strength estimate.

Also estimate match goal environment:

- High goal environment favors forwards, advanced midfielders, penalty takers, and match-level goal Risk Plays.
- Low goal environment favors favorite GK and defenders, and makes early no-goal Risk Plays more attractive.
- One-sided favorite environment supports stacking the favorite and considering team-based Risk Plays.

## Step 3: Estimate Player Availability

For each candidate player, estimate likely minutes:

- Official starter: very high availability.
- Strong predicted starter: high availability.
- Prior World Cup starter with strong minutes: high availability.
- Prior substitute with limited minutes: medium to low availability.
- Injured, suspended, unavailable, or absent from current squad news: remove from consideration.
- Unknown player with no public signal: keep only as a fallback.

Board-only approximation:

- More prior starts means higher availability.
- More prior minutes per appearance means higher availability.
- No prior stats means uncertain, not impossible.

## Step 4: Score Players By Position

Use these scoring instincts, then rank each position separately.

Goalkeepers:

- Favor high start probability first.
- Favor stronger teams and clean sheet chance.
- Add value for prior saves when available.
- Avoid keepers from heavy underdogs unless save volume is likely and clean sheet edges are weak.

Defenders:

- Favor likely starters on teams with clean sheet potential.
- Prefer defenders with prior goals, assists, set-piece threat, or attacking fullback role.
- Penalize high card risk.

Midfielders:

- Favor advanced midfielders, penalty takers, set-piece takers, and creators.
- Add value for prior goals, assists, starts, and minutes.
- Defensive midfielders are acceptable only when they are reliable starters and needed for validity.

Forwards:

- Favor confirmed or likely starting forwards on favorites.
- Strongly favor penalty takers, central strikers, elite wide forwards, and players with prior goals or assists.
- Avoid rotation forwards unless public evidence suggests they start.

## Step 5: Apply Correlation Logic

Use correlation to lift the whole lineup:

- Positive correlation: GK plus 1 or 2 defenders from a favorite with clean sheet potential.
- Positive correlation: multiple attackers from a favorite or high-total match.
- Positive correlation: an attacker stack plus a Risk Play that expects goals in the same match, if the claim has strong value.
- Negative correlation: do not overload attackers against your chosen GK or defensive stack unless that attacker is clearly elite and the match is expected to be open.
- Negative correlation: avoid mixing a clean-sheet defensive stack with a Risk Play that depends on that opponent scoring.

# Formation Search

Evaluate these legal shapes and choose the best total expected value:

- 1-3-5-2
- 1-4-4-2
- 1-3-4-3
- 1-4-3-3
- 1-4-5-1
- 1-5-4-1
- 1-5-3-2

For each shape:

- Take the best available GK.
- Take the required number of best DEF, MID, and FWD candidates.
- Prefer a different shape if it admits clearly stronger attackers or avoids weak low-minute players.
- Default to 1-4-4-2 when scores are close.

# Stacking Bonuses

Stacking is allowed because there is no budget.

- Add value for GK plus 1 or 2 defenders from a favorite with clean sheet chance.
- Add value for 2 to 4 attackers from the strongest high-goal team.
- Use 2 to 5 players from a clearly superior team if they are likely starters.
- Do not stack uncertain bench players.
- Avoid exceeding 5 players from one team unless the board is thin or that team has overwhelming matchup quality.

# Improve With Public Intelligence

After the board-only ranking, adjust only when evidence is strong:

- Replace a non-starter with a confirmed or highly likely starter at the same position.
- Upgrade to penalty takers, set-piece takers, elite creators, and central forwards from favorites.
- Prefer a favorite goalkeeper over an underdog goalkeeper unless the underdog keeper is expected to face many shots and clean sheet odds are not decisive.
- For defenders, favor clean sheet probability first, then attacking fullbacks or set-piece threats.
- For midfielders, favor advanced role, set pieces, penalties, recent goal involvement, and reliable minutes.
- For forwards, favor goal probability and start probability above everything.

# Tie Breakers

When two players are close:

1. Prefer official starter.
2. Prefer better team strength.
3. Prefer higher goal or assist upside.
4. Prefer safer minutes.
5. Prefer lower card risk.
6. Prefer the player whose match has a higher goal environment.

# Final Self-Check

Before setting `fantasy_xi`, mentally count positions and IDs:

- 11 total players.
- 11 unique player IDs.
- 1 GK.
- 3 to 5 DEF.
- 3 to 5 MID.
- 1 to 3 FWD.
- All IDs are from `players.json`.
- All players are eligible for the current matchday.

# Final Fantasy XI Output

Set `fantasy_xi` to an array of 11 player ID strings only. Do not include player names, positions, notes, or ranking details inside the array.
