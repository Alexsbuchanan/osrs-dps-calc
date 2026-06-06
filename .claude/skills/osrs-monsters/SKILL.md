---
name: osrs-monsters
description: Use when working with Old School RuneScape monster/NPC data or boss mechanics in this DPS calculator — editing monsters.json or the Monster type, monster attributes (demon/dragon/undead/kalphite/...), defensive stats (stab/slash/crush/magic and light/standard/heavy ranged defence), weaknesses, burn immunities, custom monsters, or raid/boss defence scaling (CoX/ToB/ToA/Vardorvis) and defence reductions (DWH/BGS/elder maul/arclight).
---

# OSRS Monsters (DPS calculator data model)

## Overview

A **monster** is the target the player attacks. Its data is the OSRS Wiki's source-of-truth for NPC combat stats. The calculator reads a monster's *levels* and *defensive bonuses* to compute the player's accuracy and the monster's effective HP/scaling. **The monster's own `max_hit`/`offensive` block is NOT used for player DPS** — only when the monster attacks back (`NPCVsPlayerCalc`).

Core principle: **a monster is scaled and reduced BEFORE any roll is computed.** `BaseCalc`'s constructor calls `scaleMonster(monster)` (`src/lib/BaseCalc.ts:95`), which applies raid HP/stat scaling, phase shifts, and spec defence reductions. By the time `getNPCDefenceRoll()` runs, `monster.skills.def` is already the scaled-and-drained value.

## Where the logic lives (read this first)

Monster mechanics are spread across many files. Grepping one file gives wrong/empty answers.

| Concern | File |
|---|---|
| Runtime `Monster` type + `inputs` block | `src/types/Monster.ts` |
| Attribute enum (demon, dragon, ...) | `src/enums/MonsterAttribute.ts` |
| Custom monster + default `inputs` | `src/lib/Monsters.ts` (`INITIAL_MONSTER_INPUTS`, `CUSTOM_MONSTER_BASE`) |
| Scaling orchestration (order) | `src/lib/MonsterScaling.ts` (`ORDER_OF_OPERATIONS`) |
| Raid scaling formulas | `src/lib/scaling/{ChambersOfXeric,TheatreOfBlood,TombsOfAmascut,Vardorvis,Phases}.ts` |
| Defence-reduction specs (DWH/BGS/...) | `src/lib/scaling/DefenceReduction.ts` |
| Which defensive stat a roll uses | `src/lib/PlayerVsNPCCalc.ts:147-204` (`getNPCDefenceRoll`) |
| Burn-immunity checks | `src/lib/BaseCalc.ts:684-692` |
| NPC damage caps / post-roll transforms | `src/lib/PlayerVsNPCCalc.ts:1977-2106` (`applyNpcTransforms`) |
| NPC style/attribute immunities | `src/lib/PlayerVsNPCCalc.ts:2108-2177` (`isImmune`) |
| All raid/boss/immunity ID groups | `src/lib/constants.ts` |
| Stock data (2830 monsters) | `cdn/json/monsters.json` |
| Wiki scraper | `scripts/generateMonsters.py` |

See `reference.md` for the full JSON schema, exhaustive scaling formulas, and worked examples. For how a monster's scaled levels/bonuses then enter the accuracy and damage formulas, see the `osrs-dps` skill; for the player gear that keys off monster attributes, see `osrs-weapons`.

## The defensive model (most common source of error)

A monster's `defensive` object holds **seven defence bonuses + `flat_armour`**:
`stab, slash, crush, magic` (melee + magic) and `light, standard, heavy` (the three post-rework **ranged** defence types). `flat_armour` is a separate flat damage reduction, not a roll bonus.

Which bonus a player attack rolls against (`getNPCDefenceRoll`, `PlayerVsNPCCalc.ts:184-194`):

| Player attack | Monster stat used |
|---|---|
| stab / slash / crush | `defensive[stab|slash|crush]` |
| magic | `defensive.magic` |
| ranged, thrown weapon | `defensive.light` |
| ranged, bow | `defensive.standard` |
| ranged, crossbow or chinchompa | `defensive.heavy` |
| ranged, salamander ("mixed") | `trunc((light + standard + heavy) / 3)` |

