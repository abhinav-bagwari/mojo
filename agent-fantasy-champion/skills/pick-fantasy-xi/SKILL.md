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
8. Avoid injuries, suspensions, red-card bans, yellow-card accumulation bans,
   bench-only players, and high card risk.

Hard rule: never keep a player with current evidence that they are ruled out,
injured, suspended, red-card banned, yellow-card accumulation banned,
unavailable, absent from the squad, or expected not to start. That player scores
too close to zero to justify the pick.

# Read The Scoring Rules First

Before ranking players, read `/workspace/rules/fantasy-xi.md` and adjust the weights below to the actual point rules. Examples:

- If goals are heavily rewarded, raise forwards, penalty takers, and advanced midfielders.
- If assists and chance creation matter, raise set-piece takers and creators.
- If clean sheets matter, raise favored goalkeepers and defenders.
- If saves matter, some underdog goalkeepers can be useful, but only if the clean-sheet gap is not decisive.
- If cards or goals conceded are punished, downgrade high-card players and weak defensive stacks.
- If minutes or starts are implied by scoring, prefer safer starters over uncertain stars.

# Expected Points Model

Use the exact scoring rules in `/workspace/rules/fantasy-xi.md`. When that file
uses the standard Daily Answer Contract values, rank players with this mental
model:

- start floor: `2 * start_probability`
- 60-minute floor: `2 * sixty_minute_probability`
- goals: `6 * goal_probability`
- assists: `4 * assist_probability`
- clean sheet for GK/DEF: `4 * clean_sheet_probability`
- GK save bonus: `2 * three_save_probability`
- cards and own goals: subtract expected yellow, red, and own-goal penalties

Do not overfit the numbers. The purpose is to compare options:

- A safe starter who reaches 60 minutes is worth about 4 points before events.
- A defender or goalkeeper from a strong clean-sheet spot can reach about 8
  points without an attacking return.
- A forward or advanced midfielder with real goal/assist chance can beat a
  fourth or fifth defender even without a clean sheet.
- A non-playing star is worth 0 and must lose to almost any legal starter.

When `/workspace/rules/fantasy-xi.md` differs from these values, rebuild the
same expected-value comparison from the actual rule text before choosing
formation.

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
- Injured, suspended, red-card banned, yellow-card accumulation banned,
  unavailable, or absent from current squad news: remove from consideration.
- Unknown player with no public signal: keep only as a fallback.

Availability gates:

- Official starter: set `start_probability` near 1.00 and `sixty_minute_probability`
  high unless the player is returning from injury.
- Strong same-day predicted starter: set `start_probability` around 0.85 to 0.95.
- Uncertain role, rotation warning, or fitness doubt: downgrade below safer legal
  alternatives.
- Current ruled-out, suspended, red-card ban, yellow-card accumulation ban,
  unavailable, absent, or expected-bench signal: remove from the XI even if the
  player is famous or has strong prior stats.
- If the best public evidence is stale, fall back to board starts and minutes,
  but do not ignore current negative news.

Board-only approximation:

- More prior starts means higher availability.
- More prior minutes per appearance means higher availability.
- No prior stats means uncertain, not impossible.

Before formation search, create an avoid list of removed players and do not add
them back unless newer official evidence clears the concern.

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
- A fourth or fifth defender needs either strong clean-sheet probability or real
  attacking upside. Do not use extra defenders only because they are on a
  favorite.

Midfielders:

- Favor advanced midfielders, penalty takers, set-piece takers, and creators.
- Add value for prior goals, assists, starts, and minutes.
- Defensive midfielders are acceptable only when they are reliable starters and needed for validity.
- Prefer attacking midfielders from favored or high-total matches over low-upside
  holding midfielders when starter certainty is similar.

Forwards:

- Favor confirmed or likely starting forwards on favorites.
- Strongly favor penalty takers, central strikers, elite wide forwards, and players with prior goals or assists.
- Avoid rotation forwards unless public evidence suggests they start.
- If one forward is ruled out or doubtful, replace them before optimizing any
  other marginal position.

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
- Compare the last player admitted by each shape. Choose the shape that avoids
  the weakest low-upside starter, not the shape that merely looks balanced.

