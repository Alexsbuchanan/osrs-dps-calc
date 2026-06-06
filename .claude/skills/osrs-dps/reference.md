# OSRS DPS Engine — Reference

Exhaustive companion to `SKILL.md`. All math integer-truncated (`Math.trunc`); `[a,b]` = `trunc(x*a/b)`. Citations into `src/lib/PlayerVsNPCCalc.ts` unless prefixed. Gear/spec multipliers (twisted bow, demonbane, slayer, godsword spec, ...) are catalogued in the `osrs-weapons` skill.

## 1. Integer-math helpers

- `trackFactor(base,[n,d]) = trunc(base*n/d)` (`BaseCalc.ts:109`) — apply a factor.
- `trackAdd(base, x) = trunc(base + x)` (`:124`).
- `trackAddFactor(base,[n,d]) = trunc(base + trunc(base*n/d))` (`:130`) — additive bonus (demonbane).
- `trackMaxHitFromEffective(eff, gear) = trunc((eff*gear + 320)/640)` (`:117`) — base max hit (`gear = strengthBonus + 64`).
- `Math.ts`: `lerp`, `iSqrt(x)=trunc(√x)`, `iLerp`, `addPercent(x,p)=x+trunc(x*p/100)`. `Factor=[num,div]`, `MinMax=[min,max]`.

## 2. Attack roll — per style

Public `getMaxAttackRoll()` (`:1205`) dispatches by `player.style.type`, then applies Leagues all-style accuracy. Returns 0 if ammo invalid.

