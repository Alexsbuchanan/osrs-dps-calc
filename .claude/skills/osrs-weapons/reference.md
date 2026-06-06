# OSRS Weapons & Equipment — Reference

Exhaustive companion to `SKILL.md`. All arithmetic is integer-truncated (`Math.trunc`); `[a,b]` means `trunc(x*a/b)`. Citations are into `src/lib/PlayerVsNPCCalc.ts` unless prefixed.

## 1. Full schema field mapping (`scripts/generateEquipment.py` → JSON)

| Wiki Bucket field | JSON field | Notes |
|---|---|---|
| `page_name` | `name` | canonical display name |
| `item_id`[0] | `id` | non-int → item skipped |
| `weight` | `weight` | kg, fractional |
| `version_anchor` | `version` | NMZ variants forced to `""` |
| `equipment_slot` | `slot` | `"2h"` → `"weapon"` + `isTwoHanded=true` |
| `weapon_attack_speed` | `speed` | ticks |
| `combat_style` | `category` | `EquipmentCategory` string |
| `strength_bonus` | `bonuses.str` | |
| `ranged_strength_bonus` | `bonuses.ranged_str` | |
| `magic_damage_bonus` | `bonuses.magic_str` | `int(value × 10)` = tenths of a percent |
| `prayer_bonus` | `bonuses.prayer` | |
| `{stab,slash,crush,magic,range}_attack_bonus` | `offensive.{stab,slash,crush,magic,ranged}` | |
| `{stab,slash,crush,magic,range}_defence_bonus` | `defensive.{...}` | can be negative |

11 slots: `head, cape, neck, ammo, weapon, body, shield, legs, hands, feet, ring`. Merges `scripts/manual_equipment.json` (unreleased items only).

## 2. Combat styles per category (`getCombatStylesForCategory`, `src/utils.ts`)

Each entry: `Name(type,stance)`. `{name:'Spell', type:'magic', stance:'Manual Cast'}` appended to all **except Blaster** (returns `[]` early).

- **2h Sword**: Chop(slash,Accurate), Slash(slash,Aggressive), Smash(crush,Aggressive), Block(slash,Defensive)
- **Slash Sword / Axe**: Chop(slash,Accurate), Slash/Hack(slash,Aggressive), Lunge(stab,Controlled)|Smash(crush,Aggressive), Block(slash,Defensive)
- **Stab Sword**: Stab(stab,Accurate), Lunge(stab,Aggressive), Slash(slash,Aggressive), Block(stab,Defensive)
- **Scythe**: Reap(slash,Accurate), Chop(slash,Aggressive), Jab(crush,Aggressive), Block(slash,Defensive)
- **Dagger / Claw / Multi-Melee**: stab/slash mix; Accurate/Aggressive/Controlled/Defensive
- **Spear / Partisan / Polearm / Pickaxe / Banner**: stab-leaning; Controlled options
- **Whip**: Flick(slash,Accurate), Lash(slash,Controlled), Deflect(slash,Defensive)
- **Bludgeon**: Pound/Pummel/Smash all (crush,Aggressive)
- **Blunt / Polestaff / Spiked / Staff / Bulwark / Unarmed**: crush-leaning
- **Bow / Crossbow / Thrown**: Accurate(ranged,Accurate), Rapid(ranged,Rapid), Longrange(ranged,Longrange)
- **Chinchompas**: Short/Medium/Long fuse → (ranged, Accurate/Rapid/Longrange)
- **Powered Staff / Powered Wand**: Accurate(magic,Accurate) ×2, Longrange(magic,Longrange)
- **Staff / Bladed Staff**: melee styles + Spell(magic, Autocast / Defensive Autocast)
- **Salamander**: Scorch(slash,Aggressive), Flare(ranged,Rapid), Blaze(magic,Defensive)
- **Gun**: Kick(crush,Aggressive) only. **Blaster**: `[]` (TODO).

## 3. Special-effects catalog

