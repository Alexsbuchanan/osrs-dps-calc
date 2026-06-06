# OSRS Monsters — Reference

Exhaustive companion to `SKILL.md`. All arithmetic is integer-truncated (`Math.trunc`) unless noted; order of floor operations is load-bearing.

## 1. Full monster JSON schema (`cdn/json/monsters.json`)

Runtime type: `Monster` (`src/types/Monster.ts:15-133`; data fields below at `:15-58`, the `inputs` block at `:63-132`). Data fields (everything except `inputs`) come from the wiki scraper.

| Field | Type | Meaning |
|---|---|---|
| `id` | int | NPC id. `-1` = synthetic custom monster (`Monsters.ts`). |
| `name` | string | Wiki page name. |
| `version` | string | Variant suffix after `#` (e.g. `"Awakened"`, `"Level 13"`); `""` if none. |
| `image` | string | Icon filename. |
| `level` | int | Combat level (UI display only). |
| `speed` | int | Attack speed in ticks (1 tick = 0.6s). Default 4. |
| `style` | string\|null | The monster's *own* attack style (used by `NPCVsPlayerCalc`). `"None"`→null. |
| `size` | int | Tile size N×N. Drives scythe splat count, Colossal blade, melee reach. |
| `max_hit` | string | Free-form wiki string, **display only** — calc ignores it. |
| `skills` | `{atk,def,hp,magic,ranged,str}` | Combat **levels**. `def`→defence roll level; `magic`→magic-defence roll level; `hp`→health. |
| `offensive` | `{atk,magic,magic_str,ranged,ranged_str,str}` | The monster's offensive bonuses (only used when it attacks the player). |
| `defensive` | `{flat_armour, stab, slash, crush, magic, light, standard, heavy}` | Defence bonuses by player attack type. Can be negative. |
| `attributes` | `MonsterAttribute[]` | See §2. |
| `weakness` | `{element, severity}` \| null | Elemental weakness (§3). |
| `immunities` | `{burn: "Weak"\|"Normal"\|"Strong"\|null}` | Burn immunity (§3). |
| `is_slayer_monster` | bool | True if the wiki lists slayer xp. Gates black-mask/slayer-helm bonus. |

`inputs` (runtime UI state, not in JSON) — defaults `INITIAL_MONSTER_INPUTS` (`Monsters.ts:32-58`): `isFromCoxCm:false`, `toaInvocationLevel:0`, `toaPathLevel:0`, `partyMaxCombatLevel:126`, `partySumMiningLevel:99`, `partyMaxHpLevel:99`, `partySize:1`, `monsterCurrentHp:150`, all `defenceReductions` 0, all `prayers` false.

## 2. MonsterAttribute table (`src/enums/MonsterAttribute.ts`)

| Value | What keys off it |
|---|---|
| `demon` | Arclight/Emberlight stronger drain; Silverlight/Darklight/Infernal tecpatl demonbane; **Scorching bow** demonbane + its burn DoT = 5 vs 1 (demon vs non-demon); Demonbane spells; **Bone/Burning claws** demonbane factor +5% (`demonbaneFactor(5)`) — their burn DoT is accuracy-based (max 29), NOT demon-conditional; `demonbaneVulnerability` (Duke 70, Yama 120, Void Flare 200, else 100). |
| `dragon` | Dragon hunter lance/crossbow/wand bonuses. |
| `fiery` | Pearl bolt divisor 15 vs 20; dragonstone bolts immune. |
| `flying` | **Immune to melee** unless polearm/salamander (or Leagues 2H melee-range). |
| `golem` | Granite hammer ×13/10, Barronite mace ×23/20. |
| `kalphite` | Keris accuracy/damage bonus + 1/51 triple-hit. |
| `leafy` | **Immune** unless leaf-bladed weapon; Leaf-bladed battleaxe ×47/40. |
| `penance` | Barbarian Assault classification. |
| `rat` | Rat-bone weapons +10 flat; **immune to rat-bone weapon vs non-rat**. |
| `shade` | Gadderhammer: 95% ×5/4, 5% ×2. |
| `spectral` | Spectral classification. |
| `undead` | Salve amulet / (i) / (e) / (ei) bonuses; Onyx bolts (e) do NOT proc on undead. |
| `vampyre1/2/3` | Blisterwood/Efaritay's/silver/Rod of ivandis; **vampyre2 immune** without vampyrebane/Efaritay/silver; **vampyre3 immune** without vampyrebane. |
| `xerician` | Triggers **CoX scaling**; raises tbow magic cap 250→350. |