### Melee (`getPlayerMaxMeleeAttackRoll`, `:206`)
1. `eff = trunc(atk + boost.atk)`
2. each accuracy prayer: `eff = trunc(eff × factorAccuracy)` (sequential)
3. `eff += 8` (+3 Accurate, +1 Controlled)
4. melee void → `eff = trunc(eff × 11/10)`
5. `gear = offensive[stab|slash|crush] + 64`; `baseRoll = trunc(eff × gear)`
6. gear multiplier chain (crystal, avarice/salve/black-mask exclusive block, obsidian additive, rev, demonbane, dragon hunter, keris, inquisitor's, spec) — see `osrs-weapons`.

### Ranged (`getPlayerMaxRangedAttackRoll`, `:545`)
`eff = ranged + boost`; accuracy prayers (Leagues may rescale); +3 Accurate; **+8 unconditional**; ranged void ×11/10; `roll = eff × (offensive.ranged + 64)`; then crystal bow, exclusive salve/mask block, twisted bow (`tbowScaling`), rev, DHCB, chinchompa distance, scorching bow, spec.

### Magic (`getPlayerMaxMagicAttackRoll`, `:849`)
`eff = magic + boost`; accuracy prayers; **+2 Accurate; +9 unconditional**; magic void ×29/20; `baseRoll = eff × (offensive.magic + 64)`; then crystal blessing, **additive accumulator** (avarice/salve/smoke/efaritay → `×(100+sum)/100`), dragon hunter, black mask ×23/20, demonbane, rev, tome of water, spec, spellement weakness (additive off baseRoll).

## 3. NPC defence roll (`getNPCDefenceRoll`, `:147`)
`defenceStyle = player.style.type` (spec may override: godswords/dragon weapons→slash, arclight/emberlight→stab, voidwaker→magic, dragon mace→crush). `level = (magic & not override-id) ? skills.magic : skills.def`. `eff = level + 9`. `statBonus = defensiveStat + 64` (defensiveStat per `osrs-monsters`). `roll = trunc(eff × statBonus)`, ×`(250+invocation)/250` at ToA.

## 4. Hit chance (`getHitChance`, `:1262`)
Short-circuits to 1.0/0.0 for: `GUARANTEED_ACCURACY_MONSTERS`, Doom non-Normal, Verzik P1 + Dawnbringer, P2 Wardens, Scurrius rat, TD non-Shielded, Royal Titan elementals (magic: `min(1, max(0,offensive.magic)/100 + 0.3)`, ×1.45 if void), Eclipse Moon clone, `ALWAYS_MAX_HIT_MONSTERS[style]`, Voidwaker/Dawnbringer spec, Seercull/MLB spec.

Else base = `getNormalAccuracyRoll(atk, def)` (`BaseCalc.ts:155`):
```
stdRoll(a,d) = a > d ? 1 − (d+2)/(2(a+1)) : a/(2(d+1))
// negatives: if a<0 a=min(0,a+2); if d<0 d=min(0,d+2);
//   a≥0,d<0 → 1 − 1/(−d+1)/(a+1);  a<0,d≥0 → 0;  both<0 → stdRoll(−d,−a)
```
Then overrides: Leagues crossbow double-roll, **Fang/Drygore** → `getFangAccuracyRoll` (ToA fang → `1−(1−h)²`), **Confliction gauntlets** → `getConflictionGauntletsAccuracyRoll`, Leagues max-roll talent. `getDisplayHitChance` adds the Brimstone-ring 25% lowered-def blend for display.

- `getFangAccuracyRoll` (`:170`): re-roll-once-on-miss. `a>d ? 1−(d+2)(2d+3)/(a+1)²/6 : a(4a+5)/6/(a+1)/(d+1)`.
- `getConflictionGauntletsAccuracyRoll` (`:188`): `double/(1 + double − single)`.
- `getMaxAccuracyHitChance` (`:200`): `a>d ? 1 : a/(d+1)`.

## 5. Max hit — per style (`getMinAndMax`, `:1163`)
Returns `[min,max]`; clamps negatives to 0; honors `overrides.maxHit`; `[0,0]` if ammo invalid.

### Melee (`getPlayerMaxMeleeHit`, `:342`)
`eff = str + boost`; strength prayers (Burst of Strength below L21 = flat +1); Soulreaper non-spec additive `+ trunc(base × stacks×6/100)`; `eff += 8` (+3 Aggressive, +1 Controlled); melee void ×11/10; `baseMax = trunc((eff × (bonuses.str + 64) + 320)/640)`; then chain: crystal, avarice/salve/black-mask, demonbane, obsidian additive, dragon hunter, keris, barronite/granite (golem), rev, leaf-bladed, colossal blade flat, ratbone +10, inquisitor's, fang shrink, spec, respiratory min.

### Ranged (`getPlayerMaxRangedHit`, `:656`)
`eff = ranged + boost` (atlatl/Hunter's spear use str); holy water & MSB/MLB/Seercull/Ogre-bow special early-returns; strength prayers; +3 Accurate; +8; elite ranged void ×9/8 else ranged void ×11/10; `baseMax = trunc((eff×(64+ranged_str)+320)/640)`; then crystal bow, salve/mask block (imbued mask folds rev/dragonbane/demonbane additively into numerator), twisted bow, leftover multiplicative bonuses, ratbone, tonalztics ×3/4, spec.

### Magic (`getPlayerMaxMagicHit`, `:960`)
Base from spell (`getSpellMaxHit`) or powered staff: Trident seas `max(1,trunc(magic/3−5))`, swamp `−2`, Sanguinesti `−1`, Thammaron's `−8`, Accursed `−6`, Shadow `max(1,trunc(magic/3)+1)`, Eye of ayak `−6`, Warped `trunc((8magic+96)/37)`, Bone staff `…−5)+10`, fixed crystal staves 23/31/39, salamanders via **magic level** `trunc((magicLevel*(salStr+64)+320)/640)` with salStr 56/59/77/92/104 (NOT strength — the str-scaling path is ranged-only, for the Eclipse atlatl / Hunter's spear). Then: Eye of ayak spec, chaos gauntlets +3 / charge +10, magic damage bonus `trackAddFactor(maxHit, [magicDmgBonus, 1000])` (`magicDmgBonus = bonuses.magic_str` + smoke +100 + salve/avarice + prayer magicDamageBonus), crystal blessing, black mask ×23/20, dragon hunter, rev, accursed spec, spellement weakness additive, sunfire min, tomes ×11/10.

## 6. Prayers (`src/enums/Prayer.ts`)
Applied multiplicatively/sequentially in PvN via `getCombatPrayers(filter)` (`:1144`), filtered by style. Factors as `[n,100]`:

| Prayer | acc | str | def | magicDmg‰ |
|---|---|---|---|---|
| Piety | 120 | 123 | 125 | — |
| Chivalry | 115 | 118 | 120 | — |
| Rigour | 120 | 123 | 125 (ranged str+acc) | — |
| Augury | 125 | — | 125 | 40 |
| Eagle Eye / Mystic Might | 115 | 115 (EE) | — | 20 (MM) |
| Ultimate Strength | — | 115 | — | — |
| Incredible Reflexes | 115 | — | — | — |
| Hawk Eye / Mystic Lore | 110 | 110 (HE) | — | 10 (ML) |
| Superhuman Strength / Improved Reflexes | (110 acc IR) | 110 (SS) | — | — |
| Sharp Eye / Mystic Will | 105 | 105 (SE) | — | — |
| Burst of Strength / Clarity / etc. | 105 | 105 | — | — |
| Deadeye / Mystic Vigour | 118 | 118 (DE) | 105 | 30 (MV) |

Edge cases: Burst of Strength ≤ L20 = flat +1; Sharp Eye forces +1 if the factor wouldn't change the level. Drain: `ticks = ceil(prayerLevel × (2×prayerBonus + 60) / Σ drainRate)`.

## 7. Potions (`PotionMap`, `src/utils.ts`)
Add to `player.boosts`. `floor` unless noted:

| Potion | Boost |
|---|---|
| Super combat | atk/str/def each `5 + skill×0.15` |
| Overload (+) | all `5 + skill×0.13` (`+`: `6 + ×0.16`) |
| Smelling salts | all `11 + skill×0.16` |
| Ranging / Super ranging | ranged `4 + ×0.1` / `5 + ×0.15` |
| Saturated heart | magic `4 + ×0.1` |
| Imbued heart | magic `1 + ×0.1` |
| Forgotten brew | magic `3 + ×0.08`; atk/str/def `−2 − ×0.1` |
| Super attack/strength/defence | `5 + skill×0.15` |
| Attack/Strength/Defence | `3 + skill×0.1` |
| Magic (potion) | `+4` flat |
| Moonlight | herblore-tiered atk/str/def |

## 8. Hit distributions (`src/lib/HitDist.ts`)
- `Hitsplat{damage, accurate}`; miss = `Hitsplat.INACCURATE` (0, false).
- `WeightedHit{probability, hitsplats[]}` — `zip` (cross-product: prob×, splats concat), `cumulative`, `getSum`, `getHash` (assumes damage ≤ 255).
- `HitDistribution.linear(acc, min, max)`: each `i∈[min,max]` at `acc/(max−min+1)`, plus miss at `1−acc`. **This is the standard attack.** `single(acc, splats)` for fixed outcomes.
- `AttackDistribution`: list of independent `HitDistribution`s; `zipped` (full cross), `singleHitsplat` (collapse to one cumulative splat — fed to TTK).
- Transformers (`:380+`): `flatLimitTransformer` (clamp), `linearMinTransformer`, `cappedRerollTransformer`, `multiplyTransformer(n,d,min)` (truncating, two-branch min clamp), `flatAddTransformer`. Bolt procs in `dists/bolts.ts`, claws in `dists/claws.ts`.

## 9. DPS, TTK, specs
- `getDps()` (`:2354`) `= getDpt() / 0.6`, where `getDpt()` (`:2348`) `= getExpectedDamage() / getExpectedAttackSpeed()` — i.e. `damage / (expectedSpeed × 0.6)`, `SECONDS_PER_TICK=0.6`. (Note the delegation: `getDps` does not literally contain `getExpectedDamage`.)
- `getExpectedDamage() = getDistribution().getExpectedDamage() + getDoTExpected()`.
- `getExpectedAttackSpeed()`: Blood Moon `speed − procChance`; TD unshielded `−1`; Eye of ayak spec `5`; else `getAttackSpeed()`.
- `getHtk()`: `htk[hp] = (1 + Σ_{hit=1..min(hp,max)} hist[hit]·htk[hp−hit]) / (1 − hist[0])`. `getTtk() = getHtk() × getExpectedAttackSpeed() × 0.6`.
- `getTtkDistribution()`: tick-by-tick Markov over HP states with probabilistic delays (handles variable speed; never merges different-delay timelines). `iterMax = 1000 × speed`, terminate when alive-mass `< 0.0001`.
- `getSpecDps()`: amortizes spec energy regen (Lightbearer 25 ticks vs 50; Soulreaper over stack-building).
- Prayer duration: `getPrayerTicks() × 0.6`.

## 10. Test harness (`src/tests/utils/TestUtils.ts`)
Canonical template (`src/tests/CombatCalc.test.ts`):
```ts
import { calculatePlayerVsNpc, getTestMonster, getTestPlayer } from '@/tests/utils/TestUtils';
import { ACCURACY_PRECISION, DPS_PRECISION } from '@/lib/constants';

test('Empty player against abyssal demon', () => {
  const monster = getTestMonster('Abyssal demon', 'Standard');
  const player = getTestPlayer(monster, { /* PartialDeep<Player> overrides */ });
  const r = calculatePlayerVsNpc(monster, player);   // { npcDefRoll, maxHit, maxAttackRoll, accuracy, dps, details, dist }
  expect(r.maxHit).toBe(11);
  expect(r.maxAttackRoll).toBe(7040);
  expect(r.npcDefRoll).toBe(12096);
  expect(r.accuracy * 100).toBeCloseTo(29.10, ACCURACY_PRECISION);  // accuracy is 0..1
  expect(r.dps).toBeCloseTo(0.677, DPS_PRECISION);
});
```
- `getTestPlayer(monster, overrides)`: deep-merges over `generateEmptyPlayer()`; if you set `bonuses`/`offensive`/`defensive` directly, gear-derived bonuses are bypassed (lets you pin an exact base max).
- `calc.details` (with `detailedOutput:true`) + `findResult(details, DetailKey.X)` assert on intermediate steps (`MAX_HIT_BASE`, `NPC_DEFENCE_ROLL_EFFECTIVE_LEVEL`, `DAMAGE_LEVEL_PRAYER`, ...).
- `overrides: { accuracy, attackRoll, defenceRoll, maxHit }` inject exact values to isolate one stage.
- Precision: `ACCURACY_PRECISION=2`, `DPS_PRECISION=3`. Run: `yarn jest <pattern>`.

## 11. Worked anchors
Empty player vs Abyssal demon (Standard): maxHit 11, maxAttackRoll 7040, npcDefRoll 12096, accuracy 29.10%, dps 0.677. Whip str99 → 24. Bof ranged99 → maxAttackRoll 21120, maxHit 29. Trident magic99 → 8690, 28. More in `osrs-weapons`/`osrs-monsters` references. `GeneratedTests.test.ts` (machine-generated from in-game snapshots) and `LeaguesTestDataFromJagex.test.ts` (Jagex data) are the regression ground truth.
