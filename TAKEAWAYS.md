# Building SnackShot — Project Takeaways

*This is a living retrospective, started before implementation began. The
sections below marked "TBD" fill in as the project is actually built — don't
backfill them with speculation, only with what actually happened.*

A retrospective on building SnackShot: a Telegram bot that estimates
calories, macros, and portion sizes from a photo of a meal or raw
ingredients, using a classical computer-vision pipeline (detection →
portion sizing → nutrition lookup) rather than a multimodal LLM. Built as
a portfolio/learning project. Design approved 2026-09-21; build start TBD.

## What it does

SnackShot turns a food photo into a nutrition estimate through three
distinct stages rather than one end-to-end model call:

- **Detection** — a pretrained YOLOv8 food-detection model finds and
  labels each food item in the photo.
- **Portion sizing** — a reference object in frame (a standard plate, or
  a visible utensil as fallback) gives a pixel-to-cm scale; each item's
  segmented area is converted to volume via a per-category height/density
  lookup, then to grams.
- **Nutrition lookup** — each `(label, grams)` pair is matched against
  USDA FoodData Central and scaled to the estimated grams.

Two intents are selected explicitly by the user (`/meal` vs.
`/ingredients`) rather than inferred, because CV alone can't reliably
tell "prepared dish" from "raw ingredients laid out."

TBD: whether keeping detection, portion sizing, and nutrition lookup as
separate, independently-testable stages holds up the way PikaRAG's
RAG/damage-calc split did, once real photos start hitting the pipeline.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Detection | Pretrained YOLOv8 (fine-tuned on Food-101 or similar) | Classical CV pipeline over an LLM-vision guess — portion/nutrition math needs to be traceable, not "plausibly right" |
| Portion sizing | Reference-object pixel/cm scaling + hardcoded density table | No depth sensor or 3D model available from a single phone photo; a known reference object is the only free source of real-world scale |
| Nutrition data | USDA FoodData Central API | Authoritative, free, public nutrition database |
| Bot framework | `python-telegram-bot`, webhook mode | Native Telegram integration; webhook avoids a always-polling process |
| Storage | SQLite, file-based | Simplest thing that runs on a free-tier host for a single-user-scale v1 |
| Hosting | Free-tier host (Railway, Render, or Fly.io — TBD) | Always-on, reachable from a phone without a local machine staying on |
| Testing | TBD | Plan: unit tests for portion math and label-matching against fixtures; manual end-to-end via real Telegram photos |

## Architecture

```
Telegram user sends photo (with /meal or /ingredients command context)
        │
        ▼
Telegram Bot (python-telegram-bot, webhook mode, free-tier host)
        │
        ▼
CV Pipeline
  1. Detection    — pretrained YOLOv8 food model → {label, confidence, mask/bbox}
  2. Portion sizing — reference-object scale → area → volume → grams
  3. Nutrition lookup — USDA FoodData Central, fuzzy-matched per label
        │
        ▼
Aggregate totals (calories/protein/carbs/fat) → reply to user
        │
        ▼
Persist meal to SQLite, keyed by Telegram user_id
        │
        ▼
On request (/today, /week): sum totals, compare against user's goal
```

See `docs/superpowers/specs/2026-09-21-snackshot-design.md` for full
component-level detail.

## What I actually learned, by area

*TBD — fill in per area as the corresponding piece gets built, the same
way PikaRAG's takeaways grew section by section (RAG internals, bot
framework quirks, deployment gotchas, working with an AI assistant).
Likely candidate areas based on the design, to confirm or revise once
real:*

### Classical CV pipeline vs. LLM vision
### Reference-object portion estimation in practice
### USDA FoodData Central matching quality
### Deploying a webhook-mode Telegram bot on a free-tier host
### Working with an AI coding assistant deliberately

## By the numbers

TBD — commit count, test count, detection accuracy, portion-estimate
error margins, per-query cost, once there's a build to measure.

## Quantifiable changes

TBD — numbers that moved during development (e.g. detection accuracy
after a model swap, portion-error reduction after a density-table fix),
tracked as they happen rather than reconstructed after the fact.

## What I'd do differently / next

TBD — populate from `docs/superpowers/specs/2026-09-21-snackshot-design.md`'s
"Open items deferred beyond v1" once those have actually been revisited:
fine-tuning on user-corrected labels, Postgres migration if free-tier disk
persistence becomes an issue, better handling of stacked/mixed dishes.
