# Agent Team Debate: Engagement Metrics for a Lumines-like Game
**Date:** 2026-03-19
**Team:** metric-debate
**Agents:** player-feel, game-designer, lumines-scholar, metrics-skeptic

---

## Background

We have been iterating on a headless simulation metric (200 games × 150 pieces, greedy AI) for a Lumines-inspired puzzle game called "blocks". Each metric design has been gamed by monotonically bumping a single parameter:

1. `cv_within × (1 + cv_across)` — gamed by higher power exponents on squaresCleared
2. `gini_quality × (1 + cv_across)` — gamed by fast multiplier advance rate
3. `gini_quality × progression × (1 + min(cv_across, 0.5))` — gamed by raising the level exponent (level¹ → level² → level³ kept improving)

The team was asked to debate what dimensions of engagement a better metric should capture, with particular attention to gaming resistance.

---

## Position Papers

### player-feel: Player Experience Perspective

**The Core Experience**

What makes Lumines feel good isn't high scores — it's the rhythm of *building toward something and being rewarded for it*. A satisfying session has a heartbeat: tension as the board fills, release as a well-planned clear sweeps through, then the exciting decision of what to build next. Any metric that doesn't capture this tension-release cycle is measuring the wrong thing.

**Dimension 1: Agency — Did the Player's Decisions Matter?**

What it means: The difference between a skilled player's outcome and a random player's outcome should be large. In a single session, this manifests as: did placing pieces deliberately (setting up 2×2 squares, choosing color placement) produce better results than just dropping them anywhere?

How to measure: Compare the greedy AI's score distribution against a random-placement baseline. The ratio (or gap) between them is a proxy for how much skill the system rewards. Alternatively, measure the correlation between "number of deliberate 2×2 setups" and final score within the AI's own games.

How it could be gamed: A system that gives massive points for trivially easy clears (e.g., every piece auto-clears) would show high scores for both skilled and random play, collapsing the gap. Or a system where one dominant strategy always wins regardless of board state.

Prevention: Require that the skill gap exceeds a minimum threshold AND that multiple distinct board patterns produce clears (not just one degenerate layout).

**Dimension 2: Momentum — Does Play Build Toward Crescendos?**

What it means: Good Lumines sessions have escalating intensity. Early game is methodical; mid-game builds combos; late game produces satisfying chain reactions. Score-per-piece should trend upward within a session, not be flat or front-loaded.

How to measure: Compute the slope of score-per-piece over the course of a game (split into quartiles). A positive slope means the game rewards sustained play with growing returns. Measure this as `mean(Q3_score + Q4_score) / mean(Q1_score + Q2_score)` — a "crescendo ratio."

How it could be gamed: An exponential level multiplier makes late-game points trivially larger regardless of play quality. The crescendo ratio goes up, but it's the formula doing the work, not the player.

Prevention: Measure crescendo ratio on the *raw clear counts* (squares cleared per piece), not on points. If the player is clearing more squares per piece as the game progresses, they're genuinely building momentum through board management, not just riding a multiplier.

**Dimension 3: Recovery — Can the Player Come Back from Bad Luck?**

What it means: Checker pieces (the "junk" piece) are part of Lumines' design tension. A good system lets a skilled player absorb bad piece sequences and recover, rather than making one unlucky streak fatal. This is what makes players say "one more game" — the belief that they *could have* recovered.

How to measure: Track the variance of scores across seeds. Specifically, look at the ratio of the 25th-percentile game to the 75th-percentile game. A healthy ratio (say, 0.4–0.7) means bad-luck games are worse but not catastrophically so. Too low means luck dominates; too high means pieces don't matter.

How it could be gamed: Reducing checker piece frequency trivially reduces variance. Or making all pieces score identically regardless of placement.

Prevention: Fix the piece distribution (it already is fixed by the evaluation harness). Measure recovery specifically: after a sequence of N checker pieces, how many subsequent pieces does it take to return to the pre-checker scoring rate?

**Dimension 4: Readable Board State — Does the Player Understand What's Happening?**

