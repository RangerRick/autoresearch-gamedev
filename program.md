# autoresearch-gamedev

This is an autonomous experiment loop for iterating on the **blocks** game
(`~/git/blocks`) — a Lumines-inspired puzzle game where 2×2 pieces fall onto a
grid and a sweeper clears monochromatic 2×2 squares.

The loop mirrors Karpathy's autoresearch: make a change, run a fixed evaluation,
keep if better, revert if worse, repeat indefinitely.

## Setup

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar19`).
   The branch `autoresearch/<tag>` must not already exist.
2. **Create the branch** inside `~/git/blocks`:
   `git checkout -b autoresearch/<tag>`
3. **Read the in-scope files** for full context:
   - `lib/config/game_config.dart` — all tunable constants (grid size, timing,
     scoring, level progression). Primary target.
   - `lib/game/systems/scoring_system.dart` — scoring logic.
   - `lib/game/systems/clearing_system.dart` — 2×2 detection and clear logic.
   - `lib/game/systems/gravity_system.dart` — cell gravity after clears.
   - `lib/game/models/piece.dart` — piece patterns and rotation.
   - `bin/evaluate.dart` — **fixed single-skin harness. Do not modify.**
   - `bin/evaluate_skins.dart` — **fixed multi-skin harness. Do not modify.**
4. **Initialize results.tsv** in `~/git/autoresearch-gamedev/` with header row
   and baseline entry. Run `dart run bin/evaluate_skins.dart` from `~/git/blocks`
   once to establish YOUR baseline.
5. **Confirm and go.**

## Evaluation

Run from `~/git/blocks`:

```
dart run bin/evaluate_skins.dart > run.log 2>&1
```

The script simulates 200 games per skin (1000 total) with a fixed greedy AI,
then prints per-skin engagement scores and an aggregate:

```
---
dawn    : engagement=1.502092  gini_q=0.992  progression=1.273  cv=0.190  mean_score=56367.8
classic : engagement=1.205102  gini_q=0.962  progression=0.957  cv=0.310  mean_score=42368.7
dusk    : engagement=1.124923  gini_q=0.955  progression=0.881  cv=0.338  mean_score=39698.5
midnight: engagement=0.872096  gini_q=0.927  progression=0.666  cv=0.412  mean_score=34356.8
void    : engagement=0.736437  gini_q=0.915  progression=0.580  cv=0.387  mean_score=31515.8
---
aggregate_engagement: 1.088130
difficulty_spread:    2.040  (dawn/void ratio — higher = clearer beginner/expert gap)
eval_games:           1000 (200 per skin × 5 skins)
eval_seconds:         6.4
```

**The goal: maximize `aggregate_engagement`.** Higher is better.

Each skin uses the same formula (from `evaluate.dart`):
- `gini_quality = 4 × gini × (1 − gini)` — peaks at gini=0.5 (healthy mix of scoring moments)
- `progression = mean(last_50_deltas) / mean(all_150_deltas)` — rewards late-game acceleration
- `cv_across_bonus = min(cv_across, 0.5)` — rewards replayability across seeds
- `engagement = gini_quality × progression × (1 + cv_across_bonus)`

`aggregate_engagement = mean(engagement across all 5 skins)`.

Read results with:

```
grep "^aggregate_engagement:" run.log
```

If the grep output is empty, the run crashed. Read `tail -n 50 run.log` for the
Dart stack trace.

## Skin difficulty tiers

The 5 skins span beginner → expert via piece distribution (checker% = harder):

| Skin     | mono% | checker% | Difficulty   |
|----------|-------|----------|--------------|
| dawn     | 35    | 15       | beginner     |
| classic  | 25    | 25       | medium       |
| dusk     | 25    | 30       | medium-hard  |
| midnight | 20    | 35       | hard         |
| void     | 15    | 35       | expert       |

The research goal is to find parameters that:
1. **Maximize average fun** across all 5 skins (the primary metric).
2. **Preserve difficulty spread** — dawn should feel clearly easier than void.
   `difficulty_spread` (dawn/void engagement ratio) is printed as a secondary
   signal; a drop below ~1.5 suggests the skins have converged (bad).
3. **Support early-to-late game arc** — `progression` measures whether the game
   accelerates over its 150-piece run. Good values are 0.8–2.0; below 0.7 means
   the game front-loads scoring or never builds.

## What you CAN edit

Any file in `~/git/blocks/lib/` **except** files that only affect rendering,
audio, or Flutter widgets. Focus on:

- `lib/config/game_config.dart` — easiest wins (grid size, scoring constants,
  level speed, combo thresholds)
- `lib/game/systems/scoring_system.dart` — scoring formula changes
- `lib/game/systems/clearing_system.dart` — clearing mechanics
- `lib/game/models/piece.dart` — add/remove/change piece patterns

## What you CANNOT edit

- `bin/evaluate.dart` — the fixed single-skin evaluation harness
- `bin/evaluate_skins.dart` — the fixed multi-skin evaluation harness
- Anything under `lib/game/components/`, `lib/screens/`, `lib/audio/` — these
  are rendering/UI layers that have no effect on the simulation

## Simplicity criterion

Same as autoresearch: simpler is better all else equal. A 1-point improvement
that adds 20 lines of convoluted code is not worth it. Deleting code and
getting equal-or-better results is a great outcome.

## Experiment loop

The branch lives in `~/git/blocks`. All git operations happen there.

LOOP FOREVER:

1. Check current branch/commit: `git log --oneline -5`
2. Tune one or more in-scope files with an experimental idea.
3. `git add lib/... && git commit -m "experiment: <description>"`
   (never `git add -A`)
4. Run: `dart run bin/evaluate_skins.dart > run.log 2>&1`
5. Read results: `grep "^aggregate_engagement:" run.log`
6. If grep is empty: run crashed. Read `tail -n 50 run.log`, fix and re-run.
   Give up after ~3 failed attempts on the same idea.
7. Log result to `~/git/autoresearch-gamedev/results.tsv`
8. If `aggregate_engagement` improved (higher): keep — continue from here.
9. If equal or worse: log as `discard`, then
   `git reset --hard <previous kept commit>`.

Each eval run should complete in under 15 seconds. If it exceeds 60 seconds,
something is wrong — kill it and investigate.

**NEVER STOP.** Once the loop begins, do not pause to ask whether to continue.
The human may be away. Run until manually interrupted.

## Logging results

`~/git/autoresearch-gamedev/results.tsv` — tab-separated, 5 columns:

```
commit	aggregate_engagement	difficulty_spread	status	description
```

1. 7-char git commit hash from `~/git/blocks`
2. aggregate_engagement (6 decimal places, e.g. `1.088130`)
3. difficulty_spread (3 decimal places, e.g. `2.040`)
4. status: `keep`, `discard`, or `crash`
5. short description of the experiment

Example:

```
commit	aggregate_engagement	difficulty_spread	status	description
5b7882d	1.088130	2.040	keep	baseline: 5 skins, placement bonus, soft decay
...
```
