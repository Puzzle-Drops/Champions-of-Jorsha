# IDLE ARPG PARTY — Game Design Document

**Working Title:** *Crucible* (placeholder)
**Version:** 0.1 — Foundational GDD
**Document Purpose:** Specification for engineering implementation. Hand to Claude Code (or any engineer) for build-out.

> **Reading Notes**
> Sections marked `[DESIGNER CALL]` are gaps I filled — review and adjust.
> Sections marked `[OPEN]` are unresolved questions parked at the bottom.
> Numbers in formulas are starting values for tuning, not final balance.

---

## 1. Vision

A deep-but-readable **idle ARPG party game** with a roster of 48 distinct classes. The player builds a party of 6, sends them into infinite-scaling dungeons, and watches automated combat resolve while loot rains down. Progression is *roster-wide* — every class you've ever played retains its inventory, tech tree, and unlocks forever.

The fantasy is **commanding a guild**, not piloting a single character. You are the guildmaster pulling units off the bench, gearing them up, and pushing them deeper.

---

## 2. Core Pillars

1. **Roster as Garden.** Every class is a permanent save. Switching classes never wastes time — you're tending 48 plants, not picking one.
2. **Stones Are Identity.** Equipment stats are gear. *Abilities* come from Stones. A Knight without a Spell Stone literally cannot cast — and that's a build choice.
3. **Loot Math Is Honest.** Rarity is not a hidden roll — it is a transparent function of your stat percentages. The player can read the math.
4. **Idle but Crunchy.** Combat resolves automatically, but team composition, gear allocation, and tech-tree paths reward thought. The "active" gameplay is *theory-crafting*.
5. **No Inventory Tetris.** Lists, slots, filters, sorts. Brutal readability.
6. **Infinite Vertical, Capped Horizontal.** Floors and items scale forever; abilities cap at level 100. Late game is about *gear chase*, not new buttons.

---

## 3. Core Loops

### 3.1 Macro Loop (per session)
```
Manage Roster → Pick Party of 6 → Enter Dungeon → Floor 1 → Boss → Floor 2 → ... → Death
                ↑                                                                    ↓
                └────────────── Sort loot, allocate tech, swap stones ←──────────────┘
```

### 3.2 Micro Loop (per fight)
```
Spawn → Auto-attacks tick on individual cooldowns → Spells auto-cast when off CD + mana available
      → Passives always-on → Boss dies → Loot rolls → Brief recovery → Next floor
```

### 3.3 Meta Loop (per career)
Push deeper → unlock harder dungeons → drop higher-level gear/stones → revisit earlier classes with better stones → push deeper.

---

## 4. Stats

### 4.1 Core Stats (every unit, every item)
| Stat | Abbrev | Role |
|------|--------|------|
| Health | **HP** | Pool of damage you can absorb. 0 = dead. |
| Mana | **MP** | Spell currency. Regenerates in combat. |
| Physical Attack | **P.ATK** | Scales physical damage abilities. |
| Magical Attack | **M.ATK** | Scales magical damage abilities. |
| Physical Defense | **P.DEF** | Reduces physical damage taken. |
| Magical Defense | **M.DEF** | Reduces magical damage taken. |

### 4.2 Stat Sources (additive)
1. **Class base stats** (per-level scaling, varies by class)
2. **Equipment** (weapon, armor, jewelry, stones — see §8)
3. **Tech Tree nodes** (per-class, see §13)
4. **Buffs** (temporary, from spells/passives)

### 4.3 Damage Formula `[DESIGNER CALL]`
```
rawDamage   = abilityPower × attackerStat
mitigation  = defenderDef / (defenderDef + 100 + 5×floorLevel)   // soft cap, scales with content
finalDamage = max(1, rawDamage × (1 - mitigation))
```
This makes defense *always relevant* but never invincible — high P.DEF on a floor-2000 boss still leaves a sliver of damage through.

### 4.4 Mana
- Regenerates at **5% max MP / second** during combat.
- Fully restored between floors.
- Spells gated by mana cost (defined per ability).

### 4.5 No Hidden Stats
No crit chance, dodge, or hit rate as core stats. **Crit and dodge exist only as effects granted by specific class passives or tech-tree nodes.** This keeps the stat sheet clean and makes those classes feel special.

---

## 5. Combat

### 5.1 Formation
Party arranged in **two rows of three**:
```
  [Back 1] [Back 2] [Back 3]      ← protected, ranged-friendly
  [Front 1][Front 2][Front 3]     ← takes hits
```
Player drags units into slots. Enemies mirror this layout.

### 5.2 Targeting Rules
- **Default enemy AI:** attacks lowest-HP frontline unit; if frontline empty, attacks backline.
- **Taunt:** force-targets the taunter for the taunt duration.
- **AoE:** ignores formation, hits all eligible.
- **Single-target ranged player abilities:** can hit any enemy slot.
- **Single-target melee player abilities:** prefer enemy frontline.

### 5.3 Pacing
- Each unit attacks on its own timer (set by attack speed = derived from Attack Stone level — see §7).
- Spells fire **automatically** when off cooldown AND mana sufficient AND a valid target exists.
- Player can **toggle auto-cast off** per spell if they want to manually fire (active-mode hardcore players). Default is on.

### 5.4 Speed Controls
`1× / 2× / 4×` toggles. Affects all timers but not RNG. `[DESIGNER CALL]`

### 5.5 Death & Reset
- Unit at 0 HP becomes **Downed**, contributes nothing, but isn't removed.
- All 6 downed = **party wipe** → kicked to hub, dungeon resets to Floor 1, **no item loss**.
- Surviving a floor revives downed units to 25% HP at the start of the next.

---

## 6. Class System

### 6.1 Family Overview (8 families × 6 classes = **48 total**)

| Family | Identity | Tag |
|--------|----------|-----|
| **Tanks** | Frontline absorbers — every Tank has Taunt somewhere in its kit | Defensive |
| **Fighters** | Melee DPS — about half carry brief Taunts | Bruiser |
| **Healers** | Keep the party alive — vary heal volume vs. utility | Support |
| **Marksmen** | Ranged physical DPS | Damage |
| **Rogues** | Single-target burst, mobility, status effects | Damage |
| **Magicians** | Pure magical DPS / control | Damage |
| **Mystics** | Hybrid utility — buffs, debuffs, totems, anti-magic | Utility |
| **Far Lands** | Exotic mechanics — pets, traps, RNG, risk/reward | Wildcard |

