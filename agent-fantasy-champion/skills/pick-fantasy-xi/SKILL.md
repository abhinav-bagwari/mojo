---
name: pick-fantasy-xi
description: Optimize a legal 11-player Fantasy XI from the runtime player board.
---

# Mission

Pick the highest expected value legal Fantasy XI while never violating the
schema or roster rules.

# Non-Negotiable Validity Rules

- Use exactly 11 unique `player_id` strings from `/workspace/game-board/players.json`.
- Every selected player must be eligible for the current `matchday_id`.
- Every selected player should belong to a team playing in `matches.json`.
- Formation must be legal:
  - exactly 1 GK
  - 3 to 5 DEF
  - 3 to 5 MID
  - 1 to 3 FWD
- Never output names in `fantasy_xi`; output only IDs.
- If uncertain, choose a valid 4-4-2.

# Optimization Philosophy

The tournament has no budget, captain, bench, or substitution step. That means
expected points should dominate. Stack strong teams when justified. Do not
diversify merely for aesthetics.

Use this priority order:

1. Validity and likely minutes.
2. Official or strongly predicted starters.
3. High-upside attackers from favorites.
4. Goalkeepers and defenders with clean sheet potential.
5. Midfielders with set pieces, penalties, advanced role, or prior goals/assists.
6. Prior World Cup or board stats when public data is missing.
7. Avoid injuries, suspensions, bench-only players, and high card risk.

# Board-Only Baseline Optimizer

If public research is unavailable, run this as a baseline. You may improve the
result with researched lineup/odds signals, but do not break validity.