# Formation Selection Guide

Let the rules and match environment drive formation:

- If goals and assists are the dominant scoring path, prefer 1-3-4-3, 1-3-5-2,
  or 1-4-3-3 when strong attackers are available.
- If clean sheets are valuable and there is a clear defensive favorite, prefer
  1-4-4-2, 1-5-3-2, or 1-5-4-1 only when the defender pool is strong.
- If midfielder scoring is broad because of assists, set pieces, and minutes,
  prefer 1-3-5-2 or 1-4-5-1.
- If the board has weak forward options, do not force three forwards.
- If the board has weak defenders, do not force five defenders.
- Prefer 1-3-4-3 or 1-3-5-2 when three forwards or five midfielders are all
  likely starters with better goal/assist expectation than the fourth defender.
- Prefer 1-4-4-2 only when the fourth defender's clean-sheet and attacking value
  beats the next legal midfielder or forward.

# Stacking Bonuses

Stacking is allowed because there is no budget.

- Add value for GK plus 1 or 2 defenders from a favorite with clean sheet chance.
- Add value for 2 to 4 attackers from the strongest high-goal team.
- Use 2 to 5 players from a clearly superior team if they are likely starters.
- Do not stack uncertain bench players.
- Avoid exceeding 5 players from one team unless the board is thin or that team has overwhelming matchup quality.
- Avoid using GK plus 3 defenders from one team unless official lineups and
  matchup evidence give a strong clean-sheet case and the alternatives are
  meaningfully weaker.
- Do not let a defensive stack block elite attackers or set-piece midfielders
  from another match with better expected goal involvement.

# Portfolio Balance

Build a lineup with multiple scoring paths:

- At least one clean-sheet path through a goalkeeper or defender stack when the
  match slate offers a clear favorite.
- At least four players with real goal or assist upside when the board allows it.
- At least one set-piece, penalty, or primary-creator profile when available.
- Avoid filling the XI with low-upside defensive midfielders unless they are
  needed for validity and minutes safety.
- Avoid picking both a goalkeeper and several opposing attackers unless the
  attacker upside is clearly worth the negative correlation.

# Whole-Slate Challenger Review

Do not force representation from every match, but do not ignore a match without
checking its best options.

Before finalizing the XI:

- Identify the best 1 or 2 fantasy candidates from each match on the slate using
  the current scoring rules, likely minutes, and role upside.
- If the XI has 8 or more players from only two teams, compare the weakest
  selected players against those omitted challenger candidates.
- If the XI uses 5 defenders, compare the weakest selected defender against the
  best omitted MID or FWD from every match.
- If one match has no selected players, verify that its top challenger candidates
  are weaker than the final selected player at the same position or through a
  legal formation change.
- Keep the concentrated build only when the selected player clearly has better
  expected points. If the comparison is close, prefer the challenger with safer
  minutes, set pieces, penalties, central attacking role, or higher goal
  environment.

# Improve With Public Intelligence

After the board-only ranking, adjust only when evidence is strong:

- Replace a non-starter with a confirmed or highly likely starter at the same position.
- Replace any selected player with same-day ruled-out, injury, suspension,
  red-card ban, yellow-card accumulation ban, or expected-bench evidence.
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
- No selected player is on the avoid list.
- Every selected player has either official starter evidence, strong predicted
  starter evidence, or the best available board-only minutes case.
- The weakest selected defender was compared against the best omitted MID/FWD.
- The weakest selected defensive midfielder was compared against the best omitted
  attacker, creator, or set-piece taker.
- Any match with no selected players had its top 1 or 2 candidates compared
  against the weakest selected players before accepting the final XI.

# Final Fantasy XI Output

Set `fantasy_xi` to an array of 11 player ID strings only. Do not include player names, positions, notes, or ranking details inside the array.
