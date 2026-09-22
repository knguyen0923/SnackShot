# SnackShot — Status

Point Claude at this file after `/clear` to pick up where things left off.
This is a snapshot, not a source of truth — always re-verify against the repo
(`git log`, `git status`) rather than trusting this blindly if it's been a
while. Detailed build history lives in `git log` and the design docs under
`docs/superpowers/`, not here — this file tracks current state only.

If `RESUME.md` says anything other than "nothing in progress," read it
first — it holds exact in-flight state from a session that paused mid-task,
which is more specific than this snapshot.

## Current state

Pre-implementation. The design spec
(`docs/superpowers/specs/2026-09-21-snackshot-design.md`) is approved; no
application code exists yet. Next step is turning that spec into an
implementation plan and starting on the first component (likely the
Telegram bot skeleton or the portion-math unit tests, TDD-first).

## Shipped features

None yet — nothing has been built.

## What's left

Everything in the design spec:

- Telegram bot layer (`/meal`, `/ingredients`, `/today`, `/week`,
  `/setgoal`, `/setmacros`)
- Detection module (pretrained YOLOv8 food model wrapper)
- Portion module (reference-object scaling, volume/density lookup table)
- Nutrition module (USDA FoodData Central lookup + fuzzy matching)
- SQLite storage (`users`, `meals` tables)
- Goal/macro calculation logic
- Deployment to a free-tier host (Railway/Render/Fly.io) in webhook mode

See the design spec for full detail on each.

## Useful pointers

- `TAKEAWAYS.md` — retrospective template; sections fill in as the project
  is actually built
- `docs/superpowers/specs/2026-09-21-snackshot-design.md` — architecture,
  scope, and component design
- `docs/superpowers/plans/` — implementation plans, once written

## How to re-orient fast

```bash
git log --oneline -10        # what shipped most recently
git status                   # anything in flight
```

<!-- STATUS_COMMIT: 0f50d39 -->
<!-- This HTML comment is machine-read by a Stop hook (.claude/settings.json)
     that nags to refresh this file whenever HEAD moves past this hash.
     Update it to the current `git rev-parse --short HEAD` every time you
     update this file. -->
