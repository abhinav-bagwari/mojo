---
name: final-answer-audit
description: Validate the final daily submission JSON before returning it.
---

# Mission

Before final output, audit the candidate answer. The final response must be one
JSON object and nothing else.

# Required Final Shape

```json
{
  "team_id": "runtime-team-id",
  "matchday_id": "runtime-matchday-id",
  "fantasy_xi": ["11", "unique", "player", "ids"],
  "risk_play": null,
  "strategy": "Short explanation."
}
```

`risk_play` may also be an object with a valid `claim_id` and required fields.

# Team ID

Use the exact team ID supplied by the runtime request or system prompt. Do not
use `example-team`, `cris-team`, or any sample value unless that is the actual
runtime team ID.

# Audit Checklist

Verify all of this before returning:

- Top-level object has `team_id`, `matchday_id`, `fantasy_xi`, optional
  `risk_play`, and `strategy`.
- `team_id` is non-empty and matches the runtime request.
- `matchday_id` equals `/workspace/game-board/matchday.json`.
- `fantasy_xi` has exactly 11 strings.
- All 11 `fantasy_xi` values are unique.
- Every selected ID exists in `/workspace/game-board/players.json`.
- Every selected player is eligible for the current `matchday_id`.
- Formation is legal: 1 GK, 3 to 5 DEF, 3 to 5 MID, 1 to 3 FWD.
- If `risk_play` is not null, `claim_id` exists in `claim-catalog.json`.
- If `risk_play` is not null, every required field is present.
- Every risk `match_id`, `team_id`, and `player_id` is valid.
- No `stake`, `bet_points`, `stake_percent`, comments, Markdown, or extra prose.
- `strategy` is concise and explains the selection logic and risk posture.

# Manual Validation Snippet

If possible, validate a drafted answer with this script before final output.
Replace `ANSWER_JSON_HERE` with the candidate object.

```bash
python - <<'PY'
import json, os
from pathlib import Path

answer = json.loads(r'''ANSWER_JSON_HERE''')
root = Path(os.environ.get("CONTAINER_WORKSPACE_DIR", "/workspace"))

matchday = json.loads((root / "game-board" / "matchday.json").read_text())
players = json.loads((root / "game-board" / "players.json").read_text())
claims = json.loads((root / "game-board" / "claim-catalog.json").read_text())
matches = json.loads((root / "game-board" / "matches.json").read_text())
teams = json.loads((root / "game-board" / "teams.json").read_text())

player_by_id = {str(p["player_id"]): p for p in players}
claim_by_id = {str(c["claim_id"]): c for c in claims}
match_ids = {str(m["match_id"]) for m in matches}
team_ids = {str(t["team_id"]) for t in teams}

assert answer["matchday_id"] == matchday["matchday_id"]
xi = [str(x) for x in answer["fantasy_xi"]]
assert len(xi) == 11
assert len(set(xi)) == 11

positions = {"GK": 0, "DEF": 0, "MID": 0, "FWD": 0}
for pid in xi:
    assert pid in player_by_id, pid
    player = player_by_id[pid]
    assert answer["matchday_id"] in (player.get("eligible_matchday_ids") or []), pid
    positions[player["position"]] += 1

assert positions["GK"] == 1, positions
assert 3 <= positions["DEF"] <= 5, positions
assert 3 <= positions["MID"] <= 5, positions
assert 1 <= positions["FWD"] <= 3, positions

risk = answer.get("risk_play")
if risk is not None:
    assert "stake" not in risk
    assert "bet_points" not in risk
    assert "stake_percent" not in risk
    claim = claim_by_id[str(risk["claim_id"])]
    for field in claim["required_fields"]:
        assert field in risk, (field, risk)
    if "match_id" in risk:
        assert str(risk["match_id"]) in match_ids
    if "team_id" in risk:
        assert str(risk["team_id"]) in team_ids
    if "player_id" in risk:
        assert str(risk["player_id"]) in player_by_id

print("valid", positions)
PY
```

# Final Output Rule

Return exactly one JSON object. No Markdown fence. No explanation before or
after. No comments.

