# SnackShot — Telegram Meal/Ingredient Photo Estimator

**Date:** 2026-09-21
**Status:** Approved for implementation planning

## Purpose

A Telegram bot that estimates calories, macros (protein/carbs/fat), and
portion sizes from a photo of a meal or a photo of raw ingredients.
Built as a portfolio/learning project demonstrating a classical
computer-vision pipeline (detection → portion sizing → nutrition
lookup) rather than relying on a multimodal LLM to do the estimation.
Tracks daily intake against a calorie/macro goal (2000 kcal default).

## Non-goals (v1)

- No user-facing web app — Telegram is the only interface.
- No training of a custom detection model from scratch — start from a
  pretrained model and improve later if needed.
- No handling of complex stacked/mixed dishes with high accuracy —
  acceptable known limitation for v1.
- No multi-user auth beyond Telegram's own user ID.

## Architecture

```
Telegram user sends photo (with /meal or /ingredients command context)
        │
        ▼
Telegram Bot (python-telegram-bot, webhook mode, deployed on free-tier host)
        │
        ▼
CV Pipeline
  1. Detection — pretrained YOLOv8 food-detection model (fine-tuned on
     Food-101 or similar, pulled from Roboflow/HuggingFace), returns
     bounding boxes/masks + class labels per food item.
  2. Portion sizing — reference-object scaling:
       - Detect a reference object in frame (standard plate ≈26cm
         diameter, or fallback to a visible utensil).
       - Compute pixel-to-cm scale from the reference.
       - Segment each food item's area, convert to estimated volume
         using a per-food-category height/density lookup table.
       - Convert volume to grams via food density.
       - If no reference object is detected, fall back to a fixed
         plate-size assumption and flag the estimate as "rough" in
         the reply.
  3. Nutrition lookup — for each (food label, grams) pair, query USDA
     FoodData Central, fuzzy-match the label to the best FDC entry,
     scale macros to the estimated grams.
        │
        ▼
Aggregate totals (calories/protein/carbs/fat) → reply to user
        │
        ▼
Persist meal to storage (SQLite) keyed by Telegram user_id
        │
        ▼
On request (/today, /week): sum totals, compare against user's goal
```

## Components

1. **Telegram bot layer**
   - Two photo intents, selected explicitly (CV alone can't reliably
     distinguish "prepared dish" from "raw ingredients laid out"):
     - `/meal` then send photo → whole-dish detection + portion +
       aggregate macros.
     - `/ingredients` then send photo → per-ingredient detection +
       portion + per-ingredient macro breakdown (no "dish" inference).
   - `/today`, `/week` — report consumed vs. goal.
   - `/setgoal <kcal>` — set daily calorie goal, recompute macro
     targets from the default split unless macros were explicitly
     overridden.
   - `/setmacros <protein_g> <carb_g> <fat_g>` — explicit macro target
     override.

2. **Detection module**
   - Wraps the pretrained YOLOv8 food model.
   - Input: image. Output: list of `{label, confidence, mask/bbox}`.

3. **Portion module**
   - Input: detection output + original image.
   - Output: list of `{label, estimated_grams, confidence_note}`.
   - Contains the reference-object detection, px/cm scale calc, and
     per-category volume/density lookup table (small hardcoded table
     for v1: e.g. rice, chicken, leafy greens, egg, bread, generic
     fallback density).

4. **Nutrition module**
   - Input: list of `{label, estimated_grams}`.
   - Queries USDA FoodData Central API, caches label→FDC ID matches
     (avoid re-querying identical labels).
   - Output: list of `{label, grams, calories, protein_g, carb_g, fat_g}`.
   - If no confident match, surface the top fuzzy match to the user in
     the reply with a "did we get this right?" confirm/correct button;
     corrections are logged for future model/matching improvement but
     v1 does not retrain on them automatically.

5. **Storage** (SQLite, file-based — simplest to run on a free-tier host)
   - `users`: `user_id (pk), daily_calorie_goal (default 2000), protein_g_goal, carb_g_goal, fat_g_goal, macros_overridden (bool), created_at`
   - `meals`: `id (pk), user_id (fk), timestamp, mode ('meal'|'ingredients'), items (json), total_calories, total_protein_g, total_carb_g, total_fat_g`

6. **Goal/macro calculation**
   - Default split when macros are not overridden: 30% calories from
     protein, 40% from carbs, 30% from fat.
     - `protein_g = goal_kcal * 0.30 / 4`
     - `carb_g = goal_kcal * 0.40 / 4`
     - `fat_g = goal_kcal * 0.30 / 9`
   - `/setgoal` recalculates these unless `macros_overridden` is true.
   - `/setmacros` sets `macros_overridden = true` and stores explicit
     gram targets directly.

7. **Reporting**
   - `/today`: sums today's `meals` rows for the user, compares to
     goals:
     ```
     Today: 1,340 / 2,000 kcal (67%)
     Protein: 62 / 150 g
     Carbs:   140 / 200 g
     Fat:     45 / 67 g
     ```
   - `/week`: same, summed/averaged over the last 7 days.

## Deployment

- Free-tier cloud host (Railway, Render, or Fly.io) so the bot is
  always reachable from a phone without keeping a local machine on.
- Telegram bot runs in webhook mode against that host.
- SQLite file persisted on the host's disk (acceptable for v1 single
  small-scale deployment; note as a future migration point to Postgres
  if the host's disk isn't persistent across deploys).

## Error handling / accuracy caveats

- No reference object detected → fixed-plate-size fallback, reply
  flags the estimate as rough.
- Unmatched/low-confidence food label in USDA lookup → show best
  fuzzy match with a confirm/correct button.
- Overlapping/stacked foods → known v1 weak point, not specially
  handled; acceptable limitation to document.
- Telegram photo upload failures / unsupported formats → bot replies
  with a clear error and re-prompts for a photo.

## Testing

- Unit tests: portion math (area → volume → grams) against synthetic
  geometry fixtures.
- Unit tests: nutrition label-matching logic against a fixture set of
  known food labels and expected FDC matches.
- Unit tests: goal/macro split calculation and `/setgoal` /
  `/setmacros` recalculation logic.
- Manual end-to-end test: send real meal and ingredient photos via
  Telegram, verify replies and that `/today` totals accumulate
  correctly.

## Open items deferred beyond v1

- Fine-tuning the detection model on user-corrected labels.
- Postgres migration if free-tier disk persistence becomes an issue.
- Handling stacked/mixed dishes more accurately (e.g. depth
  estimation).
