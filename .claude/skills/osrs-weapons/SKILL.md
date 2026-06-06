---
name: osrs-weapons
description: Use when working with Old School RuneScape equipment or weapons in this DPS calculator — editing equipment.json or the EquipmentPiece type, weapon categories and combat styles, attack speed, ammo applicability, equipment aliases, gear stat bonuses (str/ranged_str/magic_str/prayer, offensive, defensive), gear magic-damage scaling, or special weapon/armour effects (twisted bow, Tumeken's shadow, scythe, fang, void, slayer helm/black mask, demonbane, dragon hunter, salve, crystal, obsidian, enchanted bolts, godsword/dragon claw specs).
---

# OSRS Weapons & Equipment (DPS calculator data model)

## Overview

A player's loadout is 11 equipment slots. Each worn piece contributes flat stat **bonuses**; the equipped **weapon** also sets the attack `speed`, the `category` (which determines available combat styles and attack type), and whether it's two-handed. Many weapons/armour sets have **special effects** that multiply accuracy and/or damage under conditions.

Core principle: **the calculator detects special gear by item NAME, not by ID.** Effects are guarded by predicates like `this.wearing('Twisted bow')` / `this.wearingAll([...])` in `BaseCalc.ts` and `PlayerVsNPCCalc.ts`. If you add or rename a weapon and its effect doesn't fire, the name string almost certainly doesn't match.

## Where the logic lives (read this first)

| Concern | File |
|---|---|
| `EquipmentPiece` / `PlayerEquipment` / bonus types | `src/types/Player.ts` |
| Weapon category enum + `MAGIC_WEAPONS` | `src/enums/EquipmentCategory.ts` |
| Combat-style types, stances, ranged-damage-type map | `src/types/PlayerCombatStyle.ts` |
| Category → available styles | `src/utils.ts` (`getCombatStylesForCategory`) |
| Gear-bonus aggregation, attack speed, ammo, aliases | `src/lib/Equipment.ts` |
| Alias map (variants → base id) | `src/lib/EquipmentAliases.ts` |
| Special-effect predicates (`isWearing*`, `wearing`) | `src/lib/BaseCalc.ts` |
| Where effects multiply accuracy/damage | `src/lib/PlayerVsNPCCalc.ts` (attack-roll & max-hit methods) |
| Multi-hit / distribution effects (scythe, claws, bolts) | `src/lib/PlayerVsNPCCalc.ts` `getDistributionImpl`/`getAttackerDist`, `src/lib/dists/{bolts,claws}.ts` |
| Spec energy costs | `src/lib/Equipment.ts` (`WEAPON_SPEC_COSTS`) |
| Stock data (5306 items) | `cdn/json/equipment.json` |
| Wiki scraper | `scripts/generateEquipment.py` |

See `reference.md` for the full schema, the per-category combat-style table, and the **complete special-effects catalog** with formulas and citations.

## Equipment JSON schema (essentials)

Type `EquipmentPiece` (`src/types/Player.ts:20-34`). Each item:

| Field | Meaning |
|---|---|
| `name`, `id`, `image`, `version`, `weight` | identity; `version` disambiguates variants (`""`, `"Charged"`, ...) |
| `slot` | one of `head, cape, neck, ammo, weapon, body, shield, legs, hands, feet, ring` (wiki `"2h"` → `"weapon"`) |
| `speed` | attack speed in **ticks** (0.6s each); 0 for non-weapons |
| `category` | `EquipmentCategory` (drives combat styles); `""` for armour |
| `isTwoHanded` | true blocks the shield slot |
| `bonuses` | `{str, ranged_str, magic_str, prayer}` — `magic_str` is **tenths of a percent** (JSON int = wiki% × 10, so 5% → `50`); it enters magic max hit as a per-mille bonus `[magic_str, 1000]` |
| `offensive` | `{stab, slash, crush, magic, ranged}` — accuracy bonuses |
| `defensive` | `{stab, slash, crush, magic, ranged}` — defence bonuses (can be negative) |

`bonuses.str` feeds melee max hit; `bonuses.ranged_str` feeds ranged max hit (often comes from **ammo**); `bonuses.magic_str` feeds magic max hit; `offensive.{type}` feeds the accuracy roll for that attack type. See `osrs-dps` for how these enter the formulas.

## Weapon categories → combat styles

`EquipmentCategory` (`src/enums/EquipmentCategory.ts`) holds ~31 weapon classes (Whip, Scythe, Bow, Crossbow, Powered Staff, Staff, ...). `getCombatStylesForCategory(category)` (`src/utils.ts`) returns the available `PlayerCombatStyle[]`, each `{name, type, stance}` where `type ∈ stab|slash|crush|magic|ranged|null`. A pseudo `{name:'Spell', type:'magic', stance:'Manual Cast'}` is appended to **every** category except `Blaster` (which returns `[]` early). Full per-category table in `reference.md`.

**Ranged damage type** (`getRangedDamageType`, `PlayerCombatStyle.ts`) maps category → which monster defence stat the attack rolls against: `Thrown→light`, `Bow→standard`, `Crossbow/Chinchompas→heavy`, `Salamander→mixed` (avg of the three). `MAGIC_WEAPONS = [Staff, Powered Wand, Powered Staff, Bladed Staff, Polestaff]`.

### Stance bonuses (added to effective level, before the gear multiply)
- **Melee accuracy**: base +8; Accurate +3, Controlled +1.
- **Melee strength**: base +8; Aggressive +3, Controlled +1.
- **Ranged accuracy & strength**: base +8; Accurate +3 (Rapid/Longrange give 0 here).
- **Magic accuracy**: base **+9**; Accurate **+2** (magic max hit has no stance level boost).
- **Rapid** (ranged): attack speed −1 (`calculateAttackSpeed`). Cast stances force speed (usually 5).

## Attack speed

`getAttackSpeed()` = `player.attackSpeed ?? calculateAttackSpeed(player, monster)` (`PlayerVsNPCCalc.ts:2319`). `calculateAttackSpeed` (`Equipment.ts:249`) starts from `weapon.speed` (default 4), applies Rapid (−1 ranged), cast-stance overrides (4/5/6 depending on staff), Leagues talents, then `max(speed, 1)`. **DPS uses `getExpectedAttackSpeed()`** (proc-reduced, e.g. Blood Moon), not the raw value — see `osrs-dps`.

## Gear-bonus aggregation

`calculateEquipmentBonusesFromGear(player, monster)` (`Equipment.ts:315`): canonicalizes every slot, sums `bonuses`/`offensive`/`defensive` across all worn pieces (ammo `ranged`/`ranged_str` only count if the ammo is valid for the weapon — `ammoApplicability`), then applies special post-processing: blowpipe darts, Crystal armour, **Tumeken's shadow** (`magic_str = min(1000, magic_str × factor)`, `offensive.magic ×= factor`, factor 4 in ToA else 3), Virtus, void mage, Dizana's quiver, and Leagues talents. The totals overwrite `player.bonuses/offensive/defensive` (bridged in `state.tsx`). **Note: some effects (like Shadow's ×3/×4) live in aggregation, not in the calc file.**

## Equipment aliases

`EquipmentAliases.ts` maps a base item id → array of functionally-identical variant ids (locked/degraded/recoloured/ornament/NMZ-imbued). `getCanonicalEquipment` resolves every slot to its base before computing bonuses, so all variants share one stat profile (and `itemVars` like blowpipe dart selection are preserved).

## Adding or editing a weapon

- Stock items come from the wiki via `scripts/generateEquipment.py` → `cdn/json/equipment.json`. Don't hand-edit for real items; fix the scraper or `scripts/manual_equipment.json`, then regenerate.
- A **special effect** requires two things: (1) the item's `name`/`category`/`speed`/bonuses present in the data, and (2) a name-keyed predicate + a multiplier in the calc. To add an effect, add an `isWearingX()` predicate in `BaseCalc.ts` and apply it in the relevant `getPlayerMax*AttackRoll`/`getPlayerMax*Hit`/`getDistributionImpl` method.
- For **attribute-gated** effects (vs undead/demon/dragon/...), the target type is `this.monster.attributes.includes(MonsterAttribute.X)` (enum in `src/enums/MonsterAttribute.ts`; e.g. `UNDEAD`; see the `osrs-monsters` skill). Copy the closest existing effect as a template: **Salve amulet** (undead), **Dragon hunter lance** (dragon), or **demonbane** (demon) — each shows the attribute guard + the `trackFactor`/`trackAddFactor` multiply in both the max-hit and attack-roll chains.
- Verify with a test in `src/tests/calc/` using `getTestPlayer`/`calculatePlayerVsNpc`. See `osrs-dps` for the harness.

## Common mistakes

- **Matching gear by id instead of name.** All effect predicates use `wearing('<exact name>')`. A new/renamed item won't trigger.
- **Putting ranged strength on the bow.** For bows/crossbows, `ranged_str` comes from the **ammo**; the bow only carries `offensive.ranged`. Ammo is excluded entirely if invalid for the weapon.
- **Treating `magic_str` as a raw percent.** It's tenths of a percent (per-mille after a further /10): wiki 5% → `50` in JSON → enters max hit as `[magicDmgBonus, 1000]`.
- **Looking for Tumeken's Shadow / Twisted bow scaling in `PlayerVsNPCCalc.ts` only.** Shadow's multiplier is in `Equipment.ts`; tbow scaling is `tbowScaling()`.
- **Forgetting `isTwoHanded` blocks the shield slot** and that two-handed is derived from the wiki `"2h"` slot.
