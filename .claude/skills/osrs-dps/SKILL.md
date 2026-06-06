---
name: osrs-dps
description: Use when working with Old School RuneScape DPS, damage, accuracy, or combat formulas in this calculator — max hit, attack/defence rolls, hit chance, effective levels, prayer/potion boosts, spell max hits, hit distributions, time-to-kill (TTK), special attacks, or running, testing, or debugging the PlayerVsNPCCalc / BaseCalc engine.
---

# OSRS DPS (calculator engine)

## Overview

This calculator (OSRS Wiki's, based on the community combat pseudocode) computes how much damage a player loadout does to a monster per second. The engine is `PlayerVsNPCCalc` (player attacking NPC), extending `BaseCalc` (shared math + gear predicates); `NPCVsPlayerCalc` models the reverse (damage taken).

The pipeline, end to end:

```
attack roll (accuracy) ─┐
                        ├─► hit chance ─┐
NPC defence roll ───────┘              ├─► hit distribution ─► expected damage ─► DPS
max hit (min..max) ────────────────────┘                                          (and TTK)
```

**For the data models this consumes, see the `osrs-weapons` and `osrs-monsters` skills.** This skill is the math that ties them together.

## The two rules that cause most errors

1. **Integer truncation at every step.** OSRS combat is integer math. Almost every operation is `Math.trunc(...)` (truncate toward zero), and the *order* of floor operations is load-bearing — you cannot algebraically simplify. The universal "apply a factor" op is `trackFactor(base, [num, div]) = Math.trunc(base * num / div)` (`BaseCalc.ts:109`): multiply first, then divide, then truncate. Each prayer/gear multiplier is applied sequentially with its own truncation — they do **not** sum.

2. **Logic is scattered; grepping one file misleads.** Shadow's magic scaling is in `Equipment.ts`; all defence reductions are in `scaling/DefenceReduction.ts` (applied during `scaleMonster`, before any roll); multi-hit/proc effects are in `getDistributionImpl`/`getAttackerDist`; bolt and claw distributions are in `src/lib/dists/`.

## Where the logic lives

| Concern | Location |
|---|---|
| Shared math, accuracy formulas, gear predicates | `src/lib/BaseCalc.ts` |
| Attack rolls (per style), max hits, hit chance, DPS, TTK | `src/lib/PlayerVsNPCCalc.ts` |
| NPC defence roll | `PlayerVsNPCCalc.ts:147` (`getNPCDefenceRoll`) |
| Hit distribution math, transformers | `src/lib/HitDist.ts` |
| Bolt / claw distributions | `src/lib/dists/{bolts,claws}.ts` |
| Integer helpers (`lerp`, `iSqrt`, `addPercent`) | `src/lib/Math.ts` |
| Constants (`SECONDS_PER_TICK`, id groups) | `src/lib/constants.ts` |
| Step-by-step breakdown tracking | `src/lib/CalcDetails.ts` |
| Loadout comparison charts | `src/lib/Comparator.ts` |
| Tests / ground-truth numbers | `src/tests/calc/`, `src/tests/*.test.ts` |

`reference.md` has the full per-style derivations, prayer/potion/spell tables, distribution internals, and the test-harness template.

## Core formulas (the load-bearing ones)

**Effective level** (per style) = `baseLevel + boost`, then **each prayer factor applied in sequence (its own `trunc`)**, then `+ stanceBonus`, then `× voidFactor`. (Prayers are applied one at a time, not as a single combined factor — sequential truncation matters.) Stance/void bonuses differ by style — melee/ranged base +8, magic base +9; void melee/ranged ×11/10, magic ×29/20. (Boosts come from potions; see `reference.md`.)

**Max attack roll** = `effectiveLevel × (offensiveBonus + 64)`, then the gear multiplier chain. `offensiveBonus` is `player.offensive[stab|slash|crush]` (melee, by style), `.ranged`, or `.magic`.

**NPC defence roll** = `(defLevel + 9) × (defensiveStat + 64)`, ×`(250+invocation)/250` at ToA. Which `defensiveStat`/level — see `osrs-monsters`.

**Hit chance** (`BaseCalc.getNormalAccuracyRoll(atk, def)`, `BaseCalc.ts:155`) — memorize this:
```
if atk > def:  1 − (def + 2) / (2 × (atk + 1))
else:          atk / (2 × (def + 1))
```
The result is a fraction (0..1), **not** truncated. Variants exist: `getFangAccuracyRoll` (Osmumten's fang/Drygore — re-roll on miss), `getConflictionGauntletsAccuracyRoll`, `getMaxAccuracyHitChance`. Many monsters/specs short-circuit to 1.0 (guaranteed-accuracy list, Voidwaker/Dawnbringer specs, etc.).

**Max hit** (`trackMaxHitFromEffective`, `BaseCalc.ts:117`):
```
maxHit = trunc((effectiveLevel × gearStrengthBonus + 320) / 640)   // +320 is the +0.5 round, baked in
```
Magic max hit is spell/weapon-table driven instead (e.g. Trident `max(1, trunc(magic/3 − 5))`, Shadow `trunc(magic/3) + 1`); magic damage bonus then applies as `[magicDmgBonus, 1000]`. Then the per-style gear multiplier chain (demonbane, slayer, crystal, spec, etc. — see `osrs-weapons`).

**DPS** (`getDps`, `PlayerVsNPCCalc.ts:2354`):
```
DPS = getExpectedDamage() / (getExpectedAttackSpeed() × 0.6)      // SECONDS_PER_TICK = 0.6
```
Use **`getExpectedAttackSpeed()`** (proc-reduced — Blood Moon, TD unshielded −1, Eye of Ayak spec), not the raw `getAttackSpeed()`.

## Hit distributions & TTK (don't hand-roll these)

Damage isn't a scalar — it's a `HitDistribution` over `WeightedHit`s (each a probability + ordered hitsplats). Accuracy is folded in as a discrete **0-damage miss outcome** (`HitDistribution.linear(acc, 0, max)`), never multiplied afterward. Multi-hit weapons either pack multiple splats into one hit or zip several distributions together.

- Expected damage / max: `getDistribution().getExpectedDamage()` / `.getMax()`.
- Average TTK: `getTtk()` uses a hits-to-kill recurrence `htk[hp] = (1 + Σ p(hit)·htk[hp−hit]) / (1 − p(miss))`, ×`getExpectedAttackSpeed() × 0.6`. **Not** `hp / avgDamage`.
- Full TTK distribution: `getTtkDistribution()` is a tick-by-tick Markov walk over HP states that correctly handles variable attack speed.

Internals are in `HitDist.ts` and `reference.md`. To get the answer, call the engine — don't reimplement the distribution math.

## Running, inspecting, and testing the calc

```ts
const calc = new PlayerVsNPCCalc(player, monster, { detailedOutput: true });
calc.getMax();             // scalar max hit incl. DoT (there is NO getMaxHit(); getMinAndMax() returns [min,max])
calc.getHitChance();       // also getMaxAttackRoll, getNPCDefenceRoll, getDistribution, getDps, getTtk, getTtkDistribution
calc.details;              // ordered, human-readable breakdown of every step (CalcDetails)
```

`detailedOutput: true` records each labelled intermediate (`DetailKey`), which is the way to debug *why* a number is what it is. Tests use the `src/tests/utils/TestUtils.ts` helpers — `getTestPlayer`, `getTestMonster`, `calculatePlayerVsNpc` — see `reference.md` for the canonical template. Run with `yarn jest <pattern>`. `LeaguesTestDataFromJagex.test.ts` asserts the calc reproduces Jagex-provided DPS numbers (evidence it matches the real game; runs locally, skipped on public CI).

## Adding a new damage or accuracy effect

To add a gear/spec effect (e.g. a melee weapon that does +20% damage vs undead):
1. **Detect the item** — add an `isWearingX()` predicate in `BaseCalc.ts` (matches by item *name* via `wearing(...)`; see `osrs-weapons`).
2. **Detect the target type** if type-gated — type-conditional bonuses read `this.monster.attributes.includes(MonsterAttribute.X)` (e.g. demonbane→`DEMON`, salve→`UNDEAD`; enum in `src/enums/MonsterAttribute.ts`, see `osrs-monsters`).
3. **Apply the multiplier in the right chain at the right position** — accuracy in `getPlayerMax{Melee,Ranged,Magic}AttackRoll`, damage in `getPlayerMax{Melee,Ranged,Magic}Hit`. A flat `+X%` is multiplicative `trackFactor(roll, [100+X, 100])`; a "vs-type" additive bonus uses `trackAddFactor`. **Order matters** — truncation is per-step, so where you insert it changes the result. Copy the closest existing effect (the **Salve/undead** or **demonbane** code) as a template.
4. **Add a `DetailKey`** label so the step appears in `calc.details`.
5. **Add a test** in `src/tests/calc/` using the harness above.

## Common mistakes

- **Algebraically simplifying the formulas.** You can't — truncation at each step changes the result. Apply factors one at a time with `trunc`.
- **Computing DPS as `maxHit × accuracy / speed`.** Use the distribution's expected damage and `getExpectedAttackSpeed() × 0.6`; multi-hit, min-hit floors, bolt procs, and DoT all change expected damage.
- **Computing TTK as `hp / dps`.** Use `getTtk()` — misses and HP-dependent effects make the true value differ.
- **Forgetting accuracy is inside the distribution.** Don't multiply expected damage by hit chance again.
- **Assuming one "accuracy formula".** Fang/Drygore/Confliction and forced-1.0 cases bypass `getNormalAccuracyRoll`.
- **Editing values that scaling overwrites.** The monster is scaled/reduced in the `BaseCalc` constructor before any roll; change `monster.inputs`, not `monster.skills` (see `osrs-monsters`).