```bash
python - <<'PY'
import json, os
from pathlib import Path

root = Path(os.environ.get("CONTAINER_WORKSPACE_DIR", "/workspace"))

matchday = json.loads((root / "game-board" / "matchday.json").read_text())
matches = json.loads((root / "game-board" / "matches.json").read_text())
players = json.loads((root / "game-board" / "players.json").read_text())
teams = json.loads((root / "game-board" / "teams.json").read_text())

matchday_id = matchday["matchday_id"]
team_meta = {str(t["team_id"]): t for t in teams}
playing_team_ids = {
    str(m["home_team_id"]) for m in matches
} | {
    str(m["away_team_id"]) for m in matches
}

def number(value, default=0.0):
    try:
        return float(value)
    except Exception:
        return default

def prior(player):
    return player.get("prior_stats") or player.get("prior_world_cup_record") or {}

def availability_score(player):
    stats = prior(player)
    apps = number(stats.get("appearances"), 0)
    starts = number(stats.get("starts"), 0)
    minutes = number(stats.get("minutes"), 0)
    if apps > 0:
        start_signal = min(1.0, (starts + 0.35) / (apps + 0.7))
        minute_signal = min(1.0, minutes / max(90.0, apps * 90.0))
    else:
        start_signal = 0.55
        minute_signal = 0.50
    return 35 * start_signal + 20 * minute_signal

def team_strength_map():
    raw = {}
    for tid in playing_team_ids:
        squad = [p for p in players if str(p.get("team_id")) == tid]
        scored = 0.0
        for p in squad:
            st = prior(p)
            scored += 4 * number(st.get("goals"))
            scored += 3 * number(st.get("assists"))
            scored += 0.01 * number(st.get("minutes"))
            scored += 0.5 * number(st.get("starts"))
        meta = team_meta.get(tid, {})
        if (((meta.get("roster_checks") or {}).get("final_squad_shape")) is True):
            scored += 4
        raw[tid] = scored
    if not raw:
        return {}
    lo, hi = min(raw.values()), max(raw.values())
    if hi == lo:
        return {tid: 0.55 for tid in raw}
    return {tid: 0.25 + 0.75 * ((v - lo) / (hi - lo)) for tid, v in raw.items()}

team_strength = team_strength_map()

def player_score(player):
    pos = player.get("position")
    tid = str(player.get("team_id"))
    stats = prior(player)
    strength = team_strength.get(tid, 0.45)

    goals = number(stats.get("goals"))
    assists = number(stats.get("assists"))
    saves = number(stats.get("saves"))
    yellow = number(stats.get("yellow_cards"))
    red = number(stats.get("red_cards"))
    starts = number(stats.get("starts"))
    minutes = number(stats.get("minutes"))
    attacking = goals * 7 + assists * 5 + min(6, minutes / 90.0) + starts
    card_penalty = yellow * 1.2 + red * 5

    score = availability_score(player) + 10 * strength - card_penalty
    if pos == "GK":
        score += 18 * strength + 2.0 * saves
    elif pos == "DEF":
        score += 15 * strength + 2.5 * goals + 2.0 * assists
    elif pos == "MID":
        score += 8 * strength + 1.3 * attacking
    elif pos == "FWD":
        score += 10 * strength + 1.8 * attacking + 5 * goals
    return score

eligible = []
for p in players:
    if matchday_id not in (p.get("eligible_matchday_ids") or []):
        continue
    if str(p.get("team_id")) not in playing_team_ids:
        continue
    if p.get("position") not in {"GK", "DEF", "MID", "FWD"}:
        continue
    row = dict(p)
    row["_score"] = player_score(row)
    eligible.append(row)

by_pos = {}
for pos in ["GK", "DEF", "MID", "FWD"]:
    by_pos[pos] = sorted(
        [p for p in eligible if p["position"] == pos],
        key=lambda p: (p["_score"], p.get("display_name", "")),
        reverse=True,
    )

formations = [
    (3, 5, 2),
    (4, 4, 2),
    (3, 4, 3),
    (4, 3, 3),
    (5, 4, 1),
    (5, 3, 2),
    (3, 3, 3),
]

def roster_bonus(roster):
    bonus = 0.0
    gk = next((p for p in roster if p["position"] == "GK"), None)
    if gk:
        same_team_defs = [
            p for p in roster
            if p["position"] == "DEF" and str(p["team_id"]) == str(gk["team_id"])
        ]
        bonus += min(2, len(same_team_defs)) * 3.0 * team_strength.get(str(gk["team_id"]), 0.4)

    attackers_by_team = {}
    for p in roster:
        if p["position"] in {"MID", "FWD"}:
            attackers_by_team.setdefault(str(p["team_id"]), 0)
            attackers_by_team[str(p["team_id"])] += 1
    for tid, count in attackers_by_team.items():
        if count >= 2:
            bonus += min(count, 4) * 2.5 * team_strength.get(tid, 0.4)
    return bonus

best = None
for d_count, m_count, f_count in formations:
    if not by_pos["GK"] or len(by_pos["DEF"]) < d_count or len(by_pos["MID"]) < m_count or len(by_pos["FWD"]) < f_count:
        continue
    roster = (
        by_pos["GK"][:1]
        + by_pos["DEF"][:d_count]
        + by_pos["MID"][:m_count]
        + by_pos["FWD"][:f_count]
    )
    total = sum(p["_score"] for p in roster) + roster_bonus(roster)
    if best is None or total > best[0]:
        best = (total, roster, (d_count, m_count, f_count))

if not best:
    raise SystemExit("No legal roster can be built from the board.")

total, roster, formation = best
print("matchday_id:", matchday_id)
print("formation: 1-%d-%d-%d" % formation)
print("score:", round(total, 2))
for p in roster:
    print(p["position"], p["player_id"], p.get("display_name"), "team", p.get("team_id"), "score", round(p["_score"], 2))
print("fantasy_xi:", json.dumps([str(p["player_id"]) for p in roster]))
PY
```

# Improve The Baseline With Public Intelligence

After the baseline, adjust only when evidence is strong:

- Replace a non-starter with a confirmed or highly likely starter at the same
  position.
- Upgrade to penalty takers, set-piece takers, elite creators, and central
  forwards from favorites.
- Prefer a favorite goalkeeper over an underdog goalkeeper unless the underdog
  keeper is expected to face many shots and clean sheet odds are not decisive.
- For defenders, favor clean sheet probability first, then attacking fullbacks or
  set-piece threats.
- For midfielders, favor advanced role, set pieces, penalties, recent goal
  involvement, and reliable minutes.
- For forwards, favor goal probability and start probability above everything.

# Aggressive But Safe Stacking

Because there is no budget, stacking the best teams is allowed.

- Use 2 to 5 players from a clearly superior team if they are likely starters.
- Use GK plus 1 or 2 defenders from a favorite when clean sheet chances are high.
- Use 2 to 4 attackers from the highest goal-environment favorite.
- Do not stack uncertain bench players.

# Final Fantasy XI Output

Produce `fantasy_xi` as an array of 11 strings:

```json
["player_id_1", "player_id_2", "player_id_3", "player_id_4", "player_id_5", "player_id_6", "player_id_7", "player_id_8", "player_id_9", "player_id_10", "player_id_11"]
```

Do not include player names, positions, or explanations inside the array.