### 6.2 Class Anatomy
Every class has exactly **four ability slots**:
- **Slot 1 — Basic Attack** (driven by Attack Stone)
- **Slot 2 — Spell I** (driven by Spell Stone)
- **Slot 3 — Spell II** (driven by Spell Stone)
- **Slot 4 — Passive** (driven by Passive Stone)

If the corresponding stone is missing, **the slot is dead**. Level-1 starter stones come pre-equipped to every newly unlocked class so they always function.

### 6.3 The Roster

#### TANKS — *all carry Taunt*

**Knight** — *plate, sword & board, classic protector*
- **Attack:** Shield Bash — phys, low dmg, generates extra threat on hit.
- **Spell I:** Provoke — taunts 1 enemy 5s, +30% P.DEF self.
- **Spell II:** Shield Wall — 60% damage reduction self, 4s.
- **Passive:** Vigilant Guard — +P.DEF; regen 1% HP whenever struck.
- **Stat focus:** HP / P.DEF.

**Crusader** — *holy magic-tank, divine shield*
- **Attack:** Hammer of Faith — mixed phys+mag damage.
- **Spell I:** Divine Decree — taunts ALL enemies 3s, party +M.DEF.
- **Spell II:** Consecrated Ground — AoE around self: small heal pulse + reflect 20% mag damage taken.
- **Passive:** Sacred Aegis — M.DEF scales with HP missing (max +50% at 1 HP).
- **Stat focus:** HP / M.DEF.

**Sentinel** — *spell-tank, mana shield*
- **Attack:** Runic Strike — mag damage, applies slow.
- **Spell I:** Mana Bulwark — incoming magical damage drains MP instead of HP, 6s.
- **Spell II:** Banishment Sigil — taunts + silences single target 3s.
- **Passive:** Spell Eater — overhealing converts to MP at 50% rate.
- **Stat focus:** MP / M.DEF.

**Warden** — *nature tank, thorns and roots*
- **Attack:** Bramble Slam — phys, applies bleed.
- **Spell I:** Earthen Roar — taunts all enemies + slows 4s.
- **Spell II:** Thornwall — reflects 30% incoming phys damage 6s.
- **Passive:** Deep Roots — immune to displacement; +HP regen out of combat.
- **Stat focus:** HP / P.DEF.

**Juggernaut** — *immovable wall*
- **Attack:** Cleave — phys, hits 2 adjacent enemies.
- **Spell I:** Unbreakable — immune to crowd control 4s; next hit you take taunts the attacker.
- **Spell II:** Quaking Stomp — AoE phys + 2s taunt.
- **Passive:** Last Stand — under 30% HP, +50% all defenses.
- **Stat focus:** HP / P.DEF.

**Shieldmage** — *hybrid, mana-shield magic tank*
- **Attack:** Aether Bolt — mag damage, briefly taunts hit target.
- **Spell I:** Arcane Barrier — absorb shield equal to % max MP.
- **Spell II:** Pull — yanks enemy into your row, taunting them 3s.
- **Passive:** Mana Armor — incoming damage reduced by % current MP / 2.
- **Stat focus:** MP / M.DEF.

---

#### FIGHTERS — *melee bruisers, some have brief Taunts*

**Berserker** — *rage, glass cannon melee*
- **Attack:** Reaving Strike — phys, builds Rage stack.
- **Spell I:** Bloodthirst — heal 50% damage dealt for 6s.
- **Spell II:** Whirlwind — AoE phys; consumes Rage for bonus damage.
- **Passive:** Unbridled Fury — +5% P.ATK per missing 10% HP.
- **Stat focus:** P.ATK / HP.

**Gladiator** — *crowd-pleaser, brief taunt*
- **Attack:** Crowd Strike — phys, taunts target 2s.
- **Spell I:** Net Cast — root + phys damage.
- **Spell II:** Showmanship — party gains +20% crit damage 5s.
- **Passive:** Arena Veteran — +15% P.ATK & P.DEF when 3+ enemies present.
- **Stat focus:** P.ATK / P.DEF.

**Samurai** — *crit & parry*
- **Attack:** Iaijutsu — phys, +crit chance.
- **Spell I:** Wave Slash — line AoE phys.
- **Spell II:** Meditation — 2s wind-up; next attack guaranteed crit + bonus damage.
- **Passive:** Bushido — parry chance scales with P.ATK; parried hits return damage.
- **Stat focus:** P.ATK.

**Monk** — *speed, combo, ki*
- **Attack:** Flurry — 3 fast hits, builds Combo points.
- **Spell I:** Tiger Palm — heavy phys; consumes all Combo for damage multiplier.
- **Spell II:** Inner Peace — self HP regen + cleanse.
- **Passive:** Way of the Open Hand — attack speed scales with missing MP %.
- **Stat focus:** P.ATK / MP.

**Spellblade** — *hybrid melee, brief taunt*
- **Attack:** Enchanted Edge — scales off both P.ATK and M.ATK.
- **Spell I:** Phase Slash — teleport behind target, mag damage.
- **Spell II:** Aetheric Challenge — taunts 1 enemy 3s, mag damage.
- **Passive:** Arcane Edge — P.ATK contributes 50% of its value to M.ATK (and vice versa).
- **Stat focus:** P.ATK / M.ATK.

**Paladin** — *holy fighter, brief group taunt*
- **Attack:** Smite — phys + mag.
- **Spell I:** Word of Glory — small party heal + 2s aura-taunt around self.
- **Spell II:** Hammer of Justice — single-target stun.
- **Passive:** Aura of Valor — party +5% all attack while alive.
- **Stat focus:** P.ATK / M.ATK.

---

#### HEALERS — *vary heal volume vs. utility*

**Cleric** — *standard holy healer*
- **Attack:** Holy Bolt — mag damage.
- **Spell I:** Greater Heal — 3–6% target max HP/sec for 4s.
- **Spell II:** Group Mend — small heal to all (1–2% max HP).
- **Passive:** Devout — overhealing converts to MP at 50%.
- **Stat focus:** M.ATK / MP.