The single highest-value reference. `mattrs = monster.attributes` — these are `MonsterAttribute` enum values (`src/enums/MonsterAttribute.ts`, e.g. `MonsterAttribute.UNDEAD`, `.DEMON`); a "vs-type" effect tests `this.monster.attributes.includes(MonsterAttribute.X)` (see the `osrs-monsters` skill). Accuracy effects live in `getPlayerMax{Melee,Ranged,Magic}AttackRoll`; damage in `getPlayerMax{Melee,Ranged,Magic}Hit`; multi-hit/distribution in `getAttackerDist`/`getDistributionImpl`. `demonbaneFactor(w) = [trunc(w*vuln/100), 100]`, `vuln` = demonbaneVulnerability (Duke 70, Yama 120, Void Flare 200, else 100).

### Universal accuracy formula context
Base hit chance = `BaseCalc.getNormalAccuracyRoll(atk, def)`. Some weapons replace it: **Osmumten's fang** (stab) and **Drygore blowpipe** use `getFangAccuracyRoll` (re-roll-on-miss); at ToA, fang uses `1-(1-h)²`. **Confliction gauntlets** (1h magic) use `getConflictionGauntletsAccuracyRoll`.

### Damage / accuracy multipliers

| Effect | Condition | Affects | Formula |
|---|---|---|---|
| **Twisted bow** | always (cap 350 if xerician else 250; `mag = min(cap, max(monster.magic, monster.offensive.magic))`) | acc + dmg | `tbowScaling`; `t2=trunc((3m-f)/100); t3=trunc((trunc(3m/10)-10f)²/100); bonus=base+t2-t3; trunc(cur*bonus/100)` (acc: f=10,base=140; dmg: f=14,base=250). Applied **twice** at P2 Wardens. |
| **Tumeken's shadow** | powered staff | base max + magic_str×3 (×4 ToA) | base `max(1, trunc(magic/3)+1)`; the ×3/×4 to `magic_str`/`offensive.magic` happens in `Equipment.ts:405` (capped 1000 on magic_str) |
| **Scythe of vitur** | melee | distribution | up to `clamp(size,1,3)` splats; splat *i* max = `trunc(max/2^i)` → `max`, `max/2`, `max/4`; summed for total |
| **Dragon hunter lance** | dragon | acc ×6/5, dmg ×6/5 |
| **Dragon hunter wand** | dragon | acc ×7/4, dmg ×7/5 |
| **Dragon hunter crossbow** | dragon | acc ×13/10, dmg ×5/4 (additive +5/20 with imbued black mask) |
| **Salve (e)/(ei)** | undead | melee ×6/5; ranged ×6/5 (bow/crossbow need **(ei)**; plain **(e)** boosts ranged only via str-scaling thrown — atlatl/Hunter's spear); magic acc +20%, magic dmg +200‰ |
| **Salve / (i)** | undead | melee/ranged ×7/6; magic acc +15%, magic dmg +150‰ |
| **Black mask / Slayer helm (melee)** | on slayer task | melee acc & dmg ×7/6 |
| **Imbued black mask (ranged/magic)** | on task | ranged & magic acc & dmg **×23/20** (≠ melee's 7/6!); for ranged, rev/dragonbane/demonbane fold **additively** into the 23/20 numerator |
| **Void (melee/ranged/magic)** | full set | effective level ×11/10 / ×11/10 (elite ranged ×9/8) / ×29/20 |
| **Obsidian set + Tzhaar weapon** | both | melee acc & dmg | `+ trunc(base/10)` (additive off the base roll/max) |
| **Berserker necklace + Tzhaar weapon** | melee | distribution ×6/5 |
| **Crystal armour + crystal bow** | ranged | acc `×(20+pcs)/20`, dmg `×(40+pcs)/40`; pcs = helm1+legs2+body3 |
| **Crystal blessing** | melee / powered-staff magic | acc `×(20+pcs)/20`, dmg `×(40+pcs)/40` |
| **Inquisitor's** | crush | acc & dmg `[200+inqPcs, 200]`; Inq mace: ×5/pc (no full-set bonus); else 0.5%/pc + 1% full set (`=5` if all 3) |
| **Arclight / Emberlight** | demon | melee acc & dmg `+demonbaneFactor(70)` |
| **Silverlight / Darklight** | demon | melee dmg `+demonbaneFactor(60)` |
| **Bone/Burning claws** | demon | melee acc & dmg `+demonbaneFactor(5)` + burn DoT |
| **Infernal tecpatl** | demon | melee dmg `+demonbaneFactor(10)` |
| **Scorching bow** | demon | ranged acc & dmg `+demonbaneFactor(30)` |
| **Holy water** | demon only (else [0,0]) | ranged; eff+10, str=64+ranged_str, `+demonbaneFactor(60)` |
| **Demonbane spell** | demon + spell name has 'Demonbane' | magic acc `+demonbaneFactor(mark?40:20)` (×2 Purging staff); Mark of Darkness adds a distribution boost |
| **Bone weapons (ratbane)** | rat | melee/ranged dmg +10 flat (immune vs non-rats) |
| **Smoke battlestaff** | standard spellbook | magic acc +10% (additive), magic dmg +100‰ |
| **Amulet of avarice** | (in wildy w/ Forinthry surge) | acc/dmg `[surge?27:24, 20]`; magic dmg `+ (surge?350:200)‰` |
| **Rev weapons** (Craw's/Webweaver/Ursine/Viggora/Thammaron/Accursed, Charged, in wildy) | acc ×3/2, dmg ×3/2 |
| **Colossal blade** | melee | max hit `+ min(size*2, 10)` flat |
| **Barronite mace** | golem | melee dmg ×23/20 |
| **Granite hammer** | golem | acc ×13/10, dmg ×13/10 |
| **Gadderhammer** | shade | distribution 95% ×5/4, 5% ×2 |
| **Leaf-bladed battleaxe** | leafy | melee dmg ×47/40 |
| **Keris (breaching)** | kalphite | acc ×133/100 |
| **Keris (sun)** | ToA + hp < 25% | acc ×5/4 |
| **Keris dmg** | kalphite | ×115/100 (amascut) else ×133/100; +1/51 chance ×3 (distribution) |
| **Tome of fire/water/earth** | matching spellement, Charged | magic dmg ×11/10 |
| **Tome of water (acc)** | water spell / bind | magic acc ×6/5 |
| **Sunfire runes** | fire spell | min hit `trunc(max/10)` |
| **Spellement weakness** | spell element == monster weakness | magic acc & dmg `+ trunc(base*severity/100)` |
| **Brimstone ring** | magic | 25% of rolls use `trunc(def*9/10)` (lowered defence) |
| **Vampyrebane scaling** | vampyre | Blisterwood flail ×5/4, sickle ×23/20, Ivandis flail ×6/5, Rod of ivandis ×11/10 |
| **Efaritay's aid** | silver weapon + vampyre | melee acc ×23/20, magic acc +15% |

### Special-attack (`opts.usingSpecialAttack`) effects

| Weapon | Acc | Damage / distribution |
|---|---|---|
| Any godsword | ×2 | ×11/10, then BGS/Sara sword ×11/10, AGS/dsword/dlong/SBS ×5/4 |
| Dragon claws | (per-roll) | `dClawDist` 4-roll cascade (`dists/claws.ts`) |
| Burning/Bone claws | | `burningClawSpec` 3-roll + burn DoT (max 29) |
| Dragon/Crystal halberd | | dmg ×11/10; 2nd hit at `trunc(atkRoll*3/4)` acc (size>1) |
| Dragon dagger | ×23/20 | 2 hits |
| Abyssal dagger | ×5/4 | dmg ×17/20, 2 hits |
| Abyssal bludgeon | | dmg `×(100 + prayerMissing/2)/100` |
| Voidwaker | acc forced 1.0 | min `[1,2]×max`, max `+min` (≈1.5×) |
| Dawnbringer | 1.0 vs Verzik P1 | guaranteed `[75,150]` |
| Blowpipe | ×2 (acc) | dmg ×3/2 |
| Webweaver bow | ×2 | dmg `−trunc(max*6/10)`; 4 hits |
| Dark bow | | min 8 (Dragon arrow)/5; dmg ×15/10 or ×13/10; 2 hits |
| Magic shortbow | ×10/7 | 2 hits (acc is multiplied, NOT forced to 1.0 — only Seercull/Magic longbow force 1.0) |
| Ballista | ×5/4 | |
| Soulreaper axe | acc `[100+6·stacks,100]` | dmg `[100+6·stacks,100]`; amortized via `getSpecDps` |
| Accursed sceptre / Volatile nightmare staff | magic ×3/2 | |
| Eye of ayak | ×2 | dmg ×13/10, fixed 5-tick speed |
| Saradomin sword | | + magic splat `linear(1,1,16)` |

### Set-effect distribution mechanics
- **Veracs** (full): 25% guaranteed hit `linear(1,1,max+1)` ignoring defence.
- **Karils + amulet of the damned** (ranged): 25% second splat = `trunc(dmg/2)`.
- **Ahrims + amulet of the damned** (magic): 25% chance ×13/10.
- **Dharok** (full): `× (10000 + (maxHp−curHp)·maxHp) / 10000`.
- **Blood Moon set / Dual macuahuitl**: 33% per accurate splat → attack speed −1 (affects `getExpectedAttackSpeed` & TTK delays).
- **Two-hit weapons** (Torag's hammers, Sulphur blades, etc.): split `trunc(max/2)` + remainder.

### Enchanted bolts (crossbow, `dists/bolts.ts`)
Proc chance ×`kandarinFactor` (1.1 with Kandarin Hard Diary). Opal `5%` `+trunc(rangedLvl/10)`; Pearl `6%` `+trunc(rangedLvl/(fiery?15:20))`; Diamond `10%` effect max `trunc(max*115/100)`; Onyx `11%` `trunc(max*120/100)` (immune on undead); Dragonstone `6%` `trunc(rangedLvl*2/10)` (immune fiery/dragon); **Ruby** `6%` `min(cap, trunc(monsterCurrentHp*20/100))` cap 100 (HP-dependent). ZCB raises caps/effects and guarantees the proc on spec.

## 4. Attack speed adjustments (`calculateAttackSpeed`, `Equipment.ts:249`)
Base `weapon.speed || 4`. Then: Rapid ranged −1; cast stances → 4 (Harmonised+standard), 6 (Twinflame), else 5; Giant rat (Scurrius adds, id 7223) + bone weapon → 1; plus Leagues talents; `max(speed, 1)`. DPS uses `getExpectedAttackSpeed()` (subtracts fractional ticks for Blood Moon proc, TD unshielded −1, Eye of ayak spec = 5).

## 5. Ammo applicability (`Equipment.ts:166`)
`AmmoApplicability {INCLUDED, ALLOWED, INVALID}`. Each ranged weapon maps to allowed ammo ids. Empty list = no ammo slot (blowpipe, crystal/Craw bows). On aggregation, ammo's `ranged`/`ranged_str` count only if `INCLUDED`. Invalid ammo → max hit/attack roll forced to 0 and a user issue raised.

## 6. Worked anchors (from `src/tests/calc/`)

| Loadout | Value |
|---|---|
| Abyssal whip, str 99 | maxHit 24 |
| Bow of faerdhinen, ranged 99 | maxAttackRoll 21120, maxHit 29 |
| Trident of the seas, magic 99 | maxAttackRoll 8690, maxHit 28 |
| Osmumten's fang, max melee (id 415) | maxHit 50 |
| Tumeken's shadow, max mage (id 415) | maxHit 65; +slayer helm 70; in void 51 |
| Toktz-xil-ak + obsidian set + salve(ei), str 103 | maxHit 43 |
| Soulreaper axe, str 118, 0→5 stacks | 61/64/67/70/72/75 |
| Scorching bow vs demon + slayer(i): | `trunc(baseMax × 29/20)` — **additive**: `23/20 + 6/20 = 29/20` (= 1.15 + 0.30), demonbane +6 folds into the black-mask numerator, NOT multiplicative |
| Inquisitor's mace, 3pc, base max 200 | 215 (2.5%/pc, no full-set bonus) |

`LeaguesTestDataFromJagex.test.ts` validates full DPS numbers against Jagex-provided data (runs locally, skipped on CI).