The **roll level** is `monster.skills.def` for *all* melee and ranged attacks (including crossbow — only the *bonus* column above differs). Magic is the sole exception: it rolls against `monster.skills.magic`, unless the NPC id is in `USES_DEFENCE_LEVEL_FOR_MAGIC_DEFENCE_NPC_IDS` (`constants.ts`), which uses the defence level for magic too. Then `defenceRoll = (level + 9) * (statBonus + 64)`, ×`(250+invocation)/250` for ToA monsters.

During a **special attack** the defence style can be overridden by the weapon, independent of the player's actual style: godswords / dragon weapons → `slash`, Arclight/Emberlight/Dragon sword → `stab`, Voidwaker / Saradomin's blessed sword → `magic`, Dragon mace → `crush` (see `osrs-dps` `getNPCDefenceRoll`). The player-side multiplier that keys off `attributes` (e.g. demonbane, salve) lives in the calc — see `osrs-weapons` / `osrs-dps`.

## Monster attributes

`MonsterAttribute` (`src/enums/MonsterAttribute.ts`) has exactly 16 values: `demon, dragon, fiery, flying, golem, kalphite, leafy, penance, rat, shade, spectral, undead, vampyre1, vampyre2, vampyre3, xerician`. Gear/effects key off `monster.attributes.includes(...)` (e.g. demonbane → `'demon'`, salve → `'undead'`, `xerician` → triggers CoX scaling). Some attributes gate immunity (e.g. `leafy` immune to non-leaf-bladed; `flying` immune to non-polearm melee; vampyre2/3 immune without vampyrebane). Full table in `reference.md`.

**Attributes are fixed from JSON.** You can only toggle attributes on the **custom monster** (`id === -1`); the UI disables the toggle for stock monsters. To test demonbane gear, use a custom monster and add `'demon'`.

## The `inputs` block (UI-driven state)

`Monster.inputs` (`src/types/Monster.ts:63-132`) is added at runtime (not in JSON) and holds everything that drives scaling/effects: `toaInvocationLevel`, `toaPathLevel`, `partySize`, `partyMaxCombatLevel`, `partyMaxHpLevel`, `partySumMiningLevel`, `monsterCurrentHp` (for Vardorvis/ruby-bolt HP-dependent effects), `phase`, monster `prayers` (overhead protect), and `defenceReductions: {vulnerability, accursed, elderMaul, dwh, arclight, emberlight, bgs, tonalztic, seercull, ayak}`.

**Defence reductions are user-entered counts/flags, not derived from spec hits.** E.g. you tell it "3 DWH hits landed" via `inputs.defenceReductions.dwh = 3`; the calc does not simulate the spec.

## Scaling & defence-reduction order

`scaleMonster` runs transformers in this fixed order (`MonsterScaling.ts`): CoX → ToB → ToA → Vardorvis → Phases → **DefenceReductions (last)**. Reductions therefore drain the already-raid-scaled defence.

Within `DefenceReduction.ts` the order is: accursed **xor** vulnerability → elder maul → DWH → arclight/emberlight → tonalztic → seercull → **BGS** → ayak. BGS drains a single pool through `def → str → atk → magic → ranged`, and **stops propagating if a stat fails to fully drain** (including hitting its defence floor). Full formulas in `reference.md`.

## Adding or editing a monster

- Stock data comes from the wiki via `scripts/generateMonsters.py` (Bucket API → `cdn/json/monsters.json`). Don't hand-edit `monsters.json` for real monsters; fix the scraper or `scripts/manual_monster.json`.
- For ad-hoc combat scenarios, use the **custom monster** (`id === -1`) — it's the only one whose attributes, defensive stats, and levels are user-editable.
- After changing the `Monster` type or scaling, run the scaling tests: `yarn jest src/tests/lib/scaling` and `yarn jest GeneratedTests`.

## Common mistakes

- **Using `monster.maxHit` for player DPS.** It's display-only (`Monster.ts:23`); the calc computes its own max hit.
- **Assuming light/standard/heavy relate to attack weight.** They're selected by *weapon category* (thrown/bow/crossbow), not by how "heavy" the hit is.
- **Forgetting magic rolls against `skills.magic`,** not `skills.def` (except the override-ID list).
- **Editing `skills.def` to simulate a DWH/BGS.** Use `inputs.defenceReductions` instead — they apply during scaling in the correct order.
- **Expecting attribute toggles on a stock monster.** Only the custom monster (`id === -1`) is editable.
- **`immunities.burn` can be `null`** (most monsters) — handle null (commit #888 made this explicit).