**Druid** — *HoT specialist*
- **Attack:** Wrath — mag damage.
- **Spell I:** Rejuvenation — HoT 1–2%/tick × 8 ticks.
- **Spell II:** Wild Growth — HoT to entire party.
- **Passive:** Living Seed — HoTs that reach completion heal nearest ally for 50% of the tick.
- **Stat focus:** M.ATK / MP.

**Templar** — *low heals, big buffs* (per your example)
- **Attack:** Smite Lite — mag damage.
- **Spell I:** Blessed Ward — heal 1–5% target HP **AND** grant +M.DEF buff 6s.
- **Spell II:** Sanctuary — small heal + 1s damage immunity on target.
- **Passive:** Resolve Through Faith — every heal you cast reduces target's threat by 5%.
- **Stat focus:** M.ATK / M.DEF.

**Oracle** — *cleanse + foresight*
- **Attack:** Mind Pierce — mag, partially ignores M.DEF.
- **Spell I:** Cleansing Light — remove 1 debuff + small heal.
- **Spell II:** Foresight — next attack on target ally is dodged (5s window).
- **Passive:** Tides of Fate — random 10% chance heals critically (×2).
- **Stat focus:** M.ATK / MP.

**Bloodpriest** — *heals via damage dealt*
- **Attack:** Crimson Lash — mag damage; heals lowest-HP ally for 50% of damage dealt.
- **Spell I:** Sanguine Bond — link target ally; party damage dealt heals them too.
- **Spell II:** Hemorrhage — mag damage + target's next strike against you heals you.
- **Passive:** Vital Exchange — every 30s, sacrifice 5% own HP to grant 10% party HP.
- **Stat focus:** M.ATK / HP.

**Lifebinder** — *chain heals & damage-splitting*
- **Attack:** Soul Tether — mag damage, marks target.
- **Spell I:** Chain Mend — heal jumps 3 times, decreasing 50% per hop.
- **Spell II:** Communion — link 2 allies; damage they take is split 50/50.
- **Passive:** Tethered Souls — when ally would die, transfer 25% your HP to save them (60s CD).
- **Stat focus:** M.ATK / HP.

---

#### MARKSMEN — *ranged physical*

**Archer** — *versatile bow*
- **Attack:** Quick Shot — phys, fast cooldown.
- **Spell I:** Multishot — phys hits 3 random enemies.
- **Spell II:** Piercing Arrow — phys line AoE, ignores 50% P.DEF.
- **Passive:** Eagle Eye — +crit chance vs enemies above 80% HP.
- **Stat focus:** P.ATK.

**Crossbowman** — *slow, heavy, armor pierce*
- **Attack:** Bolt — phys, slow but high damage.
- **Spell I:** Armor Piercing Shot — ignores all P.DEF.
- **Spell II:** Repeater Burst — 5 fast bolts, then 6s reload.
- **Passive:** Heavy Draw — longer cooldowns deal more damage (idle attack timer = bonus).
- **Stat focus:** P.ATK.

**Sniper** — *patient, mark & headshot*
- **Attack:** Marked Shot — phys, debuffs target +20% damage taken.
- **Spell I:** Headshot — massive single-target, **only** lands on Marked targets.
- **Spell II:** Camouflage — party untargetable 2s.
- **Passive:** Patience — every 5s without attacking, next shot is a guaranteed crit.
- **Stat focus:** P.ATK.

**Hunter** — *beast companion*
- **Attack:** Aimed Shot — phys.
- **Spell I:** Beast Strike — pet attacks for phys damage.
- **Spell II:** Trap — root area for 4s.
- **Passive:** Loyal Beast — companion has 50% your stats and auto-attacks alongside you.
- **Stat focus:** P.ATK.

**Gunner** — *fast multishot AoE*
- **Attack:** Trick Shot — phys, ricochets to 1 extra random.
- **Spell I:** Spray and Pray — rapid fire to random targets, 3s.
- **Spell II:** Grenade — AoE phys + brief stun.
- **Passive:** Hot Hand — consecutive hits build attack speed (max +50%, decays out of combat).
- **Stat focus:** P.ATK.

**Falconer** — *bird scout, dive bombs*
- **Attack:** Talon Strike — phys, applies bleed.
- **Spell I:** Dive Bomb — target single + adjacent for phys.
- **Spell II:** Sky Watch — party +dodge 5s.
- **Passive:** Keen Sight — party +5% crit chance while you're alive.
- **Stat focus:** P.ATK.

---

#### ROGUES — *burst, mobility, status*

**Assassin** — *single-target burst*
- **Attack:** Backstab — phys, +200% crit damage.
- **Spell I:** Vanish — untargetable 3s; next attack deals double damage.
- **Spell II:** Throat Slit — execute, +50% damage to enemies under 30% HP.
- **Passive:** Shadow Step — after a kill, +100% attack speed for 2s.
- **Stat focus:** P.ATK.

**Duelist** — *sustained 1v1*
- **Attack:** Riposte Stab — phys; if dodged this attack, refunds CD.
- **Spell I:** Lunge — gap-close + bleed.
- **Spell II:** Honor Duel — locks target; both deal +30% damage to each other 8s.
- **Passive:** Parry Stance — dodge chance scales with P.ATK.
- **Stat focus:** P.ATK.

**Shadowdancer** — *mobility, dodge stacks*
- **Attack:** Twin Fangs — 2 fast phys hits.
- **Spell I:** Smoke Step — auto-dodge next 2 attacks.
- **Spell II:** Shadow Clone — creates a copy that mirrors your basic attack 4s.
- **Passive:** Liquid Movement — each successful dodge grants +5% attack speed (stacking).
- **Stat focus:** P.ATK / MP.

**Trickster** — *status effects galore*
- **Attack:** Tainted Blade — phys, applies a random debuff (slow / poison / weaken / blind).
- **Spell I:** Smoke Bomb — party +dodge 4s.
- **Spell II:** Sabotage — target's P.ATK and M.ATK reduced 30% for 5s.
- **Passive:** Mischief — debuffs you apply last 50% longer.
- **Stat focus:** P.ATK.

