# models.md — Crucible Class Implementation Guide

This document is your handoff context. You should also have **tanks.html** (the canonical reference implementation, six tank classes built and working) and **GDD.md** (the game design doc, which lists all 48 classes across 8 families with their abilities and stat focuses). Together those three documents give you everything you need to build the next class family file in this session.

## What you're building

Crucible is an idle ARPG party game in Three.js + HTML/CSS. Combat is auto-resolved; the player manages a roster of six champions, gear, builds. There are 48 total champions organized as 8 families of 6 classes each: **Tanks, Fighters, Healers, Marksmen, Rogues, Magicians, Mystics, Far Lands**.

The Tanks family is done. Your job in this session is to pick one of the remaining families (or take direction from the user on which) and produce a single HTML file with the same architecture as tanks.html — six classes of that family, swappable via a dropdown viewer, with the importable game-library code clearly delineated from the preview-viewer code.

The user works one family per session. They'll iterate per-class within the session: you build one class, show it, take feedback, build the next. By the end of the session there's a complete family file. Then the next session is a different family.

## File architecture

Each family file is a single self-contained HTML file with two clearly-delineated sections, marked with banner comments:

```
╔══════════════════════════════════════════════════════════════════════════╗
║  GAME LIBRARY — <FAMILY> FAMILY                                          ║
║  ... lifecycle contract, importable code ...                             ║
╚══════════════════════════════════════════════════════════════════════════╝

(... math helpers, materials, mesh helper, effect classes, EffectManager,
 shared builders, Champion base class, six class definitions + animation
 tables ...)

╔══════════════════════════════════════════════════════════════════════════╗
║  VIEWER — preview-only setup                                             ║
║  Game runtime: ignore everything below this line.                        ║
╚══════════════════════════════════════════════════════════════════════════╝

(... scene/camera/renderer, lights, dais, embers, post-processing,
 class registry, swap logic, UI wiring, main loop ...)
```

The reason for this split: Claude Code (the agent that builds the actual game) will import the GAME LIBRARY section into game code as a module, and ignore the viewer entirely. The viewer is just for the user to preview classes during design. So everything above the viewer banner needs to be reusable game code with no DOM dependencies; everything below it can freely touch DOM, OrbitControls, post-processing, etc.

The HTML file structure: head with Cinzel + JetBrains Mono + Sora fonts, minimal CSS for the overlay UI, body with the canvas + dropdown selector + ability buttons + demo button + loading splash, importmap for Three.js, then a single `<script type="module">` that contains both the library and viewer code in order.

## The Champion base class — the lifecycle contract

Every champion in the game extends `Champion`. Read the actual implementation in tanks.html (search for `class Champion`) — it's about 100 lines and worth understanding fully. Here's the contract:

The lifecycle:

1. **constructor(scene, position)** — call `super()`, then `this.build()`, then `this.captureRestPose()`, then assign `this.animations = <yourClass>Animations`. That's the standard order.
2. **build()** — assemble all the geometry. Add named parts to `this.parts` (e.g., `this.parts.body = torso; this.parts.leftArm = arm;`). Set `this.cape` to the cape mesh if there is one. Set `this.eyeMaterial` to the emissive material used for the character's eye glow.
3. **captureRestPose()** — auto-snapshot of every part's transform. Pose deltas during animation are added to this snapshot.
4. **playAnimation(name, time, manager)** — picks an animation off `this.animations` by name, records start time, clears fired-effects tracking. Manager is the EffectManager.
5. **update(time, dt)** — call this every frame. Handles idle breathing, cape billow, eye pulse, and active animation playback automatically. You generally don't override this; the Shieldmage in tanks.html overrides it to spin orbital rune anchors and is a good template if you need similar behavior.

Required setup in `build()`:

- `this.parts` — dictionary of named parts you want to animate. Common names: `body` (torso), `leftArm`, `rightArm`, `head` (helm), `weapon` (right-hand weapon), `shield` (left-hand shield), and whatever else makes sense (e.g., `sword`, `staff`).
- `this.cape` — set to the cape mesh built via `buildCape({material})`. The base class auto-animates this with billow + sway in update(). If the character doesn't have a cape, leave it `null`.
- `this.eyeMaterial` — set to whichever emissive material is the character's eye glow (e.g., `M.emberEye` for Knight, `M.cyanEye` for Sentinel). The base class pulses its `emissiveIntensity` in update(). Set `M.<eyeMaterial>.userData.baseIntensity = <number>` before assigning, so the pulse oscillates around that base value.

Hand-attached weapons and shields work like this: when you call `buildArm(side, angle, palette)`, the resulting arm has a `arm.userData.hand` Object3D at the wrist position. You attach the weapon/shield as a child of that hand object (e.g., `leftArm.userData.hand.add(shield)`), then position and rotate the weapon in hand-local space. When the arm rotates during an animation, the weapon goes with it because of the parenting.

## Materials (M palette)

There's a single global dictionary `M` at the top of the library section, holding all `THREE.MeshStandardMaterial` instances used by every class in the family. Each class has a "palette section" (steel/brass/red for Knight; ivory/gold/cream for Crusader; slate/silver/cyan for Sentinel; etc.) plus a shared section at the bottom (leather, blade, stone, cloth, etc.).

The naming convention is `<palette><Tone>`: e.g., `steelDark`, `steel`, `steelHi` (dark/mid/bright variants). Eye materials are named `<palette>Eye` and rune materials are `<palette>Rune`. Cape materials are `<palette>Cape` and `<palette>CapeTrim` (or just `cape` and `creamTrim` for the older tanks — the convention tightened up over time, don't worry about renaming the old ones).

Materials are shared across all instances of a class — when Knight's eye pulses in update(), it's modulating the global `M.emberEye.emissiveIntensity` directly, not a clone. This is fine because we only show one champion at a time in the viewer, and in actual gameplay you'd be modulating per-instance via shaders or per-mesh material clones. Don't worry about it for now; just use the global pattern.

When you add a new family, add new palette materials at the top of M, before the "Shared materials" comment marker. For example, the Healers family might add `clothWhite`, `goldHoly`, `silkBlue`, `lifeRune`, etc. as new entries. Reuse shared materials (`leather`, `blade`, `stone`, `wood` if added by Warden, `bone` if added by Warden, etc.) freely.

## Shared part builders

Three reusable builders that every tank class uses:

**buildLeg(side, palette)** — full leg from thigh to boot, with knee plate, greave, and toe armor. The `side` is `-1` (left) or `1` (right). The `palette` is an object `{ dark, mid, accent, trim, leather }` whose values are material references. If you pass `{}` or omit it, defaults to Knight's steel/brass.

**buildCape(opts)** — the geometry-animated cape. Returns a mesh whose vertices are auto-billowed in the Champion's update(). Opts: `{ material, width = 1.3, height = 1.85 }`. The cape is positioned at `(0, 2.36, 0)` typically, draping down behind the character. The base material should be DoubleSide so both sides render.

**buildArm(side, shoulderAngle, palette)** — full arm from shoulder to gauntlet, with a `hand` Object3D at the wrist. Palette: `{ mid, dark, hi, accent, cloth }`. The `shoulderAngle` is the resting Z-rotation (small positive angle for the off-hand at rest, 0 for the weapon arm). Returns the arm group; attach to root and assign to `this.parts.leftArm` / `rightArm`.

These builders are tuned for the heroic-plate silhouette tanks use (around 3 units tall, broad shoulders, full plate). They generally work for any plate-armored character — Crusader, Sentinel, Warden, Juggernaut, Shieldmage all reuse them with palette overrides only.

For families with substantially different silhouettes — robed Healers, lean Marksmen, hooded Rogues — you have two options:

1. **Reuse with material trickery**: pass a palette where the "armor" materials are actually cloth or leather. The leg geometry doesn't change, but it reads as cloth-pants instead of plate-greaves. Cheap and consistent.

2. **Inline the geometry**: build the leg/arm directly in the class's `build()` method without calling the helper. This is what to do if you want fundamentally different proportions (skirted robes, archer's lean build, etc.). You can still reuse the arm's `userData.hand` pattern — just create an Object3D at the wrist and add it to the arm group.

