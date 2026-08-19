---
name: design-flow
description: Sketch the flow you actually want before touching code — for any change, including docs, config, and workflows. Use at the start of a change, when an abstraction feels wrong but you can't say why, when weighing two implementations, or when a mid-change discovery tempts you to bolt something on. In a repo with its own design-flow skill, defer to that one.
user-invocable: true
---

# Design Flow

**Code follows shape.** If the shape is wrong, no amount of implementation polish saves it; if it's right, the implementation usually writes itself. A two-minute sketch costs nothing next to reverting a wrong one.

Scale it to the work — a few sentences for a config tweak, a full diagram with seam analysis for a new boundary. The discipline doesn't change, only the depth.

## Two rules, both load-bearing

**Pretend the implementation doesn't exist.** Draw what *should* flow through the system if you designed it fresh today. The temptation is to draw "what's there, plus my change" — that just re-encodes the existing constraints and teaches you nothing.

**Don't presume your constraints are real either.** When you reach for an awkward shape — a mirrored structure, a side-band registry, "we can't share this across X" — stop and ask what assumption is pushing you there, then go verify it. Thirty seconds of `grep`/`cat`/`ls` beats hours designing around a phantom limit. The hardest constraints to challenge are the ones absorbed implicitly, and most are partly or wholly false on inspection.

Both rules apply to the **premise**, not just the shape. Ask "should this behavior exist at all?" before designing machinery around it — correctly-reasoned work built on a defect is still waste.

**A repeat pass is where these silently stop firing.** A second, narrower sketch inherits the first one's mental model, and Rule 1 quietly degrades into tuning whatever already exists. The tell: every option on the table is a variation on the same mechanism — thresholds, cadences — rather than a different shape. Step one level up before continuing.

## Verify the claims your scope rests on

**"X is the only place that does Y" is the most consequential claim you'll make**, because it bounds the whole change — and an unverified "only" is the most common scoping error and the worst-timed, surviving the sketch, the plan, and the implementation before a sibling turns up at review. Enumerate before committing. "There's no precedent for this" is the same claim inside out, about absence rather than extent, and just as often false.

**Those announce themselves with a word. A bare list doesn't.** An enumeration that gates behavior — the cases that count as acceptable, what a step owns — claims nothing about its own completeness; it just lists, and the reader supplies the exhaustiveness. That's why it survives review: everything it says is true, so it never contradicts anything. When a list will be acted on, the question isn't whether its entries are right but whether a case exists that belongs and isn't there. Ask it as you write it — no later reader has a reason to.

## Then compare

Only once the ideal is on paper and the constraints are verified, compare to current state. **The delta is the work.** A small delta means the existing shape was mostly right. A huge one means the "small" change you were about to make was load-bearing on a wrong abstraction.

**Precondition: know the territory before sketching it.** You should be able to name the concrete actors in the flow, what the data looks like at each boundary, and what changed recently and why. If you can't, go read first — a sketch drawn from issue titles produces the wrong shape, which is the one thing this exists to prevent.