**Bladestorm** — *multi-hit AoE rogue*
- **Attack:** Crossblade — phys, hits adjacent.
- **Spell I:** Whirling Death — channeled AoE phys 3s.
- **Spell II:** Glaive Throw — boomerangs, 2 hits per enemy.
- **Passive:** Steel Tornado — when 3+ enemies present, each hit grants +2% attack speed (max +30%).
- **Stat focus:** P.ATK.

**Reaper** — *death-themed kill bonuses*
- **Attack:** Soul Cleave — phys, heals self on kill.
- **Spell I:** Death's Scythe — line AoE; damage scales with kills this fight.
- **Spell II:** Reaping — instant-kill enemy under 15% HP.
- **Passive:** Harvest — gain stacking +P.ATK per kill, resets each floor.
- **Stat focus:** P.ATK / HP.

---

#### MAGICIANS — *pure casters*

**Pyromancer** — *fire DoTs*
- **Attack:** Firebolt — mag, ignites for 4s.
- **Spell I:** Fireball — AoE mag + burn stack.
- **Spell II:** Combustion — detonates all burns: damage = remaining DoT × 1.5.
- **Passive:** Heat Wave — each ignited enemy grants +5% M.ATK.
- **Stat focus:** M.ATK.

**Cryomancer** — *ice control*
- **Attack:** Frostbolt — mag, slows 30%.
- **Spell I:** Blizzard — AoE mag + stacking slow.
- **Spell II:** Frozen Tomb — freeze single target 3s.
- **Passive:** Glacial Force — +30% damage to slowed/frozen targets.
- **Stat focus:** M.ATK.

**Stormcaller** — *lightning chains*
- **Attack:** Spark — mag, jumps to nearest extra.
- **Spell I:** Chain Lightning — jumps 5 times, -10% per jump.
- **Spell II:** Thunderstorm — 4s rain, random strikes hit random enemies.
- **Passive:** Static Field — every cast leaves a charge; 3 charges = next cast is free.
- **Stat focus:** M.ATK.

**Arcanist** — *pure burst caster*
- **Attack:** Arcane Missile — mag, fires 3 missiles.
- **Spell I:** Arcane Blast — heavy mag, costs charge.
- **Spell II:** Mana Surge — refund 50% of MP spent for next 4s.
- **Passive:** Arcane Mind — +M.ATK proportional to current MP %.
- **Stat focus:** M.ATK / MP.

**Necromancer** — *summons, drain*
- **Attack:** Death Bolt — mag damage; leaves a corpse on kill.
- **Spell I:** Raise Skeleton — consumes corpse → minion for 10s.
- **Spell II:** Life Drain — channeled mag, heals self 50% damage.
- **Passive:** Master of the Dead — your minions live 4s longer; on death they explode for AoE mag.
- **Stat focus:** M.ATK / MP.

**Chronomancer** — *time, cooldown manipulation*
- **Attack:** Time Bolt — mag, slows target's cooldowns.
- **Spell I:** Rewind — reset target ally's cooldowns.
- **Spell II:** Time Stop — freeze all enemies 2s.
- **Passive:** Temporal Mastery — party gains 10% cooldown reduction.
- **Stat focus:** M.ATK / MP.

---

#### MYSTICS — *hybrid utility*

**Bard** — *buff songs*
- **Attack:** Sound Wave — small AoE mag.
- **Spell I:** Song of Valor — party +20% P.ATK 8s.
- **Spell II:** Crescendo — consume an active song to deliver a large party heal.
- **Passive:** Inspiring Presence — passive +5% party attack speed.
- **Stat focus:** M.ATK / MP.

**Witch** — *curses and hexes*
- **Attack:** Hex Bolt — mag, applies a random curse.
- **Spell I:** Curse of Frailty — target -50% all defenses 6s.
- **Spell II:** Hex Toad — transforms target into a pacifist for 4s (cannot attack).
- **Passive:** Cackling — cursed enemies grant party +MP regen on hit.
- **Stat focus:** M.ATK.

**Shaman** — *totems*
- **Attack:** Totemic Strike — mag; places a small totem on the ground.
- **Spell I:** Earth Totem — 4s, taunts AoE.
- **Spell II:** Healing Totem — 4s, party HoT.
- **Passive:** Spirit Link — your totems also grant party +element resist.
- **Stat focus:** M.ATK / M.DEF.

**Inquisitor** — *anti-magic*
- **Attack:** Holy Bolt — mag, dispels 1 enemy buff.
- **Spell I:** Mantra of Silence — silences target 3s.
- **Spell II:** Anti-Magic Field — party immune to spell damage 2s.
- **Passive:** Sanctified — party +M.DEF and +20% debuff resist.
- **Stat focus:** M.ATK / M.DEF.

**Spiritualist** — *channel ancestral spirits*
- **Attack:** Ghost Bolt — mag, marks target.
- **Spell I:** Spirit of the Bear — target +30% max HP for 6s.
- **Spell II:** Spirit of the Wolf — party +attack speed 5s.
- **Passive:** Ancestral Wisdom — +50% mana regen for entire party.
- **Stat focus:** M.ATK / MP.

**Geomancer** — *earth, terrain*
- **Attack:** Stone Throw — phys damage that scales with M.ATK (instead of P.ATK).
- **Spell I:** Earthquake — AoE phys + slow + 1s taunt.
- **Spell II:** Stone Skin — target +200 P.DEF 5s.
- **Passive:** Mountain's Heart — your HP scales with M.ATK.
- **Stat focus:** M.ATK / HP.

---

#### FAR LANDS — *exotic, off-pattern mechanics*

**Engineer** — *turrets, traps*
- **Attack:** Wrench Toss — small mixed damage.
- **Spell I:** Deploy Turret — auto-attacks for 12s.
- **Spell II:** Spike Trap — delayed AoE phys (arms in 1s, lasts 8s).
- **Passive:** Tinkerer — your deployables last 50% longer.
- **Stat focus:** P.ATK / M.ATK.

**Alchemist** — *potions, transmutation*
- **Attack:** Acid Splash — small AoE mag, corrodes P.DEF.
- **Spell I:** Healing Elixir — target heal + cleanse.
- **Spell II:** Volatile Mixture — random outcome (heal / damage / random buff).
- **Passive:** Transmutation — small chance dropped Stones come at +1 level.
- **Stat focus:** M.ATK / MP.