Pick the lighter approach when possible. Visual identity at the chest/shoulders/helm/weapon level is much more expressive than tweaking leg proportions, so most families can probably get away with palette-only changes to the legs and arms, and put their distinctive silhouette work in the torso/cape/helm/weapon.

## The effect system

There's a base `Effect` class and nine concrete subclasses, all in the library section:

- **ShockwaveRing** — expanding ring with fading opacity. Color, max radius, duration. Default radial impact effect.
- **FilledShockwave** — filled disc, like a bright burst at the center. Pairs nicely with ShockwaveRing.
- **RuneRing** — shader-based ring with rotating runic glyph segments. Color, max radius, runeCount, rotateSpeed. The signature "magical circle" effect.
- **EnergyDome** — shader-based hemispherical dome with fresnel rim glow + sweep lines. The protective bubble effect.
- **HolyCircle** — shader-based ground seal with multiple ring layers, star pattern, rotating runes. Used for big AOE buff effects (Crusader's Consecration, Warden's Worldspine, Shieldmage's Starfall Seal).
- **LightPillar** — shader-based vertical column of light. Color, height, radius. Used for impact pillars and perimeter pillars.
- **ParticleBurst** — GPU particle system. Count, speed, gravity, upwardBias, size. The general-purpose particle effect.
- **ImpactFlash** — additive sphere that scales up and fades. Quick punctuation flash.

All effects take `(scene, position, opts)` in their constructor. Spawn via `effectMgr.spawn(new <EffectType>(scene, pos, opts), time)`. The manager handles lifecycle; effects auto-dispose when their duration is up.

Effects color through `opts.color`. Always tint to the class's palette — Knight's effects should be ember-orange, Sentinel's should be cyan, etc. The user notices when effects are mistinted.

If your family genuinely needs a new visual effect not covered by these nine — for example, a "healing chain" between healers, or a "trail" effect for marksmen arrows — feel free to add a new Effect subclass. Follow the same pattern: extend Effect, set `this.duration`, build geometry/material in constructor, override `tick(t01, dt)` for the per-frame update, override `dispose()` to clean up. But before adding anything new, check whether you can compose existing effects to get the look you want; the nine cover a lot of ground.

## Animation system

Animations are defined as objects in `<className>Animations`, keyed by name (`attack`, `spell1`, `spell2`):

```js
const knightAnimations = {
  attack: {
    duration: 0.85,                          // seconds
    keyframes: [
      { t: 0,    values: {} },
      { t: 0.22, values: { leftArm: { rx: 0.55, rz: -0.35 }, body: { rz: -0.1, ry: 0.15 } }, ease: ease.out },
      { t: 0.42, values: { leftArm: { rx: -0.85, rz: 0.05 }, body: { rz: 0.12, ry: -0.18 } }, ease: ease.out },
      { t: 0.58, values: { leftArm: { rx: -0.65, rz: 0.0 }, body: { rz: 0.08, ry: -0.12 } } },
      { t: 1.0,  values: {}, ease: ease.inOut },
    ],
    effects: [
      { time: 0.42, fn: (champ, mgr, time) => {
        // spawn VFX here
      }},
    ],
  },
  spell1: { ... },
  spell2: { ... },
};
```

The keyframe object format:

- `t` — normalized time, 0 to 1. Always start with a zero-keyframe (`t: 0, values: {}`) and end with a one-keyframe (`t: 1.0, values: {}, ease: ease.inOut`) — those are the rest-pose anchors. The `{}` means "back to rest pose for those parts."
- `values` — dictionary of `{ partName: { rx, ry, rz, px, py, pz } }`. Values are *deltas from rest pose* in radians (rotation) or units (position). Omitted axes inherit the rest value. Omitted parts inherit their rest pose entirely.
- `ease` — easing function applied to the *outgoing* segment from this keyframe to the next. `ease.linear`, `ease.inOut`, `ease.out`, `ease.outBack`, `ease.in`. Defaults to `ease.inOut` if unspecified. Use `ease.in` for a windup that accelerates into the impact (Juggernaut's smashes do this — the heavy weight of the hit reads through the acceleration). Use `ease.out` for a quick snap that decelerates as it lands (most strikes). `ease.outBack` overshoots before settling, good for ceremonial gestures.

The effects array fires VFX at specific timing points within the animation:

- `time` — normalized 0..1, fires once when the animation crosses this threshold
- `fn(champ, mgr, time)` — receives the champion (so you can read `champ.parts.shield.getWorldPosition(...)` to spawn effects at the shield's actual world location), the effect manager, and the current absolute time (for `mgr.spawn(new <Effect>(...), time)`)

Effects are fired exactly once per animation play — the Champion class tracks which have fired via a `firedEffects` Set. Long animations with multiple cascading effects work great: see Crusader's spell2 (Consecration) which fires four effects across three seconds, or Juggernaut's Earthshatter which fires the primary impact and then a delayed secondary cascade.

The animation pacing should reflect the character's archetype. Some examples from the tanks:

- **Knight (martial)**: standard 0.85s attack, ease.out into impact. Clean snappy reads.
- **Crusader (holy)**: longer 1.0s attack with a build, ease.out. Makes the smite feel weightier.
- **Sentinel (cold patient)**: 0.85s thrust, body twists, ease.out. Cool and precise.
- **Warden (grounded)**: 0.95s overhand chop, ease.out. Earthy and deliberate.
- **Juggernaut (brutal)**: 1.05s with long slow windup (32% of the total duration is just the wind-up) then ease.in into the smash, then a held impact pose. The `ease.in` is the trick — it accelerates into the hit, which feels heavy. Ease.out feels light by comparison.
- **Shieldmage (ceremonial)**: 0.95s lateral mace swing, ease.out. Ceremonial sweep rather than a stab.

When designing animations for a new family, give each class's three abilities a feel that matches its archetype. A healer probably has gentle slow gestures with long durations and ease.inOut. A marksman probably has quick snappy shots with short durations and ease.out. A magician probably has elaborate ceremonial windups with multiple effect-fire points.

## Design philosophy

Six classes in a family need to feel like six genuine choices, not six reskins. The user notices when classes blur together visually. Here's what separates them:

**One marquee identity feature per class.** Knight has the cross + crimson cape. Crusader has the halo + sun rays. Sentinel has the spire helm + eye-tower-shield. Warden has antlers + fur mantle. Juggernaut has bull horns + chains. Shieldmage has orbital runes + mitre helm. Each class has at least one feature that makes it instantly recognizable in silhouette alone, even with no color.

**Strong palette differentiation across the family AND across families.** Within a family, no two classes should land on near-neighbor colors. Across families, try to claim color territory that isn't already taken. The tank palettes are documented below — when you start your family, check what's claimed and pick palettes that feel distinct.

**Silhouette variation.** Different shield types (kite, sun-disc, tower, round, spiked, runed), different weapons (sword, hammer, glaive, axe, maul, mace), different helm shapes (greathelm, crown, spire, antlered, brute-closed, mitre). Even small differences in helm crest or cape length register at a glance.

**Distinctive chest emblems.** No two classes should share an emblem motif. Tanks use: cross, latin cross with rays, vertical eye, tree of life, broken chain, eight-pointed star. Pick six new motifs for your family, themed to the role. Healers might do: holy chalice, sun, dove, life-rune, blooming flower, healing hands. Pick whatever feels right for the lore but make the six distinct.

**Color flows through.** When you switch classes in the dropdown, the accent UI color, the aura color on the dais, and the ember tint of the floating particles all retint to match. The classRegistry has `accent`, `accentGlow`, `auraColor`, `emberColor` fields per class — make sure these match the class's actual eye-glow color.

**Thoroughness wins.** Going from tanks 1-3 to tanks 4-6, the marquee features got bolder (antlers, chains, orbital runes vs. just helm-and-shield variations). Don't be subtle. The orbital-rune thing on Shieldmage is a great example — it's a tiny amount of code (override update, rotate two anchor groups) but it makes the class feel actively magical at idle, not just decorated. Look for one of those per class.

## Established palettes — don't conflict with these

The Tank family has claimed:

| Class | Identity hook | Eye glow | Palette |
|--|--|--|--|
| Knight | Cross + crimson cape | Orange ember `0xff7022` | Steel + brass + crimson |
| Crusader | Halo + sun rays | White-gold `0xffe080` | Ivory + gold + cream |
| Sentinel | Spire helm + eye-tower-shield | Cyan `0x40c0ff` | Slate + silver + cyan |
| Warden | Antlers + fur mantle + tree shield | Forest green `0x80c040` | Verdigris bronze + wood + dark green |
| Juggernaut | Bull horns + chains + spiked maul | Blood red `0xdd2810` | Blackened iron + rust + dark crimson |
| Shieldmage | Orbital runes + mitre helm | Arcane purple `0xb060ff` | Deep violet + silver + purple |

For your family, pick eye-glow colors and palette accents that don't land near these. Open territories include: pale blue-white (frost), warm yellow (sunlight, distinct from Crusader's gold), pink/magenta (distinct from Shieldmage's purple), pure white (radiance), turquoise (between cyan and green), dark teal, blood-red but lighter (vampiric), bone-white, tarnished copper, sea-green, and any saturated red that isn't Juggernaut's deep crimson.

A good rule: pick a primary accent color, then check it against all 6 tank colors above. If it would feel like a sibling of any of them under bloom, shift it.

## Process for building a family file

The build process that worked for tanks:

**Step 1 — Foundation file.** Start by creating the HTML file with everything *except* the six class definitions. That means: HTML head with fonts and CSS, the canvas + UI overlay (top label, dropdown, ability buttons, demo button, loading splash), the importmap, and the script module containing math helpers, the M materials dictionary (with placeholder palette sections for your family that you'll fill as you go), the mesh helper, all nine effect classes + EffectManager, the three shared part builders, and the Champion base class. End the GAME LIBRARY section with two marker comments:

```
// ╔══ INSERT_CLASS_DEFINITIONS ══════════════════════════════════════════════╗

// ╔══ INSERT_VIEWER ═════════════════════════════════════════════════════════╗
```

The simplest way to write this foundation is to copy-paste from tanks.html (the entire library section minus the six tank classes) and adjust the family name in the banners.

**Step 2 — Insert one class at a time.** Use `str_replace` to swap the `INSERT_CLASS_DEFINITIONS` marker for the first class's definition + animations + a fresh `INSERT_NEXT` marker. Then again for the second class, and so on. Each class adds maybe 300-450 lines for the build, plus 80-150 for the animation table. Six classes = roughly 2400-3500 lines of class code, on top of the ~1100 line foundation.

**Step 3 — Insert the viewer.** After all six classes are in, replace the final `INSERT_VIEWER` marker with the viewer code (scene/camera/lights/dais/embers/aura/post-processing/classRegistry/swap logic/UI wiring/main loop). Again, copy-paste from tanks.html and update: the classRegistry needs all six of your family's classes, the dropdown HTML needs all six options, the labels (Family · I/VI through Family · VI/VI) and the role descriptions need updating per class.

**Important practical detail.** Long files (4000+ lines) sometimes get truncated when written in a single `create_file` call — the stream cuts off mid-build. Don't trust a single big create_file. Always chunk: foundation first via create_file, then class-by-class via str_replace. The marker comments make the str_replace anchors stable.

**Sanity check after each major insert.** Do a quick brace/paren/bracket balance check before presenting to the user:

```bash
node -e "
const code = require('fs').readFileSync('family.html', 'utf8').match(/<script type=\"module\">([\s\S]*?)<\/script>/)[1];
let braces = 0, parens = 0, brackets = 0;
let inStr = null, inComment = false, inBlockC = false;
for (let i = 0; i < code.length; i++) {
  const c = code[i], n = code[i+1];
  if (inComment) { if (c === '\n') inComment = false; continue; }
  if (inBlockC) { if (c === '*' && n === '/') { inBlockC = false; i++; } continue; }
  if (inStr) { if (c === '\\\\') { i++; continue; } if (c === inStr) inStr = null; continue; }
  if (c === '/' && n === '/') { inComment = true; i++; continue; }
  if (c === '/' && n === '*') { inBlockC = true; i++; continue; }
  if (c === '\"' || c === \"'\" || c === '\`') { inStr = c; continue; }
  if (c === '{') braces++; else if (c === '}') braces--;
  else if (c === '(') parens++; else if (c === ')') parens--;
  else if (c === '[') brackets++; else if (c === ']') brackets--;
}
console.log('braces:', braces, 'parens:', parens, 'brackets:', brackets);
"
```

All three should be zero. If they aren't, you've got an unclosed brace somewhere and the user won't see the viewer at all (just a blank page). Find and fix before presenting.

**Iteration with the user.** The user works one class at a time. Build the first class, copy to outputs, present, ask for feedback. They might want palette tweaks, feature adjustments, animation timing changes. Polish or move on per their direction. Then build the second class, present, and so on. By the end of the session there's a complete family file.

When presenting a class, be specific about the design decisions: what palette you chose, what the marquee identity feature is, what the chest emblem is, what the abilities are themed around. The user has been making informed feedback this way for the tanks.

## Family-specific design notes

You'll find the full ability and stat-focus information for every family in **GDD.md** — section 6 "Class Roster" lists all 48 classes with their family, base stats, abilities, and passives. Read your family's section there before designing visuals. The class names, ability names, and class roles in the GDD should match the labels you use in the dropdown and ability buttons.

Some quick design sketches for the remaining families:

**Healers** — sacred and life-themed. Likely silhouette: more cloth than plate, with armor accents rather than full plate. Common visual cues: chalices, blooming flowers, doves, sunbursts, sacred runes. Weapons trend toward staves, ceremonial maces, focus crystals, or short ritual blades. Eye glows trend warm and bright (pale gold, pure white, soft pink). Cape work is important; lots of flowing fabric.

**Fighters** — pure martial skill, distinct from Tanks' defensive focus. Silhouette: athletic plate, less heavy than tanks, more agile-looking. Weapons emphasize variety: dual-wielding, two-handers, exotic weapons (chakrams, whips, claws). Eye glows trend warm and saturated. Less ceremonial than Crusader, less brutal than Juggernaut — closer to the Knight's clean martial vibe but with personality variations.

**Marksmen** — ranged. Silhouette: lighter armor (leather + chain at most), often with a quiver, often with a hood or cloak. Weapons are bows, crossbows, firearms (if the world supports them per GDD), throwing weapons. The "shield" slot becomes interesting — maybe a buckler, maybe an off-hand throwing weapon, maybe just a focus item. Eye glows can be cold-precise (silver, ice-blue) or feral (yellow predator-eyes).

**Rogues** — stealthy, cloth + leather. Silhouette: hooded, cloaked, daggers and short blades. Weapons: paired daggers, curved blades, garrotes, throwing knives. Probably the leanest silhouette of all families. Eye glows trend cold and shadowy (deep purple, sickly green, blood-red). Lots of fabric work — cloaks that should billow, hoods that hide the face partially.

**Magicians** — pure casters in robes. Silhouette: robes from neck to ankle, ornate headwear (pointed hats, hoods, circlets), ceremonial staves or focus orbs as primary "weapons". Possibly no shield at all, or replaced by an off-hand spell focus. Lots of opportunity for floating-element identity hooks (Shieldmage's orbital runes were a tank-side echo of this — you can go bigger here). Eye glows: vivid and elemental (fire-orange, ice-cyan, void-violet, lightning-yellow). Animation pacing: long ceremonial gestures.

**Mystics** — esoteric / spiritual. Possibly the weirdest family visually. Silhouette options: monk-like, shaman-like, oracle-like. Weapons: prayer beads, ritual staves, bone implements, ceremonial weapons. Lots of opportunity for unconventional visual hooks (third eyes, halos, summoned spirits as floating elements, extra glowing limbs). Eye glows: ethereal (soft white, golden, opalescent).

**Far Lands** — exotic, foreign-coded. Pull from non-European fantasy aesthetics. Could be desert-themed, eastern-themed, oceanic-themed, ancient-mesoamerican-themed — let GDD guide you on lore. Silhouette and palette should feel deliberately unlike anything in the European-coded tanks. Weapons should feel exotic (curved swords, segmented weapons, oversized fans-as-weapons, exotic polearms). Eye glows can claim any color territory not yet used.

For all of them: the tank-family `Champion` lifecycle, materials system, effect system, and animation system carry over identically. Only the per-class build methods, animation tables, and palette materials change.

## A note on scope

Six classes per family, three abilities per class. You don't need to build a stage, lighting, post-processing, dropdown UI from scratch — copy that wholesale from tanks.html. Your work is in the six per-class build methods and animation tables, plus the dropdown options + classRegistry entries that wire everything in.

Each class build is a substantial chunk (300-450 lines). Don't try to do all six in one tool call. Break it up: foundation first, then one class at a time, present after each. The user expects iteration.

Also: don't add a feature the user didn't ask for, even if it would be cool. The architecture is what it is for reasons — the importable game library section needs to stay clean; the viewer fluff needs to stay below the marker. New things go in the right place.

## Future: baking to static game assets

The family HTML files are the **authoring source**. They contain the procedural code that builds each champion from primitives — every cylinder, sphere, extruded shape, every keyframe array. This is great for editing (change a parameter, reload, see the difference), but you don't ship procedural code to the game runtime. You ship baked assets.

Once all eight families are complete (48 champions total), Claude Code will set up a bake step that turns the procedural source into static assets:

- **`<class>.glb`** — geometry, materials, and rest pose for one champion, exported via Three.js's built-in `GLTFExporter`. Standard binary glTF, loads natively in any Three.js runtime via `GLTFLoader`. One file per champion, 48 files total.
- **`<class>.animations.js`** (or `.json`) — the keyframe animation tables exported as a data file. The keyframe pose-delta data is plain JSON-able. The `effects` arrays contain functions, so they either ship as code or get rewritten as a serializable description that the runtime interprets — CC will pick whichever fits the game's architecture.

The Champion base class itself stays as code in the game runtime — it's the engine that loads the GLB, drives keyframe playback, and dispatches effect callbacks. Maybe ~100 lines.

The bake script is a build-time tool, not a runtime dependency. Roughly:

```js
// tools/bake-champions.js — runs once at build time, not at runtime
import { Knight, Crusader, Sentinel, Warden, Juggernaut, Shieldmage } from './champions/families/tanks.js';
import { GLTFExporter } from 'three/addons/exporters/GLTFExporter.js';

for (const ChampClass of [Knight, Crusader, Sentinel, ...]) {
  const champ = new ChampClass(scratchScene, new Vector3());
  exporter.parse(champ.group, glb => writeFile(`assets/${className(ChampClass)}.glb`, glb));
  writeFile(`assets/${className(ChampClass)}.animations.js`, serializeAnimations(champ.animations));
}
```

Run once. Produces 48 GLBs + 48 animation files. Ship those with the game. The game's runtime never re-runs the procedural geometry code; it just loads the baked assets and plays the animations.

**Implication for authoring (you, this session)**: keep doing what we're doing. Build family HTML files exactly as before. Don't worry about the bake step — it's a downstream concern CC handles once all the family files are stable. The procedural code stays as the source of truth for *editing* and re-baking. If a champion's helm needs adjustment later, you edit the procedural code, re-run the bake, and re-deploy the assets.

**One constraint to maintain during authoring**: don't put behavior in `build()` that won't work in a headless bake context. No `window`, no `document`, no live game state, no DOM lookups. The procedural code should run identically in a Node bake script as it does in the browser viewer. The tank classes already follow this naturally — none of them touch DOM or runtime state — but it's worth keeping in mind, especially if you're tempted to read viewer settings or use any browser-only API in a class's build method. Build methods should be pure: take constructor arguments, produce geometry. That's it.

## When you're done

The deliverable is a single HTML file (`healers.html`, `fighters.html`, etc.) in `/mnt/user-data/outputs/`, opened with `present_files`. The user will preview it, give feedback, and either iterate (you polish) or close out the session and start a new one for the next family.

Good luck.