What it means: Players feel engaged when they can *see* their next move. A board full of scattered single cells is overwhelming; a board with clear color regions and obvious 2×2 opportunities feels inviting. This is the difference between "I'm losing and I don't know why" and "I see three possible setups."

How to measure: Track the average number of "near-complete" 2×2 squares (3 of 4 cells filled with the same color) on the board at each piece placement. More near-completes means more visible opportunities, more meaningful choices per turn.

How it could be gamed: Making the grid tiny or piece colors uniform would trivially maximize near-completes.

Prevention: Grid size and color count are fixed by the game design. Validate that near-complete count stays within a plausible band (not saturated).

**Summary**

The metric should combine: **agency** (skill matters), **momentum** (play escalates), **recovery** (bad luck isn't fatal), and **readability** (the board invites decisions). Any single dimension can be gamed; the combination resists gaming because optimizing one at the expense of others produces degenerate play that fails on the remaining dimensions. The key insight is: measure the *shape* of a game session, not just its final number.

---

### game-designer: Game Design Theory Perspective

**The Core Problem**

Every metric so far has been a single number derived from score distributions. Score is downstream of *every* parameter simultaneously, so optimizing any score-derived metric just finds whichever parameter has the steepest monotonic gradient. We need to measure properties of *gameplay trajectories*, not just final scores.

**Dimension 1: Decision Density (proxy for meaningful choices)**

What: The fraction of piece placements that *change the board state in a non-trivial way* — specifically, that create or destroy at least one potential 2×2 square. A game where most placements are "dead" (filling space with no clearing consequence) has low decision density.

Why it matters: In good puzzle games, nearly every placement matters. If 80% of pieces just pile up inertly, the game has no tension. High decision density means the board state is rich enough that placement location changes outcomes.

How to measure: For each placed piece, check whether the number of marked/markable 2×2 squares changed. Report `pieces_with_impact / total_pieces`.

How it gets gamed: Making clears trivially easy (e.g., comboThreshold=1, huge grid) means every piece trivially "matters." Constraint: Decision density must be paired with a minimum clearing difficulty — e.g., average squares-per-clear must stay above a threshold, or the grid fill percentage must stay above some floor at mid-game.

**Dimension 2: Sustained Tension (proxy for tension-and-release rhythm)**

What: The variance of grid fill level *within* a game, measured as the standard deviation of fill percentage sampled at each piece placement.

Why it matters: Good Lumines play has a rhythm: the board fills up (tension), then a well-timed clear drops it (release). A flat fill curve means the game is either trivially easy (always low) or hopelessly stagnant (always high). A game with high fill variance has dramatic swings — close calls followed by satisfying clears.

How to measure: Sample `filled_cells / total_cells` after every piece lock. Compute the standard deviation across the game.

How it gets gamed: Making the sweeper very slow would create artificial tension (board fills, then mass clear). Constraint: The sweeper speed is a fixed game constant, not a tunable parameter. Additionally, require that the *mean* fill level stays in a healthy band (say 20–60%) — too low means trivial, too high means the AI is barely surviving.

**Dimension 3: Score Acceleration (proxy for progression arc)**

What: Whether scoring rate increases over the course of a game. Measure the ratio of points-per-piece in the second half vs. the first half.

Why it matters: Good games have a *shape*. Early game should feel like building capability (combos, level advancement). Late game should feel powerful — your accumulated skill/position pays off. A flat scoring rate means no progression. An extreme ratio means early game is pointless.

How to measure: `(total_points_pieces_76_to_150) / (total_points_pieces_1_to_75)`. Call this the acceleration ratio.

How it gets gamed: A steep level exponent (level^4, level^5) trivially inflates late-game points. Constraint: Cap the level exponent as a fixed constant. Alternatively, measure acceleration on *squares cleared per piece* rather than points — this strips out the scoring formula entirely and measures whether the player's *board control* improves, which is the real design goal.

**Dimension 4: Seed Fairness (proxy for skill over luck)**

What: The correlation between a game's final score and properties of its piece sequence (e.g., fraction of checker pieces, longest monochrome run).

Why it matters: In a skill-expressive game, a good player can handle bad luck. If final scores are highly predicted by piece sequence composition, the game is luck-dominated. Low correlation means the scoring system and board mechanics give players enough agency to overcome unfavorable draws.

How to measure: Across 200 games, regress final score against piece-sequence features (checker fraction, monochrome fraction, longest same-pattern run). Report R-squared. Lower is better.

How it gets gamed: A completely flat scoring system (every piece = same points regardless of clears) would show zero correlation but would also be a terrible game. Constraint: Seed fairness is only meaningful when paired with score variance being non-trivial (cv_across > some minimum). The game must produce score differentiation; it just shouldn't be *explained by* the random seed.

**Synthesis**

A robust composite metric would combine these four dimensions multiplicatively so that gaming any single one at the expense of others collapses the product. Each dimension has a natural "healthy range" rather than a monotonic "higher is better" property, which is precisely what resists parameter hill-climbing. The key insight: measure properties of the *gameplay trajectory* (fill curves, decision impacts, scoring arcs), not just properties of the *final score distribution*.

---

### lumines-scholar: Lumines Domain Perspective

**The Core Loop Lumines Rewards**

Lumines is not a score-attack game. It is a *rhythm-of-tension* game. The sweeper creates a fixed tempo: you place pieces freely, but consequences arrive on a beat. The best Lumines sessions have a specific shape — you build up a messy board, find the key placement that links scattered same-color regions into a massive connected clear, the sweeper wipes it, and you breathe. Then the tension rebuilds. The worst sessions are when the board is either trivially empty (no challenge) or hopelessly clogged (no agency). The franchise rewards players who can sustain tension at a productive level.

This is the fundamental insight: **Lumines engagement comes from variance in board state over time, not from final score magnitude.**

**1. Board Utilization Variance (the "Breathing" signal)**

In a good Lumines session, board fill oscillates — it rises as you accumulate pieces, then drops sharply after a sweeper clear. Measure the standard deviation of board fill percentage sampled at regular intervals (e.g., every 5 pieces). A flat low fill means the game is too easy; a flat high fill means the player is drowning; healthy oscillation means the push-pull tension loop is working.

How it gets gamed: A scoring system could alternate between trivially easy and impossibly hard phases, creating artificial oscillation. Constraint: Require that the mean fill stays within a healthy band (25%–75%) and that oscillation frequency correlates with actual clears, not random noise. Measure fill only at piece-placement events, not arbitrary time steps.

**2. Clear Size Distribution (the "Chain Reaction" signal)**

Lumines's most satisfying moments are chain clears — where one 2×2 clear causes gravity to form another 2×2, which the sweeper also catches. The original Lumines scoring (50 × cells × combo) explicitly rewards this: more cells cleared in one sweep pass = disproportionately more points. A healthy game should produce a mixture of small maintenance clears (2–4 squares) and occasional large combo clears (8+ squares). Measure the Gini coefficient of clear sizes within a game.

How it gets gamed: Making clears artificially rare but huge (e.g., nothing clears until the board is nearly full, then everything clears at once). Constraint: Require a minimum clear frequency — at least one clear event every N pieces (in Lumines, going more than ~8–10 pieces without any clear usually means you're losing). Penalize games where more than 30% of pieces are placed with zero clears.

**3. Piece Survival Depth (the "Agency" signal)**

In Lumines, a good scoring/clearing system means that *how* you place a piece matters beyond the immediate moment. Pieces placed 10–20 moves ago should still be on the board sometimes, contributing to future clears. Measure the average "lifespan" of placed cells (how many pieces elapse between placement and clearing). If everything clears immediately, there's no strategic depth. If nothing ever clears, the game is broken.

How it gets gamed: Delaying all clears artificially inflates lifespan. Constraint: Combine with clear frequency (property 2) and board utilization (property 1). High lifespan is only good when clears are also happening regularly.

**4. Cross-Game Score Spread (the "Replayability" signal)**

Lumines uses randomized piece sequences, and part of the magic is that some sequences feel "hot" (lots of monochrome pieces, easy to build with) while others feel "dead" (checker-heavy, hard to organize). But the game should be winnable with any sequence — the spread in final scores across different seeds should be moderate. Too little spread means piece sequence doesn't matter (boring). Too much means the game is pure luck.

How it gets gamed: Flattening all scores to identical values. Constraint: Require a minimum CV across games (say 0.15) to ensure seeds matter, while capping it (say 0.6) to ensure skill/system matters more than luck.

**Summary Table**

| Property | Measures | Lumines Justification | Gaming Risk | Constraint |
|---|---|---|---|---|
| Board fill variance | Tension oscillation | Sweeper rhythm creates build/release | Artificial difficulty swings | Mean fill in 25–75% band; oscillation tied to clears |
| Clear size Gini | Chain reaction payoff | 50×cells×combo rewards big clears | All-or-nothing clearing | Minimum clear frequency per N pieces |
| Cell lifespan | Strategic depth | Pieces contribute to future combos | Delayed clearing inflation | Must co-occur with healthy clear frequency |
| Cross-game CV | Replayability / seed sensitivity | "Hot" vs "dead" sequences are part of the feel | Score flattening | Bounded CV range (0.15–0.6) |

The key principle: no single number should be the metric. A composite that requires *all four* properties to be in healthy ranges is much harder to game than any one-dimensional score, because optimizing one dimension at the expense of others will be caught by the others.

---

### metrics-skeptic: Measurement Theory Perspective

**The Pattern**

Looking at results.tsv, the optimization history tells a clear story. The loop found that raising the level exponent from linear to quadratic to cubic monotonically improved the composite metric (1.365 → 1.618 → 1.720). It only stopped at level^4 because gini_quality finally penalized it enough. But that "stop" was accidental — the metric didn't encode "level^3 is right"; it just happened to run out of headroom on one axis before the other compensated.

This is the structural problem: **any metric that multiplies independent components can be gamed by maximizing the component with the most remaining headroom.** The optimizer treats the metric as a Lagrangian and finds the cheapest dimension to push.

**Why Every Proposed Metric Is Gameable**

The fundamental issue is a mismatch between what we're measuring and what we want.

What the simulation measures: Statistical properties of score distributions produced by a greedy AI playing 150 pieces. That's it. The AI doesn't get bored. It doesn't feel satisfaction from a well-timed combo. It doesn't experience tension when the board fills up.

What we want: A game that produces interesting *decisions* for *humans* — moments of tension, relief, escalation, surprise. These are inherently qualitative and contextual.

The greedy AI has a fixed strategy (maximize same-color neighbors, prefer lower placement). Any scalar metric computed from its score trace is really measuring "how does this scoring formula interact with this specific heuristic?" That's a two-body problem with a small parameter space — the optimizer will always find the seam.

**Specific Structural Failures**

1. Progression is monotone in the level exponent. Higher exponents mechanically make later pieces worth more. The metric rewards this up to the point where gini_quality objects, but gini_quality's penalty is smooth and shallow — it doesn't create a hard wall.

2. cv_across is monotone in score variance amplifiers. Anything that makes scores more sensitive to early luck (high exponents, multiplicative chains) pumps cv_across. The 0.5 cap helps but just shifts the game to the other factors.

3. gini_quality peaks at gini=0.5 by design, but the optimizer routes around it. It can keep gini near 0.5 while cranking other terms. The quadratic penalty (4g(1-g)) is too gentle — it's still 0.75 at gini=0.25 or 0.75.

**What Would Actually Work: Non-Monotone Constraints**

The key insight: we need properties where **both extremes are bad** and the penalty for leaving the good zone is steep, not gentle.

Concrete non-monotone properties:

1. **Board fill fraction at game-end.** Too empty (< 20%) means clearing is trivially easy — no tension. Too full (> 80%) means the player is overwhelmed. Target: 40–60%.

2. **Fraction of pieces that trigger at least one clear.** Too low (< 15%) means most moves are boring stacking. Too high (> 60%) means there's no buildup phase. Target: 25–40%.

3. **Mean combo length (consecutive sweeps with multiplier advancing).** Too short (mean < 1.5) means combos are rare — no payoff for setup. Too long (mean > 5) means the multiplier advances trivially — no skill expression.

4. **Piece-to-piece score autocorrelation.** Near zero means scores are random noise (no momentum). Near 1.0 means the game is deterministic/snowbally. Target: 0.3–0.6 — you want streaks but not guarantees.

**The Framework: Pass/Fail Gates + Bounded Optimization**

Instead of one scalar to maximize, use:

**Layer 1: Hard gates (pass/fail).** The game must satisfy ALL of these or the experiment is rejected outright:
- Mean board fill at game-end between 30–70%
- Clear frequency per piece between 20–50%
- Mean combo length between 1.5 and 6.0
- No single piece accounts for > 5% of total score (anti-jackpot)
- Gini coefficient between 0.3 and 0.7

**Layer 2: Bounded scalar.** Only among configs that pass all gates, optimize a simple metric like `mean_score` or `progression`. Since the gates constrain the shape of the game, the scalar can be simple — it doesn't need to encode "what good looks like" because the gates already do.

This is much harder to game because the optimizer can't just push in one direction. Lowering comboThreshold to pump combo length will eventually violate the gini gate. Raising the level exponent to pump progression will violate the board-fill gate (scores get so back-loaded the AI changes placement strategy). Each gate creates a wall, not a slope.

**What the Simulation Cannot Measure**

No amount of metric engineering fixes these gaps:
- Decision quality: the greedy AI doesn't explore the decision tree.
- Aesthetic satisfaction: the visual/audio experience of a chain clear is a huge part of Lumines engagement.
- Difficulty curve: with fixed piece count and no game-over, we can't measure whether the game gets appropriately harder.

These require either a smarter AI (MCTS or learned policy) or human playtesting. The simulation can only validate that the scoring *structure* produces reasonable statistical properties — it should be a filter, not an oracle.

**Recommendation**

Replace the single engagement_score with a gate-check harness. Define 5–7 bounded constraints on measurable game properties. Reject experiments that violate any gate. Among passing experiments, use a simple metric (even just mean_score) as a tiebreaker.

---

## Cross-Responses

### player-feel cross-response

**Most compelling ideas:**

Metrics-skeptic's pass/fail gates. This is the strongest proposal on the table. Every previous metric got gamed because a single scalar always has a gradient to climb. Gates eliminate that failure mode entirely — you can't "optimize" board fill once you're inside 30–70%. This directly supports my agency and recovery dimensions: a config that passes all gates necessarily produces games where skill matters and bad luck isn't fatal. I'd adopt this as the primary architecture.

Lumines-scholar's cell lifespan. This is a dimension none of us explicitly named but it maps perfectly to player feel. Short lifespans mean pieces are cleared quickly (satisfying feedback loops). Long lifespans mean dead weight accumulating (frustration). Bounding mean cell lifespan creates a natural rhythm constraint that's hard to game without actually producing good clearing dynamics.

**One pushback:**

Game-designer's decision density metric (fraction of placements changing markable 2×2 count) risks rewarding chaotic board states where *every* placement shifts the landscape. A board in constant flux isn't necessarily readable or strategic — it can feel random. I'd want decision density bounded above as well as below, or paired with my readability measure to ensure those decisions are legible to the player.

---

### game-designer cross-response

**Strongest endorsement: metrics-skeptic's pass/fail gates.** This is the single most important structural insight across all four papers. My proposed "healthy ranges" were gesturing at the same idea, but gates are cleaner — they make the constraint *binary* rather than hoping a multiplicative term will self-balance. Gates directly prevent the monotonic hill-climbing failure mode we keep hitting. I'd adopt gates as the architecture and slot trajectory-based measurements (mine and lumines-scholar's) into them.

**Second endorsement: lumines-scholar's board fill oscillation.** This is a better formulation of my "sustained tension" dimension. Oscillation variance specifically captures the breathing rhythm of fill-up-then-clear, whereas raw standard deviation could be satisfied by a single dramatic swing. The oscillation framing is more precise about *what pattern* we want.

**Pushback: player-feel's agency metric (skill vs random baseline gap).** A greedy AI is not a skill proxy — it's a *heuristic* proxy. Widening the gap between greedy and random could be achieved by making the greedy heuristic's preferred moves disproportionately rewarded, which optimizes for *that specific AI* rather than for human skill expression. The gap measures heuristic sensitivity, not decision depth. I'd drop this or replace it with seed fairness (low R-squared of score on piece-sequence features), which measures the same underlying concern without privileging a specific play strategy.

---

### lumines-scholar cross-response

**Strongest endorsement: metrics-skeptic's pass/fail gates.** This is the single most important structural insight across all four papers. Every metric we've tried has been gamed because the optimizer finds whichever multiplicative term has unbounded headroom. Gates eliminate that failure mode entirely. I'd adopt the gate architecture as the framework and slot the other proposals' measurements into it.

**Second endorsement: game-designer's decision density.** This captures something my "cell lifespan" metric was reaching toward but more directly — does placement *matter*? A game where most placements are strategically inert is boring regardless of what the score curve looks like. Decision density is also hard to game because inflating it requires genuinely making more board states consequential.

**Pushback: player-feel's skill-vs-random baseline gap.** Measuring against a random player sounds elegant, but our greedy AI is itself a fixed strategy, not a skilled human. The gap between greedy and random tells us about the *heuristic's* exploitation of the scoring system, not about whether a human would feel agency. It risks optimizing for "greedy-exploitable" rather than "humanly satisfying." I'd drop it in favor of decision density, which measures agency at the placement level without needing a reference player.

---

### metrics-skeptic cross-response

**Most compelling ideas:**

Decision density from game-designer is the strongest proposal across all papers. It measures something the simulation *can* faithfully capture — whether piece placement actually matters — without routing through the scoring formula. A game where 80% of placements change the markable 2×2 count is genuinely more interesting than one where most moves are inert stacking. Crucially, this is naturally non-monotone: if clearing is too easy, every placement trivially changes the board; if too hard, none do.

Cell lifespan from lumines-scholar is the other standout. It captures board "metabolism" — how quickly placed material cycles through the system. Short lifespans mean instant gratification with no buildup. Long lifespans mean the board is stagnant. This gives us a direct proxy for tension/release rhythm without trying to infer it from score distributions.

**Pushback:**

Player-feel's skill-vs-random baseline gap sounds appealing but is computationally expensive (doubling simulation cost) and deeply sensitive to the choice of "random" baseline. A slightly smarter random policy dramatically changes the gap. We'd be optimizing the metric's reference point, not the game. Decision density captures agency more directly without the reference-policy problem.

---

## Synthesized Consensus

### Architecture: Pass/Fail Gates + Simple Tiebreaker

Replace the engagement_score scalar with a gate-check harness. All gates must pass for an experiment to be accepted. Among passing experiments, optimize a formula-independent tiebreaker.

### Proposed Gates

| Gate | Property | Healthy Range | Why bounded |
|---|---|---|---|
| Clear rate | % of pieces triggering a clear | 20–50% | Too low = boring stacking; too high = no buildup |
| Decision density | % of placements changing markable 2×2 count | 25–65% | Too low = inert moves; too high = chaotic noise |
| Board fill oscillation | σ of fill% sampled per piece | σ > 0.05 | Flat = no drama; must co-occur with healthy mean fill |
| Mean cell lifespan | Mean pieces from placement to clearing | 5–30 pieces | Too short = shallow; too long = stagnant |
| Score Gini | Gini of per-piece score deltas | 0.30–0.70 | Healthy distribution (not equal, not all-or-nothing) |
| Seed CV | Cross-game score CV | 0.15–0.60 | Some variation = good; luck-dominated = bad |

### Tiebreaker (score-formula-independent)

`clear_acceleration = mean(last_50_clears_per_piece) / mean(first_50_clears_per_piece)`

Measures whether the game clears *more effectively* late vs. early — based on raw clearing behavior, not on score formula. Rewards slow-burn board management without being gameable via level exponents.

### What Was Dropped

- **Skill-vs-random baseline gap**: 3/4 agents pushed back. Measures heuristic sensitivity, not human skill expression. Decision density is a cleaner proxy for agency.

### Implementation Note

Gate ranges need empirical calibration against the current baseline. Run a diagnostic first to find where the baseline sits on all six properties, then set ranges that are tight enough to be meaningful but wide enough that reasonable game variants pass.