`isVampyre(attr)` (`:20-25`) is true for any of the three vampyre tiers.

## 3. Weakness & immunities

**Elemental weakness** — `weakness: {element: 'air'|'water'|'earth'|'fire', severity: int} | null`. If the cast spell's element matches, BOTH accuracy roll and max hit get `+ trunc(base * severity/100)` (additive off the base roll). `getMonsterWeakness` (`PlayerVsNPCCalc.ts:2788-2809`): Shadowflame quadrant forces the weakness element to the cast spell; Devil's element adds +30 severity. Examples: KBD water/50, Olm head earth/50, Vardorvis (Awakened) fire/35.

**Burn immunity** — `BurnImmunity` enum: `Weak`, `Normal`, `Strong`, or `null` (most monsters). Checks (`BaseCalc.ts:684-692`): `Strong` (or hardcoded `IMMUNE_TO_BURN_DAMAGE_NPC_IDS`) blocks normal AND strong burns; `Normal` blocks normal burns. Gates Bone/Burning claws, Scorching bow, Arkan blade burn DoT. `null` allowed since commit #888.

## 4. Scaling orchestration

`scaleMonster(m)` (`MonsterScaling.ts:19`; `ORDER_OF_OPERATIONS` array at `:10-17`) applies, in order:
`applyCoxScaling → applyTobScaling → applyToaScaling → applyVardScaling → applyMonsterPhases → applyDefenceReductions`. Each no-ops unless the monster is in its id/attribute set. `scaleMonsterHpOnly` re-runs only Vardorvis when just HP changed.

Math helpers (`Math.ts`): `iSqrt(x)=trunc(√x)`; `addPercent(x,p)=x+trunc(x*p/100)`; `lerp(curr,srcStart,srcEnd,dstStart,dstEnd)=trunc((curr-srcStart)*(dstEnd-dstStart)/(srcEnd-srcStart)+dstStart)`.

### Chambers of Xeric (`scaling/ChambersOfXeric.ts`)
Gate: `attributes.includes('xerician')`. `CM_SCALE_PERCENT = 50`. Offensive skills = `atk,str,ranged` (+`magic` unless in `COX_MAGIC_IS_DEFENSIVE_IDS`); defensive = `def` (+`magic` if defensive). Skills already at 1 stay 1.

Multi (default):
```
partySize = clamp(partySize,1,100); m1 = partySize-1
highestComLevel = clamp(partyMaxCombatLevel,60,126)
highestHp = clamp(55 + trunc(44*partyMaxHpLevel/99), 55, 99)
offensive = trunc(baseOffensive * highestHp/99); defensive = trunc(baseDefensive * highestHp/99)
hp = trunc(baseHp * highestComLevel/126)
offensive = trunc(offensive * (100 + iSqrt(m1)*7 + m1) / 100)
defensive = trunc(defensive * (100 + iSqrt(m1) + trunc(m1*7/10)) / 100)
hp += hp * trunc(partySize*50/100)
// CM: offensive +50%; hp +50% (unless glowing crystal); defensive +50% (Tekton +20/35, glowing crystal none)
clamp hp[50,30000], offensive[50,5000], defensive[50,20000]
```
Singles (scavengers/vespine): `hpScaler=clamp(partyMaxCombatLevel,60,126)`, `statScaler=clamp(partyMaxHpLevel,55,99)`, CM ×1.5 each; `hp=max(trunc(baseHp*hpScaler/126),5)`, each stat `=max(trunc(base*statScaler/99),1)`. Olm: `hp=(melee/mage hand?600+300*f:800+400*f)`, `f=min(partySize-1,50)-3*trunc(min(partySize,50)/8)`.

### Theatre of Blood (`scaling/TheatreOfBlood.ts`) — HP only
Normal/HMT: `partySize=clamp(3,5)`, `hp=trunc(hp*(partySize+3)/8)`. Entry mode: factor table `1→10/40, 2→19/40, 3→27/40, 4→34/40, 5→40/40`.