**Beastmaster** — *multi-pet*
- **Attack:** Whip Crack — phys, signals pets.
- **Spell I:** Pack Hunt — summons 3 wolves, attack random for 6s.
- **Spell II:** Bear Companion — tanky pet 8s, taunts.
- **Passive:** Pack Leader — all your summons inherit 30% your stats.
- **Stat focus:** P.ATK.

**Astromancer** — *star alignments*
- **Attack:** Star Shard — mag.
- **Spell I:** Solar Flare — mag AoE; auto-crits during Day phase.
- **Spell II:** Lunar Veil — party HoT; auto-crits during Night phase.
- **Passive:** Cosmic Cycle — alternates Day/Night every 8s; bonuses match phase.
- **Stat focus:** M.ATK / MP.

**Sandwalker** — *mirages, blind*
- **Attack:** Sand Slash — phys, blinds 1s.
- **Spell I:** Mirage — creates 2 illusory copies that absorb 1 hit each.
- **Spell II:** Sandstorm — AoE mag + reduces enemy hit rate.
- **Passive:** Desert Resilience — regen 1% HP/sec; doubled while a target is blinded.
- **Stat focus:** P.ATK / MP.

**Voidcaller** — *risk/reward dark magic*
- **Attack:** Void Bolt — mag, costs 1% own HP per cast.
- **Spell I:** Sacrifice — lose 20% HP, gain +50% M.ATK 6s.
- **Spell II:** Abyssal Maw — channeled pull + AoE mag, drains your own HP each tick.
- **Passive:** From Below — under 50% HP, +M.ATK = (% HP missing).
- **Stat focus:** M.ATK / HP.

---

## 7. Abilities & Stones

### 7.1 Stone Slots (4 per unit)
| Slot | Stone Type | Drives Ability |
|------|-----------|----------------|
| Attack Slot | Attack Stone | Slot 1 (Basic Attack) |
| Spell I Slot | Spell Stone | Slot 2 (Spell I) |
| Spell II Slot | Spell Stone | Slot 3 (Spell II) |
| Passive Slot | Passive Stone | Slot 4 (Passive) |

> **Empty stone slot = dead ability.** A unit with no Spell I Stone literally can't cast Spell I.

### 7.2 Stone Levels (1 to 100)
- **Level 1 stones** (the starters): unlock the ability but provide **0 stats**.
- **Higher-level stones:** unlock the ability AND provide stats AND **scale the ability's parameters**.
- **Level 100 = max.** Floors past 100 still drop level 100 stones (with continuing stat improvement — see §8.2).

### 7.3 Ability Scaling From Stone Level
Each ability defines two endpoints:
```
Level 1  → minimum values
Level 100 → maximum values
Linear interpolation between.
```
A level-15 stone resolves at 15% from min to max for **every** parameter the ability defines.

**Example — Knight's Provoke:**
| Parameter | Lvl 1 | Lvl 100 |
|-----------|-------|---------|
| Cooldown | 18s | 8s |
| Self P.DEF buff | +15% | +60% |
| Taunt duration | 3s | 6s |
| Mana cost | 30 | 30 (some params don't scale — by design) |

A Level 50 Provoke Stone gives ~13s CD, +37.5% P.DEF, 4.5s taunt.

> **Per-class authoring:** Every ability has a JSON-style data block specifying its scaling endpoints. Engineering builds the interpolation engine once; designers tune the endpoints.

### 7.4 Stone Stat Roll (same rules as items)
Stones are **also items**. They roll all 6 stats: 0 to stoneLevel, namesake stat 0 to 2× stoneLevel. (See §8.)

### 7.5 Naming Convention
Stones have a **type** (which ability variant) and a **namesake stat**.
> *"Level 47 Vital Fireball Stone"* = Pyromancer Spell-I (Fireball variant), namesake HP, all stats 0–47, HP 0–94.

### 7.6 Multiple Variants `[DESIGNER CALL]`
For depth: each ability has 2–3 stone *variants* that drop. Same base ability, different scaling lean.
- *"Sharpened" Backstab Stone*: emphasizes raw damage scaling.
- *"Quick" Backstab Stone*: emphasizes cooldown reduction.
Player chooses which variant they prefer per build. Variants don't change *what* the ability does, just *how it scales*.

> Recommend launching v1 with **one variant per ability** and adding variants in a content patch. Don't slow ship.

---

## 8. Items & Loot

### 8.1 Equipment Slots (per unit)
| Slot | Source | Notes |
|------|--------|-------|
| Weapon | Dungeon 3 | Defines weapon class flavor (sword, staff, etc.) |
| Helm | Dungeon 1 | Armor |
| Chest | Dungeon 1 | Armor |
| Legs | Dungeon 1 | Armor |
| Gloves | Dungeon 1 | Armor |
| Boots | Dungeon 1 | Armor |
| Ring 1 | Dungeon 2 | Jewelry |
| Ring 2 | Dungeon 2 | Jewelry |
| Amulet | Dungeon 2 | Jewelry |
| Attack Stone | Dungeon 4 | See §7 |
| Spell I Stone | Dungeon 4 | See §7 |
| Spell II Stone | Dungeon 4 | See §7 |
| Passive Stone | Dungeon 4 | See §7 |

**Total: 13 equipment slots per unit.**

### 8.2 Item Generation
Every item has:
- **Item Level** = floor it dropped on.
- **All 6 stats** roll independently in `[0, itemLevel]`.
- **Namesake stat** rolls in `[0, 2 × itemLevel]`.
- **Rarity** is computed *after* the rolls (see §9).

**Example — Level 50 "Vital Sword" (namesake = HP):**
- HP roll: 0–100
- MP, P.ATK, M.ATK, P.DEF, M.DEF: 0–50 each
- Suppose rolls: HP 80, MP 12, P.ATK 22, M.ATK 5, P.DEF 33, M.DEF 17

### 8.3 Namesake Stat by Slot (loose convention)
- **Weapons** can be any namesake.
- **Armor** weights toward HP, P.DEF, M.DEF.
- **Jewelry** weights toward P.ATK, M.ATK, MP, HP.
- **Stones** namesake matches the ability's scaling lean (e.g., Fireball stones tend toward M.ATK).

> Loot tables specify namesake distribution per dungeon/floor.

### 8.4 Beyond Level 100 (Stones Only)
Stones are capped at level 100, but they continue to roll **stats** as if their level were the floor level. So a level 800 floor in Dungeon 4 drops a "level 100 stone" mechanically (ability-power-wise), but its **stats** roll on a 0–800 / 0–1600 scale.
> Implementation: separate `stoneAbilityLevel` (clamped 1–100) and `itemStatLevel` (uncapped) on the item.

---

## 9. Rarity

Rarity is a **transparent function** of how well the item rolled.

### 9.1 Roll Quality
For each stat, compute its roll percentage of its max possible:
```
rollPct(stat) = stat_value / max_possible_for_that_stat
```
The 5 non-namesake stats max at `itemLevel`; the namesake maxes at `2 × itemLevel`.

### 9.2 Total Quality Score
```
qualityScore = average of all 6 rollPcts  (0% to 100%)
```

### 9.3 Tiers
| Tier | Threshold | Color | Visual Treatment |
|------|-----------|-------|------------------|
| **Rusted** | < 20% | `#6b6b78` muted gray | flat, slight noise texture |
| **Common** | ≥ 20% | `#e4e4e7` white | clean white card |
| **Rare** | ≥ 40% | `#3b82f6` blue | subtle blue glow |
| **Epic** | ≥ 60% | `#a855f7` purple | purple glow + gradient border |
| **Mythic** | ≥ 80% | `#ec4899` pink | animated pink shimmer |
| **Legendary** | ≥ 90% | `#f59e0b` orange-gold | gold shimmer + particle drift |
| **Radiant** | ≥ 95% | rainbow shimmer | animated holographic gradient + screen flash on drop |

### 9.4 Drop Feedback
Radiant drops trigger a **screen-wide flash + slow-motion zoom on the loot card**. Make the player feel it.

---

## 10. Dungeons

### 10.1 The Four Dungeons

| # | Name `[DESIGNER CALL]` | Theme | Enemy Lean | Loot Specialty |
|---|--------|-------|------------|----------------|
| 1 | **The Iron Vaults** | Fortress, soldiers, war machines | Physical attackers | Armor (helm/chest/legs/gloves/boots) |
| 2 | **The Whispering Spires** | Floating arcane towers | Magical attackers | Jewelry (rings/amulets) |
| 3 | **The Hollowed Wilds** | Overgrown ruin, swarms | AoE-heavy enemies | Weapons |
| 4 | **The Shattered Spire** | Reality-warped peak | Mixed threats | Stones (attack/spell/passive) |

### 10.2 Floor Structure
**One boss encounter per floor.** Each encounter consists of:
- 1 boss
- 0–3 minions (scaling with floor)

Minions are flavor — the boss is the threat. Floor cleared = boss dies.

> **Why one fight per floor?** Per your design. Keeps the loop *push, push, push* with a satisfying loot beat per floor.

### 10.3 Floor Scaling `[DESIGNER CALL]`
```
enemyHP    = baseHP × (1.10) ^ floor × (1 + floor × 0.05)
enemyDmg   = baseDmg × (1.08) ^ floor × (1 + floor × 0.04)
enemyDef   = baseDef × (1.07) ^ floor
```
Mixed exponential + linear so early floors feel snappy and late floors slow appropriately.

### 10.4 Dungeon Selection
Player picks one of 4 from the hub. **All dungeons start at Floor 1 every run.** Highest-floor-reached is recorded per dungeon for bragging rights and milestone rewards.

### 10.5 Death Behavior
Wipe → eject to hub. Dungeon resets to Floor 1. Items already dropped are kept. No death penalty beyond losing the run.

### 10.6 Voluntary Exit
Player can leave a dungeon anytime between floors via a "Return to Hub" button. **No party swap allowed mid-run** (only between runs).

---

## 11. Party Management

### 11.1 The Roster
- All 48 classes are *eventually* unlockable.
- Starting classes `[DESIGNER CALL]`: 6 free at game start (one per family — pick from a list, or auto-grant Knight/Berserker/Cleric/Archer/Assassin/Pyromancer/Bard/Engineer — pick 6 out of 8 family starters).
- Additional classes unlocked via **floor milestones, dungeon completion, achievements, or rare drops**. `[OPEN]`

### 11.2 Party Slots
- 6 active slots, arranged in 2×3 formation.
- Drag-and-drop swap from roster ↔ active.
- **Locked while in a dungeon run.**

### 11.3 Per-Unit Persistence
Every class has its own:
- Level + XP
- Tech Tree state
- Equipment (13 slots)
- Inventory (its own bag)
- Cosmetic state (gear visuals, etc.)

Swapping a unit out **never loses progress**. Swap them back later, everything is exactly as it was.

### 11.4 XP Sharing `[DESIGNER CALL]`
Active party members each gain XP from kills. **Bench classes do not gain XP.** This creates a real choice: do you push deeper with your favorites, or rotate to level the bench?

> *Alternative:* "Patron" system — pick 1 bench class to passively gain 25% XP. Adds depth without trivializing the choice. Recommend this for v1.

---

## 12. Inventory & Equipment UI

### 12.1 Storage Model
Each class has its **own bag** — items sit with the unit who looted them, but you can transfer between units from the hub.

> **Decision:** items always drop to the *party shared* stash first; player drags to a specific unit. No accidental "wrong unit got it." `[DESIGNER CALL]`

### 12.2 Layout (Inventory Screen, 1920×1080)
```
┌──────────────────────────────────────────────────────────────────────────────┐
│  [Unit Portrait & Name]   [Level / XP Bar]              [Stat Sheet]         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   EQUIPPED (13 slots, paper-doll style with item cards):                     │
│   ┌─Wpn─┐ ┌Helm─┐ ┌Chest┐ ┌Legs─┐ ┌─Glv─┐ ┌─Bts─┐                          │
│   ┌Ring1┐ ┌Ring2┐ ┌Amul─┐ ┌Atk─┐ ┌Spl1┐ ┌Spl2┐ ┌Pass┐                       │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│   INVENTORY (filterable, sortable list — NO TETRIS):                         │
│   [Filter: All ▾] [Sort: Rarity ▾]   [Search...]                             │
│   ┌────────────────────────────────────────────────────────────────────┐     │
│   │ 🟧 Lvl 47 Radiant Vital Sword     | HP +89, ATK +30  | Equip [E]  │     │
│   │ 🟪 Lvl 50 Epic Strong Helm        | HP +22, P.DEF +30| Equip [E]  │     │
│   │ ⬜ Lvl 50 Common Spry Boots       | MP +5, P.DEF +12 | Equip [E]  │     │
│   │ ...                                                                │     │
│   └────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```
- **List, not grid.** Each row shows rarity, name, key stats, equip button.
- Click to expand a row → full stat breakdown + lore + compare-vs-equipped panel slides in.
- **Bulk actions:** "Equip Best By Score" button auto-equips highest qualityScore items per slot for quick gearing.

### 12.3 Item Card (universal component)
A consistent card design used everywhere items appear (loot drop, inventory row, comparison popover). Rarity color drives the border + glow; layout is identical across contexts.

---

## 13. Tech Tree

### 13.1 Structure (per-class)
Each class has a small tech tree:
- ~30 nodes
- 3 branches that bend toward the class fantasy (e.g., Knight: **Bulwark / Threat / Endurance**)
- 1 point per character level
- A **Keystone** node gates each branch's lower half (requires 5 invested in branch)
- A **Capstone** node at branch end (build-defining)

### 13.2 Node Types
- **Stat node:** flat or % stat increase (+10 P.DEF, +3% HP).
- **Modifier node:** changes how an ability behaves (e.g., "Provoke also slows targets").
- **Keystone:** powerful effect, build-defining.
- **Capstone:** transforms a class identity at the extreme (e.g., Knight Capstone "Living Wall": when below 30% HP, all party damage redirects to you and you take 50% less damage).

### 13.3 Resetting `[DESIGNER CALL]`
- Free reset until level 10.
- After level 10, costs gold (or Spirit currency, see §15).
- Cost scales with level.

### 13.4 Level Cap
**No hard cap.** XP per level scales smoothly so deep levels still feel rewarding but don't break ability scaling (which is capped to 100 anyway).

---

## 14. Progression Curves

### 14.1 What Caps and What Doesn't
| Variable | Cap |
|----------|-----|
| Stone level | **100** |
| Ability scaling | **100** (driven by stone) |
| Class level | None (XP curve gets steep) |
| Floor | None |
| Item stat rolls | None (scale with floor) |
| Tech tree points | = class level (no cap) |

### 14.2 Why This Works
You stop unlocking *new ability mechanics* at stone L100, but you keep finding **better rolls on stones and gear forever**. Endgame is a stat-roll chase. Late-tier players grind for that perfect Radiant Level-2000 Vital Sword.

---

## 15. Currencies & Economy

### 15.1 Currencies `[DESIGNER CALL]`

| Currency | Source | Sinks |
|----------|--------|-------|
| **Gold** | Loot, sold items | Tech tree resets, basic upgrades |
| **Dust** | Salvaging unwanted items | Re-rolling individual stats on equipment |
| **Spirit** | Boss kills (more from higher floors) | Class unlocks, premium re-rolls, stone-variant swaps |

### 15.2 Salvaging
Right-click any item → **Salvage**. Returns Dust scaled to item level + rarity.
> Filter "salvage all below rarity X" buttons for endgame QoL.

### 15.3 Re-Rolling
Spend Dust to re-roll a single stat on an equipped item. Costly but lets you fix one bad stat on an otherwise perfect drop.

---

## 16. Idle ≠ Offline

**The game is "idle" in the sense that combat resolves automatically while the player watches** — they don't input every attack. There is **no offline progression**: when the game is closed, nothing happens. When the player returns, they pick up exactly where they left off (the active run is paused, or they were back in the hub).

This is a deliberate design choice:
- Keeps the loot economy honest — every item came from a fight the player witnessed.
- No "while you were gone" summary screens to design or balance.
- Encourages active sessions where the player engages with combat speed, gear swaps, and theory-crafting.
- Eliminates the design pressure to make offline-AFK strictly weaker than active play (a recurring balance trap in idle games).

> Pause-on-blur is fine; resume-where-you-left-off is the contract. No background ticking.

---

## 17. Visual Direction

### 17.1 Aesthetic
**Dark-fantasy + clean data UI.** Think *Path of Exile's mood meets Diablo IV's polish*, but the menus read like a well-designed broker terminal.

### 17.2 Color System
```
--bg-deep:       #0a0b14   /* near-black background */
--bg-panel:      #14152a   /* navy panels */
--bg-panel-hi:   #1d1f3a   /* hover state */
--ink:           #e8e6d8   /* off-white text */
--ink-muted:     #8a8b9a   /* secondary text */
--brass:         #c89860   /* primary accent (gold-ish, not yellow) */
--brass-hot:     #e3b878   /* highlights, edges */
--ember:         #d63b3b   /* HP, damage, danger */
--azure:         #4a8fc0   /* Mana, magical */
--verdant:       #6b9e4a   /* healing, nature */
--rune:          #8b5cf6   /* magical effects */
```

Rarity colors per §9.3.

### 17.3 Typography
- **Display / headers:** *Cinzel* (medieval, but readable). Used for class names, dungeon titles, floor numbers.
- **Body / UI:** *Sora* or *Geist* — distinctive but neutral. Anything BUT Inter.
- **Numbers / stats:** *JetBrains Mono* — locked-width so columns line up beautifully.

### 17.4 Sprite Direction `[DESIGNER CALL]`
For v1, use **procedurally drawn pixel-art-ish silhouettes on canvas** (32×48 figures with palette swaps per class). Cheaper than commissioning 48 sprite sheets, looks intentional. Phase 2: real pixel art commissions.

### 17.5 Animation Principles
- **Combat:** lunge → impact frame → particle burst → return. Even simple sprites feel alive.
- **Loot drop:** card flips in from above, rests for ~600ms, settles. Higher rarity = longer settle + more particles.
- **Damage numbers:** float up, scale-in then drift. Color matches damage type.
- **Death:** sprite desaturates → fades → "[Downed]" tag overlays.
- **Boss kill:** brief slow-mo + flash.

### 17.6 Sound `[DESIGNER CALL]` `[OPEN]`
Out of scope for v1 prototype — placeholder hooks throughout. Plan for later: ambient dungeon track + SFX library.

---

## 18. UI Architecture

### 18.1 Top-Level Screens
1. **Hub** (default): roster on left, party on right, dungeon select bottom.
2. **Battle**: combat scene center, party portraits left, enemy portraits right, log + speed controls bottom.
3. **Unit Detail**: inventory, equipment, tech tree (tabbed).
4. **Roster Browser**: grid of all classes with unlock state.
5. **Settings**.

### 18.2 Persistent UI
- Top bar: gold / dust / spirit / settings cog / profile.
- Bottom bar (during battle): speed (1×/2×/4×), pause, return-to-hub-after-floor toggle.

### 18.3 Layout Target
1920×1080 reference, scales fluidly down to 1280×720 with breakpoints. No mobile target for v1.

### 18.4 Readability First
- All damage / heal numbers **always visible** during combat.
- Stat changes from gear hover use **green/red diffs** for instant compare.
- No skill icon ever shown without its cooldown timer overlay.
- Hover **anything** → tooltip explaining it.

---

## 19. Save System

### 19.1 What Persists
- Roster (every class's level, XP, equipment, inventory, tech, unlocks)
- Party composition
- Currencies
- Dungeon high-water marks
- Settings (audio, speed default, etc.)

### 19.2 When to Save
- After every floor cleared.
- On return to hub.
- On voluntary exit / window close.
- Auto-save every 60s as belt-and-suspenders.

### 19.3 Storage Backends
- **Web build (in-Claude):** `window.storage` API.
- **Local file build:** `localStorage` (5MB cap is plenty for JSON state).
- **Future native:** swap for filesystem.

Wrap in a single `storage.get/set` adapter so engineering can swap backends without touching gameplay code.

### 19.4 Format
JSON, single root object. Versioned (`saveVersion: 1`) so future migrations are clean.

---

## 20. Implementation Roadmap

### Milestone 1 — Vertical Slice
- 1 dungeon (Iron Vaults)
- 6 classes (1 per family — Knight, Berserker, Cleric, Archer, Assassin, Pyromancer)
- All 4 stone slots functional
- Item drops + rarity
- Inventory & equipment UI
- Save/load
- 50 floors of content scaling

### Milestone 2 — Feature Complete
- All 4 dungeons
- All 48 classes
- Tech tree
- Currencies + salvage + reroll

### Milestone 3 — Polish
- Sprite art commissioning
- Sound design
- Achievements
- Class unlock progression
- Endgame leaderboards

### Milestone 4 — Live Ops `[OPEN]`
- Seasonal dungeons
- New class drops
- Cosmetics

---

## 21. Open Questions

1. **Class unlocks** — How do you unlock the other 42? Floor milestones? Specific boss drops? Crafting from family currency?
2. **PvP / competitive layer** — None planned for v1. Worth a leaderboard for highest floor per dungeon?
3. **Crit/Dodge**: confirmed as effects (not stats) — is that final?
4. **Mid-run leaving** — do you keep the loot if you bail before dying? *(Current GDD: yes.)*
5. **Boss variety** — should each dungeon have a finite roster of bosses that cycle, or procedural? Mix?
6. **Stone variants** — ship in v1 (more depth, more design) or v2 (faster ship)?
7. **XP "Patron" system** — confirm preference: shared XP only across active party, or 1 bench-patron getting 25%?
8. **Premium currency / monetization** — out of scope for now, but worth a stub plan if this ships commercially.
9. **Sound design budget** — placeholder vs. real? Affects scope of M3.
10. **Class roster final-final** — any of the 48 you want renamed / removed / replaced? List is opinionated.

---

## Appendix A — Data Schema Sketches

### A.1 Class definition (data, not code)
```json
{
  "id": "knight",
  "family": "tank",
  "displayName": "Knight",
  "tagline": "plate, sword & board",
  "baseStats": { "hp": 120, "mp": 40, "patk": 18, "matk": 6, "pdef": 22, "mdef": 12 },
  "perLevelStats": { "hp": 18, "mp": 4, "patk": 2.0, "matk": 0.5, "pdef": 3.0, "mdef": 1.5 },
  "abilities": {
    "attack":   "shield_bash",
    "spell1":   "provoke",
    "spell2":   "shield_wall",
    "passive":  "vigilant_guard"
  }
}
```

### A.2 Ability definition
```json
{
  "id": "provoke",
  "type": "spell",
  "school": "physical",
  "manaCost": 30,
  "scaling": {
    "cooldown":         { "min": 18, "max": 8 },
    "selfPDefBuffPct":  { "min": 15, "max": 60 },
    "tauntDuration":    { "min": 3,  "max": 6 }
  },
  "effects": [
    { "type": "taunt", "targets": "single_enemy", "duration": "tauntDuration" },
    { "type": "buff",  "targets": "self", "stat": "pdef", "amountPct": "selfPDefBuffPct" }
  ]
}
```

### A.3 Item / Stone instance
```json
{
  "id": "uuid-...",
  "kind": "stone_attack",
  "abilityId": "shield_bash",
  "abilityLevel": 47,
  "statLevel": 47,
  "namesake": "patk",
  "stats": { "hp": 12, "mp": 5, "patk": 88, "matk": 0, "pdef": 9, "mdef": 14 },
  "qualityScore": 0.62,
  "rarity": "epic"
}
```

### A.4 Save root
```json
{
  "saveVersion": 1,
  "currencies": { "gold": 0, "dust": 0, "spirit": 0 },
  "roster": { "knight": { ... }, "berserker": { ... }, ... },
  "party": ["knight", "cleric", "archer", "assassin", "pyromancer", "berserker"],
  "dungeons": {
    "iron_vaults":      { "highestFloor": 47, "currentRunFloor": null },
    "whispering_spires":{ "highestFloor": 12, "currentRunFloor": null },
    "hollowed_wilds":   { "highestFloor": 0,  "currentRunFloor": null },
    "shattered_spire":  { "highestFloor": 0,  "currentRunFloor": null }
  },
  "settings": { "speed": 2, "autoCast": true }
}
```

---

*End of GDD v0.1.*