### Tombs of Amascut (`scaling/TombsOfAmascut.ts`) — HP here; off/def scale at roll time via `(250+invo)/250`
```
newHp = (ejected core? 4500 : skills.hp)
invoFactor = trunc((ejectedCore?1:4) * toaInvocationLevel / 10); newHp += trunc(newHp*invoFactor/100)
if pathMonster and pathLevel>=1: newHp = trunc(newHp * (100 + 3 + 5*toaPathLevel)/100)
partySize=clamp(1,8); if >=2: partyFactor = 9*(>=3?2:1) + (>=4? 6*(partySize-3):0); newHp = trunc(newHp*(10+partyFactor)/10)
if newHp>100: round to nearest (newHp>300?10:5), half-up
```

### Vardorvis (`scaling/Vardorvis.ts`) — str/def lerp with current HP
Versions: Quest (maxHp 500, str 210→280, def 180→130), Awakened (1400, 391→522, 268→181), default (700, 270→360, 215→145). `str=lerp(currHp,maxHp,0,strStart,strEnd)`, `def=lerp(...)`. As HP→0, str rises and def falls.

### Phases (`scaling/Phases.ts`) — only if `inputs.phase` set
Araxxor 'Enraged': `def+=35, magic+=28, ranged+=31`. Yama: `defensive.magic = 60` if 'Tank using magic' else `-30`. Other phase effects (TD, Hueycoatl pillar ×13/10, Abyssal Sire transition ÷2, Doom shielded immunity) live in `PlayerVsNPCCalc.ts`. Phase lists per boss: `MONSTER_PHASES_BY_ID` in `constants.ts`.

## 5. Defence reductions (`scaling/DefenceReduction.ts`) — runs last

`baseSkills` snapshotted up front (arclight/emberlight drain off this original; DWH/elder maul drain the running value). Defence floors (`getDefenceFloor`): Verzik/Vardorvis→own def, Sotetseg 100, Nightmare 120, Akkha 70, Baba 60, Kephri 60, Zebak 50, P3 Warden 120, ToA obelisk 60, Nex 250, Araxxor 90, Hueycoatl 120, Yama 145, else 0.

Order:
1. **Accursed sceptre** (xor vuln): `def=trunc(def*17/20)`, `magic=trunc(magic*17/20)` (−15% both).
2. **Vulnerability**: `def=trunc(def*9/10)` (−10%).
3. **Elder maul** ×iters: each `def -= trunc(def*35/100)`.
4. **DWH** ×iters: each `def -= trunc(def*3/10)`.
5. **Arclight/Emberlight** (off baseSkills): per stat (atk/str/def) `-= iter*(trunc(num*base/den)+1)`; factor Arclight `[2,20]` if demon else `[1,20]`, Emberlight `[3,20]` if demon else `[1,20]`.
6. **Tonalztic** ×iters: each `def -= trunc(magic/10)`.
7. **Seercull**: `magic -= seercull`.
8. **BGS**: single pool drains `def→str→atk→magic→ranged`; if a stat fails to fully drain (incl. floor) the BGS stops propagating; else overflow carries to the next stat.
9. **Ayak**: drains the magic **defence bonus** (`defensive.magic`), not the level: `defensive.magic = max(0, defensive.magic - ayak)`.

## 6. Worked anchors (from tests)

- **Abyssal demon (Standard)** vs empty player: `npcDefRoll = 12096` (def level 135 → effective 144; `144 × (defensiveStat + 64) = 12096`), `maxAttackRoll = 7040`, `maxHit = 11`, `accuracy ≈ 29.10%`, `dps ≈ 0.677` (`CombatCalc.test.ts`, `DefenceRolls.test.ts`).
- ToA HP scaling, CoX stat blocks, Vardorvis HP-lerp all regression-tested in `src/tests/lib/scaling/*` and `src/tests/calc/*`.

## 7. Wiki scraper (`scripts/generateMonsters.py`)

Pulls `infobox_monster` from the wiki Bucket API (`action=bucket`), paginated by 500. Maps levels/bonuses/defences (incl. the three `*_range_defence_bonus` → light/standard/heavy), `elemental_weakness(+_percent)`, `burn_immune` (substring → Weak/Normal/Strong), `slayer_experience`→`is_slayer_monster`. Skips CoX CM variants, Deadman, discontinued, etc. Merges `scripts/manual_monster.json`. (Note: `poison_immune` field was renamed `poison_resistance`, commit #889; poison isn't modeled in DPS.)
