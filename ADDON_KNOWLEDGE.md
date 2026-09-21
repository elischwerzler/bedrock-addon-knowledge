# Minecraft Bedrock Add-On Making: Everything We Learned

A field guide to building Minecraft **Bedrock Edition** add-ons (behavior packs, resource packs and `@minecraft/server` scripts), written from roughly 200 real add-ons and the faults that actually shipped in them. Most Bedrock failures are **silent**: a wrong value means a blank icon, a mob that never spawns, or a feature that quietly does nothing, with no error anywhere. This guide is organised around finding those faults before players do.

**Targets:** Bedrock 1.21.x, `format_version: 2` manifests, item files `1.21.0`, `@minecraft/server` **2.0.0**, plain JavaScript. Generators are written in Python (Pillow for sprites), tests in Node.

**How to read it**
1. **Part 1** is what the platform accepts and what it silently ignores.
2. **Part 2** covers mobs, models, animation and art.
3. **Part 3** is a catalogue of script-logic bug classes with an audit checklist.
4. **Part 4** is the build, test and red-team workflow, with reusable harness sketches.
5. **Part 5** is case studies of finished projects, plus RimWorld, Farming Simulator 22, Borderlands 2 and Bloons TD 6 notes.

> Notes are written down as they were learned. Where something was never confirmed in game, the text says so. Verify against the current game version before relying on any single detail, because Bedrock changes between releases.

## Contents

- [Part 1 — Platform facts](#part-1--platform-facts)
  - [1. Pack structure and manifests](#1-pack-structure-and-manifests)
  - [2. A minimal worked example (generator + engine split)](#2-a-minimal-worked-example-generator--engine-split)
  - [3. Item JSON facts and traps](#3-item-json-facts-and-traps)
  - [4. Custom blocks, ores and world generation](#4-custom-blocks-ores-and-world-generation)
  - [5. Recipes](#5-recipes)
  - [6. Entities, spawning and mob AI](#6-entities-spawning-and-mob-ai)
  - [7. Scripting API quirks (`@minecraft/server` 2.0.0)](#7-scripting-api-quirks-minecraftserver-200)
  - [8. Damage, enchants, drops, mining](#8-damage-enchants-drops-mining)
  - [9. Identifiers, references and unverifiable claims](#9-identifiers-references-and-unverifiable-claims)
  - [10. Deploy, pack loading and "my change did nothing"](#10-deploy-pack-loading-and-my-change-did-nothing)
  - [11. Web app <-> Bedrock bridge](#11-web-app---bedrock-bridge)
  - [12. Textures (PIL) for 16px icons](#12-textures-pil-for-16px-icons)
  - [13. Silent-failure trap index (quick scan)](#13-silent-failure-trap-index-quick-scan)
- [Part 2 — Mobs, entities, models, animation and art](#part-2--mobs-entities-models-animation-and-art)
  - [1. Anatomy of a custom entity (what has to line up)](#1-anatomy-of-a-custom-entity-what-has-to-line-up)
  - [2. Custom mob AI traps (all look like "the AI is just bad")](#2-custom-mob-ai-traps-all-look-like-the-ai-is-just-bad)
  - [3. Mob patterns](#3-mob-patterns)
  - [4. Animation](#4-animation)
  - [5. Models and UV](#5-models-and-uv)
  - [6. Hit tests, projectiles and geometry that silently never connect](#6-hit-tests-projectiles-and-geometry-that-silently-never-connect)
  - [7. Procedural textures and icons (PIL)](#7-procedural-textures-and-icons-pil)
  - [8. Telling a big roster apart (mobs, items, blocks, drinks)](#8-telling-a-big-roster-apart-mobs-items-blocks-drinks)
  - [9. Drawn-extent, mirroring and looking (for programmatic art)](#9-drawn-extent-mirroring-and-looking-for-programmatic-art)
  - [10. Numbers, playability and test fixtures that affect mobs](#10-numbers-playability-and-test-fixtures-that-affect-mobs)
  - [11. Quick checklists](#11-quick-checklists)
- [Part 3 — Script logic bug classes](#part-3--script-logic-bug-classes)
  - [3.1 State and ownership](#31-state-and-ownership)
  - [3.2 Transfers, kits and phase changes](#32-transfers-kits-and-phase-changes)
  - [3.3 Timing, ticks and queues](#33-timing-ticks-and-queues)
  - [3.4 Config with no other end](#34-config-with-no-other-end)
  - [3.5 Dispatch and names](#35-dispatch-and-names)
  - [3.6 Safety of building and destructive features](#36-safety-of-building-and-destructive-features)
  - [3.7 Reporting honestly](#37-reporting-honestly)
  - [3.8 Generators and patches](#38-generators-and-patches)
  - [Audit checklist](#audit-checklist)
- [Part 4 — Testing, verification, build and workflow](#part-4--testing-verification-build-and-workflow)
  - [Project layout and generators](#project-layout-and-generators)
  - [The build pipeline](#the-build-pipeline)
  - [Static guards to put in `check()`](#static-guards-to-put-in-check)
  - [The sim harness: run the shipping scripts in Node](#the-sim-harness-run-the-shipping-scripts-in-node)
  - [How to write fakes](#how-to-write-fakes)
  - [Proving a guard can fail (red-teaming)](#proving-a-guard-can-fail-red-teaming)
  - [Test anti-patterns (quick catalogue)](#test-anti-patterns-quick-catalogue)
  - [Things to look at with your own eyes](#things-to-look-at-with-your-own-eyes)
  - [Silent-failure debugging on Bedrock](#silent-failure-debugging-on-bedrock)
  - [Deploying and staging safely](#deploying-and-staging-safely)
  - [Workflow habits](#workflow-habits)
- [Part 5 — Case studies and other games' modding](#part-5--case-studies-and-other-games-modding)
  - [Minecraft Bedrock add-on case studies](#minecraft-bedrock-add-on-case-studies)
  - [Other games](#other-games)

---

## Part 1 — Platform facts

Everything here was learned building Minecraft Bedrock Edition add-ons (Windows, Bridge-style editor workflow, `@minecraft/server` 2.0.0, format versions around 1.21). Bedrock fails **silently** far more often than it fails loudly, so most entries are written as symptom -> cause -> fix. Items marked **(unverified)** were never confirmed in a running game.

---

### 1. Pack structure and manifests

**Layout.** One add-on is a paired behavior pack (BP) and resource pack (RP):

```
MyAddon/
  behavior_pack/
    manifest.json
    scripts/main.js            # only if scripted (ES module, plain JS, no transpile)
    items/<id>.json            # "minecraft:item"
    blocks/<id>.json           # "minecraft:block"
    entities/<id>.json         # "minecraft:entity"
    recipes/<id>.json
    loot_tables/blocks/<x>.json
    features/<x>_feature.json          # ore feature (auto-loaded)
    feature_rules/<x>_rule.json        # ore placement rule (auto-loaded)
    spawn_rules/<mob>.json
  resource_pack/
    manifest.json
    textures/item_texture.json         # item icon short name -> textures/items/<file> (no .png)
    textures/terrain_texture.json      # block texture key -> textures/blocks/<file> (no .png)
    textures/{items,blocks,entity}/*.png
    entity/<mob>.json                  # "minecraft:client_entity"
    models/entity/<geo>.json           # custom geometry
    sounds.json / sounds/sound_definitions.json   # optional
    texts/en_US.lang
    texts/languages.json               # ["en_US"]
```

**Manifest rules**

- Manifest `format_version` is `2`. Per-type JSON files use their own version **string** (items/blocks/recipes `"1.21.0"`, `client_entity` `"1.10.0"`, geometry `"1.12.0"` / `"1.16.0"`), never the number `2`.
- `min_engine_version: [1, 21, 0]`.
- Generate **five unique UUIDs** per add-on: BP header, BP data module, BP script module (if scripted), RP header, RP resources module. Content identifiers (`ns:name`) are not UUIDs.
- **Reciprocal header dependencies** pair the packs: BP `dependencies` lists the RP *header* UUID, RP `dependencies` lists the BP *header* UUID. Backwards = packs do not pair.
- A script BP declares `{"module_name": "@minecraft/server", "version": "2.0.0"}` (add `@minecraft/server-ui` `2.0.0` if it uses forms). The version must match the API surface the script uses. **An invalid or unshipped version (e.g. `1.14.0`) makes the entire script module fail to load and the add-on silently does nothing.** Match a known-working sibling.
- A data-only add-on must NOT declare a script module or an `@minecraft/server` dependency.
- Deterministic UUIDs are handy: `uuid.uuid5(uuid.NAMESPACE_URL, "myaddon/bp.header")`. Changing the seed mints a whole new pack identity (see section 10).

Minimal BP manifest (from a generator; `v` is the version counter):

```json
{
  "format_version": 2,
  "header": {
    "name": "My Addon V3 BP",
    "description": "...",
    "uuid": "<bp-header-uuid>",
    "version": [3, 0, 0],
    "min_engine_version": [1, 21, 0]
  },
  "modules": [
    {"type": "data", "uuid": "<bp-data-uuid>", "version": [3, 0, 0]},
    {"type": "script", "language": "javascript", "entry": "scripts/main.js",
     "uuid": "<bp-script-uuid>", "version": [3, 0, 0]}
  ],
  "dependencies": [
    {"module_name": "@minecraft/server", "version": "2.0.0"},
    {"uuid": "<rp-header-uuid>", "version": [3, 0, 0]}
  ]
}
```

RP manifest: same shape, one module `{"type": "resources", ...}`, dependency `{"uuid": "<bp-header-uuid>", "version": [3,0,0]}`.

**Versioning: two conflicting observations, choose deliberately.**
- Bumping a visible version label on every rebuild (V1 -> V2 -> V3) is useful so a tester can tell the new build loaded. Stamp it in: both manifest `version` arrays, manifest header `name`, the `.lang` pack name, and a `VERSION_LABEL` constant the script shows on the action bar at first spawn. Re-running the generator bumps all in sync.
- **Trap:** a pack whose *display name* counted up to V61 while `header.version` was hard-coded `[1, 0, 0]` looked identical to the game on every rebuild (the number the game compares never moved). Derive the manifest version from the same counter that names the pack (e.g. `[1, VERSION // 100, VERSION % 100]`), in the generator that writes the manifest, not in a packager that the next build overwrites.
- **Counter-observation from another project:** a pack version that *moved* dropped the pack out of existing worlds, so that project pins the version at `[1, 0, 0]` forever and changes the UUID seed (`GEN` constant) only when it must force a fresh identity. Cost: the world sees a new pack and it must be switched on again. Which behaviour you get is project-specific; test the upgrade path in a world that already has the pack enabled.

**Packaging.** Zip with Python `zipfile` and forward-slash arcnames (`relpath.replace(os.sep, "/")`). PowerShell `Compress-Archive` writes backslash paths Bedrock rejects. A `.mcaddon` is a zip containing both packs; `.mcpack` is a single pack.

**Other file facts**

- `texts/languages.json` must list every locale (`["en_US"]`) or `.lang` names silently do not load.
- Texture references in `item_texture.json`, `terrain_texture.json` and client entities omit `.png`.
- Lang keys: items `item.<ns>:<id>=Name`, blocks `tile.<ns>:<id>.name=Name`, entities `entity.<ns>:<id>.name=Name`. (A `minecraft:display_name` component with `{"value": "..."}` on the item also works.)
- **No `blocks.json` is needed for custom blocks in 1.21.** An RP-root `blocks.json` only assigns break/place sounds.
- `geometry.<name>` and texture short names must exist in the RP before a block/client_entity references them.
- Two ways a custom mob looks like something: **custom** (own geometry + texture) or **vanilla-borrow** (BP `description.runtime_identifier: "minecraft:zombie"`, RP reuses vanilla geometry/render controller). Note: a zombie runtime id also renders held items.
- Mob voices cost no audio files: borrow vanilla sound *events* in RP `sounds.json` (`{"entity_sounds": {"entities": {"ns:mob": {"volume":1, "pitch":1, "events": {"ambient": "...", "hurt": "...", "death": "..."}}}}}`) **and** add `minecraft:ambient_sound_interval` to the BP entity or the ambient sound never fires. With neither, a custom mob is completely mute.

---

### 2. A minimal worked example (generator + engine split)

Pattern used by a finished add-on with 17 tools: **one Python generator writes every JSON file, the script holds only behaviour that JSON cannot express.** Everything under `behavior_pack/` and `resource_pack/` except `scripts/main.js` is generated; stats are exported to the script as a generated `tools_data.js`.

Sword/pickaxe item JSON as generated:

```json
{
  "format_version": "1.21.0",
  "minecraft:item": {
    "description": {
      "identifier": "ns:emerald_pickaxe",
      "menu_category": {"category": "equipment", "group": "itemGroup.name.pickaxe"}
    },
    "components": {
      "minecraft:display_name": {"value": "Emerald Pickaxe"},
      "minecraft:icon": "emerald_pickaxe",
      "minecraft:max_stack_size": 1,
      "minecraft:hand_equipped": true,
      "minecraft:damage": 7,
      "minecraft:enchantable": {"value": 15, "slot": "pickaxe"},
      "minecraft:durability": {"max_durability": 1500},
      "minecraft:digger": {
        "use_efficiency": true,
        "destroy_speeds": [{"block": {"tags": "1"}, "speed": 12}]
      },
      "minecraft:tags": {"tags": ["minecraft:is_pickaxe", "minecraft:is_tool", "minecraft:diamond_tier"]}
    }
  }
}
```

Swords use `minecraft:allow_off_hand: true` and `minecraft:can_destroy_in_creative: false` instead of `digger`/`tags`. Pickaxe/axe tags: `minecraft:is_pickaxe` / `minecraft:is_axe` plus `minecraft:is_tool` plus a tier tag (`stone_tier`, `iron_tier`, `diamond_tier`, `netherite_tier`).

Shaped recipe (note the `unlock` field, see section 5):

```json
{
  "format_version": "1.21.0",
  "minecraft:recipe_shaped": {
    "description": {"identifier": "ns:emerald_pickaxe_recipe"},
    "tags": ["crafting_table"],
    "unlock": {"context": "AlwaysUnlocked"},
    "pattern": ["xxx", " s ", " s "],
    "key": {"x": {"item": "minecraft:emerald"}, "s": {"item": "minecraft:stick"}},
    "result": {"item": "ns:emerald_pickaxe", "count": 1}
  }
}
```

`item_texture.json`:

```json
{"resource_pack_name": "myaddon", "texture_name": "atlas.items",
 "texture_data": {"emerald_pickaxe": {"textures": "textures/items/emerald_pickaxe"}}}
```

Generator habits worth copying:
- Write every file as **temp file + `os.replace`** (an interrupted run never leaves a half file).
- Keep the script's stats in one generated module (`export const OP_TOOLS = {...}`); never hand-edit generated files, or the next run reverts the fix (an inline patch not written back into the generator is undone by the next rebuild).
- Guard patch scripts: a `str.replace` that matches nothing is a silent no-op that ships a lie; assert the replacement count.

Script skeleton conventions (all proven in shipped packs): `import { world, system, ItemStack } from "@minecraft/server"`; handlers via `world.afterEvents.*`; APIs in regular use: `getEntitiesFromViewDirection`, `getBlockFromViewDirection`, `teleport`, `applyDamage`, `applyImpulse`, `applyKnockback`, `addEffect(name, ticks, {amplifier})`, `spawnParticle`, `playSound`, `onScreenDisplay.setActionBar`, `runCommand`.

---

### 3. Item JSON facts and traps

- **`minecraft:icon` on `format_version "1.21.0"` must be a plain string.**
  ```json
  "minecraft:icon": "canteen_regeneration"          // renders
  "minecraft:icon": {"texture": "canteen_..."}       // pre-1.21 form: renders BLANK, no error
  ```
  Symptom: item exists, no texture, nothing in any log. Copy item JSON from a known-working pack rather than from memory or a pasted template.
- **Item `events` cannot run `run_command`.** AI-generated templates that do this are wrong. Implement active behaviour (guns, teleport, effects, area-of-effect) in the script.
- **Not real components** (do not use): `minecraft:mining_speed`, `minecraft:weapon`. **Real ones:** `minecraft:allow_off_hand`, `minecraft:digger.destroy_speeds` (supports `q.any_tag(...)` and Molang), `minecraft:enchantable {value, slot}`, `minecraft:durability` (just `max_durability`; the `{numerical,denominator}` damage_chance form is invalid), `minecraft:hand_equipped`, `minecraft:glint`, `minecraft:max_stack_size`, `minecraft:tags`, `minecraft:use_modifiers`, `minecraft:projectile`, `minecraft:consumable`.
- **`minecraft:damage` is stored in a byte**: values above ~127 wrap (266 -> 10 in game, 532 -> 20). Both tooltip and real damage use the wrapped value. Fix: clamp native damage to 127, export the overflow (`intended - 127`) in a map, and apply it from script in `entityHitEntity`: `target.applyDamage(bonus, {cause: "entityAttack", damagingEntity: attacker})`. Script `applyDamage` is not byte-capped.
- **Unbreakable = omit `minecraft:durability` entirely.** There is no `minecraft:unbreakable` component in 1.21. `damage_chance: {min:0, max:0}` does NOT make an item unbreakable (it only interacts with the Unbreaking enchantment). A tool that relied on it was "probably not unbreakable, just had 12000 durability so nobody noticed" (unverified in game).
- **Hold-to-fire needs `minecraft:use_modifiers`.** `itemUse` fires once per right-click; there is no "is mouse held" query. An item only raises `itemStartUse` / `itemStopUse` if it carries:
  ```json
  "minecraft:use_modifiers": {"use_duration": 3600, "movement_modifier": 0.9}
  ```
  Subscribe to **both** `itemStartUse` and `itemUse`, rate-limit by tick, and use a fallback flag so if hold-to-fire never fires you still get click-to-fire and never both:
  ```js
  let holdToFireWorks = false;   // itemStartUse sets it true
  // itemUse acts only while holdToFireWorks is false
  ```
  **(unverified)** The docs say `itemStartUse` fires for a "chargeable item" without defining it; whether a custom item with `use_modifiers` raises it was never confirmed in the game, hence the fallback.
- **Off-hand items deal NO melee damage** in Bedrock (only the main hand swings; `minecraft:damage` on an off-hand item does nothing). Script it:
  ```js
  world.afterEvents.entityHitEntity.subscribe(ev => {
    const off = player.getComponent("minecraft:equippable").getEquipment("Offhand");
    ev.hitEntity.applyDamage(bladeDamage, {cause: EntityDamageCause.entityAttack, damagingEntity: player});
  });
  ```
  Rate-limit to ~10 ticks or a fast click becomes a machine gun.
- **Fast weapon left-click:** players swing swords with left-click. If your weapon is designed around right-click, vanilla left-click damage is what happens (and a damage-sensor may absorb it), so the weapon "does nothing". Wire `entityHitEntity` to trigger the weapon too.

---

### 4. Custom blocks, ores and world generation

Block (1.21):

```json
"minecraft:block": {
  "description": {"identifier": "ns:my_block"},
  "components": {
    "minecraft:material_instances": {"*": {"texture": "my_block", "render_method": "opaque"}},
    "minecraft:geometry": "minecraft:geometry.full_block",
    "minecraft:destructible_by_mining": {},          // value shape illustrative; copy from a working block
    "minecraft:destructible_by_explosion": {},
    "minecraft:map_color": "#808080",
    "minecraft:loot": "loot_tables/blocks/my_block.json"
  }
}
```

RP: add `"my_block": {"textures": "textures/blocks/my_block"}` to `textures/terrain_texture.json` (`texture_name: "atlas.terrain"`) and the PNG to `textures/blocks/`.

**Naturally spawning ore** = a block + two auto-loaded files (no manifest change):
- `features/<x>_feature.json`: `minecraft:ore_feature` with `count` (vein size) and `replace_rules: [{"places_block": "ns:ore", "may_replace": ["minecraft:stone", ...]}]`. The giant `places_on_*` lists seen in some AI examples are hallucinated; the real schema is just `count` + `replace_rules`.
- `feature_rules/<x>_rule.json`: root key `minecraft:feature_rules`; `description.places_feature`; `conditions.placement_pass: "underground_pass"`; `conditions.minecraft:biome_filter: [{"test":"has_biome_tag","operator":"==","value":"overworld"}]`; `distribution.iterations` (veins per chunk) + uniform x/y/z extents.
- Block loot table drops the material item.

**Custom blocks cannot emit redstone power.** No component does it (`minecraft:redstone_conductivity` only controls conduction and may not even be accepted by the stable pack format; leave it off rather than risk the whole block JSON failing to load - **unverified**). Working technique: while the block should power, `block.setType("minecraft:redstone_block")` and swap back when off; dust, pistons, doors, lamps and droppers then react for free. Costs to handle:
- The record is the intent, the block is the truth: re-read the block before trusting the saved record (a node blown up by TNT fires no break event). Have `apply()` return `ok / gone / unloaded`; never reap on "unloaded".
- Cancel the break while powering (`beforeEvents.playerBreakBlock`) or the player walks off with a real redstone block; send the warning inside `system.run()` because a before-event is read-only.
- A toggle in an unloaded chunk must still land: store desired state, reconcile a few nodes per pass, round robin.
- `system.currentTick` resets on world reload, so a pulse expiry stored as an absolute tick never arrives; sweep at boot and force those off.

---

### 5. Recipes

- **Custom recipes are hidden from the recipe book by default.** Hand-placement works but the item looks uncraftable. Always add `"unlock": {"context": "AlwaysUnlocked"}` inside the recipe object. Bake this into the generator's shaped/shapeless helper so no recipe can forget it.
- Give every weapon/tool/armor a recipe unless it is intentionally creative-only, or a raw ore-drop material that must be **mined only** (a cobblestone -> ore-material recipe trivialises progression). Generator pattern: `cost=None` for those materials skips the recipe and a cleanup step deletes any orphan recipe JSON; tools made *from* the material stay craftable.
- **A custom recipe with the same grid as a vanilla one makes ONE of them uncraftable.** Bedrock picks one recipe for a duplicate grid, and nothing logs it. Real cases: 2x2 brick (vanilla: bricks), 2x2 string (white wool), 2x2 honeycomb (honeycomb block), 2x2 sand (sandstone). Result: either your block is uncraftable or a player with the pack can no longer craft bricks/wool/sandstone. Fix: add a second ingredient.
- **Shapeless recipes collide with shaped vanilla recipes** using the same items, because arrangement does not matter: three wheat shapeless = vanilla bread. So compare **ingredient multisets** across shaped and shapeless together, not grids. Danger zone: any uniform 2x2 (4x brick/string/honeycomb/sand/quartz/snowball/clay ball/glowstone dust/prismarine shard/nether wart/planks) or 3x3 (9x coal/ingots/gems/wheat/bone meal/slime balls/dried kelp/raw ores), plus small mixed recipes (bread, book = 3 paper + leather, mushroom stew, pumpkin pie, magma cream, cookie, golden apple, blaze rod, sugar cane, bone, melon). No reference file carries vanilla recipe data, so a *stated* list of these is the guard (incomplete is safe: it can miss a collision but never invent one).
- Also confirm no recipe of yours **outputs** a vanilla item.
- **Nearest-match crafting** (parts carry stat deltas, the game returns the closest real item): reachability is a property of the whole roster. Adding a new item can *evict* an existing one so no combination resolves to it, and the item still exists, has a recipe and shows in the creative menu. Cause: a profile only distinguishes what it models (e.g. a burst-fire gun with no burst term is indistinguishable from cheaper rifles). Guard at **build time**, sweep exhaustively (45k combinations is under a second; sampling gave confident wrong answers), and also check that dismantling item X returns parts that rebuild X.

---

### 6. Entities, spawning and mob AI

- **`minecraft:damage_sensor` causes must be real `ActorDamageCause` values.** `"attack"` is not one (it is `entity_attack`). An unknown cause makes the whole entity file fail to load: the mob never spawns and there is no error. Keep a whitelist of causes in the builder and validate.
- **Unknown JSON keys are ignored, not rejected.** `"filter"` instead of `"filters"` in `nearest_attackable_target.entity_types` meant no filter at all: each mob attacked the nearest thing, including its own kind. Check keys against a vanilla file (an installed pack's `wolf.json` etc.) and assert the key name itself, not that a value appears somewhere in the stringified block.
- Mob AI gotchas: duplicate JSON keys collapse; panic fires on ANY hit; `attack_interval` is in seconds; target family `"mob"` makes a mob hostile to all mobs.
- **Tameable pattern:** `component_groups` + `minecraft:tameable` + a `tame_event`.
- **Custom mobs swamp the vanilla spawn pool.** Spawn weights of 5-9 look normal next to a cow's 8, but custom spawn rules use `has_biome_tag` with the blanket `overworld` tag (competing in every biome, while a cow is biome-limited). Thirteen mobs, combined weight 85, took ~63% of the daylight-surface animal pool (vanilla ~50). Also custom passives keep respawning where vanilla animals mostly spawn once at worldgen. Fix: measure the share **per pool** (daylight surface, dark surface, underground, monster), use weights of 1-2 for daylight, move mobs to a pool vanilla does not use, freeze a stated budget per pool and fail the build over it.
- **A custom entity's spawn egg is not craftable.** A mob with no spawn rule exists only in the creative menu. Check obtainability as its own property.
- **A self-spawning mob recurses exponentially.** A "calls friends" power that spawns 2 copies of its own type (each copy also has the power) goes 2 -> 4 -> 8 and exhausts memory in ~30 simulated seconds. The per-entity "only once" flag does not help because each new entity fires once too. Tag script-spawned entities (`addTag("summoned")`) and skip the power for tagged ones; add a crowd cap counted from nearby mobs; same rule for on-death splitters (tag the halves).
- **Idle animation freeze:** distance-moved walk cycles lock limbs when a mob stops; custom mobs are static unless vanilla animation controllers are wired in.
- **Borrowed player geometry renders untextured.** A client entity that borrows `geometry.humanoid.custom` (belongs to the player) may not resolve and renders untextured with no error. Ship your own copy under your own name (`geometry.my_bot`); never ship one under a vanilla name, which replaces that model for every entity including players.
- Box-UV scales its patch with the cube; per-face UV is needed when painting marks so they land on the visible patch (a mark painted then covered by a later face draw hides the eyes).

---

### 7. Scripting API quirks (`@minecraft/server` 2.0.0)

**Events and timing**
- `playerInteractWithBlock` fires **twice per click** (once per hand). Guard: `if (ev.isFirstEvent === false) return;` (compare to `false`, not falsy: older builds omit the field).
- Sneak means build, not use. A custom block the player interacts with is also the block they place things against. Add `if (player.isSneaking) return;` or every attempt to build next to it pops the menu.
- **`afterEvents` are read-write**, so no `system.run` deferral is needed. `beforeEvents` are read-only: wrap writes in `system.run()`.
- `entityHurt` and `entityHitEntity` both fire for one melee hit and **their order is not guaranteed**. Snapshot values (e.g. "when was this target last hurt") before the hit counts.
- `system.runInterval(fn, 10)`: a counter inside counts **passes**, not ticks. `beat - last > 90` meant 900 ticks (45 s), not 4.5 s. Name the unit, define the period as one constant used by both the gate and `runInterval`.
- **`system.currentTick` resets on world reload.**
- Capture `ev.player` / `ev.dimension` into locals before any deferred callback.
- **Reflect effects recurse.** Damage dealt from inside a damage event (thorns-style) re-fires the event; two wearers can bounce a hit unboundedly if dispatch is synchronous. Guard with a depth counter (bail past depth 2). Setting health directly does not re-fire the event; `applyDamage` does.
- Bulk block work: drain ~20 blocks/tick from a queue with a global cap (e.g. 4000) rather than hundreds of commands in one tick. A drain function must **always re-arm** in a `finally`, or the flag wedges and every later job silently does nothing. Also apply a per-player share, not just a global cap, or one player's big job starves everyone else's tools.

**Entities and players**
- **`entity.location` is the FEET**, not the centre.
- A removed entity throws on touching **any** property (`entity.nameTag` after `entity.remove()` included). `isValid` is a property, not `isValid()` (a defensive helper can accept both).
- **`entity.fallDistance` is not a real property.** Use downward velocity (`-getVelocity().y`) as the fall proxy; e.g. falling if `getVelocity().y < -0.4`.
- Event fields: `entityHitEntity` gives `ev.damagingEntity` and `ev.hitEntity` (NOT `damager`); `applyDamage` option is `damagingEntity` (NOT `damager`).
- Equipment: `player.getComponent("minecraft:equippable").getEquipment(EquipmentSlot.Mainhand)` (or `"Offhand"`).
- `player.setGameMode("Survival")` needs no permission; **`/gamemode` needs operator**, so on a realm or someone else's world a command-based respawn leaves non-hosts stuck spectating.
- `BlockPermutation` has no `typeId`: use `ev.brokenBlockPermutation.type.id`. `ev.itemStackBeforeBreak` is a better tool gate than re-reading the main hand after the break.
- `container.addItem(stack)` **returns the leftover stack** when full. Ignoring it silently voids items; drop the overflow with `dimension.spawnItem`.
- **Rounding at half-block positions:** an entity standing at x.5 and stepped with `Math.round(x + dir)`: `Math.round(32.5)` = 33 (same block) going negative, `Math.round(34.5)` = 35 (two blocks) going positive, so bots heading north/west froze and others moved double speed. Work on the block: `Math.floor(x)` for the cell, then add a sign, one axis at a time; test all four directions.
- There is **no light-level query** (no `getLightLevel`) and no block hardness / `isUnbreakable` property in 2.0.0. Do not promise "only in the dark"; test what you can ("solid block overhead"). A comment or player-facing string that claims an unimplementable condition reads like a verified decision.
- `Player.isSleeping` is documented but was never proven to fire in any shipped add-on **(unverified)**: pair it with a proven event (e.g. `beforeEvents.playerInteractWithBlock` on a bed at night).

**Forms (`@minecraft/server-ui` 2.0.0)** dropped positional defaults on `ModalFormData`:
```js
.dropdown(label, items)                                   // options object is the 3rd arg
.slider(label, min, max, {valueStep: 1, defaultValue: 1}) // NOT (..., 1, 1)
.toggle(label, {defaultValue: false})                     // NOT (..., false)
```
Passing the old positional numbers throws at form construction.
- `ModalFormData.submitButton()` does not exist in every build; guard optional methods with `typeof f.method === "function"`.
- A `showForm` helper that catches every error and retries turns a crash into "the menu reopens forever". Retry only on `cancelationReason` containing "busy"; log anything else and resolve `null`. **Never treat a caught error the same as a user cancel.**
- A form `await` lasts minutes. The player can swap slots or stash the item while it is open: re-read the slot after the await and confirm it is the same item before writing; reload any lists AFTER the last await; charge only after the write lands (verify by re-reading).
- Forms are one screen per step and cannot combine a list picker with a text input; keep flows shallow (no hub screens that only route; after a repeatable action return to the list the player came from; optional buttons shift every later index).

**Sounds:** there is no `stopSound` in `@minecraft/server`. Remember the id you started and `player.runCommand("stopsound @s <soundId>")`; route every stop/pause/track change through one helper so no path can skip it (otherwise a long track keeps playing after unequip and re-equipping layers a second copy).

**Vanilla music suppression:** override real event names; see section 9.

**Networking:** a behavior pack script has **no network access at all**. See section 11 for the websocket bridge.

---

### 8. Damage, enchants, drops, mining

**Reading enchants (v2.0):** `item.getComponent("minecraft:enchantable")?.getEnchantments()` returns `[{type: {id}, level}]` (the old `getComponent("minecraft:enchantments").enchantments` shape is out of date). `sharpness`, `looting`, `fire_aspect`, `unbreaking`, `mending`, `knockback` work natively on a sword-slot item. Others must be scripted (density = damage per fall block, breach = damage multiplier, wind_burst = launch attacker up via `applyKnockback`, efficiency = grant Haste while held, silk_touch/fortune = cancel `beforeEvents.playerBreakBlock` + custom drop via `system.run`, riptide/loyalty/thorns).

**"Any enchant on any item" needs real scripted actions per enchant.** Vanilla's `addEnchantment` refuses incompatible pairs (Riptide on a sword does nothing), and lore-only "cosmetic" enchants are dead weight. Wire per kind: `entityHitEntity` (damage/knockback/fire/AoE), `entityHurt` (protection/thorns; damage cannot be reduced after the fact, so heal back a fraction), `itemUse` (lunge via `applyKnockback`, arrow-firing), `projectileHitEntity`, `playerBreakBlock` (silk touch: cancel drops and spawn the block; fortune: duplicate drops), `entityDie` (looting). Breach bypasses armour by setting health directly rather than `applyDamage`.

**Enchant-handler gotchas (each silently does nothing forever inside a try/catch):**
- Wrong parameter position: a break handler written `(p, l) => ...` when the table passes `(player, ev, level, drops)`; `l` receives the event, `l - 1` is `NaN`, `addEffect` throws, the catch swallows it.
- `applyKnockback({x:0,z:0}, 0)` does NOT cancel knockback. There is no player knockback resistance; push back along the current velocity instead.
- `jump_boost` at amplifier 128 as a "cannot jump" trick is a Java folk trick, **unconfirmed on Bedrock** and possibly the reverse.
- Effect ids must be real; a typo'd id throws inside your try/catch.
- Effect amplifier accepts 0-255 only: `2 * level` at level 255 throws. Clamp amplifier and duration in one helper.
- Entity-query radius and loop counts must never scale freely with a level (`20 + 10 * l` scans a 2500-block sphere every second; `6 * l` lightning bolts is 1530 in one tick). Budget every repeat count by what its body does (~150 particles, ~40 projectiles, ~24 explosions, ~15 summons per tick).
- Resolve enums once at load (`EntityDamageCause.entityAttack`) with literal fallbacks; an inline read of a missing enum kills every damage enchant.
- Hard-coded max health (20) is wrong once health boost exists; read the entity's real max.
- Passive auras hit friendly tamed animals unless you skip family `animal`.
- DIE handlers only receive the drop pile if they run in the deferred pass; otherwise `drops` is `undefined`.
- Derived tables (`GROUPOF` built from `CATALOG`) must be built after the last `CATALOG.push(...)`.
- Per-entity status maps cleaned only on death leak forever (mobs mostly despawn); sweep periodically (e.g. every 600 ticks).
- A generated module is only as fresh as its last generate: have the build run the generator.

**Effects never stack.** `addEffect` for the same effect twice keeps the **higher amplifier**; two Strength I sources are still Strength I. Slowness is a single slot shared by everything that toggles it. To make gear stack: one interval owns worn/held effects, walk every slot, look each item up in a table of `{effect: level}`, **sum levels per effect**, then apply once with `amplifier = min(cap, total) - 1` (cap e.g. 5). Only describe effects as stacking if the code sums them.

**Mace-style smash (fall-scaled bonus):**
- `entityHitEntity`: if the attacker holds the item and is falling, `target.applyDamage(smash, {cause: "entityAttack", damagingEntity: attacker})` scaled by fall speed, plus a small AoE; record the attacker id in a Map with an expiry tick.
- Negate own fall damage: healing back in `entityHurt` still plays the hurt sound/flash. Instead apply `slow_falling` at the moment of the smash; it prevents fall damage entirely, so no hurt event fires.
- Instant "trident": hitscan with `getEntitiesFromViewDirection({maxDistance: big})`, damage first few, particle tracer.

**Custom throwable trident:** item gets `minecraft:projectile {"projectile_entity": "minecraft:thrown_trident"}` + `minecraft:consumable {consume_seconds, animation: "trident"}` (or shooter/chargeable). `world.afterEvents.entitySpawn` fires with `typeId === "minecraft:thrown_trident"`; track in a Set and drive with `system.runInterval`, reading `getVelocity()` / `location`. Do not "trail" a projectile with thousands of `setType` calls per second (griefs the world, lags hard); use a hitscan instead.

**Drops and mining for custom tools (verified against docs; plausible guesses were wrong):**
- **Instant-mine everything:** `destroy_speeds` `block` accepts a Molang expression in `tags`, so a constant-truthy `{"block": {"tags": "1"}, "speed": 100}` is a catch-all. Put specific block entries BEFORE the catch-all. `speed` must be an integer.
- **Digger alone makes ores drop NOTHING** (speed is not harvest tier). Add a tier tag to `minecraft:tags`: `["minecraft:is_pickaxe", "minecraft:is_tool", "minecraft:netherite_tier"]`. Without a tier tag, obsidian/ancient debris/diamond ore break instantly and drop nothing.
- **Bedrock has a `/loot` command with a `mine` source.** `loot give @s mine <x> <y> <z> mainhand` gives Silk Touch / Fortune-correct drops straight to the inventory; `loot spawn <x y z> mine <x y z> mainhand` drops them on the ground. Order matters: run `loot ... mine` FIRST (it reads a block that must still exist), then clear with `setblock <x> <y> <z> air replace`. Using `destroy` after a loot call **double-drops every block**. Keep `setblock ... air destroy` only as the fallback if `/loot` is unavailable (drops on the ground, ignores enchantments).
- **Bedrock usually refuses by returning `successCount: 0` and NOT throwing.** A command that runs and refuses leaves the drop taken and the block standing; a pack that pays XP after an unverified `setblock` pays it forever (8 payouts for 8 blocks still standing, farmable by standing still). Check that the block actually went (re-read it, or check `successCount`) before paying.
- **Neither `/loot` nor `spawnItem` awards XP.** Award it with `player.addExperience(n)` from a per-ore range table (vanilla: coal 0-2, diamond/emerald 3-7, lapis 2-5, redstone 1-5).
- **Area-mining safety:** with no hardness property, a typeId **denylist is mandatory**: bedrock, barrier, command blocks, portal/end-portal frames, structure blocks, reinforced deepslate, liquids, and **containers** (chest, trapped chest, ender chest, barrel, shulker boxes, furnaces, hopper, dispenser, dropper, brewing stand, beacon, spawners, vault, bed, lectern, jukebox, decorated pot); otherwise a 5x5x5 swing next to a storage room voids its contents.
- Tree felling: BFS over the 26 neighbours from the broken log, bounded by a reach box (e.g. 16 horizontally, 32 vertically) and a block cap; only start when the broken block is actually a log.
- Block-placing features must fill **air only** and skip occupied space (never suffocate or overwrite builds); a per-block safety rule cannot catch a fault built across two blocks.

**Reach/radius limits:** the game only returns entities in loaded chunks, at most the simulation distance (32 chunks = 512 blocks). A "reach anywhere" setting of 1024 covers that twice; a huge number looks identical in a fake but a value the real game refuses makes a sensor silently stop seeing. Never let "anywhere" apply to block-area loops (they walk every cell in reach, which stops the game) or to anything that takes something from someone else (a lock that would shut every chest within 1024 blocks).

---

### 9. Identifiers, references and unverifiable claims

- **Never hand-guess a list of vanilla identifiers.** 23 of 45 guessed music-event names were fictional (`music.game.desert`, ...) while the count matched. Pull the authoritative list from Mojang's `bedrock-samples` repo (e.g. `resource_pack/sounds/sound_definitions.json`, every entry with `category: "music"`; `mojang-item.d.ts` in `@minecraft/vanilla-data` for item ids) and verify by **set difference**, not by count. Real music cues include oddities: `music.overworld.*` biome cues, `music.game.swamp_music`, `music.game_and_wild_equal_chance`.
- Java and Bedrock names differ: `minecraft:rooted_dirt` is Java; Bedrock calls it `dirt_with_roots`.
- A reference snapshot (items, ids, sounds, geometries, animations, filter tests) is worth checking against: recipe ingredients, loot entries, ore drop tables, block lists, `using_converts_to`, mob sound events, every `playSound`. All fail silently (bad sound id = mute mob, bad item id = uncraftable recipe). A missing or truncated reference must FAIL the build (assert minimum counts), or every check passes on anything.
- If a whole id class cannot be checked (no particle data in the reference), do not use it.
- **Block tags** in `destroy_speeds` molang queries are a separate unverifiable class: a tag that does not exist makes the entry a silent no-op. Use an allowlist of tags you are confident about (stone, metal, wood, sand, gravel, dirt, grass, snow, clay, web, wool, plant, leaves, rail) and express exceptions as explicit block ids (`{"block": "minecraft:sponge", "speed": 20}` is valid and an id can be verified).
- `minecraft:damage_sensor` causes: see section 6.
- **`Player.isSleeping`, scoreboard cross-pack channel and `itemStartUse` for custom items are unproven** (see sections 3, 7, 9).

**Dynamic properties do not cross packs.** A dynamic property is scoped to the behavior pack that set it, so two add-ons cannot talk through `world.getDynamicProperty("shared:x")` even if both agree on the name. The cross-pack channel that is world data is a **scoreboard objective**: `world.scoreboard.addObjective(id)` / `o.setScore("x", n)` (integers only; publish block coordinates as three participants `x`, `y`, `z`; never set a display slot). Pin the objective name as a string literal in a test in **both** projects (checking it against your own constant renames both halves together and catches nothing). **(Both scoreboard use and per-pack scoping were unproven by any shipped add-on at time of writing.)**

---

### 10. Deploy, pack loading and "my change did nothing"

**Where the game reads packs (verified on a current Windows Bedrock build, v26.45+):**

```
%APPDATA%\Minecraft Bedrock\Users\Shared\games\com.mojang\
```

**NOT** the old UWP `%LOCALAPPDATA%\Packages\Microsoft.MinecraftUWP_8wekyb3d8bbwe\LocalState\games\com.mojang` (Preview: `Microsoft.MinecraftWindowsBeta_8wekyb3d8bbwe`). The old folder still exists with `behavior_packs` / `development_*` subfolders, so writing to it looks perfectly successful while the game never reads a byte. Symptoms: "all my blocks disappeared" after a "successful" update; in-game pack list stuck at an old version; **"Duplicate pack detected"** on every import while the old folder shows nothing to duplicate. **Tell:** `minecraftWorlds` is missing or empty in the folder you are writing to; if the game has worlds and that folder has none, it is the wrong folder.

There can also be numbered per-user profiles beside `Shared` (`...\Minecraft Bedrock\Users\<numeric id>\games\com.mojang\`). A deploy should write `Shared` **and** every numbered profile it finds. Write a `mojang_roots()` helper that returns every layout on the machine, newest first, and copy it into every new Bedrock project.

**Delivery methods**
- Development folders: copy into `com.mojang\development_behavior_packs\<Name>_BP` and `development_resource_packs\<Name>_RP`, then `rm -rf` and re-copy on update. The pack shows up under "My Packs"; the tester enables it in the world's Behavior/Resource pack lists. Minecraft **scans the dev folders only at startup**: a fresh stage needs a full game restart, not just leaving the world.
- `.mcaddon` import: opening (double-clicking / `Start-Process`) the file launches Minecraft and imports, even with the game closed. **But** a launched `.mcaddon` is ignored while you are **inside a world** (it worked when the title screen was showing). Cold start takes ~30 s; wait and re-poll before relaunching. Importing an unwanted large pack can hang the game.
- Re-importing a same-UUID `.mcaddon` every version creates confusing duplicate/ghost entries (one can show EMPTY while the real pack is fine). Every import makes another copy (`Pack`, `Pack(1)` ... `(7)`, all one UUID), and that pile refuses new imports; delete old copies, keep the newest. **Minecraft remembers pack ids that are no longer on disk**: bumping the UUID seed is the only reliable way past a stuck duplicate (cost: switch the pack on again in each world).
- **Never let a dev-folder copy and an imported copy coexist** (same UUIDs = two packs fighting over one id).
- **Never delete a working staged copy before its replacement is confirmed in place.** Clearing the dev folders to "make room" for an import that then silently wrote nothing left the add-on installed nowhere. If a UUID clash is the worry, build the new pack with new UUIDs and a new name (v2) alongside, and delete the old one only after the new one is verified in game.
- Verify an import by finding the installed copy matched on **manifest UUID, not folder name** (import renames the folder after the pack). A packager's `stage()` should update whichever copy exists.
- In game a BP+RP pair with matching names shows as ONE entry, not separate "BP" / "RP" lines.

**Checklist when an add-on "isn't working" / an update seems ignored (in this order):**
1. **Is the pack on in the world?** Check `minecraftWorlds\<world>\world_behavior_packs.json` and `world_resource_packs.json`. A world can list the **resource** pack while its **behavior** list is `[]`: textures load so the pack looks present, but there are no blocks/items and **no scripts run at all** (so nothing answers a web bridge either). Minecraft rewrites these files when saving a world and that rewrite has dropped the behavior entry; hand-edits only stick while the world is closed.
2. **Does the world carry its own copy?** `minecraftWorlds\<world>\behavior_packs\<Name>\` and `resource_packs\<Name>\` **win over My Packs.** Every install, import and rebuild then lands somewhere the game never reads and the world keeps running an old build in total silence. Update these too (find by pack UUID). Check first, not last: `ls minecraftWorlds/<world>/behavior_packs minecraftWorlds/<world>/resource_packs`. Do not reinstall before finding what the game actually loads. Old builds kept there are also a handy archive to cross-check a theory against a version that worked.
3. **Is it the right com.mojang folder?** (above)
4. **Item count.** An add-on grown to ~270,000 item files silently registered NOTHING: the import hung the game, dev staging produced a pack that listed but registered nothing (`/give @s ns:pistol` gave a syntax error, i.e. the id did not exist), no log line, every filesystem check perfect. ~1,100 items (a 755 KB `.mcaddon`) works. Keep item counts sane.
5. **Manifest / script module.** Bad `@minecraft/server` version, a cross-file import of an unexported name (one bad import kills the whole script module: no handlers register, add-on looks absent), or a top-level throw. `node --check` does not catch cross-file import errors (and exits 0 on some unparseable ESM); load the real module graph in Node against stubbed `@minecraft/server`.
6. **Boot beacon.** Bedrock writes no content log by default (the `LocalState\logs` folder stays empty), so a script that failed to load looks identical to one that worked. Print a beacon (`world.sendMessage(...)` on the first tick / action bar version label) so "the script loaded" is visible.

**Other Windows/API facts**
- `/say` rejects a message whose first character is `[`; prefix a colour code (`§8[tag]...`) when using chat as a channel.
- `dimension.fillBlocks` / `getBlock` / `setType` in **unloaded chunks silently do nothing**. Building "far away for safety" (3000 blocks, Y210) and then teleporting a player there dropped them into open sky: no error, only "I keep falling from above the clouds". Build set-pieces near a player (directly overhead is always loaded); **verify the build before moving anyone into it** (read back a floor block under the destination, abort and tear down if not solid); run a fall catcher (below floor Y -> put them back with Slow Falling); give Slow Falling / Resistance on any scripted teleport to a height.

---

### 11. Web app <-> Bedrock bridge

A behavior pack cannot call the network, so a web page reaches the game via the **Code Connection websocket**: the game dials OUT (`/connect <ip>:<port>`) to a server you run, which sends commands.

- **page -> game:** the bridge sends a `commandRequest` with `commandLine: "scriptevent ns:remote <args>"`; the add-on handles it in `system.afterEvents.scriptEventReceive`.
- **game -> page:** the bridge `subscribe`s to `PlayerMessage`; the add-on answers by saying things: `dimension.runCommand("say §8[tag]...")` is the only channel a websocket can read back.
- Windows blocks the UWP Minecraft from reaching `127.0.0.1`. Print the LAN IP and `/connect 192.168.x.x:19131`. (Loopback exemption `CheckNetIsolation LoopbackExempt -a -n=Microsoft.MinecraftUWP_8wekyb3d8bbwe` needs admin.) Cheats must be ON and Settings -> General -> "Require encrypted websockets" must be OFF.
- Chat is the return channel and it is visible: stay silent until the bridge says `hello` (a broadcast window that expires), send **deltas** for one node and a full snapshot only when asked.
- Snapshots must be **chunked** (a header line plus a line per record; chat has a length limit) and a half-arrived snapshot must not replace the good one.
- **On Windows a second server can bind a port that is already listening** (`SO_REUSEADDR` / `HTTPServer.allow_reuse_address`); the two then split requests randomly (an old bridge served a deleted page while the new one looked fine). Set `allow_reuse_address = False`, skip `SO_REUSEADDR` on `nt`, let the second one fail loudly.
- The websocket server is ~60 lines of stdlib (sha1 handshake, masked frames in, unmasked out); no pip needed. Fake the *client* half in a test so the whole loop is testable without the game.

---

### 12. Textures (PIL) for 16px icons

- **`ImageDraw` replaces pixels, it does not alpha-blend.** `fill=(0,0,0,130)` over a finished icon writes a half-transparent pixel = a **hole** in Minecraft's UI. Use solid computed tones (e.g. `(46,44,58,255)` dark, `(176,176,192,255)` mid). For real translucency draw into a scratch RGBA layer over the shape's bounding box and `alpha_composite` it back.
- Stacked translucent shapes accumulate toward opaque (16 concentric alpha-140 circles = a flat white disc). Do a glow/vignette as ONE composite of a gradient alpha mask.
- A 16px glyph in one tone is a blob. Use three opaque tones (bright face, mid body, dark seam); put details **outside** the shape they belong to; leave a dark column between adjacent 2px buttons; a pickaxe head must sweep (high at ends, low in the middle) with a diagonal handle or it reads as the letter T; gear teeth must overlap the ring.
- Preview on a **neutral inventory-like background**, not near-black (a blade the exact grey of an inventory slot passed many "look at it" passes on a black preview). A contact sheet (every icon scaled x6 with its name) catches all of this at a glance and no scripted check does.
- Adjacent-colour duplicates: compare with an RGB distance, not equality (nine bottle pairs passed equality and were indistinguishable).

---

### 13. Silent-failure trap index (quick scan)

| Symptom | Cause | Fix |
|---|---|---|
| Item has no texture, no error | `minecraft:icon` is an object on 1.21 | plain string |
| Custom recipe not in recipe book | missing `unlock` | `"unlock": {"context": "AlwaysUnlocked"}` |
| Sword shows 266 dmg, hits for 10 | `minecraft:damage` byte wrap | clamp 127, apply overflow in script |
| Custom pickaxe breaks ores, no drops | no tier tag | add `minecraft:netherite_tier` (etc.) |
| Area-mined blocks drop twice | `setblock ... destroy` after `loot` | use `air replace` after loot |
| Area mining gives no XP | `/loot`, `spawnItem` grant none | `player.addExperience` |
| Mob never spawns, no error | invalid `damage_sensor` cause / bad key | validate causes; check keys vs vanilla file |
| Mob attacks its own kind | `"filter"` instead of `"filters"` | correct key |
| Mob is mute | no `sounds.json` entry / no `ambient_sound_interval` | add both |
| Whole script module absent | bad `@minecraft/server` version, or bad cross-file import | match working siblings; load module graph in Node |
| Form throws on open | old positional `slider/toggle/dropdown` args | options objects |
| Custom block menu opens twice | `playerInteractWithBlock` per-hand | `isFirstEvent === false` guard |
| Enchant does nothing forever | handler param order / NaN in try/catch | fixed arity, audit level slot |
| Off-hand weapon does no damage | Bedrock only swings main hand | script via `entityHitEntity` |
| Held-fire item fires once | no `use_modifiers` | add it + click-fire fallback |
| Update ignored | wrong com.mojang folder / world-embedded copy / pack off in world | section 10 checklist |
| Vanilla recipe broken by the pack | duplicate grid or shapeless multiset | second ingredient; guard list |
| Two effects "stack" but do not | highest amplifier wins | sum levels, apply once |
| Feature built far away, player falls | unloaded chunks no-op | build near players, verify floor |
| Two add-ons cannot share state | dynamic properties are per-pack | scoreboard objective (unproven) |
| Vanilla sounds still play | guessed event names | derive list from Mojang samples, set difference |
| `/gamemode` respawn fails for non-hosts | command needs op | `player.setGameMode("Survival")` |
| Speed/range wrong by a constant | one quantity in two units (per-step vs per-second) | test at stated range |
| Facing check fails at steep view angles | fake player always level | test at steep angles (fails past ~75 degrees) |


---

## Part 2 — Mobs, entities, models, animation and art

Everything here was learned building Bedrock add-ons (behaviour pack "BP" + resource pack "RP") with generated content: armed humanoid mobs, wave-survival enemies, catalogue-style packs with dozens of mobs/tools/drinks, plus procedurally drawn textures. Most of the traps are **silent**: nothing errors, the build validates, and the game just looks or feels wrong. Each trap below is written as symptom -> cause -> fix. Claims that came from a single observation or that I could not re-verify are marked *(unverified)*.

---

### 1. Anatomy of a custom entity (what has to line up)

**Files (Bedrock):**

```
behavior_pack/entities/<mob>.json        "minecraft:entity"        identifier ns:name  (AI, stats, spawn events)
resource_pack/entity/<mob>.json          "minecraft:client_entity" same identifier; geometry + textures + render controller + animations
resource_pack/models/entity/<geo>.json   custom "minecraft:geometry"
resource_pack/textures/entity/<mob>.png
resource_pack/animations/<anim>.json     custom animation(s)
resource_pack/texts/en_US.lang           entity.ns:name.name=Display Name   (+ spawn egg name)
resource_pack/texts/languages.json       ["en_US"]
behavior_pack/spawn_rules/<mob>.json     optional; without it the mob exists only via the creative-menu egg / script
```

- **The link between BP and RP is only the shared `ns:name` identifier string.** Nothing else ties them.
- Format-version strings are per file type and are strings, never the number 2: typical values are `"1.21.0"` for entity/items, `"1.10.0"` for client_entity, `"1.12.0"` / `"1.16.0"` for geometry. (Manifests use `format_version: 2`.)
- Texture references in client_entity / `item_texture.json` / `terrain_texture.json` omit `.png`; putting `.png` in makes the lookup fail silently.
- `languages.json` must list every locale (`["en_US"]`) or the `.lang` file is ignored and mobs show raw ids. A pack that had `en_US.lang` but no `languages.json` silently dropped 525 display names.
- A custom entity's spawn egg is created by the engine but is **not craftable**. A mob with no spawn rule therefore exists only in the creative menu; a survival player never sees it even if your in-game guide describes it. Treat "obtainable in survival" as its own property to check.

**Two render strategies (both halves required):**

1. **Custom**: own `geometry.<name>` + own texture + own animation. Needs art but you control everything.
2. **Vanilla-borrow (zero new art)**: BP entity sets `"runtime_identifier": "minecraft:zombie"` inside `description`; RP client_entity reuses `geometry.zombie`, the zombie texture and `controller.render.zombie`. You get custom identity/stats with vanilla visuals. Bonus: held items render for free (see 3.7).

Sketch of a client entity for a custom rig (generic shape, adapt names; not copied from a shipped file):

```json
{
  "format_version": "1.10.0",
  "minecraft:client_entity": {
    "description": {
      "identifier": "ns:my_mob",
      "materials": { "default": "entity_alphatest" },
      "textures":  { "default": "textures/entity/my_mob" },
      "geometry":  { "default": "geometry.my_mob" },
      "animations": { "walk": "animation.my_mob.walk" },
      "scripts": { "animate": ["walk"] },
      "render_controllers": ["controller.render.default"],
      "spawn_egg": { "base_color": "#8a5a2b", "overlay_color": "#ffd24a" }
    }
  }
}
```

**Ordering rule:** `geometry.<name>` must exist in the RP before a client_entity references it; texture short names must exist before use.

**Build-time guard worth having (the "invisible mob" class):** resolve every client_entity reference (geometry, animation, animation controller, render controller, texture) against the files that exist. An unresolved reference doesn't crash; the mob is just invisible or a magenta/T-posed shell.

---

### 2. Custom mob AI traps (all look like "the AI is just bad")

#### 2.1 Two behaviour keys of the same name collapse to one
- **Symptom:** mobs hunt only the player, never each other, though you wrote a target rule for each other; build validates fine.
- **Cause:** `minecraft:behavior.nearest_attackable_target` written twice in one components dict keeps only the last (dict/JSON key semantics, in Python or JSON alike).
- **Fix:** put every target in ONE behaviour's `entity_types` array.

#### 2.2 `is_family` with `"operator": "!="` does not exclude
- **Symptom:** "attack any clone that is not my kind" produced the opposite: a mob hunted only its own kind.
- **Fix:** use `none_of`:
```json
{"all_of": [
  {"test": "is_family", "value": "FAMILY"},
  {"none_of": [{"test": "is_family", "value": "own_family"}]}
]}
```

#### 2.3 ...but excluding your own kind was the wrong design anyway
- **Symptom:** a field of identical mobs stood around ignoring each other.
- **Cause:** it worked exactly as written; the design was wrong for free-for-all.
- **Fix:** for free-for-all mobs exclude nothing. **An entity never targets itself**, so no self-exclusion is needed.

#### 2.3b "Hostile to everything" is one filter: `is_family: mob`
- Most vanilla mobs carry family `mob`; your custom mobs do too **if you put `mob` in their `type_family`**. Two halves must agree and neither is visible from the other: the mob must CARRY `mob`, and its targeting must ASK for it. Guard both.
- **Anything else you spawn (bullets, projectiles) must NOT carry `mob`**, or every shot pulls every mob's aim onto the round in mid-air.
- Players are family `player`, not `mob`: list them as a second entry.

#### 2.4 Default targeting feels like a delay
- **Symptom:** "they aren't hostile until a few seconds in".
- **Cause:** `nearest_attackable_target` scans every 10 ticks by default and `target_in_sight_time` adds more.
- **Fix:** `"scan_interval": 1` and `"reselect_targets": true`. If the design says "hostile immediately", never add a wind-up.

#### 2.5 `behavior.panic` fires on ANY damage
- **Symptom:** mob bolts the instant it is scratched, the fight never happens, AI reads as stupid.
- **Cause:** panic is not health-gated.
- **Fix:** remove it; gate retreating on actual health in script. 15% of max health worked; 30% still read as cowardice.

#### 2.6 `minecraft:shooter` `attack_interval_min/max` are in SECONDS, not ticks
- **Symptom:** a fast gun (6-tick cadence) fired one shot a second.
- **Cause:** `ticks // 20` floors 6 ticks to 0, then a `max(1, ...)` clamp makes it 1.
- **Fix:** `round(ticks / 20.0, 2)`.

#### 2.7 `alert_same_type` on `hurt_by_target`
- Calls its own kind to help, which is nonsense once they fight each other. Turn it off for free-for-all mobs, and list the `mob` family in `hurt_by_target.entity_types` or a mob hit by another mob never retaliates.

#### 2.8 `nearest_attackable_target` hunts EVERY player in range
- **Symptom (no error):** a bystander building near a wave-survival arena was swarmed by mobs from someone else's run; the wave also *walked away* from the player actually fighting it whenever the bystander was closer (which quietly trivialises the fight).
- **Cause:** a filter `{"test":"is_family","subject":"other","value":"player"}` means any player. There is no per-instance "my owner" filter.
- **Fix:** script puts a tag on a player for the length of their run; the entity filter requires it:
```json
"filter": {"all_of": [
  {"test": "is_family", "subject": "other", "value": "player"},
  {"test": "has_tag",   "subject": "other", "value": "ca_player"}
]}
```
- **Tags outlive a logout.** Someone who left mid-run returned still tagged and got hunted by the next person's arena. Strip the tag on spawn when the player has no live run, and on world load.
- **Test the generated data, not the script.** A test that only asserts "the player got the tag" passes while the entity JSON still hunts everyone. Read the filter off disk and assert it names the tag; prove the test can fail by reverting both halves.

#### 2.9 Melee damage should be the weapon's damage
- 1 damage = half a heart. A stick-wielding melee mob hitting for 14 was wrong; a stick adds nothing so it should hit for 1 (its gimmick is speed). A melee mob's contact "shove" uses the same number.
- Example table used for armed humanoids (damage = the weapon's own damage): stick 1, light gun 3, pistol 5, diamond sword 7, auto shotgun 8, gun-blade 9, machine gun 10, sniper 13. Gun-carriers that bump you do 1.
- One mob can be both ranged and melee: `does_melee = (not ranged) or melee` lets it carry `minecraft:shooter` and a melee attack together.
- Ranged mobs point `minecraft:shooter` at the pack's own projectile entities, so damage is engine-dealt and identical to the player's version of that weapon.

#### 2.10 Behaviour keys that don't exist
- Verify a behaviour against vanilla data before writing it. `behavior.random_hover` and `behavior.dig` were on a shortlist and **do not exist**. Real ones used successfully for per-mob variety: `behavior.tempt` (specific item), `behavior.avoid_mob_type`, `fire_immune`, `rideable`, `behavior.eat_block`, `behavior.move_to_water`.

---

### 3. Mob patterns

#### 3.1 Tameable custom mob
A tameable mob = a `component_groups` "tamed" set that a `minecraft:tameable` event adds on success.

```json
"components": {
  "minecraft:tameable": {
    "tame_items": ["minecraft:bone", "minecraft:cookie"],
    "probability": 0.4,
    "tame_event": {"event": "ns:on_tame", "target": "self"}
  }
},
"component_groups": {
  "ns:tamed": {
    "minecraft:is_tamed": {},
    "minecraft:persistent": {},
    "minecraft:behavior.follow_owner": {"priority": 4, "speed_multiplier": 1.2, "start_distance": 5.0, "stop_distance": 2.0},
    "minecraft:behavior.sit": {"priority": 2}
  }
},
"events": {
  "ns:on_tame": {"add": {"component_groups": ["ns:tamed"]}}
}
```
Right-click while holding a tame item -> chance to tame -> event adds follow-owner + sit. Put `minecraft:persistent` in the tamed group so pets never despawn; wild ones keep `minecraft:despawn`. A generator can branch on a `kind` field: hostile kinds get attack + target AI, friendly kinds get this pattern plus panic + tempt.

#### 3.2 Target-dummy mob
Post, base, chest with a painted bullseye, head, two stub arms. No walk cycle: it sways gently and wobbles on `query.hurt_time` (counts down from 10 after a hit, so the wobble dies away by itself). Any exporter must know the "dummy" kind, or a dummy exported through it comes out as a normal melee mob.

#### 3.3 Mobs that build things (cover, pillars)
Anything that places blocks must: fill **AIR only**, skip occupied cells, cap how many it places (6 per mob worked), and **remove every block it placed when the fight ends or it dies**. Test all of it: "a mob that rearranges the world is a menace." (Also see the arena/site-safety rules elsewhere in this knowledge base.)

#### 3.4 Contact shove
Anything within ~2.4 blocks (player, or a mob of a DIFFERENT kind) takes 1 damage and a real knockback once a second. Same-kind mobs are exempt so groups still cluster.

#### 3.5 Balance settled by feedback for armed mobs (example numbers)
Mobs above ten hearts (24-34 hp; slow heavy ones highest) so they don't fold to a couple of hits; guard both the floor and table-vs-pack drift in the build. Player-strength (20 hp) mobs died too fast. Feedback direction was consistently "tougher": hostile immediately, then attack own kind, then all mobs, then never flee, then more health. Lean tougher.

#### 3.6 Enchants and runes on entities (things with no inventory)
- Store on the entity: `entity.setDynamicProperty("pb:ench", "key=lvl,key=lvl")`, with a `Map` fallback keyed by entity id if the property API refuses (fallback does not survive chunk unload). Use the same `key=lvl` string format as item lore so one parser serves both.
- Only wire HIT / HURT handlers for entities: a mob has no inventory to tick and no blocks to mine, so TICK/BREAK/USE entries would be dead weight.
- Build the random-mob-enchant pool **from the handler tables themselves**, never a hand-written list, minus rare/legendary and minus a denylist of world-altering ones.
- **`entitySpawn` fires again when a chunk reloads.** Skip the roll for any entity already carrying a map or every mob reshuffles as you walk.
- **The action bar is a shared resource.** A look-at readout running 4x/second silently wiped every message other code showed. Give messages a quiet window (2.5 s stamp) that the readout respects; any new always-on HUD must do the same.
- Player "runs": stats on the player re-applied on an interval with a duration longer than the interval (200-tick reapply, 16 s duration) so effects never flicker off; no refund on removal so there is no buy/sell loop.

#### 3.7 Held items render for free with the zombie runtime id
- With `"runtime_identifier": "minecraft:zombie"` the entity already draws whatever is in its hand. An earlier note said to bake the weapon sprite into the skin on an arm quad; that was **wrong**: it gave a pistol mob two pistols in one hand. Bake a quad only for an entity with **no** runtime identifier.

#### 3.8 Spawn pool budget (custom mobs swamp vanilla)
- **Symptom:** "this add-on replaced the animals." Every consistency guard was green.
- **Cause:** thirteen custom mobs each with weight 5-9 (looks normal next to a cow's 8) and the blanket `overworld` biome tag competed in EVERY biome and claimed ~63% of the daylight-surface animal pool (combined weight 85 vs vanilla ~50). Custom passives also keep respawning where vanilla animals mostly spawn once at world generation, so domination grows over time.
- **Fix:** measure the share per POOL (daylight surface is the scarce one; dark surface; underground where vanilla animals barely spawn; monster pool in the hundreds). Drop daylight weights to 1-2, move mobs to pools vanilla doesn't use where it suits the creature, freeze a stated budget per pool, fail the build over it and print pools in the balance report.
- Nine mobs also had NO spawn rule at all (see section 1, egg not craftable). Check obtainability separately.

---

### 4. Animation

#### 4.1 Borrowed vanilla animation controllers do not animate a custom entity
- **Symptom:** custom mob referencing `geometry.zombie` (or other vanilla geometry) renders static, T-posing or sliding; looks worse than a plain box.
- **Cause:** `geometry` alone is not animation. The vanilla controllers (`controller.animation.humanoid.move`, `look_at_target`, `animation.humanoid.bob`) depend on the full vanilla animation short-name map being present, which a custom client entity does not replicate. **Verified in-game: referencing them did NOT animate the custom entity** (mobs stayed static/sliding). Reproducing the maps blind for non-humanoids (slime, bat, golem, creeper, guardian, vex, bee, phantom, chicken, wolf, pig, cat each have their own ids) is error-prone.
- **What works reliably: own the whole rig.** Define a custom `geometry.<name>` with bones you name (`head, body, rightArm, leftArm, rightLeg, leftLeg`, standard 64x32 humanoid UV), write your own animation targeting those exact bones, put it in `RP/animations/` and reference it from the client entity's `animations` + `scripts.animate`. Applied to 14 humanoid mobs. For non-humanoids build archetype models (blob / quad / wing) with your own animations, same pattern.
- **Alternative tool:** the Bridge. editor with the "more vanilla entity presets" + "custom entity syntax" plugins copies the COMPLETE vanilla client entity (geometry, textures, animations, animation controllers, render controllers), so mobs animate out of the box. *(Tooling recommendation, reported by the add-on author; I have not re-verified it.)*

Walk-cycle building blocks that worked:
```
legs/arms swing:  math.cos(query.modified_distance_moved * 38.17) * 40
body bob:         query.anim_time
```

#### 4.2 Bone names must line up between geometry and animation
- **Symptom:** one limb frozen; identical look to the borrowed-controller failure; no error, no log line.
- **Cause:** animations address bones by NAME; a renamed bone or a typo leaves that limb static.
- **Fix (cheap build check for every rig):** every bone an animation targets must exist in the geometry (**failure**); every geometry bone with NO animation is frozen by definition (fine for a fixed part, suspicious for a limb: **note**, not failure).

#### 4.3 Distance-moved walk cycles freeze mid-stride when the mob stops
- **Symptom:** a mob that stops stands like a statue frozen mid-step at a random angle; no idle.
- **Cause:** `math.cos(query.modified_distance_moved * N)` doesn't go to zero when movement stops, it goes to a CONSTANT (distance stops changing, cosine returns whatever it last evaluated).
- **Fix (two parts, baked into the same expression):**
```json
"rightLeg": { "rotation": [
  "math.cos(query.modified_distance_moved * 38.17) * query.modified_move_speed * 42
   + math.sin(query.anim_time * 17) * 1.5", 0, 0 ] }
```
1. `* query.modified_move_speed` (0..1) fades the stride out as the mob slows.
2. A small `query.anim_time` term on every bone (breathing on the body, slow head turn, a couple of degrees of limb sway). `anim_time` keeps advancing while standing still, so this is what makes a stationary mob look alive.
- Vanilla mobs get idle by blending a separate idle animation in via controllers. With one hand-written animation, bake both into one expression: stacking two animations on one bone in `scripts.animate` blends unpredictably.

---

### 5. Models and UV

#### 5.1 Box UV scales its texture patch with the cube
- **Symptom:** a parametric/slider-driven model renders garbage at bigger sizes; geometry file still "valid"; enlarged limbs overflow the sheet at slider extremes.
- **Cause:** `"uv": [u, v]` derives the patch from the cube: a cube (w,h,d) at (u,v) claims **2*(w+d) wide by (h+d) tall** (the whole unfolded box). Double the cube and the patch doubles and walks off the sheet.
- **Fix:** per-face UV (geometry `format_version` 1.12.0+):
```json
"uv": {
  "north": {"uv": [12, 20], "uv_size": [8, 12]},
  "south": {...}, "east": {...}, "west": {...}, "up": {...}, "down": {...}
}
```
The patch is pinned regardless of cube size; texture stretches to fit. `mirror` is ignored under per-face UV: give both sides the same patch.

Where box UV would have put each face for a part at `[u,v]` with nominal `w,h,d`:

| face | region | is |
|---|---|---|
| `up` | `[u+d, v, w, d]` | top |
| `down` | `[u+d+w, v, w, d]` | bottom |
| `west` | `[u, v+d, d, h]` | -x side |
| `north` | `[u+d, v+d, w, h]` | **the front: where a face/eyes go** |
| `east` | `[u+d+w, v+d, d, h]` | +x side |
| `south` | `[u+d+w+d, v+d, w, h]` | back |

Two silent mistakes: the head's `[u+d, v]` patch is the **top of the head**, not the face (eyes ended up on the scalp); and enlarged limbs overflowed a 64x64 sheet. A UV-bounds check over every rig at min/mid/max slider values caught both. Write it before trusting a parametric generator.

#### 5.2 Overlap rules (what to guard)
- Two cubes whose rectangles **partially** overlap share pixels: painting one face repaints part of another. A rectangle running past the sheet samples whatever is at the edge. Both silent.
- **Mirrored limbs share a patch on purpose** (both arms at one uv, all four legs of a quadruped at another). So an IDENTICAL rectangle is correct and a PARTIAL overlap is the bug. A guard refusing all overlap flags every rig ever written.
- A front-face composite (the usual cheap rig preview) says nothing about sides, top or back; those can be wrong for a long time unseen.

#### 5.3 Paint markings on the patch a player actually sees
Found only by composing a **front view** from the real box-UV patches and looking at it:
1. **Marking invisible head-on.** Spots were scattered over the whole body texture region (0,0)-(31,15), but the body's front is only the 6x6 window at (10,10)-(15,15). The mob read plain red from the front, and the second distinguishing signal was gone. Fix: paint the mark over the whole region for sides AND again over the front patch explicitly.
2. **Draw order paints over the face.** `_face()` then `_mark()` put a chest plate across the eyes (eyes on row +2, mark covering rows +0..+2). Fix: mark FIRST, face on top; restrict a head mark to the top two rows so a hat/helmet band can never reach the eye row.
- Box-UV front-face arithmetic worth keeping: for a cube (w,h,d) at (u,v): front `(u+d, v+d)` size `w x h`; top `(u+d, v)` size `w x d`; right `(u, v+d)` size `d x h`; left `(u+d+w, v+d)`; back `(u+2d+w, v+d)`.
- **Always render a front-on composite next to the flat sheet.** A flat 64x32 sheet tells you almost nothing about what a player sees.

#### 5.4 Derive the hitbox from the geometry; never hand-write it
- **Symptom:** every blob mob shipped with a `minecraft:collision_box` exactly TWICE the height of the visible model (swinging at empty air above one connects; they bump ceilings that look clear). 10 of 25 mobs were badly out of step, worst ratio 2.15.
- **Cause:** collision boxes were hand-written in a table while `minecraft:scale` multiplied the model underneath; nothing tied them.
- **Fix:** walk the geometry's cubes, take the real extent, scale it, emit that.
```python
def drawn_extent(arch):          # blocks, 16 model units to a block
    xs, ys, zs = [], [], []
    for bone in geo["bones"]:
        for cube in bone.get("cubes", ()):
            ox, oy, oz = cube["origin"]; sx, sy, sz = cube["size"]
            xs += [ox, ox + sx]; ys += [oy, oy + sy]; zs += [oz, oz + sz]
    # height from y=0, not from the lowest cube: an entity's origin is its
    # FEET, and a rig whose lowest cube starts above zero is floating
    return (max(max(xs)-min(xs), max(zs)-min(zs)) / 16.0, max(ys) / 16.0)
```
- **Height and width are not symmetric.** Height should track the drawn extent almost exactly (a box taller than the model is the bug that hurts). Width should be NARROWER: a vanilla zombie draws 1.0 wide including arms and collides as 0.6, since limbs at the sides aren't bumped. Factors used: humanoid 0.6, critter 0.7, blob 0.95.
- **Check the other end too:** deriving boxes immediately produced mobs under a fifth of a block tall (smaller than a silverfish at 0.3 x 0.4): effectively unhittable. Floor at vanilla's smallest.

#### 5.5 Rig tests that run headless
A model editor's pure code can be run in Node: slice the region between two marker strings out of the HTML, stub a tiny canvas (real PNG encoder via `node:zlib`), and check every rig: every cube has a skin patch, no patch runs off the sheet, no two patches partially overlap, the animation only names bones that exist; an `--export` mode writes the real data. The marker strings are load-bearing: moving them breaks the tool.

#### 5.6 Multi-project storage note for browser model editors
A single-file HTML tool storing skins as base64 PNGs in `localStorage` will hit the ~5 MB quota with several projects. Toast on quota failure instead of swallowing it. When migrating an old storage key to a multi-project key, read once, migrate, and **leave the old key in place as a backup**.

---

### 6. Hit tests, projectiles and geometry that silently never connect

- **`entity.location` is the FEET.** An orbital at `y + 1.1`, a bullet from the head, an explosion at chest height, measured against feet all carry ~1 m of phantom vertical distance. A star orbiting at radius 2.4 sailed over a mob standing exactly 2.4 away every time (true 3D distance 1.45 vs reach 1.2). Fix: one `bodyCentre(rec)` helper returning `y + 0.85 * scale` used by the single "things within range" function so melee, orbitals, lasers, fields and splash all inherit it. Scale matters: a scale-2.0 boss has its centre 1.7 up.
- **A rotating hitbox tunnels like a fast bullet.** Arc length per step is what counts: radius 2.4 at 1.9 rad/s = 1.14 blocks per engine step vs reach 1.2, so it sweeps through a target between two sampled positions. Subdivide until each substep is under half a block.
- **Fast projectiles tunnel.** A bullet moving 4 blocks/tick with a 1.3 hit radius flies through anything between start and end when you check once per tick. Walk the step in sub-hops no longer than the hit radius: `hops = ceil(speed / HIT_DIST)`.
- **A test stub that fakes distance is worse than no test.** A harness filtered `getEntities({location, maxDistance})` on a synthetic `rayDistance` field instead of real geometry, so proximity hits never registered and charge tests silently passed by *claiming* a small distance. Making the stub do real distance maths failed 6 tests that had been green for weeks. When a stub models something differently from the game, every test through it only tests the stub.
- My first substep test passed with substepping disabled (at shipped speeds proximity caught it anyway). Write the test so it FAILS without the fix.
- **Scripted projectile hygiene:** a tick budget that removes them on EVERY exit path (a never-expiring summon hard-freezes the game), a per-player live cap with a graceful fallback, cleanup on `playerLeave`. Test the freeze invariants: "40 rapid shots are capped", "everything expires within its budget", "nothing leaked after the failure path".
- **Homing projectile tip:** spawn `minecraft:shulker_bullet` inert and steer it from script; vanilla's own homing picks its own target, deals its own 4 damage and forces levitation, all usually wrong.
- Scripted, self-spawning mobs recurse exponentially: cap and budget spawns.

#### A wind-up nobody can see
- **Symptom:** one of three charging enemies had a 1 s charge with no root, no particle, no sound; dodging was luck.
- **Cause:** a "holding" flag was read in exactly one place (it pins the enemy down), so rooting *was* the only tell; the third enemy forgot it.
- **Method that found it:** don't ask "is this flag set consistently?"; ask "what does this flag DO, and is any other channel carrying the same information?"
- **Guard mistakes to avoid:** (1) over-specifying "must be phase X and rooted" rejected a correct enemy that telegraphs with particles instead; accept "rooted OR emitting particles". (2) counting all particles in the world passed with the bug restored (bullet trails drowned the signal); restrict to particles within 2.5 blocks of the enemy. Always revert the fix and confirm the guard goes red.

---

### 7. Procedural textures and icons (PIL)

#### 7.1 The 16x16 sprite approach (reusable recipe)
- Shapes as dicts `{(x, y): shade}` with shade codes (`m` main, `l` light, `d` dark, `s` stick, `t` stick dark, `g`/`h` guard shades), origin top-left, drawn by small functions per silhouette (sword, pickaxe, axe...).
- Palette as `(main, light, dark)` per material; a shared dark `OUTLINE` colour.
- **Write pixels directly (`px[p] = colour + (255,)`), no alpha blending**, so nothing translucent turns into a hole.
- **Outline via the neighbour ring:** for every filled cell, add each 8-neighbour that is not filled and is inside the 16x16 bounds to a `ring` set; write ring pixels in the outline colour first, then write the cells on top.
- Optional halo: same trick one ring further out, painted only on a checker (`(x+y) % 2 == 0`) for a glow; sparkles as a few white pixels on empty cells.
- Vary from a table (tiers x kinds) so a whole family renders in one run; scale ×4 with `Image.NEAREST` for pack icons.
- Contact sheet for eyeballing: every sprite ×4-6 nearest-neighbour on a **mid-grey (139,139,139) inventory-slot background**, with names, saved somewhere you will open it.

```python
img = Image.new("RGBA", (16, 16), (0, 0, 0, 0)); px = img.load()
ring = set()
for (x, y) in cells:
    for dx in (-1, 0, 1):
        for dy in (-1, 0, 1):
            n = (x + dx, y + dy)
            if n not in cells and 0 <= n[0] < 16 and 0 <= n[1] < 16:
                ring.add(n)
for p in ring: px[p] = OUTLINE + (255,)
for p, col in cells.items(): px[p] = col + (255,)
```
Seed any random texture noise with `zlib.crc32(name)`, never `hash(str)`: Python randomises string hashes per process, so ore textures were redrawn differently on every build.

#### 7.2 PIL ImageDraw replaces, it does not blend
- **Symptom:** a "shadow" drawn with `fill=(0,0,0,130)` over a finished icon does not darken it; it writes a half-transparent pixel, which in Minecraft's UI is a hole.
- **Fix:** every shadow tone is a solid colour computed by hand (e.g. `(46,44,58,255)` dark, `(176,176,192,255)` mid). If you truly need translucency, draw into a scratch RGBA layer over that shape's bounding box and `alpha_composite` it back; wire that into every drawing primitive so no call site can forget.
- **Stacked translucent shapes accumulate toward opaque.** A "soft glow" of 16 concentric circles at alpha 140 is a flat white disc. A radial falloff must be ONE composite of a gradient alpha mask. Same for vignettes: build a mask; don't stack ellipse outlines (which also throw `y1 must be >= y0` once insets cross on a non-square canvas).

#### 7.3 A 16px glyph needs three opaque tones
A chest as one white rectangle with a black bar reads as a white box. What fixed it:
- three tones per object: bright face, mid body, dark seam/detail;
- details must sit **outside** the shape they belong to (a backpack's straps drawn inside the body get covered by the flap and vanish);
- two adjacent 2px buttons merge into one bar: leave a dark column between;
- a top bar plus vertical stem is the letter **T**, not a pickaxe: the head must sweep (high at the ends, low in the middle) with the handle running diagonally out of it;
- gear teeth must **overlap** the ring, not touch it, or it is a compass rose.

#### 7.4 Preview on the ground the thing ships against
- **Symptom:** a blade colour `#8c8c8c` was 1.7 away from the vanilla inventory-slot grey (~`#8b8b8b`) and invisible in game, yet looked fine for about forty "render it and look" passes; seven more tools were under 30 away (all the stone/gravel ones).
- **Two faults:** (1) the preview drew items on a near-black card (38,40,46), and the distinguishability guard was purely tool-vs-tool; nothing measured against the slot background. (2) The silhouettes had **no outline**: vanilla gives every item a dark outline, which is why a grey stone pickaxe reads on a grey slot. Five original sprites called `outline()` by hand; thirteen added later all forgot.
- **Fix:** render on BOTH grounds (dark that flatters, real mid-grey), include **the background as one of the things every item must differ from**, and move the outline into the one shared finishing function (`Canvas.finish(tier)` = outline then tier pip) so a sprite can't end any other way. If the first few members of a family do a step by hand, that step belongs in the shared path. A palette solver must also know the background, or it resolves a clash by moving a colour onto it.

---

### 8. Telling a big roster apart (mobs, items, blocks, drinks)

The recurring failure is not a crash: the roster "feels like ten things instead of sixty".

#### 8.1 Equality is not distinguishability
- Two guards (no shared egg colour, no shared name) passed while a contact sheet showed near-identical orange bottles and three brown ones. Swapping equality for an RGB distance threshold found **9** drink pairs and **7** spawn-egg pairs indistinguishable.
```python
COLOUR_MIN = 60
def colour_gap(a, b):
    ra, rb = hex2rgb(a), hex2rgb(b)
    return sum((ra[i] - rb[i]) ** 2 for i in range(3)) ** 0.5
```
- **Scope it to items that share their other signals:** every drink shares one bottle silhouette so all pairs must be far apart; two foods clash only if they also share a shape; two blocks only if they share a pattern.
- **Calibrate against the artifact.** Raising the same-silhouette threshold to 105 (to catch one bad pair at 88) flagged nine pairs that read clearly on the contact sheet. 80 matched judgement by eye; the one real pair was fixed by a different PATTERN, not a nudged hue. A guard that fires on things that look fine gets deleted. Extending the rule to tools by kind flagged a dozen swords, because tools share material colours on purpose; it was scoped to tools with an explicit sprite override, where the real case lived.
- Same reasoning applies to names, icons, sounds, silhouettes: ask what the player must do with it, then measure that.

#### 8.2 Past ~20 entries add a second signal keyed to behaviour, and guard the PAIR
- 62 enemies: 78 pairs closer than the just-noticeable threshold, two pairs byte-identical. Colour can't carry it (a hue wheel only holds so many, and theming *wants* related enemies in one colour).
- **Second signal = chest marking chosen by what the mob DOES:** charges up -> crosshair, dashes -> chevrons, leaves poison -> drips. It teaches the player something true (the marking predicts the attack). Size is a legitimate third signal.
- **Guard rule:** *no two entries of the same size may share BOTH a near-identical colour AND the same marking.* Guarding colour alone forces apart hues theming wants together; guarding markings alone forbids two dash enemies existing.
- Compare colours **as rendered** after any brightness floor (near-blacks that differ in the table lift to the same grey in game).
- Watch marking families collapsing too ("everything that freezes" = five enemies wearing one snowflake; subdivide into aura / lobbed / dropped / on-death).

#### 8.3 Derive the icon FROM the thing
- Spawn eggs hand-picked to be distinct hid that the mob **skins** underneath weren't (two gold humanoids 18 apart, two same-sized fire dogs 29 apart). Derive the egg from the skin (base = skin's main colour, overlay = the tier's pip colour) so the real clash shows up and the second colour is an independent axis.
- **Confusability needs all three factors together:** same archetype, within 1.35x scale, under 45 apart in colour. A scale-0.35 critter and a scale-1.3 one are different animals in the same brown.
- **Calibrate against vanilla, not tidiness:** vanilla eggs cluster worse (cow, horse, donkey, llama, mule are all brown; people use the tooltip). A rule stricter than vanilla pushes naturalistic colours towards neon for no gain.
- When a solver says the palette is full, check the claim: three "impossible" colours needed only 11-15 drift when solved against both constraints at once.

#### 8.4 Pairwise separation is not enough: measure the group's spread
- 63 tools, 4 kinds, 3 tiers: same kind+tier share silhouette and tier pip, so colour is the only signal: 26 pairs under 55 apart, four under 25. Fixing pairs wasn't enough: four pale warm shovels each 50 apart still read as four pale warm shovels because the group sat in one corner. Metric that matched the contact sheet:
```python
span = sum((max(c[i] for c in cs) - min(c[i] for c in cs))**2 for i in range(3))**0.5
need = 120 + 22 * (n - 2)      # a bigger family has to spread further
```
- Check an exemption is load-bearing before trusting it (a "material identity matters" exemption protected a principle the tables weren't following).
- A hypothesis (shovel pans look smaller, need more contrast) died on measurement: shovels had MORE blade pixels than swords (28.8 vs 22.7). Measure the cause before writing the cure.
- Build a resolver that re-solves a whole family at once, minimising drift, using a second colour channel where cheaper, re-checking every constraint after each move.

#### 8.5 When one axis runs out, add an axis; don't accept the colour
- 37 drinks, one bottle silhouette: best solver answer for "Soot Syrup" was **teal**. The tell is the absurd answer, not an error (bright purple for a scrap pile, lime for kelp, cyan for chalk).

| symptom | axis added |
|---|---|
| 10-11 tools per kind+tier, absurd hues | its own silhouette (18 now) |
| 4 blocks per pattern | move it to another pattern |
| 37 drinks, one bottle | four vessels: flask, jug, bottle, vial |
| foods | already grouped by shape (that was the model) |

- The separation floor then applies WITHIN a group ("all 15 flasks differ" instead of "all 37 differ").
- **Make the new axis mean something.** Balancing vessel counts for four rounds was the wrong question. Rule that stuck: every bad drink comes in a vial (vessel = a third tier signal beside the pip and name colour), others take the vessel their name implies (syrup in a jug, tonic in a flask).
- **When the group forces the art, move the NAME:** "Cinder Gravel" landed on the only pattern with colour room, which draws a woven mat; renaming it "Cinder Mat" made name and art agree.
- **Guard both directions:** a vessel requested but not drawn silently falls back to the default shape; a vessel drawn but unused is dead art.

#### 8.6 The newer thing yields
- A palette solver that minimises drift still must pick WHICH of a clashing pair moves. Picking the first id pushed a long-established pale-pewter pickaxe to magenta (drift 103) to admit a glass pickaxe added ten minutes earlier. Tie-break by table order so the newest row moves:
```python
order = {t["id"]: i for i, t in enumerate(rows)}
pick = min((a, b), key=lambda i: (moved.get(i, 0), -order[i]))
```
- Applies to any solver over a growing catalogue (colours, names, ids, key bindings, recipe grids). When the palette is genuinely full give the new thing a different SHAPE or PATTERN rather than pushing hues.

#### 8.7 Vary material, not numbers
- Six generated tracks varying only tempo/gain read as "all the same". Same trap for 200 items differing only in damage, mobs differing only in health, levels differing only in spawn rate.
- Change the material: different silhouette/kit, harmony, **form**, palette, a per-item **motif**, a wide range (some slow, some fast).
- **Mob audit (3 lines) that found six mechanically identical pairs at 29 mobs:**
```python
for a, b in itertools.combinations(MOBS, 2):
    if a["arch"] == b["arch"] and a["mood"] == b["mood"] and abs(a["hp"] - b["hp"]) <= 2 and a["dmg"] == b["dmg"]:
        print(a["name"], b["name"])
```
- Fix = give each something to DO (real behaviour components, section 2.10), not a nudged number; make the guard permanent, counting traits and speed as real differences and colour as none.
- Tool audit: group by (kind, tier); flag equal-damage pairs where NEITHER has a gimmick or speed quirk. Eight points of durability is not a difference.
- **The same-item-with-different-numbers check:** an item's profile is the SET of things it is good at, compared within kind and tier (not the numbers). Six pairs in a 146-tool catalogue were each other with smaller numbers (three axes quick at wool, two at plant, ...). Compare namespace-stripped ids (one tool named the TAG `web`, another the BLOCK `minecraft:web`). Print the free options before assigning: it revealed `metal` was used by nothing. Scope: foods vs drinks with the same effect are different acts; mobs compared within tier; decorative blocks that differ only in appearance are NOT duplicates (their job is looking different). A de-dup rule's scope is "what makes these two the same EXPERIENCE".
- **Applies to copy too:** three drinks whose blurbs all opened with the same seven words read as one item listed three times. Check that two descriptions don't share 24+ opening characters (overall similarity can't tell a template from a deliberately parallel pair). Read the whole set at once sometimes; batch-to-batch patterns are invisible from inside one batch.
- A blurb can promise something the field doesn't do (a "very quick" sword whose `speed` field fed mining speed). Read strings against what fields actually drive.
- **One shape, two things (art):** in a flat-vector comic, shells and round props read as people because the routine that drew them was basically the figure routine (a blob with smaller blobs). Give each class of thing one structural signature (a deer = long body on four thin legs with a wedge head; a hooded bird = hooded wedge with a beak; a villager = cloak triangle); props flatter, wider or more angular than anything alive. Check by rendering a prop beside a figure at panel size, since most lookalikes only collapse when small.

#### 8.8 Presentation carries meaning
- Message colour codes are data: a good tool announced success in `§c` red (reads as warning/damage), a deliberately useless tool said `§aSqueak.` in green, and programmer shorthand `thing(s)` reached the player. No test could fail on it.
- Guard: map each `tell()` to its handler and tier, compare the leading colour code (bad tools must not speak in green; good/mid must not report success in red; red is allowed for a refusal identified by failure words). Dump ALL runtime strings and read them in one place; one-at-a-time is how these got in. (Wider version: read every player-visible string as an assertion and test it.)

---

### 9. Drawn-extent, mirroring and looking (for programmatic art)

- **A drawing routine's nominal box is not its drawn extent.** A dragon specified in 200x140 had wings reaching 60 above and fire breath 108 past the right; placing by the box corner cropped wings off nearly every panel. Measure: draw each pose alone on an oversized transparent canvas, read `im.getbbox()`. Measure **per pose** (key the table on whatever changes the silhouette), **write the table back automatically** between `# <<< CALIBRATED` markers (temp file + `os.replace`), **clamp** placement to it (cap size and nudge the anchor inward, leaving a few px INSET, since flush to the edge puts pixels under the frame and reads as a clip), **anchor by the feet**, and treat a rider as part of the figure. One world scale beats per-panel fitting ("largest that fits" made a girl as tall as a house; one rule such as adult = 0.44 of panel height, everything derived).
- **Mirrored must be a mirror.** With `px(v) = cx + s*S(v)` (s = -1 flipped), points mirror correctly but `rect(px(-26), y, S(52), h)` still grows RIGHTWARDS from its corner: 49 such bugs across four characters, and only one had been spotted by eye. Points are flip-safe; spans are not. Fix: an `lx(a, b) = min(px(a), px(b))` helper for the left edge. Do NOT return `(x, width)` and splat it `*span(...)` into `rect(x, y, w, h)` (it shifted every later argument one place, silently, at 25 call sites). Test mechanically: draw each pose twice, mirror the flipped copy about the anchor, fail if more than 0.4% of pixels differ (took 37 broken to 0).
- **A sed-style fix fixes only the formatting it matched:** arms in the coat colour were invisible; the first pass matched only single-line call sites and 11 multi-line ones survived (figure looked armless for several builds). After a mechanical edit, grep for the ORIGINAL pattern again and expect zero hits, with a regex that tolerates line breaks.
- **Show geometry, not numbers.** For 3D/CAD-like work render a colour-coded, labelled PNG and open it. Bounding-box numbers meant nothing (negative coordinates read as "negative parts"); a render with each feature in its own colour settled in one reply what many rounds of words had not. A stdlib-only software rasteriser (orthographic projection + z-buffer + flat shading, PNG via `zlib` + `struct`) is enough. Colour-code the features under discussion and ask by colour. Render the *assembled* pose too. Save it somewhere the human will actually see; images in tool results may not be visible to them. If the human reports something the file doesn't contain, check the viewer.

---

### 10. Numbers, playability and test fixtures that affect mobs

- **10,500 passing assertions can describe an unplayable game.** Printing the numbers found: seconds to be defeated from contact 3.8 (casual, wave 0), 1.3 (normal, wave 10), 0.4 (hardest, wave 20); a late boss taking 131 s to defeat in a 20 s wave; a class whose whole design was "start with nothing" opening as joint strongest. Root cause: a growing quantity (enemy damage x mode x entropy x wave) aimed at a fixed one (player health). Write a readout script that prints curves for a person, compute damage per second from the *effective* rate (hits/s is capped by invulnerability frames; a boomerang hits twice; shotgun pellets all count), then freeze conclusions as assertions ("casual wave one gives at least five seconds", "no class opens at more than 3x another"), and assert any constant copied into the readout equals the code's.
- **One scaling axis per item:** items that scaled both cooldown and damage reached 36x at level 6 vs 6x for single-axis items, making every single-axis item worthless late. Damage-over-time pools compound with fire rate: scale the NUMBERS not the clock. A cooldown divided by level can fall under the engine minimum (0.25 s / 6 < 0.05 s) and fire 20x/second. Measure single-target and crowd separately (a piercing laser vs 16 packed dummies looks 16x better than it is) and put the lone dummy inside orbital range or orbitals measure as nothing.
- **Is the reward attached to what the game is about?** Coins came only from surviving the wave clock, so killing enemies paid nothing and running in circles equalled clearing the field; the first shop's cheapest item (30) cost more than the first income (28). Check total run income vs total catalogue price and whether the FIRST shop can sell anything.
- **A pack where nothing is dangerous:** every hostile mob was bad-tier and held feeble by the pack's own tier rules, and good mobs were forbidden to hunt players, so no mob could hurt anybody: add a mid-tier hostile band. Top-tier tools identical to netherite with lower durability made an ore chain pointless. "Must beat vanilla" must be scoped by what the thing is for (sword = damage, pickaxe = mining), and a gimmick alone isn't a defence.
- **A balance report should print** each item next to the vanilla ladder it competes with, hits-to-defeat both ways, a bar chart, tier/mood population counts, then a few FROZEN conclusions that exit non-zero (prove they can fail).
- **Shared test worlds are a budget.** One rich fixture (7x7 material patch, worn tool, crowd of mobs, water, lava) shared by 80+ behaviours broke another tool's test four times as blocks were added: crops above every neighbour left no soil with air above; a torch took the last free face; removing a water block left the only water at 0.87 blocks from a fixture mob vs a 0.8 safety radius; moving water put crops on the last plantable cell. A block has six faces and once handlers read neighbours they are a resource. **Write the allocation down in the fixture:**
```js
/* THE NEIGHBOUR BUDGET
   (1, Y, 0)   clear soil with AIR above  -> till, terrace, trench
   (-1, Y, 0)  lava                       -> quench
   (0, Y, -1)  water, away from the crowd -> diver, rime, drain
   (0, Y, 1)   torch                      -> snuff
   (0, Y+1, 0) air (or wood)              -> graft, lampwright
   A tool that needs a cell takes one from this list and says so here. */
```
When a sweep reports a regression in something you didn't touch, suspect the fixture before the code.

---

### 11. Quick checklists

**New mob, before you call it done**
1. BP identifier == RP client_entity identifier; every geometry/animation/texture/render-controller reference resolves (build guard).
2. Name lines in `.lang` for the entity and its egg; `languages.json` present.
3. Spawn rule exists (or the mob is intentionally creative-only) and its weight fits the per-pool budget.
4. AI: one `nearest_attackable_target` with all entity types; no `!=` family filters; `scan_interval: 1` if it must react instantly; no `behavior.panic` unless you want fleeing on any hit; shooter intervals in seconds; player-targeting scoped by tag if others may be nearby.
5. Animation: your own bones + your own animation; every animation bone exists in the geometry; stride multiplied by `query.modified_move_speed`; idle term from `query.anim_time`.
6. UV: per-face UV if the model is resizable; bounds check on the sheet; only IDENTICAL rectangles may overlap; marks painted on the front patch; face drawn after marks.
7. Hitbox derived from geometry (height ~ drawn height, width narrower, floor at vanilla-smallest).
8. Render a front-on composite of the skin and open it.
9. Anything the mob places is air-only, capped, and removed afterwards; projectiles are not in the `mob` family and expire on every path.

**New catalogue entry, before you call it done**
1. Does it differ from its nearest neighbour in colour by a measured distance, on the real background, AND in something the player experiences (behaviour, set of strengths, silhouette)?
2. Is there a second signal beyond colour (silhouette, marking, vessel, pattern)? If the solver answers with an absurd colour, add an axis.
3. Does every sprite pass through the shared outline/finish path?
4. Does the name agree with the art? Does the blurb agree with what the fields do? Do the message colours agree with the tier?


---

## Part 3 — Script logic bug classes

Every class below shipped in a real add-on, and most passed a green test suite first. The common thread: the code runs, and nothing throws, but the feature quietly does nothing, does it twice, or does it to the wrong thing. Project names are left out on purpose. Numbers (caps, tick budgets, counts) come from the incidents they were found in.

### 3.1 State and ownership

#### Stale state across an `await`

**Symptom:** A player's coins or items silently revert to an earlier value after a menu closes.

**Cause:** Any `await` on a form (`show`, `pick`, `confirm`) can last **minutes**, because the player is staring at a dialog. State loaded before the await is a snapshot from another era. Background writers (a 30-second economy tick, mail arriving, another player buying from your shop) write the same profile in the meantime, and saving the old snapshot erases their work.

```js
const d = load(player);              // coins: 100
if (!await confirm(...)) return;     // meanwhile the 30s economy tick pays +30
d.coins -= price;                    // 100 - 20
save(player, d);                     // writes 80, the +30 is gone
```

**Fix:** Load for display before the await if you like, but call `load()` again **after the last await** and mutate that fresh copy. For shared world lists (sites, feeds, likes), re-find the record by id rather than reusing the object reference, because the array you loaded may no longer be the array on disk.

**Same shape elsewhere:** a test file that captured a config alias before a module-graph reload kept pointing at the old module's arrays. `.includes(item)` is reference equality, so it matched nothing, `cat` was `undefined`, and the restriction check short-circuited to "allowed". Re-fetch data from the live module after any reload, and match by a stable field (`item.id`), not by object identity. A restriction that is skipped and one that correctly allows look identical from outside, so the test needs a case where the restriction must fire for a reason nothing else would also produce.

#### A stale timer releases a lock it no longer owns

**Symptom:** Menus flicker or double-open, but only after the world has been running for about five minutes.

**Cause:** Each menu chain did `MENU_BUSY.add(id)` and scheduled `system.runTimeout(() => MENU_BUSY.delete(id), 6000)` as a safety release. Those timeouts keep firing long after their own chain ended. Once the world was older than the timeout, an OLD release deleted the lock belonging to the chain in use right then. `playerInteractWithBlock` fires every tick while the button is held, so each hole in the lock started a fresh chain.

**Fix:** Store a token, and let a release act only if it still owns the lock.

```js
const token = ++menuToken;
MENU_BUSY.set(p.id, token);
const done = () => { if (MENU_BUSY.get(p.id) === token) MENU_BUSY.delete(p.id); };
```

**Rule:** any release scheduled on a timer must prove it still owns what it is releasing. A boolean shared between generations is always a race.

**Testing:** a harness that runs every `runTimeout` immediately can never reproduce "an old timer fires during a NEW chain". Park long timers (over 100 ticks) in a list and fire them at a chosen moment. A symptom that appears only after the world has been up a while points at a timer, a growing collection or a wraparound, not at the feature.

#### A shared effect needs one owner

**Symptom:** Freezing an enemy mid-charge silently does nothing for the rest of the freeze.

**Cause:** Bedrock has no runtime setter for `minecraft:movement`, so everything that wants to slow an entity uses the slowness effect. That makes slowness one slot with several writers, each with its own boolean (`rec.slowed`, `rec.held`):

1. A charge-up hold adds slowness 6 and sets `held`.
2. A freeze handler sees `!slowed` and adds slowness 2, **overwriting** the hold, because `addEffect` replaces rather than stacks.
3. The charge finishes and its `removeEffect` takes the freeze away too.
4. `slowed` is still `true`, so the freeze handler thinks it has already applied and never puts it back.

**Fix:** Each system records only its INTENT (`rec.holding = true`). One function computes the strongest reason the entity has to be slow, compares it to one remembered level, and applies or removes once. Booleans per feature cannot express "two reasons at once"; a level can.

**Check:** any time a feature calls `addEffect` or `removeEffect`, ask who else writes that same effect id. If more than one, add an owner before adding the second. In this incident, an exported duplicate of the freeze handler that nothing called was how the third owner crept in.

**Testing note:** to prove the fix, restore the desynced flags in the injected break. Merely returning early from the fixed function left the freeze working, so the injection proved nothing.

#### Two answers to one question

**Symptom:** One pulse counts as two, or a printed number disagrees with the real one.

**Cause:** Two places each work out the same fact with their own copy of the rules.
- Redstone-style add-on: `channelOn()` and `markEdge()` each computed a node's output channel. `channelOn` had special cases for stepping blocks, `markEdge` did not. A listener block stamped a rising edge on the channel it listens to, so one flick read as two edges, and Round Robin skipped every other output while Queue, Repeat and Chance counted one pulse twice. Fix: one shared `outChannels(rec)`.
- Prose is a second answer too. The count of things that could be wrong went from 8 to 12, and three places still said "eight" (an in-game line, two module comments, a renderer caption). Fix: count the list (`` `Only the ${CHANGES.length} things I built count.` ``) and add a mutation that hard-codes the number back.
- Version numbers: a pack whose display name counted up to V61 while `manifest.header.version` stayed `[1, 0, 0]`. The visible number moved every build and the number the game compares never did, so the game could keep the old pack on update ("my change did nothing"). Derive the manifest version from the same counter that names the pack, in the generator that writes the manifest.
- Rules disagreeing: a "no free rerolls" option changed the price function, but the reroll routine still branched on `freeRerolls > 0` before checking the price, so only the label moved.

**Rule:** anywhere a sentence, caption, README table or comment states a number or a name that a list already knows, that sentence is a second answer waiting to drift. Derive, do not retype.

**Open item from the source:** a `WIRED` table in the redstone-style add-on was noted as missing nine listener types, so those still report themselves as driving the channel they listen to. That was recorded as not chased down. Treat as unverified.

#### A record outlives the block it describes

**Symptom:** A shop keeps selling from chests whose drone port was blown up. Or a capped list fills with dead entries and crowds out real ones.

**Cause:** A feature remembers a placed block in a dynamic property. The record is removed only when the block is broken **by hand**, because `playerBreakBlock` is the only hook that fires. TNT, creepers, pistons, water and `/fill` all leave the record with no block under it.

**Fix:** Treat the block as the source of truth. Before acting on a record, `getBlock()` and confirm the `typeId` still matches. Distinguish two failures: `undefined` means the chunk is not loaded (keep the record, do nothing); a real block of the wrong type means it is gone (drop the record). Reap confirmed-dead records on an existing slow tick.

**In tests:** a fake `getBlock` that returns `undefined` for empty space models "chunk unloaded", not "air", and hides this bug. A precondition that checked a flag stored on the player object (`isInside(player)`) still passed after the fake world was wiped. Preconditions and guards must check the WORLD (is the floor block still there), not a record that outlives it.

#### Deferred work outlives its authority

**Symptom:** Something happens long after the thing that permitted it is gone.

**Cause:** Queued work holds a reference to state that has since changed. Four instances from one add-on:

1. **The tool broke mid-job.** Area mining ran `/loot ... mine` on the player, so drops depend on the MAINHAND when the job runs, not when it was queued. After the pickaxe broke, the queue kept going bare-handed. `setblock` removed the blocks and nothing dropped. Fix: a broken tool cancels that player's outstanding work and says how much was dropped.
2. **The player changed dimension.** `dim.getBlock` checked the dimension the job was queued in, but `player.runCommand` runs wherever the player is NOW. Walking through a portal with work outstanding meant 26 `setblock ... air` commands landing at the same coordinates in the Nether. Since commands must run on the player (so Fortune and Silk Touch resolve), refuse the job: compare `player.dimension.id` with the job's, skip if different, and drop the rest of that player's queue.
3. **A location without its world.** A waystone stored `"x y z"` and no dimension, so two waystones at the same coordinates in different dimensions were "the same one". Coordinates are only half an address.
4. **A deadline against a clock that restarts.** A cooldown stored `currentTick + 100` on a persistent mob. `system.currentTick` counts from world load but the dynamic property survives the reload, so a sheep sheared at tick 50,000 was un-shearable for forty minutes of the next session. A stored deadline further out than the cooldown itself is impossible, so treat it as stale.

**Rule:** for any queue, timer or batch, ask what the work DEPENDS on (tool held, player present, block still there, permission, dimension, session) and cancel on the event that invalidates it, and say so. When work runs *through* an actor, every implicit property of that actor is part of the job's preconditions. A persistent record compared against a session-scoped clock is always wrong eventually.

#### "Pay once" rewards keyed to the wrong thing

**Symptom:** A one-time reward pays 4 times in one play session, or pays again after every reload.

**Cause:** A shelter bonus was tracked in a claimed-set keyed by `floor(playerX/12), floor(playerZ/12)`: the player's position, not the shelter's. The detector scans ±6 blocks, so walking around one base entered four cells that each saw the same shelter. The set also lived only in memory, so a reload re-paid.

**Fix:**
- Key "already rewarded" by the **thing** (store positions, use a distance radius larger than the detector's scan; 48 blocks there).
- **Persist** it (world dynamic property), capped.
- Skip checks for a few seconds after a player joins; chunks are still loading.
- When a repeat-reward report comes in, ask "standing still or moving?" first. It splits the two causes immediately.

#### A world-singleton is really one per place

**Symptom:** The second player gets a permanent "It did not open. Try again." from a once-per-world structure.

**Cause:** One room was saved per world, built in the sky above whoever used it first. A player 3,000 blocks away was handed that saved site, whose chunks were not loaded, so the build half-failed, the landing-safety check said no, and every attempt failed. Every test passed because every test used one player standing in one place.

**Fix:** Before reusing a saved world-singleton site, check it is both loaded and within a sane distance of the person about to use it; otherwise rebuild where they are. Test with a SECOND player thousands of blocks away and the fake's `loaded()` restricted to their chunks, then with the player who logged out inside the old one.

#### A sentinel must be unrepresentable

**Symptom:** A cooldown or "never happened" state misbehaves at tick 0, on a fresh world or after a reload.

**Cause:** Using a number for "no value". Three features, three passes:
- A run counter stored `0` for "never swung" and tested `last > 0`. `system.currentTick` really starts at 0, so a run begun in the first seconds after load could never start.
- `-1` for "never swept", then `Number.isFinite(quiet)` before a cooldown. `-1` is finite, so at tick 0 the cooldown branch fired and the tool did nothing.
- The sheep cooldown above.

**Fix:** Use `null`/`undefined`, which cannot be confused with a reading.

```js
let quiet = null;                       // not -1
const raw = player.getDynamicProperty("ns:stow_quiet");
if (raw !== undefined) {
  const n = Number(raw);
  if (Number.isFinite(n)) quiet = n;    // a real reading, or nothing
}
if (quiet !== null && now - quiet < COOLDOWN) return;
```

**Spot it:** any `if (x > 0)` or `Number.isFinite(x)` guarding something read from storage. Ask what the value means at tick zero, on a fresh world and after a reload.

### 3.2 Transfers, kits and phase changes

#### Take-then-give must roll back

**Symptom:** Items or health vanish when a purchase or transfer fails halfway.

**Cause:** A transfer has two halves and the second can fail (full inventory, bad id, player left). Refunding only the currency destroys the items.

```js
if (!takeStock(seller, id, n)) return;   // items leave the chest
charge(buyer, price);
if (!give(buyer, id, n)) {
  refund(buyer, price);                  // money is fine...
  return;                                // ...and n items just ceased to exist
}
```

Taking first is correct (giving first allows double-spend), so the take must be undoable. Write the paired `putStock`/`restock` before shipping the sale path and call it on every failure branch after the take. Say "returned to the shop" only when it is true. Test by making the give fail on purpose.

**The price is not always money.** An item read "Set your HP to 1 and gain a random Legendary item" and did `setHp(1); grantRandom("legendary"); removeItem(...)`. `grantRandom` returns null when every Legendary is owned, so late in a run the player was left on 1 HP with the item gone and nothing given. Charge after you know you can deliver:

```js
onBuy: (c) => {
  if (!c.grantRandom("legendary")) return;   // nothing to give: nothing is taken
  c.setHp(1);
  c.removeItem("space_sandwich");
}
```

An empty pool must SAY so, otherwise "you own them all" and "this item is broken" look the same. Near-miss from the same fix: inferring "is this a reroll?" from whether a shop object already existed would have leaked a bonus item into every shop. Pass intent as an argument; do not deduce it from incidental state.

#### "No empty slots" is not "no room"

**Symptom:** A hopper-style block stops dead while the target chest is visibly not full.

**Cause:** `if (to.emptySlotsCount <= 0) return false;` reads like "the chest is full", but 27 slots each holding half a stack of stone is zero empty slots and roughly 1,700 stone of room.

**Fix:** The only honest test of "no room" is to try. `addItem` puts in what fits and returns the remainder:

```js
const sending = item.amount;
const over = to.addItem(item);
if (over && over.amount >= sending) return false;   // truly nothing fitted
from.setItem(i, over ?? undefined);                 // keep what came back
```

Both halves matter: dropping the `over.amount >= sending` test claims moves that never happened; using `setItem(i, undefined)` instead of `over ?? undefined` **destroys items** (a stack of 64 snowballs never fits one slot, since they cap at 16). Also watch `emptySlotsCount` on a player's inventory before handing something over. Found by mutation testing: flipping `<=` to `<` left the whole suite green, meaning no test held that line.

#### A second way in hands out the kit twice

**Symptom:** Rejoining mid-game doubles a player's starting kit (32 stone became 64).

**Cause:** `giveKit()` assumed an empty inventory. Two of three callers guaranteed it (game start with the inventory stashed, and after a death that clears everything). Rejoining mid-game marked the player dead so a respawn timer ran, the respawn called `giveKit`, and nothing had been cleared because nothing died.

**Fix:** For any function that issues items, list every path that reaches it and what the inventory holds on each. Clear on the rejoin path (keep currency, and only clear when the real inventory is safe in a vault; never clear the only copy). Test by driving the odd path and asserting the COUNT, not existence ("has stone" passes on 64).

#### A phase change needs a teardown

**Symptom:** The shop opens while the last round's fire is still burning, a rocket is still incoming, and acid sits on the floor. Players are defeated in a menu by something that no longer exists.

**Cause:** Ending a wave removed the enemies but not the burn and poison pools on the player, enemy bullets in the air, ground fields and puddles, or a purge timer that then ate most of the next wave. Each piece was individually correct; the bug was the absence of a teardown, visible only by reading the whole tick end to end after eleven targeted passes had found 39 other bugs.

**Fix:** Every phase transition gets one explicit function that empties everything the phase accumulated, listing the fields by name. The same read found a guard testing `run.phase === "dead"` immediately after a status tick that nothing above ever sets, so it could never fire. A condition that cannot be true is the quiet cousin of a field nothing reads.

**Habit:** when targeted passes stop paying, read the main loop top to bottom and ask what survives each boundary.

#### Record "the event arrived" before any branch that returns

**Symptom:** A fallback (click-to-fire) never retires, so every later action runs twice, but only for some classes.

**Cause:** The engine may or may not deliver `itemStartUse`, so the add-on sets `holdToFireWorks = true` on the first one and retires the fallback. The ability branch ran first and returned before that line:

```js
const def = weaponDef(key);
if (def?.kind === "ability") { useAbility(...); return; }   // returns first
holdToFireWorks = true;
```

Three of eleven classes, only when the first click was their ability, left the fallback armed for the rest of the run.

**Rule:** a flag that records *that an event arrived* goes on the first line of the handler. A flag recording *what the event did* belongs with the branch. Two probe lessons: fire each weapon as the class that owns it, and make sure a "did anything change?" snapshot includes every field the thing can touch.

### 3.3 Timing, ticks and queues

#### A tick's writes vanish unless the tick ends dirty

**Symptom:** A real bug is "unreachable" in tests, and its injection comes back MISSED.

**Cause:** When the store re-parses its saved string on every read, every tick works on fresh objects. Anything the tick writes onto a record is discarded unless the tick ends by saving, which it does only when something changed (`dirty`). A test that pokes the world from outside (`setPower`, `configure`) saves on its own path and leaves the tick with nothing to keep.

**Fix / check:** To exercise anything a tick writes, put something in the world that changes itself *inside* the tick (a blinker or timer node, a sensor, a countdown). A retry after a skipped removal pass likewise needs `touched = true` set explicitly. If an injection comes back MISSED, ask whether the state ever got saved before concluding the code is safe.

#### Repeats inside one tick collapse

**Symptom:** A "hits twice" attack behaves exactly like one hit.

**Cause:** A loop applied knockback N times in one tick, all in the same direction. `applyKnockback` **sets** velocity rather than adding, so only the last call is felt. The fake recorded every call, so a total-count assertion passed with the bug put back.

```js
for (let i = 0; i < (s.pushes ?? 1); i++) {
  ctx.player.applyKnockback({ x: d.x * 2.2, z: d.z * 2.2 }, 0.5);
}
```

**Fix:** Spend one push per engine step (which also reads as a sustained gust). Test **per step**: snapshot the count before and after each step and assert no single step produced more than one. When a fake cannot represent the failure, move the assertion to a dimension it can (here, time). Applies to any set-not-add API (velocity, health, a timer, an effect duration).

**Same routine, second fault:** the attack reached 18 blocks instantly with no particle, sound or wind-up and could shove a player out of a boundary ring unannounced. Every ranged attack without a projectile in flight needs a visible wind-up; a bullet's travel time is itself the tell.

#### `runInterval` counts passes, not ticks

**Symptom:** An effect meant to fire "every ~4.5 seconds" fires every 45.

**Cause:** With `system.runInterval(fn, 10)`, the counter inside counts PASSES. `beat - last > 90` is 90 passes = 900 ticks = 45 seconds.

**Fix:** Name the unit, define the period as a constant used by both the gate and the `runInterval` call, state in a comment how many ticks a beat is, and assert in the sim that the effect fires inside a realistic window.

#### Self-spawning powers recurse exponentially

**Symptom:** Heap exhaustion in the simulator in about 30 simulated seconds (would have wrecked a real world).

**Cause:** A `calls_friends` power spawned 2 copies on spotting a player, and each copy also had `calls_friends`. The "only once" guard was per-entity, which lets each new entity fire once too.

**Fix:** Tag anything a script spawns (`friend.addTag("summoned")`) and skip the spawning power for tagged entities. Add a crowd cap counted from nearby mobs as a second limit. Same for on-death splitters (tag the halves). Make the test harness's `spawnEntity` **throw** past a few hundred entities, so a runaway reports as a failure.

#### One shared queue starves everyone else

**Symptom:** After one player fells a huge tree, every other player's area tools "quietly do nothing" for up to 30 seconds.

**Cause:** One global FIFO array capped at 600 jobs, 20 blocks a tick. One big job fills the queue, `enqueue` starts returning false for everyone else, and FIFO means the first player's 600 blocks are all taken before anyone else's first. 350 single-player checks could not see it.

**Fix (keep what was right, change what was not):**
- a queue PER PLAYER with a per-player ceiling (200) and the global cap (600) as a backstop;
- still ONE interval (an interval per player is how a server starts stuttering) and still 20 blocks a tick in total, since that budget is about frame time;
- share the tick's budget round-robin, moving the front of the rotation to the back each pass;
- a throttled message when work is refused;
- drop the player's queue on `playerLeave`, or the maps grow one entry per player who ever used the tool.

**Rule:** any shared, capped resource in a multiplayer feature (work queue, cooldown pool, rate limit, entity budget) needs a per-actor share. Test with TWO actors; a single-actor test cannot tell "fair" from "first come, first served".

#### Removal needs the same budget as building

**Symptom:** A structure built at 256 blocks per tick was torn down 681-in-one-tick inside an `itemUse` handler, nearly three times the project's own budget, in the tick the player is waiting on.

**Fix:** Slice removal at the same per-tick constant as building. Test: measure one slice's writes and assert `<= PER_TICK`, assert the first slice does NOT finish, then pump and assert it does. Also guard with a `siteLoaded()` check first; it changes no outcome (reads return undefined anyway) but saves about 1,100 wasted `getBlock` calls. Assert on the count or the guard is untestable. Remove a block only if it is one of the ids *that cell* is allowed to hold (see 3.6).

#### A slow tick fails nothing

**Symptom:** The game stutters late in a run; every assertion is green.

**Cause:** Nothing had ever asked how LONG a tick takes. Worst realistic load (40 enemies, late wave, every heavy shooting item at max level): 561 shots in flight and 1.32 ms per engine step in Node. Two causes: the projectile list was uncapped (every shot scans the field each substep, so cost is quadratic in how well the player is doing) and a snapshot array was rebuilt inside the hot loop, per projectile per substep.

**Fix:** Cap the list and drop the OLDEST (nearest to expiring): 400 shots, 0.61 ms. Build the snapshot once per step and pass it down. Bedrock's script engine is several times slower than Node, so a millisecond in Node is already trouble.

**Testing:** write the probe as an assertion (`ok(ms < 1.0)`), not a benchmark. Print the conditions next to the number; one probe ran through a 20-second wave that ended partway, and only the "0 droids" in the summary line gave it away.

#### A hard wall-clock ceiling catches the host, not bugs

**Symptom:** A "no tick over N ms" assertion across ~80,000 synchronous ticks failed at 38 to 1,300 ms, in clusters, at a different place each run.

**Cause:** First blamed on garbage collection (disproved with `node --trace-gc`: every pause under 4 ms). The OS was descheduling the process for tens to hundreds of ms while the game and a browser ran. Wall-clock timing on a shared machine measures the machine.

**Fix:** A real regression is SUSTAINED, so assert on counts: slow ticks as a small fraction of the run, and "huge" ticks (over 250 ms) a handful at most. After every over-budget tick, time a fixed CONTROL (pure arithmetic, about 0.5 ms); if the control is slow too, the machine stalled, not the add-on. Prove it still bites: injecting a 25 ms spin every 20th tick turned both count checks red (3,440 slow ticks against a budget of 150). Check what else is running before theorising about the code.

#### A fake clock reset must rewind everything scheduled on it

**Symptom:** Six assertions fail with "0 launches" for a block whose code is correct.

**Cause:** The harness `reset()` set `sim.tick = 0` but left each interval's `next` deadline (around 180 by then) alone, so no interval fired again for the rest of the run. The failure points at the feature; the bug was the harness.

```js
for (const iv of system._intervals) iv.next = sim.tick + iv.ticks;
```

**Rule:** a `reset()` must leave the fake in a state the real system could be in. The same applies to ids: every test player was `p_Tester`, so a per-player cooldown map (correctly keyed on `player.id`) leaked into the next test. Hand out unique ids. When several tests fail at once on a feature you just wrote, suspect the shared fixture first.

#### Right about what, wrong about how often

**Symptom:** The game is correct but nags: the same line repeats every 2 to 3 seconds, or 25 times in 25 ticks.

**Instances:** a threshold line repeated on laps 5, 6, 7 and 8 (said whenever the condition held, not when it changed); a fix comparing only to the PREVIOUS line let a line return when another came between; "Someone else is in there" every two seconds from a walk-in loop; the same line every three seconds to a sleeping player who cannot act on it; "It does not come apart" 25 times while holding the break button.

**Fix:** Keep the ACTION unconditional and throttle only the TELLING. Key the throttle by what happened (or one message swallows a different one that matters). Track each line as a set, once per visit. Say nothing to someone who cannot act on it. Test by counting occurrences over time (`bar.filter(m => m.includes(...)).length <= 1`), not by checking one appeared.

### 3.4 Config with no other end

A config-driven system has three ends to a wire: something writes the field, something reads it, and something ties the two together. Most "the feature silently isn't there" bugs are a missing end.

#### Fields nothing reads

**Symptom:** A class, item or enemy ships with a texture, icon and description and does nothing. Or its one defining trick is missing.

**Instances:**
- A class core set `clone: true` and `cloneDamage: 0.5`. Nothing read either.
- An ability wrote `run.imagesRecalled`, read by nobody: a ten-second cooldown that printed a line.
- Options-bag typos: `explode: 3` where the projectile code reads `splash`, and `homingPlayer: true` where it reads `homing`. A rocket that did not explode and a homing bolt that flew straight.
- Party Hat wrote a field for a mechanic ("fiestas") the add-on never built; Magnet Coil promised coin collection in a game with no coin pickups.
- **A drawback item became a pure buff:** a "consumes double the stamina" item wrote `staminaCost *= 2`, but there is no stamina meter, so it gave a free 150% dash speed.
- Two functions disagreeing about a rule: the price changed, the branch on `freeRerolls > 0` did not (see two answers, 3.1).

**Audit:** grep every field for readers; anything reading `0` is a feature that is not there.

```bash
for f in clone cloneDamage images swordsPerWave; do
  printf "%-16s %s reader(s)\n" "$f" \
    "$(grep -rn "\.$f\b" src/*.js | grep -v 'blank\|apply:' | grep -c .)"
done
```

Pitfalls: a hand grep like `^\s*(\w+):` misses fields sharing a line with another. Use a guard that reads the real `Object.keys(blankStats())`. Also guard that no item's text promises a mechanic the port lacks. Assertion totals are not a coverage signal; record assertions per check and fail any check that asserted zero.

**Related fault, gating on the class instead of the stat:** `if (run.classKey !== "blademaster") return []` meant that a core stolen by another class did nothing. Gate on the stat the core writes (`if (!(run.core.swordsPerWave > 0)) return []`) so the feature travels with whatever grants it.

**Best guard:** behavioural, one probe per class that grants the core and asserts something observable changes, failing loudly when a new class has no probe. Prove the probe can fail: one clone probe passed with the fix reverted because the test enemy sat inside melee reach anyway.

#### Fields nothing writes

**Symptom:** A whole build path is unreachable. A melee multiplier was applied by four damage paths and raised by none of 97 items, so melee builds drafted from a smaller pool of scaling items with no visible reason.

**Why harder to see:** a dead write drags a lying description along; a dead read changes no text, throws nothing and reads as ordinary code.

**Check behaviourally, not by grep:** a regex reported four dead reads and three were false positives (writes like `s.tagMult.shots = ...`). Run every item's `mods()` and every class core's `apply()` against a fresh stat block and diff.

| guard | catches |
|---|---|
| every declared field has a reader | an item that does nothing; a drawback that is not real |
| every declared field has a writer | a feature the engine supports that no content reaches |

#### Exports with no consumer

An `export const ALL = { ...HIT, ...DIG }` shipped to players and was imported by nothing and named by no test. It read like a feature, and the import-graph check only proves each FILE runs. The guard refuses any `export` that no other pack script imports and the simulator never names. Test-only exports are legitimate (a `queued()` accessor exists for the simulator alone). Dropping the keyword from four module-internal exports made the truly dead one visible. Also relevant: a module that nothing imports never runs (walk the import graph from the entry file).

#### An opt-out flag needs every consumer

**Symptom:** Adding a `creative_only` flag to one gun (no recipe) produced raw `KeyError` tracebacks in four places that all assumed `g["pattern"]` existed (recipe writer, two helpers, balance guards). The crash also hid the guard that would have named the real problem.

**Fix:**
- grep for every read of the field the flag makes optional; keep one central list (`CRAFTABLE_GUNS = [g for g in GUNS if not g.get("creative_only")]`);
- consumers **skip** a malformed entry so validation can report it in words;
- prove the flag both ways: give the exempt entry recipe data (guard must fire) and remove the flag (guard must fire);
- assert the exemption itself (the creative-only gun exists AND has no parts) instead of skipping the id.

#### A borrowed feature carries the owner's baseline

**Symptom:** A "steal another class's ability" reward pays exactly zero for the thief.

**Cause:** One class's core paid 1% more damage per HP **above 100**. The owner had 120 HP so it always paid. The thief had exactly 100, so it paid zero, and because he never stole the same core twice, taking it burned a boss reward on a guaranteed nothing. Bookkeeping (one per boss, never twice) was thoroughly tested; whether the borrowed core *did* anything was not.

**Check:** any "X gains Y's ability" feature needs testing on X, not on Y. Reuse the per-owner probes on the borrower (ten of eleven cores passed; the failure was the bug). Two traps: the fix must not become an exemption list (the test asserts the core comes back once the condition is met), and an absolute threshold in the probe (`damageMult > 1.001`) silently encodes the owner's baseline. Compare the run against itself with the condition off.

#### A table's value shape is its contract

**Symptom:** A new handler yields nothing, and it looks exactly like the legitimate "nothing to do" path.

**Cause:** New code read an ore table as `id -> item id`; it is `id -> [item, count]`. `ORE_DROPS[typeId] ?? DROPS_AS[typeId] ?? typeId` returned the ARRAY. JavaScript compared an array to a string without complaint, existing readers either indexed `[0]` or only tested key existence, and the guard "every field has a reader" was satisfied since the field WAS read, just wrongly. Four wrong hypotheses were spent before printing the value from inside the handler.

**Fix:** print one entry before reading an unfamiliar shared table; read tuple-valued tables through one accessor (`oreDrop(id)` returning `{item, count}`); write the test that asserts the POSITIVE outcome (the chest contains one raw iron), not just the absence of an error.

#### The lookup list outgrew the reader

**Symptom:** A promise ("It starts where you left it.") silently breaks for the players who got furthest.

**Cause:** A saved id was written from the long list (`ALL_CHANGES`) and read back with `CHANGES.find(...)`, the short list. Deep-tier ids exist only in the long list, so `find` returned `undefined` and a `?? pickWrong(0)` fallback covered it.

**Rule:** when a list gets a second tier, grep every reader of ids from it and make each search the superset. Drive the deep tier specifically in a test; the shallow test passes either way.

#### Two settings, each clamped, whose SUM indexes something

**Symptom:** A stepping block on channel 6 with 8 steps drove channels 6, 7, 8 and then 9 to 13, which no block can listen to. Five of eight steps did nothing; the only symptom was a chase of lights dying halfway round.

**Fix:** Wrap, do not clamp: `((base - 1 + step) % 8) + 1`. Anything that already fitted is unchanged (6 with 3 steps is still 6, 7, 8); clamping the step count would give fewer steps than the menu offered. The off-by-one `(base + step) % 8 + 1` looks right and shifts every existing setup by one; injection-test both the wrong wrap and no wrap.

### 3.5 Dispatch and names

#### The string that joins them

In data-driven formats the halves usually exist and the join fails: a name matched at runtime, missing, with no error. Thirteen bugs in one add-on had this shape:

| the join | what a miss looks like in game |
|---|---|
| tool `fx` to handler | the gimmick never happens |
| handler to its dispatch MAP | fires on the wrong action, reads undefined |
| animation to bone name | that limb never moves |
| entity to sound event | the mob is mute |
| `minecraft:icon` to texture shortname | blank square item |
| texture shortname to PNG file | invisible item or block |
| `minecraft:loot` to path | drops nothing, looks like bad luck |
| ore feature to feature rule | the ore generates nowhere |
| recipe pattern letter to key | the engine refuses the recipe |
| block `sound` to category | falls back silently |
| families filter to declared family | matches nothing, forever |
| mob `mark` to painter branch | a blank mob |
| block `pattern` to painter branch | the WRONG thing, plausibly |

**The check is always the same two lines:** collect the names one side OFFERS, collect the names the other side ASKS FOR, and diff both ways. Missing is a failure; unused is usually a note (dead content) but occasionally the real bug. Write the diff in the same commit as any new wiring.

#### If/elif chain with no else

**Symptom:** A typo'd name (`"chevrons"` for `"chevron"`) paints a plain body with no error, and a "no two share a marking" check is happy because every typo is unique.

**Fix:** derive the valid set from the chain's own source and check every table row against it:

```python
def mark_kinds():
    import inspect, re
    return frozenset(re.findall(r'kind == "(\w+)"', inspect.getsource(_mark)))
```

A hand-written list next to the chain is another second answer. Keep generation tolerant so the BUILD GUARD reports with a message naming the mob, rather than a traceback far from the cause. **Worse:** a chain ending in a real default (`else: # speckle`) ships a plausible wrong result, and a separation rule then compares the colour of what was *claimed*, not what was rendered. A silent fallback to a plausible result is harder to spot than a silent fallback to nothing. Red-team with a near-miss spelling.

#### The validity list and the dispatch list

**Symptom:** Six new markings passed ~110 guards, a 534-check simulation and a 106-case red team, and shipped as flat single-colour bodies. Found by rendering the mobs.

**Cause:** The legal names were derived from both painter functions' source, but the delegation from the first painter into the second was a hand-written tuple. A name can be VALID and UNREACHABLE. Deriving validity is not deriving reachability.

**Fixes, in order of value:**
1. Delete the list. The delegation is a plain `else:` at the end of the chain.
2. Make the end of the road loud: the final `else` records the name and paints magenta hatching, which nobody ships by accident. Painting nothing is indistinguishable from a feature not existing.
3. Prove the painter paints: run every legal name through the real painter on a flat patch and require pixels to change. "A branch exists" and "a branch does something" are different claims.

Harness notes: report through the normal channel rather than `raise` (a red-team harness driving one build per injected break cannot read anything from a process that died), and have `restore()` clear any module-level accumulator.

#### A handler in the wrong dispatch map

**Symptom:** A "take wool off a sheep" effect never fires, in silence, forever.

**Cause:** Tool effects live in two maps with different payloads: `HIT` (from `entityHitEntity`, context `{player, target, dim, def}`) and `DIG` (from `playerBreakBlock`, context `{player, dim, x, y, z, typeId, def}`). The handler was inserted into DIG because the insertion anchor happened to be a dig handler. "Every fx has a handler" and "every fx is in exactly one map" were both satisfied; the effect fired on block breaks and read `ctx.target`, which is undefined there.

**Guard:** read the BUILT script, split the two map literals, and compare the context fields each handler destructures against the fields its map provides (a DIG handler touching `target`, or a HIT handler touching `typeId`/`x`/`y`/`z`, is in the wrong map). A guard about what ships must read what ships; the first version read source while the injection edited the built copy. The same mistake happened one pass later and the build refused it by name: the insertion anchor decides which map you land in, and two map literals in one file look identical from "insert before this function". "Is it registered?" and "is it registered in the right place?" are different questions.

### 3.6 Safety of building and destructive features

#### Never suffocate players or eat builds

Two hard rules for any block-placing feature:
1. **Only fill `b.isAir`.** Never overwrite an existing block.
2. **Build a blocked-coordinate set from every nearby player first** (feet plus headroom plus a block of margin) and skip those positions, before the placement loop.

Also budget the loop (one terrain feature capped at 700 blocks; a full chunk of open air would otherwise place thousands in one tick) and scale by level (8x8 at I, 12x12 at II, 16x16 at III, mounded not flat).

**Teleports:** `entity.teleport()` does NOT check for solid blocks, so blind random hops drop the player into rock. Use a `safeTeleport()` that scans upward from the requested spot for a two-block air gap, then a little below, and **refuses to move the player at all** if none is found. Staying put beats suffocating. This includes destinations that were safe when written down (typed portal coordinates, a pad someone has since paved over, a bed spawn walled in months later). Shoves (`applyKnockback`) are exempt; the game's collision stops them at a wall.

**Spawning:** persistent mob spawns need a crowd ceiling (`crowded(entity, radius, max)`) or holding right-click fills the chunk (one summon was 18 permanent bees per click). Particles, projectiles, fireworks and lightning expire on their own.

#### A per-block rule misses the ring

**Symptom:** Two link blocks pointing at each other (`1 to 2`, `2 to 1`) hold a circuit on forever after the source lets go.

**Cause:** The rule "a block may not repeat onto a channel it listens to" was checked one block at a time. It had already been widened twice (first output only, then all three, then a Round Robin's whole span), and each widening was still per block.

**Fix:** walk the channel graph forward from the proposed output and refuse it if any path returns to one of that node's own inputs, so a ring of any length is refused when it would be closed. When a rule exists to stop a *structure* (cycle, duplicate, deadlock), checking one participant can only catch the one-participant case.

#### A sampled safety probe

A "is the sky clear?" check sampled corners, centre and edge midpoints of each fill box (33/39/45). A leaf at x=35 sat between samples, so the arena was built over it. The whole arena was under 2,000 cells checked once, so sampling saved nothing and the guard was hollow. Count the real cost of a full scan before sampling; if sampling is genuinely needed, test with an obstruction placed BETWEEN sample points.

#### A skip rule can trap what it protects

A build skipped every cell a player occupied. (1) The skip included the cell UNDER the feet, so a visitor floating down onto a broken landing blocked the very floor block that would catch them. (2) Even feet plus head only: a visitor drifting down during a slow mend floated INTO the hole as it was filled, the build skipped that cell, and they stood in the hole forever. A protective skip is evaluated against a moving target. Skip only feet and head; move the player clear and lay the critical cells (the landing pad) FIRST in the same tick; retry until the post-condition (`landingSafe`) holds. Simulate gravity and slow falling in the fake or none of this shows.

#### Forget only after the world agrees

**Symptom:** An invisible, unmineable barrier wall (and four similar blocks) left stranded after switching a block off from far away.

**Cause:** Blocks that place things keep a note of what they put out (wall height, walkway length, reach). Each cleared the note at the end of the removal pass whether or not it had done anything. `dim.getBlock` on an unloaded chunk answers nothing, so the loop skipped everything, the note was cleared anyway and nothing knew about the leftovers.

**Look for:** a `catch { continue; }` or `if (!block) continue;` inside a removal loop, followed by an unconditional `rec.something = 0`.

**Fix:** `let sawItAll = true`, set false on every skip, clear the note only `if (sawItAll)`; the block retries each turn until the chunk is back (and the retry needs `touched = true`, see 3.3). Test with the harness's unloaded flag: switch off while unloaded, assert the thing is STILL there and the note is still set, then reload and assert it cleans itself up.

#### "Ignore it" and "allow it" are the same code

**Symptom:** Blocks bridged past an arena edge became permanent litter: nobody could break them, no clean-up removed them.

**Cause:** A handler opened with `if (!insideArena(...)) return;`. The same handler registered the block in `S.placed`, which makes it breakable ("only break blocks players placed") and is what the end-of-game clean-up walks. An early return meant to say "not our business" actually said "allow it and keep no record".

**Fix:** for any early return in a handler that also *registers* something, ask who owns the thing if this path is taken; if nobody, refuse instead of ignoring. Put bounds checks after the refusal helper is defined so refusing is as cheap as returning. Test from the clean-up end (after a game, nothing left near the site). Found by a load test with a full lobby spread across all four spawn islands, not by any placement test.

#### Removal allows only what that cell could hold

Reverse an air-only build with a **per-cell allow-list**: remove a block only if it is one of the ids *that cell* may hold (plan id, the wrong variant of anything on a change or decoy list, a bricked-up doorway). A flat "ids this structure uses" set would eat a player's own planks. See also the removal budget in 3.3.

#### Be pessimistic when the action is destructive

**Symptom:** A harvesting tool destroys immature crops.

**Cause:** Two features asked "is this crop ripe?" via `permutation.getState("growth")` and both treated "the API will not answer" as ripe. Fine for a report (the crop-counting tool), destructive for a harvester (an unripe wheat returns the seed only, and the growing time is gone).

**Rule:** an unanswerable question has two correct defaults. Reporting: optimistic, and say what you could not read ("3 ripe, 2 it cannot read"). Destroying, spending or overwriting: pessimistic, and say why (a different message from "did nothing"). Worse: the fake returned `undefined` for every block state, so every test only ever exercised the fallback and the real `growth >= 7` comparison had never run. Giving the stub real states failed four existing tests that had been passing for the wrong reason. Grep for `catch {`, `?? true`, `|| 0` in anything irreversible.

### 3.7 Reporting honestly

#### Coped with, and told nobody

The commonest fault: something goes wrong, the code handles it correctly, and nobody is told. To the player that is identical to their blocks vanishing on their own. Four instances in one night:
- a block threw during its turn: caught and skipped, counted in `stumbles()`, shown to nobody;
- records would not fit in the save: counted in `dropped` with a comment "so somebody can be told", and nothing read it;
- the save could not be parsed: start clean (right, since a throw kills the pack before the world loads), but every placed block is gone and it looks like a world nobody built in;
- the world refused the write: swallowed by an empty `catch`, losing everything since the last good save at the next reload.

**Find them:** grep `catch { /* ignore */ }` and counters that are incremented but never read. Ask of each: if this fires, what does the player see? **Two traps:** put the message where the failure's own consequences lead (an unreadable save means ZERO nodes, which takes the empty-list screen, so a warning on the populated list is never seen), and give the fake the failure path first or the test is theatre. A related case is a refused job (queue full, pool empty) that looks exactly like a broken tool.

#### A refusal can arrive as HTTP 200

**Symptom:** Every button on a companion web page does nothing, says nothing, and looks like it worked.

**Cause:** The page's `send()` did `fetch(...).then(r => r.json())` and checked nothing else. The bridge answers a command it cannot pass on with HTTP 200 and `{"ok": false, "why": "minecraft is not connected"}`. Four separate failures need four checks:
1. the request rejects (bridge not running): `.catch`
2. a bad status (`!r.ok`): a proxy answers 502 with no useful body
3. a body that is not JSON: `r.json()` throws, landing nowhere if nobody awaits it
4. **a 200 whose body says `ok: false`**: the easiest to miss and usually the only one carrying a useful sentence

**Testing:** each fixture must be one that ONLY the check under test can catch (a 400 with `{"ok":false}` is caught by check 4 even with check 2 deleted; use 502 with `{}` for the status check). Run the page's own code (slice the `<script>`, `new Function(...)` with a stubbed `fetch`) and assert the message reaches the page's own log list.

#### A promise in the text is a spec

A character line said "If you leave now, I will remember where you stopped." `leave()` reset the lap counter to 0 and `enter()` set it to 0 again. No test had read the sentence as a claim. Read the player-visible strings as a list of assertions and ask "is this true?" of each; promise lines (*I will remember*, *it starts where you left it*, *only the N things I built count*) each need a test. Two real bugs came from that pass.

#### The message describes the old behaviour

Player-facing copy is a second copy of the logic. A dowsing tool scanned a radius-4 cube and said "nothing within four blocks"; the scan became six eight-block rays and the message stayed, while the test asserted `includes("nothing within")`, which passes either way. A consume routine said "nothing happens" whenever no EFFECT landed, so eating a plain-food item reported that nothing happened (fix: `if (!landed && !def.nutrition)`). After changing behaviour, grep every string the module emits and re-read it. Pin the words that encode the behaviour (reach, count, direction), not a prefix. **Flavour text counts:** an axe that "hits harder than anything" did 9 damage next to an 11-damage sword. Keep a short list of ranking phrases ("the fastest", "lasts longer than anything") and fail the build when the claimant is not top of its stat.

#### Message frequency

See "Right about what, wrong about how often" in 3.3: count the messages over time, throttle the telling and not the action.

### 3.8 Generators and patches

#### A silent patch no-op ships a lie

Running `s.replace(a, b)` from a shell heredoc silently matches nothing when the target contains backticks, `${...}` or backslashes (the shell eats them). Returning the string unchanged is not an error, so the script printed success and exited 0. It happened three times in one session, and one shipped: a spawn greeting undercounted its pack for several versions.

```python
def edit(rel, old, new, count=1):
    body = open(rel, encoding="utf-8").read()
    found = body.count(old)
    if found != count:
        raise SystemExit(f"PATCH FAILED in {rel}: expected {count}, "
                         f"found {found}. Nothing written.")
    ...
```

Make the no-op an exception, prefer the editor's own find-and-replace for anything with template literals, and assert every player-visible string in the harness. (A related trap: `\b` inside a heredoc becomes byte 0x08, so a regex matches nothing.) It catches a no-op, not a malformed insertion.

#### An inline fix is reverted by the generator

A music switch (from overriding vanilla events to custom `trackNN` events) was applied as a one-off inline patch of the OUTPUT, while the generator still contained the old logic. The next full rebuild silently restored all 45 vanilla overrides and deleted the custom events the script plays. It was caught only because a post-build check printed "vanilla overridden: True". The generator is the source of truth: fix in the generator, grep every `gen_*.py` for the old logic after changing an approach, and keep a post-build assertion that fails if the old behaviour returns.

### Audit checklist

Ask these when reviewing an add-on. Each maps to a class above.

**State and ownership**
- After every `await` on a form, is the state reloaded before it is mutated and saved?
- Does every timer that releases a lock hold a token proving it still owns it?
- For each effect id (`addEffect`/`removeEffect`), how many systems write it, and is there one owner?
- Is any fact (channel, count, version, "how many things") computed or typed in two places?
- Is every saved record verified against the block or entity it describes before it is acted on, and reaped when dead?
- What does each queue, timer or batch depend on (tool held, player present, dimension, permission), and what cancels it when that changes?
- Are stored coordinates paired with a dimension, and stored deadlines paired with a clock that survives reloads?
- Is a "once only" reward keyed by the thing rewarded (not the actor's position), persisted, and radius-checked?
- Does a world-singleton work for a SECOND player standing thousands of blocks away?
- Does any "no value" sentinel use `0`, `-1` or `Infinity` instead of `null`/`undefined`?

**Transfers and phases**
- If the give half of a take-then-give fails, is everything taken (items, health, currency) put back?
- Is anything charged before it is known that delivery is possible?
- Is "no room" answered by trying to add and reading the remainder, not by counting empty slots?
- Does every path that reaches an item-issuing function leave the inventory in the state it assumes?
- Does each phase transition run one teardown listing everything the phase accumulated?
- Is every "the event arrived" flag set before any branch that can return?

**Timing and queues**
- Does every value a tick writes get saved (does the tick end dirty)?
- Do repeated calls in one tick go through a set-not-add API (velocity, health, timer, effect duration)?
- For each `runInterval`, is the period in ticks defined once and named?
- Can a spawning power spawn something that also has the power (tagged, capped)?
- Does every shared queue or budget have a per-player share, and was it tested with two players?
- Is teardown sliced at the same per-tick budget as building?
- Is there a cap on every list a hot loop scans, and a probe timing the worst realistic load by counts, not wall-clock max?
- Does a repeated correct action produce a repeated message?

**Config wiring**
- Does every declared field have a reader AND a writer, checked behaviourally?
- Is any effect gated on a class name when it should be gated on the stat that grants it?
- Does every exported symbol have an importer or a named test?
- After adding an opt-out flag, did every consumer of the old field get updated, and is it proven both ways?
- Was a borrowed or stolen feature tested on the borrower, with relative rather than absolute thresholds?
- When a shared table is read in new code, was one entry printed to confirm the value shape?
- When a list gains a second tier, does every reader search the superset?
- Do two independently clamped settings have a sum or product that indexes something?

**Dispatch and names**
- For every name-based join, has the offered set been diffed against the asked-for set in both directions?
- Does every if/elif dispatch have a derived valid-set check and a loud terminal `else`?
- Is every name that is legal also reachable, and is that proven by running the painter/handler?
- Is each handler in the map whose payload provides the fields it reads, checked against the built script?

**Building and destructive features**
- Does every placement fill air only, skip nearby players (feet and head), and stay within a per-tick budget?
- Does every teleport check for room and refuse (stay put) if none exists?
- Does a structural rule (cycle, duplicate, deadlock) consider two or three participants, not one?
- Is any safety probe sampled instead of scanning every cell, and was it tested with an obstacle between samples?
- Can the protective skip rule itself trap the player it protects?
- Is a removal note cleared only after every block it lists was actually seen and removed?
- Does any early return sit in a handler that also registers something?
- Does removal allow only ids that cell may hold?
- For anything irreversible, what happens when the engine will not answer, and does the fake ever return a real answer?

**Reporting**
- Does every `catch`, dropped counter and refused job produce something a player will actually see?
- Does the client check request rejection, bad status, non-JSON body AND an `ok: false` body?
- Is every promise, count and superlative in the text true, tested, and derived rather than typed?
- After changing a behaviour, were all the strings that describe it re-read?

**Generators and patches**
- Does every scripted patch fail loudly when it matches nothing?
- Was each fix made in the generator, not only in its output, with a post-build assertion for the old behaviour?
- Does the manifest version come from the same counter as the visible pack name?


---

## Part 4 — Testing, verification, build and workflow

Bedrock add-ons fail silently. There is no content log by default, a bad import kills every handler with no message, an unknown JSON key is ignored, and `try/catch` around every handler turns a broken feature into a feature that quietly does nothing. So the workflow below has one goal: **move every fault from "silent in game" to "red in the build"**, and then prove the red actually happens.

### Project layout and generators

Treat the pack as a build product, not something you hand-edit. One folder per add-on:

```
MyAddon/
  gen_pack.py          # writes every JSON/lang/manifest except hand-written scripts
  gen_textures.py      # writes every PNG (PIL); previews go to a contact-sheet PNG
  build.py             # bump version -> generate -> check() -> atomic zip
  version.txt          # one integer, read and bumped by build.py
  tools/sim.mjs        # runs the shipping scripts in Node against a fake @minecraft/server
  tools/check.mjs      # (optional) parses every script as an ES module + resolves imports
  behavior_pack/       # generated, except scripts/main.js and other hand-written modules
  resource_pack/       # generated
  dist/MyAddon.mcaddon
```

Rules that pay for themselves:

- **One table is the only place a stat lives.** Tiers, damage, durability, recipes, colours, mob stats: one Python table (`TIERS`, `KINDS`, `OP`, ...) feeds the item JSON, recipes, lang, item_texture and the script's data file. A generated `tools_data.js` (`// GENERATED - do not edit`) hands the numbers the script needs to the engine side, so the JSON and the script cannot drift.
- **Generated files are never patched inline.** A fix applied to output but not to the generator is undone by the next rebuild. Fix the generator, rebuild.
- **Everything worth re-running lives in the project** (`art/`, `tools/`), addressed relative to the script (`os.path.dirname(os.path.abspath(__file__))`). The session scratchpad is deleted between sessions and sometimes mid-session; generators kept only there are lost (only the PNGs they made survive). Scratchpad is for throwaway dumps.
- **Deterministic UUIDs are fine.** `uuid.uuid5(uuid.NAMESPACE_URL, "myaddon/bp.header")` gives stable ids per add-on and part; the five ids you need are BP header, BP data, BP script (if scripted), RP header, RP resources. Different add-ons must use different name prefixes. (A workspace-wide UUID collision scan is a short Python walk over every `manifest.json`.)
- **Derive the pack version in the build, not in the packager.** If the pack *name* says `V61` but `header.version` stays `[1,0,0]`, the game and the label disagree. `gen_pack.main(version)` writes the label and `[v, 0, 0]` on every manifest/module from the same number.
- **Bedrock pack cross-references** the build should verify (each is a real silent failure): BP dependencies list the RP header UUID and the RP lists the BP header UUID; the script module's `@minecraft/server` version must be one that actually ships (use the version your working add-ons use; an unshipped version, e.g. `1.14.0` in one old pack, makes the whole script module fail to load); item icons are plain strings (an object silently renders blank in 1.21); texture references omit `.png`; `texts/languages.json` = `["en_US"]` must exist next to the `.lang` or no display names load; data-only add-ons must not declare a script module.
- **Fabric mods** (same ideas, different toolchain): the verification command is `gradlew compileJava` (it catches real errors inspection misses, e.g. a `RegistryEntry<SoundEvent>` passed where a `SoundEvent` is needed), run from inside the mod folder. Register blocks, then entities, then items, then event handlers; a spawn egg needs the entity type registered first. 1.21.x needs the `assets/<id>/items/<name>.json` model-definition layer or the icon is magenta/black, and data folders are singular (`loot_table`, `recipe`, `tags/item`) or they are silently ignored.

### The build pipeline

Order matters and each step is small. This is the real shape of a working `build.py`:

```python
def main():
    old = int(open(VERSION_FILE).read().strip()); new = old + 1
    write_version(new)
    try:
        gen_pack.main(new); gen_textures.main()
        bad = check()                       # every silent fault becomes a string
    except BaseException:
        write_version(old); raise           # a crashed run must not burn a number
    if bad:
        write_version(old)                  # a failed build does not burn a number either
        print("BUILD FAILED:"); [print("  -", b) for b in bad]
        sys.exit(1)                         # nothing is zipped
    zip_pack()
    print(f"built {OUT} V{new} {os.path.getsize(OUT)} bytes")   # stat the artifact
```

Details worth keeping:

- **Auto-bump the add-on version on every rebuild**, and **roll the bump back on any failure** (exception or failed check). Bump first because the generators write the version into the manifests.
- **Atomic writes everywhere.** Opening a file for write truncates it before the new content exists; if building the string throws afterwards, the target is left at zero bytes (this destroyed a 1,100-check test suite that had no git behind it). Build the whole string first, write a temp file, then `os.replace`:

  ```python
  def write(path, obj):
      os.makedirs(os.path.dirname(path), exist_ok=True)
      text = obj if isinstance(obj, str) else json.dumps(obj, indent=2) + "\n"
      tmp = path + ".tmp"
      with open(tmp, "w", encoding="utf-8", newline="\n") as f:
          f.write(text)
      os.replace(tmp, path)
  ```
  Keep a dated backup of any long-lived generated/test file before a run of patches. The session transcript records every patch verbatim, which is the last-resort recovery path.
- **Zip with Python `zipfile` and forward-slash arcnames**, never PowerShell `Compress-Archive` (it writes backslash paths Bedrock rejects). Zip to `OUT + ".tmp"` then `os.replace` so a half-written `.mcaddon` never replaces a good one:

  ```python
  with zipfile.ZipFile(tmp, "w", zipfile.ZIP_DEFLATED) as z:
      for src, prefix in ((BP, "MyAddon_BP"), (RP, "MyAddon_RP")):
          for dp, _, fs in os.walk(src):
              for f in fs:
                  p = os.path.join(dp, f)
                  z.write(p, os.path.join(prefix, os.path.relpath(p, src)).replace(os.sep, "/"))
  os.replace(tmp, OUT)
  ```
- **A partial build must never write the deliverable's path.** A "build just this range" flag still builds one object and saves to the one output path; verifying one chapter replaced a finished 700-page epub with a 36-page one and nothing warned. Same shape for addons: give partial/dev builds their own output path, and after your last edit-verify cycle re-run the **full** build and stat the artifact (size, item count), not just the exit code.
- **Never overwrite output you have not read.** A verification step that writes to the deliverable's path is not read-only; the last thing to touch the path decides what ships.
- **Print collected notes/warnings LAST**, after everything that can add to them. A `NOTES` list printed before `package()` and `stage()` ran never showed the one note that said an imported copy had just been overwritten. A silent channel looks identical to "nothing to say".
- **Put every check ABOVE the report.** A guard appended below `if errors: ...; sys.exit(1)` computes the right answer, appends a perfect message, and the build passes anyway. The only tell was deliberately breaking the input and getting exit 0.
- **Have a control-byte gate.** Walk every `.py/.js/.mjs/.json` in the project and fail if `chr(8)` (and 7, 27) appears. Write it as `chr(8)`, not as a literal, or it reports itself. Scan `tools/` and helper scripts too: "a checker must check the checkers".
- **Sanitise generated tokens where they are embedded.** A random base64 id containing `--` is illegal inside an XML comment (about 1 build in a few hundred produced an unparseable page). Constrain the generator AND sanitise at the point of use, then test the generator in bulk (tens of thousands of ids), not once.
- **`sys.path`**: when a script borrows another project's module, `sys.path.append`, never `insert(0, ...)`; the borrowed folder shadows your own same-named modules and the error surfaces far away as a missing name. Inserting your **own** folder at the front (as in the example build script) is fine. When a traceback cites an unexpected directory, print `mod.__file__`.
- **Write patch scripts as files, not heredocs.** Through a shell heredoc `\b` becomes byte 0x08, `\n` inside a string becomes a real newline, unquoted backticks are command-substituted away, and a multi-line `old` pattern silently fails to match. Write the patch with the file-writing tool and run the file; locate blocks with `str.index` and splice by slice when exact-text matching is fragile; and only write after every assertion has passed so a partial failure cannot leave a half-patched file. A silent `str.replace` that matched nothing shipped a wrong build for several versions: always assert the pattern was found.

### Static guards to put in `check()`

The example build validates, and fails on, each of these (the list generalises to any generated pack):

- duplicate item/recipe identifiers; file name must equal the identifier's short name
- `minecraft:icon` is a plain string and its key exists in `item_texture.json`
- every icon PNG exists, is 16x16 and has a non-empty alpha bounding box (an empty sprite is an invisible item)
- the `.lang` entry exists and equals the `display_name` (lang is a second copy of the name; guard drift)
- `minecraft:damage` <= 127 (the byte wraps negative above that)
- every recipe is `AlwaysUnlocked`, its result is an item in the pack, ingredients from your own namespace exist, `pattern` letters equal `key` letters
- **no two recipes share the same grid** (one silently becomes uncraftable; also check your shaped recipes against vanilla's, since a 2x2 of common items can shadow a vanilla recipe)
- design invariants as strings (e.g. "OP tools must be unbreakable, so no `minecraft:durability`")
- existence of specific required items, so a generator edit cannot drop them silently

More guards that earned their place elsewhere:

- **Import graph reachability.** A `.js` file that `main.js` never imports never runs: the pack loads fine, every other feature works, and two finished, tested systems are simply not in the game. `build.py` copying a file into `scripts/` feels like installing it; it is not. Walk the graph from the manifest entry and fail on any unreachable script:

  ```python
  reach, todo = set(), ["main.js"]
  while todo:
      name = todo.pop()
      if name in reach: continue
      reach.add(name)
      body = open(os.path.join(scripts_dir, name), encoding="utf-8").read()
      todo += re.findall(r"""^\s*import\s+(?:[^'"]*?from\s*)?['"]\./([\w.]+\.js)['"]""", body, re.M)
  for f in os.listdir(scripts_dir):
      if f.endswith(".js") and f not in reach:
          bad.append(f"scripts/{f} is never imported from main.js")
  ```
  Side-effect-only modules (`import "./blocks.js";` whose job is `world.afterEvents.X.subscribe(...)`) must be imported explicitly, with a comment that the import IS the installation, or someone tidies it away as unused.
- **Syntax and cross-file imports.** `node --check` parses each file in isolation, so it cannot see that `main.js` imports a name another file never exports. In Bedrock that is fatal and silent: the module graph fails, no handler registers, every feature looks dead. Also, on the Node version where this was observed (Node 24, `.js` files containing ESM syntax) `node --check` exited 0 on files with plain syntax errors (a raw newline inside a string, an injected `new Set(;`), and exited 0 on a missing path. Do not use it as a gate. Parse each file with `vm.SourceTextModule` (run under `node --experimental-vm-modules`), which is how the loader parses it, and also check that each import's file exists and the imported name is exported. Prove it by injecting a break and watching the build go red.
- **Read-but-never-written / written-but-never-read fields.** A config field nothing reads is a feature that silently is not there; a value read but never written is the mirror. Guard "is this key consumed anywhere" over nested keys too, and **strip comments and docstrings before searching** (a field named only in a comment satisfies a naive text search, which is exactly the state the guard exists to catch). Strip with `tokenize` in Python (a naive `#.*$` eats `"#1a2b3c"` colour strings) and with a quote-aware scanner in JS (a `//` inside a string must survive). Exempt only sub-dicts whose keys are read by iteration (e.g. a recipe's grid-letter map).
- **Distinguishable, not merely different.** A duplicate-colour check (`!=`) passed nine drink pairs and seven spawn-egg pairs nobody could tell apart. Use an RGB distance:

  ```python
  COLOUR_MIN = 60
  def colour_gap(a, b):
      ra, rb = hex2rgb(a), hex2rgb(b)
      return sum((ra[i] - rb[i]) ** 2 for i in range(3)) ** 0.5
  ```
  Scope it to items that share their other signals (all drinks share a bottle silhouette so all pairs must be far apart; two foods clash only if they also share a shape). Calibrate the threshold by looking at the contact sheet, not by fitting one bad pair: raising the threshold to 105 for one bad pair flagged nine pairs that read clearly, and a guard that cries wolf gets deleted. Check the premise where you apply it: tools intentionally share material colours, so the rule was scoped to sprites with an explicit override. Same idea for names, icons, sounds and mob silhouettes: measure what the player has to do (tell them apart), not equality.
- **Reference data floors.** Anything validated against vanilla data (ids, textures, sounds, entity component names) reads a folder the game may repack. Once, the game's readable data folders vanished into `__brarchive` containers mid-session and a check dropped from 2,500 known ids to 480 while still passing. Every such check needs (1) **floors**: refuse to run if the reference is implausibly small; (2) a **dated cache** in the repo to fall back on, refreshed only when a fresh read passes the floors; (3) **never write the shrunken list**, and have the harness enforce the floors too rather than report a hollow pass. A `.brarchive` is a name index followed by plain file contents; decoding it as latin-1 and running regexes recovers ids and paths, but the json inside is minified, so match on what FOLLOWS a key rather than on indentation. A `warn:` line that scrolls past under `| tail -2` is not a check.
- **Compare like with like.** A texture-collision guard compared our shortnames (`mypack_sunsteel_sword`) with vanilla texture PATHS (`textures/blocks/stone`); the sets can never intersect, so "zero collisions" and "cannot express a collision" looked the same. Key against basename, path against path.
- **Guess lists are not references.** In one project 23 of 45 guessed sound-event ids did not exist. Pull identifier lists from Mojang's `bedrock-samples` or the installed pack and check by set difference.
- **Generated tokens/JSON keys checked against a vanilla file.** `"filter"` instead of `"filters"` inside `nearest_attackable_target.entity_types` is silently ignored by Bedrock (no filter, so every mob attacked the nearest thing, another mob). Compare keys against an installed vanilla entity file, and assert the key name itself, not that a value appears somewhere in a stringified block.
- **Per-content-type hygiene**: every string a player sees is a promise. Read them all, and test the ones that promise behaviour ("I will remember where you stopped").

### The sim harness: run the shipping scripts in Node

The single most valuable tool is a Node script that loads the **real built** `behavior_pack/scripts/main.js` (and its whole import graph) against a hand-written fake `@minecraft/server`, then drives events and reads the fake world. It covers what `node --check` cannot: the module graph, handler registration, and behaviour.

Minimal loader (from the small finished example; the fake here is deliberately tiny):

```js
import vm from "node:vm"; import fs from "node:fs"; import path from "node:path";
import { fileURLToPath, pathToFileURL } from "node:url"; import { spawnSync } from "node:child_process";

// SourceTextModule needs a flag: re-launch ourselves with it if it is missing.
if (!vm.SourceTextModule) {
  const r = spawnSync(process.execPath, ["--experimental-vm-modules", "--no-warnings",
    fileURLToPath(import.meta.url)], { stdio: "inherit", env: process.env });
  process.exit(r.status ?? 1);
}
const HERE = path.dirname(fileURLToPath(import.meta.url));
// SIM_BP=<dir> runs a mutated COPY of scripts/ instead -- the red-team hook.
const SCRIPTS = process.env.SIM_BP ? path.resolve(process.env.SIM_BP)
                                   : path.join(HERE, "..", "behavior_pack", "scripts");

const serverExports = { world, system, ItemStack, BlockPermutation, EquipmentSlot }; // your fakes
const cache = new Map();
async function load(file) {
  if (!cache.has(file))
    cache.set(file, new vm.SourceTextModule(fs.readFileSync(file, "utf8"),
                                            { identifier: pathToFileURL(file).href }));
  return cache.get(file);
}
async function link(spec, ref) {
  if (spec === "@minecraft/server") {                       // the fake package
    if (!cache.has(spec)) {
      const names = Object.keys(serverExports);
      cache.set(spec, new vm.SyntheticModule(names, function () {
        for (const n of names) this.setExport(n, serverExports[n]);
      }));
    }
    return cache.get(spec);
  }
  const file = path.resolve(path.dirname(fileURLToPath(ref.identifier)), spec);
  if (!fs.existsSync(file)) throw new Error(`bad import "${spec}" from ${ref.identifier}`);
  return load(file);                                        // a missing file = loud failure
}
const entry = await load(path.join(SCRIPTS, "main.js"));
await entry.link(link);          // a missing export or bad import throws HERE
await entry.evaluate();          // top-level subscribe() calls run; handlers are now registered
```

(For a fake that answers every property, a Proxy-based stub of `@minecraft/server` and `@minecraft/server-ui` that `import()`s `main.js` is enough for a pure load test: report event-subscription and interval counts and fail on any load error. Keep that as a separate always-on gate run by the packager, and make the strict per-name fake for behaviour tests.)

The rest of the harness is a hand-written fake world plus a tiny test runner:

- **A fake clock.** `system.runTimeout` pushes `{at: now + max(1, delay), fn}`; `pump(n)` advances `now` and fires due timers. Assert `timers.length === 0` at the end of a scene to prove a queue drained and nothing leaked.
- **A fake dimension** backed by `Map` (`"x,y,z" -> typeId`), with `getBlock`, `runCommand` (parse only the exact commands the code is allowed to issue and **throw on any unexpected command**), `spawnItem`, `getEntities`.
- **A fake player** that records what happened to it: inventory adds, `xp`, loot commands, and counters like "loot ran on an already-cleared block" (an ordering bug you can only see if the fake records the block state at command time).
- **Event helpers** that reproduce the platform's timing: the real `playerBreakBlock` after-event fires after the block is gone, so the helper deletes the block first, then calls every registered handler.
- **`ok(cond, msg)` + `section(name)`** printing `FAIL: msg` and a final `N check(s) FAILED`, exiting non-zero. Assertion messages carry the measured values (`stone left inside the 5x5x5: 3`).
- **Fixture sanity checks first.** Assert the scene is what you think (`12 logs, >60 leaves`) before acting on it. A precondition assertion is what exposes a test that never set up the thing it tests.
- **Assert conservation, not movement.** "It merged something" passes on a bug; "the total is conserved" fails on it (a missing `maxAmount` on the fake produced NaN item amounts, and only the conservation check went red).
- **Assert both directions of every rule**: the tool that should act does (positive case), and the plain one does nothing (`only the 3 broken blocks are gone`, `no commands were issued`). A negative-only test cannot tell "correctly skipped" from "never ran".
- **Test the jobs that queue.** A 124-block job must spread over several ticks (`pump(3)` leaves blocks), finish (`pump(60)`), and a **second** job must still work afterwards, which catches a wedged queue.

Sim discipline:

- **The sim reads the built pack, not `src/`.** Editing source and running the sim without rebuilding tests yesterday's code. Three rounds of debugging went into a stale build. Make the harness police the gap itself: compare the newest mtime in `src/` against `behavior_pack/scripts/` and exit 2 with `STALE BUILD`.
- **One pump can be many ticks.** In one harness `pump(n)` advanced the clock by 20 ticks per iteration (one second), not one tick, so `pump(12)` = 240 ticks. Timing tests (cooldowns, one-second kicks, drain rates) lost three passes to this. Read what your `pump` really does; for a short-lived output loop `pump(1)` and check each time; when a timing test fails, print the block's own timestamps before believing the block is broken.
- **A tick's writes vanish unless the tick ends dirty** and **N repeats inside one tick collapse to one** (a set, not an add): assert per step, not per run.
- **Do not `await` a queued `system.run` callback** in a harness; it often awaits a form whose resolution comes from a later queued callback and deadlocks. Fire and flush microtasks instead.
- **Settle after every event whose handler defers.** A handler that defers with `system.run` leaves timers and a parked promise chain that resume three tests later, in the middle of an unrelated menu flow, where they eat that test's scripted form answer (symptom: "no button matching X on screen Y" on a screen the test never opened). Use one helper that does both: `const settle = async (t = 2) => { mc.pump(t); await flush(); }`.
- **Async tests need an awaiting runner.** `function check(name, fn){ try { fn() } ... }` given an `async` function drops the promise; the assertions run after the summary and usually after `process.exit`. Collect async checks and run them **sequentially after** the sync pass (a shared `CURRENT` label makes concurrent ones report under whichever name was set last), and make the sync runner **report** a returned promise instead of dropping it.
- **Loops in harnesses need a cap.** `while (count < max) grow();` spun forever once `grow()` started refusing (the game had ended), and a hang reports nothing. Loop on the call's return value (`if (grow() < 0) break;`), bound every unbounded `while`, and in mutation runs treat a timeout as a suite failure, reported separately from a caught mutant.
- **Randomised fixtures**: pick sentinel values the code under test cannot produce (a test set the world clock to 6000 and asserted it moved, but two of nine randomly-ordered scenes set 6000 themselves, so it flaked). Run the suite five times in a row after adding a random fixture.
- **A hard wall-clock tick ceiling catches the host, not bugs** (in one run the failures came from the game hogging the CPU, not GC). Assert operation counts against a control instead.
- **Freeze what a "slow tick" is** by counting work (reads per handler, entities scanned), not milliseconds; a slow tick fails nothing on its own.

### How to write fakes

A fake built by the same person who wrote the code shares their assumptions. Most silently vacuous suites are a fake problem. The rules, each learned the hard way:

1. **Match identifiers exactly.** A fake that answered `getComponent(n)` with `n.includes("is_baby")` returned a component for a typo'd `"minecraft:is_babyX"`. Normalise the namespace at most (`n.replace(/^minecraft:/, "")`), then `===`. The same shape in a form driver: picking a control by label *substring* set a toggle instead of a slider. Exact stripped label wins; a substring must match exactly one control, and matching several is a reported error.
2. **Never stub a hook as `() => {}`. Record it.** An empty `onArenaChange` hook meant "was it called?" was not a question the suite could ask, so an early return that skipped the notification (the arenas stopped changing every lap) was invisible. Two lines to record calls made a real test possible.
3. **Implement every method the code calls, especially inside `try { } catch { return false }`.** A fake with no `matches({families})` threw, the catch returned false, the feature did nothing in the sim, and the assertion that mattered ("it never blinds an animal") passed for the wrong reason. Assert the positive case beside the negative one. Give the fake a real family model (zombie: mob/monster/undead, cow: mob/animal).
4. **Implement every property too.** A missing `maxAmount` produced NaN stack amounts. The fake should also report the platform's real limits (max stack 1 for tools; a fake saying 64 lets a merge test pass on something the engine would never allow).
5. **Record every field the real API accepts.** A message-form fake recorded `{who, title, buttons}` but not `body`, and the reader's `f.body || ""` turned the gap into a blank line instead of an error, so every confirm body escaped the read-every-string pass. When a fake records a call, enumerate the real API's setters and record all of them; when a reader defaults a missing field, ask what "never set" looks like; cross-check by counting (one body per form shown).
6. **Apply the effects, not just record the call.** A double that logged `runCommand("time set 6000")` and left its own clock alone made "the run changed the world time" impossible to fail. Likewise a stub that recorded `setblock` but never removed the block left every later test reading a world only the pack believed in. Apply the handful of commands the code depends on.
7. **Give timed things a real end.** A fake whose `addEffect` pushed to an array that never aged out could not tell "the block took it away" from "it ran out on its own", made "wears off" inexpressible, and hid the difference between "gave it" and "still carrying it". Store `until = currentTick + ticks` and drop entries as the clock advances (in `pump()`, not in a filtering getter, because tests do `alice.effects.length = 0`). Keep a separate never-expiring `given` record, cleared wherever scenes clear the live list, and make "did this do anything" fingerprints count both. Match real semantics (a re-applied effect of the same type REPLACES, highest amplifier wins) and record an application COUNT per type so a top-up is distinguishable from a single application. Modelling expiry honestly made three "passing" tests fail; none tested what it claimed.
8. **Fail the way the platform fails.** Bedrock commands usually refuse by returning `successCount: 0` and NOT throwing. A fake that only fails by throwing makes upstream `catch` paths reachable and the "did the block actually go?" check look like dead code you added out of caution (removing it broke nothing), which is how it gets deleted. Model the real dialect: status code, empty result, partial success, thrown error. With the fake refusing correctly, the same bug was a farmable one (8 payouts for 8 blocks still standing).
9. **Do not let the fake enforce the thing under test.** A sword that heals to at most max health was asserted by reading the health afterwards, and the assertion stayed green with the clamp removed from the handler because the fake also clamps (as the engine does). Keep the fake faithful and assert on what the CODE uniquely decides (here: the handler stays silent on a full-health target).
10. **A fake missing an axis of the real world hides bugs on that axis.** A fake player that always looked perfectly level (`view = {x:1,y:0,z:0}`) is the one angle where a raw-3D facing check agrees with a flat one; real players look down at small mobs and up at tall bosses. Test at steep angles (and note gentle angles passed on the buggy code; it failed only past about 75 degrees). Likewise, test ranged things **at their stated range**: a speed quoted in two units (blocks per engine step vs per second) made every shot travel a quarter as far, hidden because every test fired at targets 2-4 blocks away.
11. **Redundant code paths defeat single-mutant testing.** Naming a killer worked through two routes (death-event damager AND remembered last hit), so mutating either alone stayed green and the guard looked vacuous. Mutate every path that can answer the question, together.

### Proving a guard can fail (red-teaming)

**A green suite that cannot go red is worse than no suite**, because it retires the suspicion. Every new guard and every new test needs a break that makes it fail, and you must check that the break is real.

The mutation loop, small enough to reuse (tested against the example add-on: emptying the protected-block list turned two checks red):

```python
# mutate.py <scripts_dir> <scratch_dir>
import os, shutil, subprocess, sys
SRC, SCRATCH = sys.argv[1], sys.argv[2]
MUTANTS = [("protected blocks list emptied", "bedrock", "zzz_none")]   # (label, old, new)
for label, old, new in MUTANTS:
    tmp = os.path.join(SCRATCH, "m")
    shutil.rmtree(tmp, ignore_errors=True)
    shutil.copytree(SRC, tmp)
    p = os.path.join(tmp, "main.js")
    s = open(p, encoding="utf-8").read()
    assert old in s, f"mutation did not apply: {label}"        # a silent no-op is the enemy
    open(p, "w", encoding="utf-8").write(s.replace(old, new))
    r = subprocess.run(["node", "tools/sim.mjs"], env={**os.environ, "SIM_BP": tmp},
                       capture_output=True, text=True, timeout=120)   # a hang is a failure
    print(label, "->", "CAUGHT" if r.returncode else "MISSED")
```

The `SIM_BP` variable points the harness at a modified copy so the shipping pack is never touched. For build-side guards the equivalent is to inject the break into the generator's tables (in-process, then build into a temp directory) and assert `check()` complains.

How to make the proof honest:

- **Ask "what exactly is now different?" and verify that specific thing.** Make the replacement raise once to confirm the injection took effect at all. Add `assert old in s` to every text mutation.
- **Prefer reverting the real fix over hand-crafting a broken input.** A revert is far harder to make a no-op. After a pass, back up the changed sources, undo all of the pass's fixes with one script, rebuild, run the suite, restore. Three tests written in one sitting were all vacuous on the reverted build (a laser test whose target was inside range anyway, a heal test with slack enough to hide a doubling, a retrigger test that called `fire()` by hand instead of through the path where the bug lives). It takes about thirty seconds.
- **Rebuild between injection and run.** A harness that loads the shipped artifact does not see a `src/` edit until the build runs, so "injected, still green" is ambiguous between "vacuous check" and "the break never reached the artifact". Rebuild (or point the harness at the mutated copy), and revert plus rebuild afterwards so a poisoned build is not left behind. The mirror also bites: fixing `src/` and not rebuilding shows tests failing for a bug already repaired.
- **The tighten-the-threshold trick.** For any budget or count check, lower the limit until the check MUST fail. If it still passes it is not measuring anything (a sweep asserting "no handler reads more than 400 blocks" still passed at 20 because it prefixed the namespace twice and dispatched to nothing).
- **Every loop-shaped check must count its own work** and assert on that (`seen > 300 && spoke >= 10`): reads performed, handlers that answered, rows compared. A sweep can iterate the right number of times and touch nothing.
- **Precondition assertions** for any test with a setup chain: "the shot actually connected", "the form actually opened". A test that wrote a round to `tg_rnd_<short_id>` where the code reads `tg_rnd_<full_type_id>` never loaded the round; no assertion could fail.
- **Break the thing the code depends on.** Renaming a definition and its reference together (one sed hitting both) breaks nothing. Flattening a stat's size left the material multipliers varying the same stat, so the slot was never inert.
- **Inject where the guard looks, not where the data also lives.** Three ways a WORKING guard was reported broken by a bad injection, each costing a wild-goose chase:
  1. A **dispatch table captured the function by reference** (`SPRITE = {"sword": sword_sprite}`); rebinding `art.sword_sprite` changed the name, not the dict entry. Patch the entry point the caller reaches or the table entry.
  2. An **index-addressed fixture** (`MOBS[3]["skin"]...`) silently changed meaning when one row was inserted above it. Address rows by id (`_by_id(rows, "x")`, raising if missing).
  3. A fixture edited **the file on disk** while the guard read the **in-memory record** of what was written.
  The tell is identical each time: a brand-new guard you just watched work by hand comes back "stayed quiet on a real break". Check the injection before believing the verdict.
- **A fixture over a family must be derived, not listed.** A "group must span this much colour space" case named five pickaxes and tightened their colours; three more pickaxes added later kept the group wide, the injection stopped injecting, and 75/76 pointed at a nonexistent bug. Filter the table (`[t for t in TOOLS if t["kind"]=="pickaxe" and t["tier"]=="good"]`, assert `len(fam) >= 3`) and cover every member. Add a name filter to the runner so one case re-runs in seconds.
- **Do not patch a module that subscribes at import.** Writing a modified copy and importing it fresh re-runs top-level code, subscribing a SECOND set of handlers; the original handler then quietly does the right thing, so the injection reports CAUGHT for a reason unrelated to the patch. **A misleading CAUGHT is worse than a MISSED.** If you cannot make it sound, delete the check and leave a comment saying why.
- **A regex written through a generating layer can be silently different** (byte 0x08 from `\b`). Print the compiled pattern (`[c for c in fn.__code__.co_consts if isinstance(c, str)]`), or grep the file's bytes; never trust rendered text (the file-view tool strips control characters; `cat -A` shows `^H`). When you cannot avoid an escape, use `chr(92) + "b"`, lookarounds like `(?![a-z])`, or `String.fromCharCode(10)`. `\d` and `\w` survive a Python heredoc; `\b` and `\n` do not.
- **Keep the gate fast or it gets skipped.** A red team that rebuilt the whole project once per case took ~30 minutes at 118 cases and drifted into "run it later" (twice a number was reported from a run before the last edit). The cases were independent (each resets tables, applies its break, builds into its own temp directory), so they fan out with a process pool: **152 seconds for 118 cases, same coverage**. Use **processes not threads** (cases mutate module-level tables), keep an escape hatch back to serial (`GUARD_JOBS=1`), and collect results then print in order. When a verification step gets slow enough that you start running it "later", treat that as the bug.
- **A guard's own protection can hide behind another.** Test a protection in the ONE scenario where it is the only thing in the way. A "chest inside a radius of stone is left alone" test stayed green with the denylist emptied, because a same-type rule had excluded the chest first. Break a chest surrounded by chests (same-type lines every neighbour up) so only the denylist can save them; now emptying it turns three assertions red. Ask of every guard "what else would have caught this?" and build the case that disables everything else.

### Test anti-patterns (quick catalogue)

| Anti-pattern | Symptom | Fix |
|---|---|---|
| A check appended after `sys.exit` / the report | Right answer computed, build passes | Put checks above the report; break input, confirm non-zero exit |
| Injecting by renaming both sides / breaking an unused input | "Proof" is vacuous | Verify what changed; revert the real fix instead |
| "Must not happen" scene where the fault cannot win | Guard deleted, test still green | Make the wrong candidate reachable, first in the list, in range (a domino at the same coordinates as its partner can never be "one apart"; a decoy placed last is never reached) |
| Setup line deletes the precondition | Test named after the bug cannot see the bug (`clearAllRuns()` erased the very leak) | Read setup lines; drive the real sequence (leave and rejoin) instead of hand-building end state |
| Skip/exempt list with a reason attached | The exemption IS the bug, written down as design ("its two modes may both be true at once" was the bug) | Delete the entry, run the suite, read what fails; keep only world-based reasons |
| Asserting on what the platform also guarantees | Green with the code removed | Assert on what the code uniquely decides |
| `apply(status, "burn", 0)` and similar | Helper guards `if (!(x > 0)) return`, so a zero argument is silence; the boss's whole second phase did nothing | Grep call sites for literal `0` / `undefined` arguments; write a strong version of every throwaway assertion (a `!== undefined` that could never fail hid it) |
| Early `return` above a line every other path runs | Notification / flag / rage check skipped on one path | Search for a `return` inside a conditional directly above an every-path line; record "the event arrived" BEFORE any branch that returns |
| "Did anything change?" in a noisy room | Passed with every ability disabled, four times over (contact damage, wave ended mid-measure, spawner still adding units, short effect expired) | Quiet everything you are not testing, sample every step and stop at the first difference, then disable the subject and watch it go red |
| Protection tested where another rule saves it | Deleting the denylist changed nothing | Build the scene where only that rule stands in the way |
| Sentinel the system can also produce | Flaky by random draw | Sentinel nothing under test can generate |
| Test that hangs | No output, a 100% CPU process | Loop on return values, cap every `while`, timeouts on mutants |
| Comparing two different kinds of strings | Zero hits forever | Same-kind comparison; assert the check can express a hit |
| Deferred work from an earlier test | Failure on a screen the test never opened | `settle()` after any deferring event |

The commonest fault across all of these is the same: **handled correctly, reported to no one**, or checked in a way that could never fail. When a hunt dries up, **grep for untested exports**: list every `export function NAME` in `src/` and count occurrences of `NAME` in `tests/`. In one project, 144 exports had 50 never mentioned; two gaps were real (an entire shop form had never executed, and an on-death effect was called only from damage-over-time paths, so an enemy killed by anything else died silently; the fix was to call it from `kill()`, the one place a death happens, capturing the position before the entity is removed). The grep is crude (a name mentioned once counts as covered) but it asks the question no other technique does: does anybody ever run this code?

### Things to look at with your own eyes

Assertions test rules. Pacing, repetition, tone, balance and "can I tell these apart" are properties of things a test does not know to ask about.

- **Render sprites and geometry to a PNG and open it.** Build a contact sheet of every item. The habit has caught invisible items, five identical-looking ores, and near-identical bottles that passed every duplicate check. Preview icons on a background matching the real inventory slot, not near-black: a blade the exact grey of a slot passed forty "look at it" passes on a dark preview.
- **Read every string the player will see** (names, lang, form titles and bodies, chat lines, action-bar text), all of them, in one dump. A mob called "Big Target Target" was found this way. A promise in the text is a spec: "I will remember where you stopped" was a lie for forty rounds until each string was read as an assertion and tested.
- **Print the transcript, not the assertions.** A playthrough script that runs the fake world start to finish and prints one line per thing the player sees (chat, action bar, sounds near and far, in order) found in seconds what 297 green checks and every injected bug caught could not: the same sentence on four laps running. Fix: lines speak once when they change (edge-triggered), not every time a condition holds. Also count messages over time; throttle the telling, not the action.
- **Print the curves and read them.** Ten thousand passing assertions said nothing about a game you die in 0.4 seconds. Write a readout (`balance.mjs` / `balance.py`) that prints seconds-to-die and seconds-to-kill by setting and wave, each item beside the vanilla tier ladder it competes with, hits-to-kill both ways, a bar chart so outliers show, and population counts (tier/mood). Then **freeze the conclusions as assertions that exit non-zero** ("Casual wave one gives at least 5 seconds", "no class opens at more than 3x another", "at least one hostile mob can actually hurt someone") and prove them red by restoring the old numbers. Root cause is usually a growing quantity aimed at a fixed one. Compute damage per second from the effective rate (capped by invulnerability frames, pellets all count), measure single-target and crowd separately, and if the readout keeps a copy of a constant that lives in code, assert the two agree. A "must beat vanilla" rule must be scoped by what the thing is for (a sword's job is damage, a pickaxe's is mining), and a gimmick alone is not a defence.
- **Ask what the reward is attached to.** If coins came only from surviving a timer, killing paid nothing and every weapon was optional; nothing was broken, the incentive pointed away from the game. Check the first shop can sell you something.
- **One axis per item.** Items that scale both cooldown and damage with level are 36x at level six versus 6x for a single-axis item; a damage-over-time pool compounds with rate; a cooldown divided by level can fall under the engine's minimum tick.
- **Get a real playtest early.** A fake shares its author's assumptions about input, units and geometry. After 32 bug-hunting passes, one real play session found four things 12,000 assertions missed: a misspelled JSON key (`filter`), one speed in two units, the control players actually use (left-click) never wired to anything, and a fake that only looked level. Ask how a player who never read the instructions would use the thing.
- **Read the artifact's bytes when a claim matters.** Stat the file (size, entry count, page count); a build's exit code is not evidence the output is right.

### Silent-failure debugging on Bedrock

- No content log is written by default, `try/catch` hides broken handlers, and three failure modes are each invisible: a **module-graph error** (a cross-file import of a name that is not exported: no handlers register at all), a **wrong parameter position** (a handler declared `(p, l)` when the table passes `(player, ev, level)`: `l` becomes an object, arithmetic goes NaN, the API throws, `try/catch` eats it), and a **missing API method** (`ModalFormData.submitButton()` does not exist in every server-ui build; guard optional methods with `typeof f.method === "function"`).
- **Never treat a caught error the same as a user cancel.** Resolve `null` for errors and a real response for cancels, branch differently, or a crash is indistinguishable from pressing Escape (and the menu just walks backwards).
- **Surface errors in-game** on the action bar (`"§cForm error: ..."`); it is the only channel you can actually read.
- **Drive the real code in Node** (the load test, the form-flow test that stubs the UI classes and scripts form replies, the sim).
- **When you cannot test an assumption, make its failure survivable.** If every weapon depends on an event (`itemStartUse` for a custom item with `use_modifiers`) that the docs do not define for that case and you cannot load the game to check, add a second path: hold-to-fire where it works, click-to-fire (`itemUse`, which fires for any item on a plain right-click) where it does not, never both, with a flag set by the primary path that the fallback consults. Test that the fallback fires and that it stands down once the primary path has proved itself. Same for an unproven API such as `Player.isSleeping`: pair it with a proven event or the feature is silently absent.
- **Custom mobs swamp the spawn pool**: normal-looking spawn weights took 63% of a vanilla animal pool; budget weights per pool. **Self-spawning mobs recurse exponentially**; cap them.
- **Other silent traps found in playtests**: `/gamemode` needs op (use `setGameMode`); borrowed player geometry renders untextured; removed entities throw on every property access; distance-moved walk cycles freeze limbs when a mob stops; one shared work queue lets one player's big job starve every other player's tools (give a per-actor share, not just a global cap); deferred work must re-check its authority when it runs (a broken tool left a queue mining bare-handed and destroying every drop); a handler wired into the wrong dispatch map passes "has a handler?" but fires on the wrong event and reads undefined fields; unloaded chunks make far-away `fillBlocks` a silent no-op (build near players and verify the floor first).
- **A name matched at runtime, missing, with no error** is the shape of most of these. Diff the set of names offered against the set asked for (recipes vs items, lang keys vs identifiers, dispatch tables vs valid names, fled-id written from one list and read from another). A name can be VALID and UNREACHABLE (six markings passed every guard and painted nothing): derive the valid set from the dispatch chain itself, and give an `elif` chain an `else` that fails loudly.
- **A table's value shape is its contract.** A new reader took `id -> [item, count]` for `id -> item` and every old reader hid the shape.

### Deploying and staging safely

- **Stage, do not launch.** Produce the `.mcaddon` and wait for the go-ahead. Never auto-import it, never auto-launch the game, and never force-quit a game the user is in. Do not start auto-clickers or macro bots without an explicit, verified go-ahead.
- **Iterate through the development folders.** Copy the packs into `com.mojang/development_behavior_packs` and `development_resource_packs` (the real path is under the app's own data folder, so look it up rather than guessing) and restart the world; import an `.mcaddon` only for the final delivery. `python build_mcaddon.py --dev` is the convention in the reference workspace.
- **Check where the world reads its packs from.** A world whose `world_behavior_packs.json` is empty has no add-on regardless of what is installed; and a copy embedded inside the world's own `behavior_packs` folder (one-line note, not re-verified here) can win over what you just deployed. When the game "shows nothing new", suspect stale/duplicate packs before your code.
- **Bump the version on every rebuild/deploy** so old and new coexist in the pack list and you can tell which one loaded; a new numbered folder gets fresh UUIDs, a same-folder rebuild only changes the version array.
- **Do not touch what you did not create.** Block-placing features fill AIR only and skip occupied space; removal needs the same per-tick budget as building (681 writes in one handler tick versus a 256/tick build) and removes only allow-listed ids per cell; "one per world" is really one per place, so reuse a saved site only if it is loaded and near.
- **Friend-safe rule for multiplayer add-ons**: no kick/ban, no removing op, no trapping players, and no takeover of player control (teleporting, forced effects, gamemode/inventory/input overrides) from scripts, even for myth-flavoured content; apply effects to mobs, keep atmosphere to sound/particles/chat. Spot-check before shipping: `grep -rinE "setGameMode|inputPermissions|player\.addEffect|\.teleport\(" **/behavior_pack/scripts/*.js` (opt-in item-use self-buffs are fine). Every new version-family copy can silently reintroduce a previously fixed bug, so re-audit each copy.
- **Kid-appropriate everything.** Titles, file names, mob/item names, in-game copy and code comments are clean and game-flavoured (`midnight_arcade`, `turbo_static`) rather than visceral. Aggressive music is fine; the words attached to it are not. Name it clean the first time.

### Workflow habits

- **Run the whole pipeline, then verify after the LAST edit.** For a Bedrock add-on: build (`python build.py`), then the sim (`node tools/sim.mjs`); if either exits non-zero, fix and rerun. Any later partial run invalidates an earlier "done". Stat the artifact.
- **After each build, name the specific untested or fragile spots** (unverified API calls, entities never seen in game, performance cliffs, interactions that stack unexpectedly), ranked by likelihood of biting, rather than repeating cautions already explained.
- **Save bug CLASSES (not instances) as you review**, mid-review, to a notes file, and record when a check is added to a validator (what it catches) so nobody rebuilds it.
- **Coped with, and told nobody**: if you handled something surprising (a vanished reference folder, a wiped scratchpad, a stale build), say so.
- **When effects pile up, sum them in one owner.** Effects do not stack in Bedrock (highest amplifier wins), so sum levels per effect in ONE owner loop; a shared slot (slowness) needs one owner, and a token rather than a boolean so a stale timer cannot release a lock it no longer owns.
- **When the bug hunt dries up**: grep for untested exports, read the transcript, print the numbers, do a real playtest, then add content.


---

## Part 5 — Case studies and other games' modding

Each case study below is a finished (or nearly finished) project. Read them for the design decisions and for the things that broke; copy the reusable pattern at the end of each. "Not playtested" means the project was only verified by scripts and simulations, never by playing it in the real game.

### Minecraft Bedrock add-on case studies

#### ArsenalAddon (78 to 180+ items from one generator)

**What it is.** One combined add-on: 8 melee tiers (bronze to celestial) x sword/spear/pickaxe/axe/shovel = tools, bows and crossbows, 8 script-driven ranged weapons, 4 gadgets (grappling hook, blink shard, panic-heal relic, void berry), 11 potion-style drinks, then later special ammo, ores, armor and "kinetic" spear-swords.

**Key design decisions.**
- A deterministic generator (`gen_arsenal.py`) writes all item JSON, manifests, lang files, `item_texture.json` and a `scripts/arsenal_data.js` table. A separate `gen_textures.py` draws icons. The hand-written engine (`main.js`) imports the generated data table, so a stat exists in exactly one place.
- Generator UUIDs are idempotent (`uuid5`, reuse existing) so re-running never changes the pack's identity. Texture regeneration skips PNGs that already exist so hand-painted art is never overwritten; a cleanup step prunes orphans.
- Special ammo: guns auto-load the first special round found in the inventory (an `AMMO` map) and consume one per trigger pull; arrows detonate on impact via `projectileHit` events (an `ARROWS` map). A passive HUD shows the loaded ammo.
- Universal instant mining uses `minecraft:digger` with `destroy_speeds [{block:{tags:"1"},speed:100}]`. The tag `"1"` matches every block.
- Visible armor needs **attachables** (`RP/attachables/<item>.json`) using vanilla `geometry.humanoid.armor.<slot>` and `controller.render.armor.v2` (not the plain `controller.render.armor`, which fails on 1.21), plus per-material layer textures. Native protection is `minecraft:wearable {slot, protection}`; the script only grants set-bonus perks.
- Bows and crossbows will not fire without `minecraft:use_modifiers` next to `minecraft:shooter`.
- Full-auto weapons use `minecraft:food` plus `itemStartUse`/`itemStopUse` events to run a hold-to-fire loop.
- A custom ore that is "rock hard" (60 seconds to destroy) drops its material only through a loot `match_tool` condition for specific pickaxes.

**What went wrong.** A bug-hunt pass found six real bugs, each caught by a 38-assertion behavioural simulation: single-pellet weapons randomly missed locked shots (the random "stray" was not gated on pellet count > 1); splash and lingering potions were never consumed on throw; a shotgun fired its on-hit effects once per pellet (9x explosions) instead of once per target; the blink shard was never consumed; arrows were processed twice when both `projectileHit` events fired (fix: dedupe by projectile id). A known remaining limit: special ammo in the off-hand slot is not detected because only the main inventory is scanned.

**Reusable pattern.** Generator + data table + hand-written engine; idempotent ids; a stubbed-API simulation (`tests/simharness.mjs` stubs `@minecraft/server`) that counts failing assertions as its metric. Deferred ideas (not built): a real magazine/reload system, true optical scope zoom (needs an RP overlay), physics-thrown splash bottles (need custom projectile entities).

#### OP Tools addon (a 9-of-a-tool crafting ladder)

**What it is.** Fill all 9 crafting slots with one tool to get its "OP" form; 9 OP tools make an "OP OP" tool, up to 7 levels. 5 tool kinds x 6 tiers x 7 levels = 210 items and 210 recipes.

**Key design decisions.**
- Everything is generated (`gen_pack.py`, `gen_textures.py`, `build.py`); `main.js` is hand-written and reads a flat generated `TOOLS` table. Higher levels get extra glow rings so all seven are distinguishable at 16x16.
- Level N is always crafted from 9 of level N-1, so no level can be skipped. If a request skips a level, the skipped one must still be built or the chain becomes uncraftable.
- **Balance ceiling.** One weapon in the world was meant to stay the strongest (864 damage = `minecraft:damage` 127 plus 737 from a script bonus table). Every OP Tools weapon is hand-tabled under it (top is 700). Damage is a hand-balanced table frozen at the top level, not a multiplier: a x3 ladder clears 864 by level 2. Only reach (mine radius, tree cap, field size) scales per level.
- An instant-defeat ability was removed because it outranks any number and cannot coexist with a ceiling.
- Reach caps exist for tick cost, not taste (`MAX_MINE` 8 = 15^3 blocks, `MAX_TREE_CAP` 16384, `MAX_HARVEST` 24 = 49x49). Axes and hoes therefore saturate around level 3-4; pickaxes and shovels still climb all 7. This was stated openly rather than shipping dead levels silently.

**What went wrong.** A different addon's weapon (1000 damage) actually beat the "best in world" sword, so the ceiling promise was false until its bonus table is lowered. Lesson: a global ceiling must be checked across all packs in the world.

**Reusable pattern.** A craft-N-of-the-previous-level ladder; freeze damage tables and scale reach; tell players when a cap makes upper levels cosmetic.

#### Basic Tools addon (small, fully guarded)

Five gem tiers (lapis/quartz/amethyst/emerald/obsidian using vanilla materials) x sword/pickaxe/axe, plus an OP Pickaxe (5x5x5 area mine, auto-smelt, magnet) and OP Axe (fells a whole tree including leaves, cap 1024) crafted from 9 of the obsidian tool. Copper was deliberately skipped because vanilla has copper tools and a copper recipe would shadow one. The plain gem tools are pure JSON; only the OP tools need script.

Pipeline: `python build.py && node tools/sim.mjs`. `build.py` bumps a version file (rolled back on failure). `sim.mjs` loads the shipping scripts via `vm.SourceTextModule`, and an environment variable (`SIM_BP=<dir>`) runs a mutated copy, which proved 7 sim guards and 8 build guards go red when broken. Not playtested.

#### PvP Buffs addon (potion tiers and a scripted enchanting block)

**What it is.** 10 PvP potions x 3 tiers as reusable containers: 4 vanilla potions craft a Canteen (10 uses, 2x duration); 4 canteens craft a Jug (100 uses, 4x, +1 amplifier, Slowness I while held); 4 jugs craft a 20-Gallon Jug (1000 uses, 8x, +2 amplifier, Slowness V while held). Uses are `minecraft:durability` spent in `itemCompleteUse`. The **True Enchanter** block (2 enchanting tables + 3 lapis blocks + 1 obsidian) applies 89+ scripted enchants at any level up to 255 to any item.

**Key design decisions.**
- Every enchant needs a genuinely distinct mechanic (new trigger, interaction or resource), not another flat bonus with a new name. A catalogue entry with no real scripted action is a build failure. Batches of 15-20+ are better than a cautious handful because every enchant multiplies across the whole inventory.
- Stripping enchants pays "Enchant Essence" (1 per normal level, 4 per legendary level); the legendary group is bought with essence instead of XP. Recycling gear is the only source.
- The block looks like a colour-inverted enchanting table: build.py draws vanilla-style then does `255 - v` per channel.
- Every menu screen carries a one-click Exit (players got stuck without one).
- A "Speed Mining" ability breaks a block and hands out `new ItemStack(id)`, which bypasses the command validator and can produce items `/give` refuses. Indestructible and admin blocks must stay in a `MINE_SKIP` list (reinforced deepslate, bedrock, barrier, light block, structure void, allow/deny/border, command and structure blocks, portals, spawners, containers).

**What went wrong.**
- `build.py` wiped `behavior_pack/` on every run and destroyed hand-written scripts once. Fix: hand-written code lives in `src/` and is copied in, with a hard failure if `src/` is missing. Never edit `behavior_pack/scripts/` directly.
- `package.py` now guards three things that each broke once: every catalogued enchant has a handler; `minecraft:icon` is a plain string; no ALL_CAPS identifier is used without a declaration (`node --check` cannot see an undeclared constant).
- A clone mob ("husk") tamed to the player who made it attacks mobs only; `is_owner` is not a documented filter test, so there is no reliable way to spare the owner.
- Bedrock potion aux values for recipes must come from the wiki, not a generator site that groups them by effect (healing 21, harming 23, poison 25, regeneration 28, strength 31, weakness 34, swiftness 14, fire resistance 12, leaping 9, slowness 17).

**Reusable pattern.** Toolchain order `gen_mega.py` (generated enchant module) -> `build.py` -> `package.py` (validates JSON, checks recipe identifiers resolve, verifies icon keys, syntax-checks JS, runs the load test, then stages), plus `audit.py` for the catalogue and `tests/loadtest.mjs` / `tests/menuflow.mjs`.

#### True Guns addon (data-table-driven ranged weapons, ores, gunsmithing, armed mobs)

**What it is.** About 30 hand-written ranged weapons plus 400 generated ones, hundreds of gunsmith parts, 12 ores, 8 armed player-clone mobs, craftable ammo and invincible target dummies. (Written up here as a systems study; the design lessons apply to any weapon pack.)

**Key design decisions.**
- `guns_table.py` is the only place a weapon number exists. `build.py` turns each row into item JSON, recipe, 16x16 sprite and `scripts/generated.js`, which `src/*.js` imports. Never hard-code a stat in a script.
- Damage is dealt by the engine, not by script: weapons spawn real projectile entities with `minecraft:projectile` + `on_hit.impact_damage`, one entity per damage "rung" (`pipeline/bullets.py` TIERS). Scripted `applyDamage` did nothing in game. A weapon whose damage has no rung silently drops to the nearest one.
- Targeting was originally a hitscan ray (`getEntitiesFromRay` + `getBlockFromRay`, nearest wins) so a 1100 rounds-per-minute weapon would not spawn ~20 entities a second; the ray picks the target and a spawned projectile deals the damage.
- The magazine is a **player** dynamic property keyed by weapon id + hand, not an ItemStack property (Bedrock cannot persist per-item state). Two identical weapons in one hand share a magazine; per-hand keys make dual-wielding work.
- Reload removes rounds from the inventory when it finishes and credits the magazine with what actually came out, so a cancelled reload has nothing to roll back.
- Right-click is the fire button. A sidearm in the off hand makes clicks alternate main/off, each with its own magazine key and cooldown. Only secondary weapons get `minecraft:allow_off_hand`. Bedrock raises no use event for the off-hand slot, so the main hand must hold a weapon for the off-hand one to alternate with.
- Target dummies are meant to be invincible: `minecraft:damage_sensor` refuses every cause except projectile and entity_attack, and the script applies `instant_health` amplifier 255 for 40 ticks whenever health drops below a backup line, rate-limited to once per 2 seconds.
- Sounds are recorded per class (primary/secondary x shot/reload/ready/empty, plus hit and headshot markers). Several takes in one slot become several entries under one sound definition, which is how Bedrock randomises them. build.py synthesises placeholder audio for any slot with no takes so the pack is never silent.
- One `fx` vocabulary (explode/poison/fire/frost/glow/heal/smoke/knock) serves both rounds and thrown charges. Adding an fx means a `KNOWN_FX` entry plus a branch in `fire.js`; build.py checks both directions so a round cannot ship doing nothing.

**The parametric gunsmith.** `parts_table.py` generates 62 parts from grids (size x material), giving 40,320 combinations. There is no lookup table: each part carries stat deltas, `profile()` multiplies them and `match()` returns the nearest of the 30 real weapons. Lessons:
- A profile can only distinguish what it models. One heavy weapon was unbuildable for two passes because the only difference from another was piercing, which the profile did not include. If two things feel different in game, the model needs the stat that makes them different.
- Check coverage exhaustively, never by sampling. A stepped sample said 26/30 reachable and named four as unreachable; the full 40,320-combination sweep said 29/30.
- Never let a test reimplement shipping logic. A cross-language guard compared Python against a copy of the matching maths written inside the test; the copy went stale and "proved" agreement with nothing. It now imports the real function from `smith.js`.

**Ores** (`ores_table.py` + `ores.py`). 12 real minerals feed a real-chemistry chain: sulfur + saltpeter + charcoal -> gunpowder, copper + zinc -> brass (cartridge cases), galena -> lead, wolframite -> tungsten (armor-piercing rounds). Ore-based ammo recipes sit alongside the plain ones so nothing already craftable breaks. Two silent-failure traps:
- **No tier tags on custom ore blocks.** An ore gated behind a tier tag drops nothing when mined. Anyone can break it; a pickaxe is just faster via `item_specific_speeds`.
- Ore generation is two files that must agree: `minecraft:ore_feature` (what) and `minecraft:feature_rules` (where). A mismatched identifier means it never generates and nothing says so. `validate()` cross-checks every rule against every feature plus texture entries and y ranges.
- Texture rule: keep the stone quiet (gentle noise, not speckle) and give any ore darker than the stone (luminance < 100) a bright rim; the rim is what you see from six blocks away. Raw drops get furnace recipes only, no crafting recipe.

**Special rounds.** Pressed ingots (9 raw -> 1), then black powder, then explosive charges and rounds. Special rounds load into any weapon and each has two recipes on purpose (a mob in a bucket or a brewed alternative). The magazine remembers which ammo went in (`tg_rnd_*` property). Reload preference is table order; there is no in-game selector because Bedrock gives scripts no spare button. Explosive rounds are limited to one blast per 3 ticks with `breaksBlocks: false` (20 explosions a second is a slideshow); the explosive charge is the tool for holes (`breaksBlocks: true`, 30-tick fuse).

**Armed clone mobs** (`clones_table.py`, `pipeline/clones.py`, `src/clones.js`). Humanoid mobs spawn holding a weapon, wander and fight each other as readily as the player.
- Health 24-34 (above ten hearts); a player-level 20 folded in a couple of shots. build.py guards the floor and table-vs-pack drift.
- The targeting filter is just `is_family: mob`, which clones carry themselves, so identical clones fight each other. Naming a clone family and excluding its own kind was the bug that made identical clones ignore each other. Projectiles must never join the `mob` family or every shot yanks every clone's aim onto the round in flight.
- No `behavior.panic`: it fires on any damage, so a clone bolted the instant it was scratched. Backing off is script-side and health-gated at 15%.
- Damage equals the weapon's damage (1 damage = half a heart), and a melee clone's shove uses that number. Contact shove: anything within 2.4 blocks of a different kind takes 1 damage and knockback once a second.
- Clones build cover: under fire they place a block between themselves and the threat and pillar up if the target is above. Capped at 6 blocks each, air only, every block removed when the fight ends or they are defeated. A mob that rearranges the world needs all of that tested.
- **`attack_interval` is in seconds, not ticks.** `interval // 20` floored a 6-tick cadence to one shot a second. Use `round(ticks / 20.0, 2)`.
- Held items render for free with `runtime_identifier: minecraft:zombie`; baking a weapon quad into the skin as well produced two weapons in one hand.

**What went wrong (bug classes found by auditing).**
- Silent overwrite was the dominant class: two recipes with the same grid, the same id (same filename), or the same display name. Minecraft never tells you; the item is simply absent. build.py now guards all three.
- Numbers that lie: a range past simulation distance can never hit anything; 12 ores at their first vein counts added 84 veins per chunk, more than doubling vanilla's whole ore budget (now guarded at 96 blocks range and 55 veins).
- A `burstGap` field meant the gap inside a burst instead of between bursts, so a "3-round burst" was three shots 1.3 seconds apart.
- `random.Random(hash(str))` is not deterministic across runs (Python randomises string hashing per process), so every ore texture redrew on every build. Use `zlib.crc32(name.encode())`.
- "Prefer the special resource" picks must take the required amount into account, not just `count > 0`: one poison round won a whole reload and left a 1-round magazine while 64 plain rounds sat unused.
- An item with no `use_modifiers` raises no event on an air right-click (same trap as the ranged weapons).
- `minecraft:potion` in a shaped recipe matches any potion including water bottles; use the brewing ingredient instead.
- Maps keyed by player id leak unless `playerLeave` cleans them up. A HUD that reads one source while the logic reads another lies (ammo reserve counted plain rounds only). A creative `menu_category` group `itemGroup.name.miscFood` put ore and metal bars in the Food tab.
- **Item-count ceiling:** about 1,100 items works; 270,000 silently registered nothing. Keep generated catalogues modest.
- Two of the eleven build guards were wrong on their first attempt (a creative-tab guard matched on substrings and flagged `bayonet_rifle` as a part). After writing a guard, prove it fires on a deliberately injected bug and stays silent on correct code.
- Rendering everything and reading every string caught an invisible Smoke Charge, five identical black ore blobs and a mob called "Big Target Target" that no scripted check found. Dark art on a dark background needs a lighter rim.

**Reusable pattern.** One table per concept; generate JSON, sprites and script data from it; validate both directions (table to pack and pack to table); simulate every weapon (`node tools/sim.mjs` fires every weapon, empties a magazine, reloads, hits a dummy, checks a held trigger stops on its own); `python build.py` fails on the silent stuff (bad icon, missing sound definition, dead import).

#### ClaimAddon (owner-locked doors, item-taking beds, safety locks)

**What it is.** A custom two-block Claim Door owned by whoever places it, beds that claim themselves on first right-click and move other players' inventory items to the owner (stashed in a dynamic property if the owner is offline), a `doge:safety_lock` item that locks any block to you, and a master key that releases only your own claims. Vanilla doors are deliberately left alone.

**How the custom door works.**
- Both halves are the same block; state `doge:top` says which. Facing comes from the `minecraft:placement_direction` trait (`minecraft:cardinal_direction`, `y_rotation_offset: 180`), not a hand-rolled yaw state.
- Geometry is a 16x16x3 slab on the block's north edge. Opening is just +90 degrees more of `minecraft:transformation` rotation, so 4 facings x open/closed = 8 rotation permutations and no second model.
- Collision is a full block when closed and `false` when open, which is rotation-invariant and immune to uncertainty about Bedrock's transformation sign convention.
- On place, script grows the top half via `permutation.withState("doge:top", true)`; on break it clears the partner with `setType("minecraft:air")` (silent, no drop) so exactly one door item drops.

**Protection mechanics.** Locked blocks behave like bedrock to non-owners: mining cancelled, filtered out of `beforeEvents.explosion` via `setImpactedBlocks`, `pistonActivate` cancelled, interaction denied. Claim-on-place (not first-click) prevents a friend from stealing your door. An all-powerful key would defeat locks in multiplayer, so the key only frees claims you own; an admin escape hatch is `/scriptevent doge:clearclaims`.

**Redstone bypass.** Cancelling an interact event does nothing against a button, lever or pressure plate, and Bedrock has no block-state-change event. Custom doors are immune by construction (custom blocks are not wired to redstone), but locked vanilla doors need a `system.runInterval` sweep that force-shuts (`withState("open_bit", false)`) any claimed door open while its owner is not within `GUARD_RADIUS`. Claims are cached in memory and each entry remembers whether it is really a vanilla door, so locking hundreds of blocks costs nothing per tick.

**Data.** Claim key `c:<dim>:<x>,<y>,<z>` as a world dynamic property. Solo testing with `/scriptevent doge:fakeclaim <name>`, `doge:claiminfo`, `doge:clearclaims`.

**Reusable pattern.** One model plus rotation instead of many models; cache protection lookups; use a periodic sweep when no event exists.

#### Cyber Arena (a Roblox game rebuilt in Bedrock)

**What it is.** A co-op wave roguelike: pick a class, survive four waves per arena, shop between waves, a boss every fourth. Six arenas a run (five random plus one fixed final), modes casual/normal/brutal and an "entropy" 0-9 difficulty layer. Namespace `ca:`.

**Key design decisions.**
- All HP and damage are in the source game's wiki units (a Tank has 1500 HP, a laser hits for 600). Minecraft's health bar is fixed at 20 for players, so the real pool is script-side and the vanilla bar is only a mirrored percentage. Enemies are immune to every vanilla damage cause and are removed only via script.
- Projectiles are records in a list, not entities. Hundreds stay cheap and the hit test runs in Node. Substep at half a block so fast shots do not tunnel.
- One entity, `ca:droid`, with a component group and event per enemy and a render controller indexing `query.variant`: 62 enemies from one model.
- The model is a hovering drone with no legs, so there is no walk cycle and none of the "distance-moved limb freeze" (a walk animation driven by distance moved locks limbs when the mob stops). Animate from `query.anim_time`.
- `tools/export_data.mjs` emits `build_data.json` from the real source modules so `build.py` never retypes the roster; `package.py` fails if it is stale.
- Pipeline: `src/*.js` copied not generated, `build.py` then `package.py`, `tests/loadtest.mjs`, `tests/sim.mjs`, `tests/preview.py`.

**Sourcing game data.** Some wiki sites block `WebFetch` but the MediaWiki API works with `curl` (`api.php?action=query&generator=allpages&prop=revisions&rvprop=content`). Community wikis often have the fullest stat tables.

**Bug classes.** An entity's `location` is its feet, not its centre (hit tests must aim higher), and a rotating hitbox tunnels exactly like a fast projectile. A 62-enemy roster needs behaviour and colour and a mark to tell enemies apart; 78 lookalike pairs were found.

**Reusable pattern.** Keep game-scale numbers in script and mirror them; use records instead of entities for mass projectiles; export data from source modules rather than retyping.

#### Bedwars addon (a mini-game in the sky)

**What it is.** A basic Bedwars game (namespace `bw:`) with an arena, custom bed blocks, generators, shops, a points-and-loadout meta layer, bots, eight game modes and optional classes. Pipeline: `python build.py && node tools/sim.mjs`, plus `python tools/preview.py` rendering beds and a top-down arena to a PNG. Wording is deliberately soft ("knocked out", "FINAL KNOCKOUT").

**Key design decisions.**
- The arena is built by script in the sky (player y+30, y 90..290, Overworld only) after checking every target cell is air, so it never overwrites a build. A ticking area keeps far generators running. Beds are custom blocks `bw:bed_<team>` because vanilla bed colour cannot be set by command; a per-second truth check counts a bed that vanished without a player breaking it.
- Real inventories are stored in a barrel vault sealed in barrier blocks 40 below the arena (2 barrels per player), verified before clearing the player, and restored at game end or on rejoin.
- Map protection: only player-placed blocks break (the set persists in chunked dynamic properties); explosions are filtered the same way; glass is immune.
- Shopkeepers borrow the vanilla villager model and animations (no custom art).
- The item picker scans `ItemTypes.getAll()` and sorts into 18 categories with 20-per-page paging; `NEVER`/`NEVER_PREFIX` remove game-breakers. Powerful items (netherite, elytra, totem, mace...) are reachable only through a challenge.
- Unlock price = `max(floor, damage^2 x 1.5, armour x8)` rounded to 5, with per-material floors.
- **Decoy Bed:** identical to the real bed in model, texture, hardness, sound, map colour and empty loot table; build.py fails if any of those drift apart.
- **Modes are config, not code.** A mode is a few fields in `MODES`, read through `mode()` (`bedsPerTeam`, `teams`, `genSpeed`, `shopPrice`, `respawnSeconds`, `events`, `noMidGens`, `timeScale`, `holdToWin`, `shopClosed`, `startKit`). Eight modes ship (Classic, Dimension Clash with stolen beds, Chaos, Horde waves, Rush, One Life, King of the Middle, Geared Up); each new mode needed one new knob, not new plumbing.
- **Bots** are one entity `bw:bot` with a component group per team and `nearest_attackable_target` filters keyed on `has_tag` against `bw_team_<id>` tags the game puts on players. Vanilla AI chases and hits; script does what a mob cannot (walls the bed in, bridges to the middle by teleport-stepping, walks to the nearest enemy bed).
- **Turret:** a block that becomes a stationary entity the instant it is placed (the "TNT lights itself" trick), capped per player by counting live entities with a matching owner property rather than a separate counter.
- Classes (Builder, Attacker, Trickster) reissue their kit through one helper on every spawn; shop restrictions are enforced in `buy()` itself, not only in the menu.

**What went wrong.**
- A per-bot scan of the whole placed-block set made 20 bots a 30 ms tick; a short `S.decoys` list brought it to 3.5 ms. A sim check "no tick over 20 ms" caught it.
- keepInventory stayed on forever if a game never ended: save the player's own rules once and restore them on load if the state is stale.
- Building outside the arena was silently ignored instead of refused, so the block was never recorded and never cleaned up. Refuse, do not ignore, when the handler is also what registers the thing. A Realm-sized load test (12 players, 4 bots, 240 placed blocks) found it.
- A skin's solid hat/jacket overlay layer drawn over the head hides the face behind blank skin tone; paint base layers only.
- A test fake was stricter than the game (its effect whitelist held six effects, so a valid `jump_boost` failed). A fake stricter than the game is as wrong as one that is kinder.
- A config alias captured before a reload silently defeated the class-restriction system for several iterations (stale module reference).
- `terrain_texture.json` was written to disk before the turret texture was added to the in-memory dict, so a check passed while the shipped file lacked the entry. Write files after everything is registered. Pull numbers such as turret range and health out of the config module with a small regex so the entity's real stats cannot drift from what scripts use.
- Reference-data gotcha: no single game file lists every item. Union BP data, RP `blocks.json`, `en_US.lang` keys and a catalogue; `terrain_texture.json` has the same shape as `blocks.json`, so a loose regex passes texture names off as blocks.

**Design lesson: leave the fun parts.** Scripting the whole arena so it builds itself from a menu took away the part players enjoy (building the map). For anything map-shaped (arenas, bases, dungeons, tracks), default to giving markers and tools (a bed block, a generator block, a shop spawner, a wand that sets a spawn) so the game runs on a map the builder made; offer auto-generation as an option, not the only path.

**Not playtested.** Untrusted real-API spots: leather dye (`minecraft:dyeable`), `playerInteractWithEntity` on a custom entity, a villager model on a 1.21 entity, `setSpawnPoint(undefined)`.

#### Cheat Menu addon (one-screen owner-locked panel)

**What it is.** A single-item cheat menu that opens three ways so you cannot lock yourself out: an item, `!cheat` in chat, or `/scriptevent doge:menu`.

**Key design decisions.**
- **Owner-locked.** `access.js` gives the world one owner, claimed by the first player who spawns. Everyone else gets no menu, an inert item and no cheats applied. The engine re-checks access every tick because settings live on the player and outlive a revoke. `/scriptevent doge:claim` seizes ownership; that is safe because running a scriptevent already needs operator permission.
- Code shape: `state.js` (23 toggles + 8 sliders in one `DEF`, stored as one JSON dynamic property per player), `cheats.js` (the whole engine on one `runInterval(..., 1)` with work spread by `tick % n`), `menu.js` (every screen built from `{label, icon, run}` row objects, never a switch on a selection index), `catalog.js` (data only: 321 give items, 8 kits, 59 mobs, 40 enchants, game rules).
- The cheat panel is one `ModalFormData` with every toggle and slider, keeping click count down. World Control is also one modal (time/weather/mode/difficulty dropdowns + 14 game-rule toggles). Only game rules that changed are sent; the screen reads current values from `world.gameRules` (command name and script property are spelled differently; the mapping lives in `catalog.GAMERULES`).
- `/enchant` is used rather than `addEnchantment` so it refuses what an item cannot take and the menu can report it.
- Area Breaker runs two fills that skip the layer at feet-1 so it never removes the floor. Vein Miner only accepts blocks that come in veins (accepting stone would tunnel) and breaks with `setblock ... destroy` so blocks drop.
- Everything that can throw is caught and recorded in `cheats.problems`, which the Settings screen prints.

**Verification.** `python build.py && python package.py` (runs `tests/loadtest.mjs` + `tests/sim.mjs`, 102 behaviour checks), then `python tests/injections.py`: 24 deliberate breaks that must each make the sim fail on a named check.

#### DogeOS Computer addon (an in-game computer with apps)

**What it is.** Blocks (computer, TV, server rack) and items (laptop, phone, camera) opening a shell of apps: a video site, mail, a shop, maps, games, a search engine and a page builder, all in Bedrock forms.

**Key design decisions.**
- The camera records real gameplay: `camera.js` collects notes from real events (break/place/defeat/hurt/dimension plus a 1 Hz sampler), keeps the highest-priority ones and turns them into timestamped video frames. Published clips keep frames in a separate world property with a size budget, not in the feed entry (the feed is one capped string).
- Scripts are authored in `src/` and only copied by `build.py`. Apps receive a `back` callback instead of importing the shell, so the module graph stays acyclic (`main -> os -> {tube, apps, games} -> {data, store, ui, playback}`).
- Everything persists in dynamic properties (per-player profile, world lists). All arrays are capped in `store.save()` because an uncapped profile eventually throws on save and looks exactly like "the computer forgot everything".
- Search (`dogle.js`) indexes articles, videos, shop, players, places, files and apps into one ranked list; results act (play/buy/travel), so `purchase()` and `travelTo()` are exported separately from their menus. Bedrock has no `eval`, so calculator answers use a hand-written expression parser.
- The page builder maps a page to a list of elements picked from a menu: static elements render into the form body, interactive ones become buttons. That mapping is what makes a Bedrock form work as a web page.
- Shops are physically stocked: a drone port claims vanilla chests and a Buy Button reads and removes real items, so an empty chest means out of stock. Stock lookups use a snapshot per render (a time-based memo let a just-filled chest read as empty), and the sale always re-reads the chests before charging.
- One currency shared by every app.

**Bug classes (all with regression tests).** A form await lasts minutes: reload lists after the last await. Take-then-give must roll back. A saved record outlives the block it describes. Identify records by a stored id, never by display name (auto-generated clip titles collided, so deleting by title deleted every match; switching to name lookup to dodge a stale-index bug just trades one bug for another).

**Reusable test idea.** `tests/sim.mjs` drives the real scripts through fake players, blocks and forms, with the form stub matching buttons by label text, so a renamed or reordered menu button fails loudly instead of pointing at the wrong app. `package.py` runs the load test and the sim before it will pack.

#### Upgraded Redstone addon (150 signal blocks and a web remote)

**What it is.** A standalone pack (`ur:`) with 150 blocks plus a handheld remote. Every block powers the same way: while "on" it swaps to (or acts as) a `minecraft:redstone_block` so all six sides push power. Custom blocks cannot emit redstone themselves, which is why the swap is used. Blocks differ only in what decides "on".

**The shape of the system.**
- **Sources** broadcast on a channel `ch`: sensors of people (player/sneak/key/armour/ride/crowd/...), the world (mob/time/moon/crop/chest/furnace/...), events (chat/alarm/defeat/hurt), plus manual and free-running ones.
- **Wired blocks** listen on `ch` and repeat on `oc` (`oc === ch` is refused): link, delay, latch, counter, logic gate, majority, memory, flip-flop, edge, debounce, chance, duty, sync, inverter, splitter, one-shot, extender.
- **Outputs** do something: sound (a row plays a tune), particle, effect, message, title, firework, light, water, ice, bridge, harvester, magnet, vault, sorter, dropper, weather, teleport pad, elevator, countdown, score and more.

**Rules the whole thing runs on.**
- Sensors read every 10 ticks off `system.currentTick`; a `null` reading (unloaded chunk, API refused) means "cannot tell" and must never be treated as "no", or a door slams shut whenever a chunk goes quiet.
- Channels record when they last rose (`edgeAt`) because a 2-tick flick is invisible to a block that looks twice a second.
- Any record that listens to a channel must stamp "I start listening now" when it is pointed at one, or the block fires on something that happened before it existed (this cost three separate bugs).
- Anything that writes to the record from the read path must report `touched`, or the write is discarded when the list is re-read next tick.
- Paired blocks (teleport pads, mob pads, a vault between two chests) need a direction, not a cooldown, or they ping-pong. Anything that fills a container needs a stop condition.
- Blocks that place things fill air only and remove only what they placed.
- Test traps: checking a short output after its window closed, and checking a count on something that drains while you look. Assert on transitions, not leftovers.

**Verification.** `tests/loadtest.mjs` (module graph, every expected subscription, `NODE_BLOCK` agreeing with the declared block) and `tests/sim.mjs` (425 checks). One sim check walks every type in `TYPES`, places it and asserts it registers as itself. `package.py` also cross-checks addon and bridge wire formats, that every block id the remote drives is declared, and that no two recipes share a grid (two pairs collided during the build).

**Working vs untested.** The blocks, remote and redstone-block swap were tested in the real game. The websocket half was only driven by a fake Minecraft (untested in game).

**Installation note.** Install over the live `com.mojang\behavior_packs` folder once it exists and only use `development_*_packs` before the first install; having both means editing a copy nobody enabled. `.mcaddon` may have no file association on a machine, so copy the two pack folders in and restart Minecraft.

#### Web app to Bedrock bridge (a general technique)

A Bedrock behaviour-pack script has no network access, so a web page cannot call it. The way in is `/connect <ip>:<port>`: the game dials out to a websocket server and runs commands sent over it (the Code Connection protocol).
- **Page to game:** the bridge sends `commandRequest` with `commandLine: "scriptevent ur:remote <args>"`; the addon handles it in `system.afterEvents.scriptEventReceive`.
- **Game to page:** the bridge subscribes to `PlayerMessage` and the addon replies by saying things (`dimension.runCommand("say ...")` is the only channel a websocket can read back).
- Windows blocks the UWP Minecraft app from reaching 127.0.0.1: print the LAN IP and connect to that (or use the loopback exemption via `CheckNetIsolation`, which needs admin). Cheats must be on and "Require encrypted websockets" must be off.
- Chat is the return channel and it is visible. Stay silent until the bridge says `hello` (a window that expires), send deltas for single changes and full snapshots only when asked. Snapshots must be chunked (header line plus a line per record) because chat has a length limit, and a half-arrived snapshot must not replace the good one.
- On Windows a second server can bind a port that is already listening and the two split requests at random (an old bridge served a deleted page). Set `allow_reuse_address = False`, skip `SO_REUSEADDR` on `nt` and let the second one fail loudly.
- The websocket server is about 60 lines of stdlib (sha1 handshake, masked frames in, unmasked out). A test that fakes the client half makes the loop testable with no game running.

#### Good Bad & Mid addon (a modpack whose labels are enforced)

**What it is.** A grab-bag pack (`gbm:`) of tools, foods, drinks, blocks and mobs, each labelled Good, Mid or Bad, with no guns and not a randomiser. About 122 tools, 47 foods, 32 drinks, 72 blocks, 52 mobs (3 rigs: humanoid/blob/critter, own geometry and animations, borrowed vanilla voices), 3 ores -> raw -> bars -> storage blocks, hundreds of recipes and generated textures.

**The premise is the constraint.** Every entry carries a tier and `build.py` refuses to finish if the label is a lie:
- Tools: within a kind, every good out-damages and outlasts every mid, which out-does every bad.
- Food: ordered on nutrition; food and drinks are checked on the KIND of effect, not count (a good one may only do nice things, a bad one only unpleasant ones or nothing; counting effects was useless because one drink had two effects and both were punishments).
- Mobs: health strictly ordered; a good mob may never hunt players; a bad mob may not out-hit the weakest mid one.
- Three signals carry the tier, never one: name colour, a bracketed word in the display name, and a coloured pip in the sprite corner that `validate()` reads back out of the PNG pixels.
- A Field Guide item browses the whole catalogue from the same tables, with a guard that fails if anything is missing. `validate()` proves the progression closes: it walks recipes forward from vanilla items, ore drops and mob loot and fails if any pack item is unreachable.
- "Potions" are crafted drinkable items, not brewing-stand potions: Bedrock's stable format has no custom brewing recipes.

**Pipeline.** `python build.py && node tools/sim.mjs && python tools/guard_check.py && python tools/balance.py`. `guard_check.py` proves the 96 build guards each go red on a real injected break (with a name filter to re-run one case quickly). `sim.mjs` runs the BUILT scripts (a red-team run against a stale build can look green). `tools/vanilla_ref.json` is a copy of real vanilla data and every `minecraft:` string is checked against it; guessed identifier lists proved half wrong elsewhere. `art/sheet.py`, `art/mob_sheet.py` and `art/spot.py <ids...>` render art big to look at; colour clashes are re-solved by tool (`resolve_palette.py`, `pick_colour.py`), not by hand.

**Coverage sweep.** Every tool effect (~60) and every block feature is driven by at least one sim test; `grep` the sim for each effect's ids to prove it. That sweep is how a cow-dragging "Clumsy Axe" was found, and untested effects are where comment and code drift apart.

**Bug classes it produced.** A script nothing imports never runs (walk the import graph from `main.js`); a protection test that passes because something else saves it; box-UV marks painted off the visible patch; `reset()` that zeroed the fake clock but left interval deadlines in the future; a monkey-patched name that a dispatch table had already captured by reference.

**Untested.** Nothing has been in Minecraft yet. Riskiest: `item_specific_speeds` molang strings (shape-checked only; block tags cannot be verified, so a stated 14-tag allowlist applies), spawn-rule weights never tuned against vanilla.

#### THE HOLLOW (a mob-less spooky add-on with a "dread meter")

**What it is.** A spooky pack (`hollow:`) with zero real creatures: the atmosphere is the world. Six items, a settings menu, and a 5-minute "Nightmare" survive-and-loot mode.

**Key design decisions.**
- **Dread meter** 0-100 per player (dynamic property, survives relog). It rises in darkness, below Y0 and at night; it falls near light, in a ward or holding a candle. There is no light-level API in `@minecraft/server`, so `lightScore()` samples a precomputed offset list inside radius 6 for known light-emitting block ids. That heuristic is the load-bearing piece of the pack.
- Four tiers (20/45/70 thresholds) of escalating, reversible effects: whispers, footsteps, doors swinging (`open_bit` flip), a blackout (`/camera @s fade` to black), "the turn" (`/camera @s set minecraft:free` snaps the view behind the player), fake "left the game" chat, a stone cairn built behind you. At 92 the "Hollowing": 30 seconds of all light within 8 blocks put out, until you stand near light for 3 seconds.
- **Nothing is destroyed.** Snuffed torches become `minecraft:unlit_redstone_torch` (facing carried over) and are recorded; campfires get `extinguished`; built cairns are recorded; both lists are restored by a timer.
- **The camera rule:** every free-camera call must be able to give the camera back. A `cameraUntil` timestamp is force-cleared by the main loop and `playerSpawn` also clears camera and fog. A stranded free camera would make the pack unplayable.
- **The Figure** is an entity with no AI at all: no behaviours, no navigation, no attack, no loot, `is_spawnable: false`, a 0.2 collision box you walk through, not pushable, and `damage_sensor` `{"cause": "all", deals_damage: false}` (`"all"` is the enum; `"any"` breaks the whole entity file). Not in the `monster` family. All motion is `teleport()` from script. A custom geometry 2.38 blocks tall makes it taller and thinner than a player. A billboard-particle version is the fallback if `spawnEntity` fails.
- Lifecycle: a map of entity id to ticks left; a 20-tick sweep across all three dimensions removes anything expired or unrecognised, so a crash or reload cannot leave one standing.
- Damage: the watcher and hunter deal real damage via `applyDamage(n, {cause: "entityAttack", damagingEntity: ent})` with a knockback helper that tries both `applyKnockback` signatures (it changed across API versions). Timer damage is suppressed while a hunter is alive: two damage sources at once is unfair, not spooky.
- The "stalker" is a statue-style pursuer: standing still for 6 seconds makes it step closer each tick of stillness; moving resets it. It punishes stopping to look.
- **Visuals without shaders.** Custom `.material` shader packs can crash phones and consoles. Four fog definitions pushed/popped by tier plus one custom particle give the whole look at zero rendering cost.
- Settings menu (one `ActionFormData`, cycle-on-click): intensity, "Empty World" (removes `monster`-family entities within 64), Endless Night, reset.
- Sounds: mono OGG only (stereo files are not positioned in Bedrock). Intelligibility of processed voice lines was measured with speech recognition instead of guessed (the first pass scored 0% while sounding fine by RMS/peak); processed effects should be layers, not replacements, and size should come from the room, not from slowing the words. A NaN trap: filtering `abs(x)` can ring negative and `negative ** 1.15` is NaN, silently poisoning a whole file (clamp and assert `np.isfinite`).
- Geometry gotcha: long limbs overflow the vanilla skin UV layout. Box UV needs `2*depth+2*width` across and `depth+height` down; build.py checks every cube's UV bounds and every bone's parent.

**Nightmare run (5 minutes).** Sleeping starts a survive-and-loot run.
- `beforeEvents.playerInteractWithBlock` cancels the bed only at night; in the day a bed still sets spawn (cancelling always would quietly break respawn points).
- It happens where you sleep, not in a generated arena: 8 chests are placed 14-44 blocks out, filled from a weighted table and restored to their original block afterwards. You are teleported back to the bed on waking.
- Hunters ignore light (the distinction from the Hollowing): only distance helps. Each unlooted chest emits a particle and the action bar shows distance and compass bearing.
- The "ghost" state (fail a run: spectate until the next night) is a `{mode, seenDay, ticks}` dynamic property. Release needs `seenDay && isNight()` (waiting for "night" alone would release instantly, since you fail at night). `gamemode spectator` is re-asserted every second and the original mode restored. **Escape hatch:** a 10-minute real-time maximum releases the player anyway, because with Endless Night or a frozen day cycle "the next night" never arrives.

#### IT WAKES add-on and the ScareCraft world (a spooky overhaul)

**What it is.** A large spooky overhaul (`dread:`, ~46 MB, almost all audio): 24 replaced biomes, 168 authored structures, a "presence" that learns each player, challenges, an arena, relics and a script-played soundtrack. Build order matters: `gen_pack.py` -> `gen_structures.py` -> `gen_relics.py` (appends to lang/item_texture that gen_pack rewrites) -> `gen_music.py` -> `build.py`, then copy to the dev folders.

**Custom biomes.** `generate_for_climates` at weight 200 does nothing in a 1.18+ overworld; it only feeds legacy worldgen. `minecraft:replace_biomes` is mandatory:

```json
{"replacements":[{"amount":1.0,"dimension":"minecraft:overworld","noise_frequency_scale":20.0,"targets":["desert","..."]}]}
```

Targets are written without a namespace. There is no "Custom Biomes" experiment toggle in 1.26 (only Beta APIs is needed) but biomes still need a fresh world. Each biome has fog and a client biome.

**Structures.** 288 procedural features (scatter columns, spires, ruins, ore growth, hanging things) plus 168 authored `.mcstructure` files (7 designs x 24 biomes) written by a small little-endian NBT writer in `gen_structures.py`, round-trip verified. `tree_feature` was avoided because commonly pasted fields are fake.

**The presence (`brain.js`).** A persisted profile (mine/build/fight/roam/dig_deep/night), a bandit over 8 tactics scored 3 seconds after each scare by measured reaction, per-player spatial memory and inferred territory (visits plus decay), a mood machine (dormant -> curious -> stalking -> hunting -> toying -> sated), and a per-player "toughest mob" damage table.

**Design rules from the world it runs in.**
- **The "mosquito principle":** mining is fine (it is how a player gets started). Mining costs only 0.05 "awareness" per block; depth costs nothing. The grudge comes from refusal: a scare that lands with no reaction (+3), refusing a trial (+45), failing or ignoring a challenge (+20), refusing the arena. Building earns favour (-0.04 per block) but never immunity. It must not become an environment mod: no protected blocks, no build restrictions.
- Acts by `dayNumber()`: day 1 only music and a warning card; day 2 dimming and rain, presence limited to voice and whispers; day 3+ full repertoire and the arena.
- Trial: the only spoken line is "Stop digging" (7.5-minute cooldown); 3-5 weak mobs; defeat them for a reward, stop digging and it leaves, keep digging 12+ blocks and a heavier one arrives.
- Challenges matched to the player's profile every ~10 minutes with traps mid-task. Miner gauntlet: a 70-block stone corridor with a fast pursuer, reach the lantern.
- Arena as an offering: a player's favourite builder gets an Arena Button; wall a room and press inside; no room means the arena never fires. 2% roll, once per day, last one standing with escalating waves; the winners' arena spawns each player's personal toughest mob.
- 6 unbreakable relics whose levels sum (see "effects do not stack": the highest amplifier wins, so sum levels per effect in one owner loop).
- keepInventory is on in that world, so no mechanic may rely on item loss as a deterrent. Sleeping is disabled; never write text assuming sleep. A small shelter reward is once per base (keyed by a base id, not the player's grid cell, or walking pays again).
- **Music.** Vanilla music events cannot be overridden per day, so custom events `dread.trackNN` are played from script and players set the Music slider to 0. Overriding vanilla was abandoned.

**Fragile / untested.** The pack has no `tools/sim.mjs`: verification was JSON parse, cross-reference checks and `node --check` only. A real module-graph load test was never added. The arena and gauntlet were never exercised by a sim. Traps and the gauntlet both use `fillBlocks` near players (air-only filters used, but the gauntlet razes a full volume to air afterwards, which would remove anything a player built at Y207-216 overhead). `openArena` uses the first player's dimension for everyone.

#### Scary Map addon and MonsterMaker (a friends' story map with a monster editor)

**What it is.** A found-footage style spooky map: a friend group explores a forest and films clips; the map builder builds the world, the add-on supplies mechanics. Multiplayer is the design target: powers play off having buddies (dormant until you are alone, hunts whoever wandered off).

**Story mechanics.** Night 1 is empty. A **Story Start** block triggers the opening; a **Night 2 Gate** block only opens when everyone has tried a bed; then 7 nights x 10 minutes = 70 minutes, and a night only counts if at least one clip was filmed (the bed only banks a night when the clock is up AND there is footage). Other blocks: Camp Marker (no spawns within 22 blocks), Evidence Board (reads clips back), an area trigger, and an ending block. The **Escape Seal** is a solid visible unbreakable wall that only unseals once the run is won, a few blocks at a time as players approach, with scanning that starts only after the win so it costs nothing before.

**Invisible trigger blocks.** Use `render_method: alpha_test` with a fully transparent texture (`blend` can still draw a faint box). That also blanks the creative-menu icon, so each block is hidden from the menu and placed via a `minecraft:block_placer` item with a visible icon; a `/scriptevent scary:show` command finds placed ones.

**Adventure mode.** All players play in Adventure mode, so players cannot craft: the camera item must be handed out at story start and re-checked each night, or the run is unwinnable. Assume the same for anything a run requires a player to hold. Toggling the camera needs it in hand because Bedrock fires no event for an empty-handed right click; you have to look at a monster to film it.

**MonsterMaker.html** (standalone offline editor). 7 body plans, shape sliders, a 128x128 pixel skin editor with auto-paint, a live CSS-3D preview, stats/AI/powers/spawn/loot, autosave to localStorage. It exports `monsters.json` with geometry and animation baked in so the preview and the in-game model cannot disagree. Custom parts (tails as chains of tapering segments, horns, wings, extra limbs, fins, claws) attach to any bone. Parts are dragged in the 3D preview (CSS-3D faces are real DOM nodes, so picking is `e.target.dataset.pid`). `screenAxes(yaw)` and `axisOnScreen(axis, S, yaw, pitch)` map mouse movement to world axes and are unit-tested at every camera angle, because a sign error makes the editor feel broken in a way that is hard to diagnose by eye. Rigs own the top-left 64x64 of the skin sheet at their original coordinates, which is why growing to 128 did not break old skins; part patches are shelf-packed below y=64. `assemble(m)` is the single source of the bone list and UV map for preview, geometry and animation.

**Build side.** `build.py` consumes `monsters/monsters.json` and generates both packs; `src/main.js` is the only hand-written code; packs are wiped and regenerated each run; every monster gets a spawn egg for hand-placing. `tools/refresh.mjs` re-derives baked geometry from saved rigs when the rigs change. `tools/gen_test_monsters.mjs` slices real rig code out of the HTML and checks geometry and painter at slider extremes; it overwrites `monsters.json` (it is a fixture generator). `tools/sim.mjs` runs the shipped `main.js` against a stubbed Bedrock API and asserts each power fires.

**The "scarer" encounter.** A scripted one-shot spawns in front of one random player who is not already next to a friend; stare 5 seconds and it hunts you, look away and it hunts you sooner; the only escape is reaching another player. "Once" lives in a world property, but a fizzled attempt (never noticed) deliberately does not consume it, or the set piece would be burned on a player who never saw it.

**Voices.** A browser voice recorder writes `voices/<slug>/*.wav` straight into the add-on via the File System Access API; `build.py` converts to OGG with ffmpeg (mtime-cached) and emits `sound_definitions` plus a table. Line ids in the recorder and in `main.js` are a contract: change both together. Everything degrades to chat text when a line was not recorded.

**Reusable pattern.** The handoff between designer and engine is one JSON file. A power the editor can toggle but the build ignores is worse than not having it: add a feature to the editor UI and to `build.py`/`main.js` in the same pass. Not tested in game as of the first build.

#### Claude's Dreamland and Claude's Nightmare (two "one shared place" add-ons)

**Dreamland.** A single shared sky island per world (not a per-player base), reached by a Dream Lantern item: a library with a long table, a letter box for visitor notes, a question shelf (60 questions) and a guest book; a lighthouse; an observatory with a telescope (37 true star facts) and a wishing-star pedestal (wishes become real sea-lantern stars in the sky); a pond, a garden, a bridge to a quiet place, a cat called Books, monsters removed while visitors are present, day/night particle moods and per-visit greetings. Design lesson: when asked for "your" thing, design from one coherent perspective and voice, and make it a place visitors come to. A skip-rule ("skip occupied cells") can trap what it protects: a floating visitor blocked their own landing pad, so lay the pad first and retry. Building far from players with `fillBlocks` in unloaded chunks is a silent no-op; build near players and verify the floor first. Verify with `python build.py && node tools/sim.mjs && node tools/inject.mjs && python tools/render.py && python package.py`, where `inject.mjs` must report every mutation CAUGHT (STALE means a mutation stopped testing anything).

**The Same Room (Claude's Nightmare).** A spooky puzzle add-on, deliberately unlike the mob-less Hollow: one room with four doors; every door leads back into the same room. Each lap exactly one of eight things is different from how it was built; leave by the side it is on, three times running, and the door lets you out. No dread meter, no whispers, no figure, no camera tricks (check the earlier pack before adding anything to avoid repeating it). Pieces: `plan.js` (room + 8 changes + 4 subtle ones from lap 10 + sealed dead-end passages), `dread.js` (escalation: lights out, decoys that never count, a sign that lies except on laps 5 and 13, ambient sounds), `night.js` (enter, lap, leave, occupancy, refuse breaks/places), `main.js` (wiring only). You enter by carrying "The Same Key" and going to sleep, with a second path (a bed used at night, a proven event) because `Player.isSleeping` was unproven (pair any unproven API with a proven event). Invariants held: the key always works from anywhere; nobody is ever put down on a block that is not there; nothing outside the room is changed; it never touches health; one person at a time. Lesson: `node tools/playthrough.mjs` prints the experience in order as the player sees it. 297 green assertions once coexisted with a room that repeated itself four laps running; read the transcript, not the assertions.

### Other games

#### RimWorld (XML mods with optional C#)

**Setup.**
- Game 1.6 (Core, no DLC) at `...\steamapps\common\RimWorld`; mods live in `RimWorld\Mods\<ModName>` with `About/`, `Defs/`, `Patches/`, `Textures/`. Core textures are packed in Unity assets, so a mod must ship its own PNGs.
- A Python generator pattern works for every mod: `art/gen_textures.py` draws every PNG (PIL, 4x supersampling) and a contact sheet; `build.py` validates against the real Core defs, prints every player-visible label, then copies into `Mods/`. `--no-deploy` validates only; `--selftest` injects breaks (14 to 48 of them across the mods) and proves each check goes red. Deploying only stages files; enabling and launching is left to the player. Never edit `ModsConfig.xml` while the game is running.
- Generated XML must never be hand-edited (a freshness check in build.py fails it). Adding a species is one dict in `species.py` plus one feature drawer.
- **Inspecting real game code:** `dotnet tool install ilspycmd --version 8.2.0.7535 --tool-path <dir>` (the latest needs a newer .NET), then `ilspycmd -t Verse.PawnGenerator Assembly-CSharp.dll`; `-p -o dir` decompiles a whole mod DLL. Vanilla textures: `pip install UnityPy` and read `resources.assets`. Use the Windows Python, not the Git Bash one (a different install). In PowerShell `$pid` is read-only.

**Cozy Corner (furniture and food).** Lanterns, beds, recipes, all defined in XML. Bug class: RimWorld XML inheritance appends a child's `<li>` entries to the parent's list. Inheriting `LampBase` gave a lantern Electricity as a research prerequisite and a second Flickable component. build.py resolves inheritance and asserts the final research list and component classes. Not verified in game: bed rotation art (pillow end per `Rot4`), list-append behaviour beyond the resolver, recipes actually showing at stoves.

**Friendly Aliens (species without DLC).** 15 friendly species, 8 quirk traits (commonality 0.4), scenarios, wanderer incidents (0.048 each), and hybrids in `Compat/` folders loaded only through `LoadFolders.xml IfModActive`.
- Why Core-only works: xenotypes and forced genes are gated on `ModsConfig.BiotechActive`, but Core honours these `PawnKindDef` fields: `skinColorOverride`, `forcedHairColor`, `forcedTraits`, `nameMaker`, `fixedChildBackstories`/`fixedAdultBackstories`, `backstoryFiltersOverride`. A trait's `renderNodeProperties` draws head art (`PawnRenderNode_AttachmentHead`, parent tag `Head`, a 128x128 canvas identical to vanilla heads; `skipFlag Hair` hides it under hats).
- Scenarios with specific pawn kinds: Core has `ScenPart_ConfigPage_ConfigureStartingPawns_KindDefs` but no ScenPartDef for it, so define one; `pawnChoiceCount` must equal the total or spare humans are added. The game caps a start at 10 colonists (build.py checks this).
- Bug classes: `ParentName` matches a `Name=` attribute, never a `defName` (vanilla `Colonist` has no Name), so repeat the base's settings in an abstract def. Child `<li>` lists append. A partner mod's race can be tagged `AlienRace.ThingDef_AlienRace`, so a validator that only indexes `ThingDef` says the race is missing. A trait label can collide with a vanilla one. A self-test mutation that leaves an equivalent fault behind proves nothing; mutate to a real fault. Distinguishability is measured (skin/skin 70, hair/hair 50 for 15 species, hair/skin 60 RGB distance, plus different stat sets).
- The load order of a big mod list can throw red XML errors that are not from your mod (a life stage that only exists with Biotech broke another mod's world generation); disable other mods in turn to test.
- **C# add-on (needs Harmony).** `Source/FriendlyAliens` builds `Assemblies/FriendlyAliens.dll` (net472, `dotnet build -c Release`, references Assembly-CSharp, Unity modules, the workshop Harmony's `0Harmony.dll` and NuGet `Microsoft.NETFramework.ReferenceAssemblies`). Two Harmony patches: a postfix on `Pawn.GetGizmos` adds a "Have a child" command, and a postfix on `StartingPawnUtility.DrawPortraitArea` draws gender buttons that set `PawnGenerationRequest.FixedGender`. Without Biotech there are no baby stages, so the child arrives adult (a tuning constant). Inheritance logic (`Blend.cs`, `Lineage.cs`) is pure C# with no game types so a plain console test project links the same file (13 then 21 tests plus injected faults that must fail). Tuning numbers live in `species.py` and are generated into `Tuning.g.cs`, so XML and C# cannot drift; build.py also checks every `"FA_..."` string in the C# is a real def. `StartingPawnUtility.StartingAndOptionalPawns` is private; use `Find.GameInitData.startingAndOptionalPawns`. Compiling against the real DLL is the API check. Lineage rule: same species -> pure; mixed species or a human parent -> a hybrid trait.
- Not verified in game: incident rate, west-facing art (the game mirrors east), the pawn-kinds scenario page (most likely to need a fix), custom-colour tinting, hybrids, the character-screen button layout (estimated from a screenshot), the gizmo, the letter text, whether `def.ConflictsWith` and `GainTrait` behave as read from the decompile, and the child render tree refreshing after traits are added.
- Tooling gotcha: heredocs through Git Bash mangle backslash-n inside Python strings (a `\n` became a real newline and broke a data file); write such scripts with a file-writing tool instead.

**Hive Friends (furniture and tameable animals on top of another mod).** XML-only, depends on a parent race mod, so it is furniture, floors, a research project, and five gentle tameable animals. Costs use the partner's item names and build.py prices each thing from the partner's real market value (things 15-260, floors 2-30 per tile). Animals are copied from Core's `Hare`/`Megascarab`: `AnimalThingBase`, `AnimalKindBase`, `hasGenders false` (so `milkFemaleOnly` must be false), `useMeatFrom Megaspider`, `LeatherAmount 0`, life stages AnimalBaby/Juvenile/Adult, plus `dessicatedBodyGraphicData`. In 1.6 an animal declares its own biomes:

```xml
<wildBiomes><TemperateForest>0.12</TemperateForest></wildBiomes>
```

with no BiomeDef patch. build.py checks body, food type, life stages, sounds, biomes, trade tags, tool groups, textures, manhunter chance (<= 0.05), pack animals needing `CarryingCapacity` and daily production value 1..20 (36 injected breaks). A kid-safe wording filter bans scary words even in "no gore" sentences; the parent mod is spooky, so the child-friendly versions are gentle. Real animals cannot be drafted or do colonist work, and a full humanlike race needs full body and head art, so the "colonist" form is the Human race plus a species trait and head art. Not verified in game (the parent mod cannot fully load on a Core-only install).

**Parent-mod fact worth remembering:** a mod's end-game trigger that grows daily can re-fire every day; a settings slider (growth factor 0) stops it.

#### Farming Simulator 22

- Steam v1.14. One folder per mod (`FS22_*`); `python build.py [name]` zips to `dist/` and copies into the game's mods folder. The log shows `ModDesc Version: 80`, so use `descVersion="80"`. The game scans mods only at launch, so relaunch to pick up changes; copying fails or goes stale while the game is running.
- Lua source is packed in `dataS.gar` (not readable). Ground truth for data: `<install>\data\maps\maps_fruitTypes.xml`, `maps_fillTypes.xml` and `shared\xml\schema\*.xsd`. Verify Lua logic by stubbing the engine in `lupa` (pip) against those real files.
- Built so far: a hello-world mod, a key-bound mod with random news, a yield mod (x3 yield via `fruit.literPerSqm`, x1.25 price) and a re-badging mod (sorghum shown as "Alien Wheat", grass as "Alien Smoky Grass").
- **Hard limit:** the three base maps (US/FR/Alpine) have exactly 18 baked foliage layers and no spare ones, so a truly new growable crop cannot come from a script mod; it needs a custom map made in Giants Editor. Offer re-badging existing crops or a Giants Editor map project.
- Untested in game (as of the time of writing): `fillType.title` rename, `fruit.literPerSqm`, `fillType.pricePerLiter`. Check `log.txt` lines starting with your mod's tag. Do not run two yield mods together: the boosts stack.

#### Borderlands 2

- Real mods are console `set` files (BLCMM format), not a custom `#header` format. A draft that used invented object names (`PopDef_VarkidAdult`, `WeaponPool_Legendary`, `HealthAttributeModifierMultiplier`) would fail silently in the console, because guessed names simply do nothing.
- Verify names by regex-scanning `WillowGame\CookedPCConsole\*.upk` for `GD_[A-Za-z0-9_]+\.[...]`. Item pools (`GD_Itempools.*`) are readable this way (Pearlescent weapons are `Pool_Weapons_All_05_VeryRare_Alien`, legendary are `Pool_Weapons_All_06_Legendary`). PawnBalance and creature balance objects are compressed and cannot be found this way; find them in game with `obj list AIPawnBalanceDefinition`.
- A vanilla install has no console enabled and no BLCMM; a mod file keeps a clearly named placeholder until the real object name is confirmed, and a health/damage buff waits until the field name is verified. Say which names are unverified.

#### Bloons TD 6 (MelonLoader, BTD Mod Helper, Harmony, IL2CPP)

**Setup.**
- Steam game (BTD6 56.1, Unity 6000, Il2Cpp x64), MelonLoader 0.7.3, BTD Mod Helper 3.6.8 (`Btd6ModHelper.dll` in `Mods/`, `UpdaterPlugin.dll` in `Plugins/`), .NET 8 SDK (winget can die with exit 1602 if the UAC prompt is dismissed; download the SDK installer and launch it so elevation can be clicked). No Visual Studio needed: `dotnet build`.
- Projects must live in `Documents\BTD6 Mod Sources\<ModName>\`: the `.csproj` does `<Import Project="..\btd6.targets" />` and that targets file is auto-generated there by Mod Helper on first run (resolves game DLL references, embeds PNGs, post-build-copies the dll into `Mods/`). Target `net6.0` even though the SDK is 8. Scaffold new mods with Mod Helper's in-game creator rather than hand-copying the template.
- Build: `dotnet build <csproj> -c Release`. **Close BTD6 before rebuilding**: a running game locks `Mods\<mod>.dll`, the copy fails with `MSB3061 ... Access ... denied` while compilation succeeds, so `bin\Release` is fresh and `Mods\` silently keeps the old dll. Compare timestamps before testing.
- The template does not enable implicit usings: add `using System.Collections.Generic;` and `using System.Linq;`.
- Reading real field names: run Mono.Cecil (ships in `MelonLoader\net6`) over `MelonLoader\Il2CppAssemblies\Assembly-CSharp.dll` and list Properties, not Fields (Il2Cpp interop exposes fields as properties and leaves only `NativeFieldInfoPtr_*` as fields). When a type will not resolve, find its namespace with Cecil (`GetTypes() | Where Name -eq X | select FullName`); for instance `BloonProperties` lives in the bare `Il2Cpp` namespace.
- **MelonLoader installer can succeed while skipping files.** In one install 7 files were skipped (5 `MonoMod.*` DLLs plus 2 in `Dependencies/MonoBleedingEdgePatches`). Symptom: game will not start and `MelonLoader\Latest.log` ends with `Could not load file or assembly 'MonoMod.RuntimeDetour'`. Fix: download the same version's zip and `Compare-Object` file names in `net6`, `net35`, `net472` and `Dependencies` (correct counts for 0.7.3: 56, 26, 30, 47); copy back what is missing. Version numbers in the error are a red herring. Escape hatch: MelonLoader injects only via `version.dll` next to the game exe; rename it and the game is vanilla again.

**The mental model.** BTD6 is data-driven. A tower is a tree of plain data (`TowerModel -> AttackModel -> WeaponModel -> ProjectileModel -> DamageModel/behaviours`). Modding is cloning an existing tree and mutating nodes, or `Duplicate()`-ing a behaviour off another tower and grafting it on. An upgrade is a function that edits the tower model: `ModUpgrade.ApplyUpgrade(TowerModel)`. Keep Model (blueprint, where ~90% of the work happens) separate from Simulation (`Tower`, `Bloon`, live objects); editing a model mid-round does nothing until it is re-read, and changing live behaviour means Harmony patches.

Useful calls: `Game.instance.model.GetTowerModel(TowerType.GlueGunner, 0, 0, 0)` (any tower at any tiers), `.GetAttackModel().weapons[0].projectile`, `.GetBehavior<SlowModel>()`, `.Duplicate()`, `.AddBehavior(...)`. Confirm the upgrade path before copying a behaviour (a dart monkey's Triple Darts is 0-3-0, the middle path; copying the wrong one fails silently), and prefer scanning for the behaviour you want and logging when it is not found.

**Freeze safety (bugs that stop the game with no error).**
1. Every duration is stored twice, in seconds and in frames at 60 fps: `WeaponModel.rate`/`rateFrames`, `SlowModel.lifespan`/`lifespanFrames`, `BloonModel.speed`/`speedFrames`, `TowerExpireModel.lifespan`/`lifespanFrames`, `TimeTriggerModel.interval`. Never scale the two independently: `rateFrames = (int)(rateFrames * factor)` across stacked upgrades truncates to 0, meaning firing or spawning every frame. Set both from one clamped number:

```csharp
private static void SetRate(WeaponModel weapon, float seconds, float floor)
{
    var clamped = Math.Max(seconds, floor);
    weapon.rate = clamped;
    weapon.rateFrames = Math.Max((int) Math.Round(clamped * 60f), 1);
}
```

2. Anything summoned must be able to expire. Vanilla sentries carry a `TowerExpireModel`; some other summons do not. Copy the expiry from the original summon, or fall back to the vanilla Sentry's.
3. Do not free-spawn ground pet towers (they expect track or water placement driven by a pet model; spawned standalone they froze the game and rendered as green placeholder cubes). Safe creature summons are units the game already spawns temporarily (`WizardPhoenix`, `DarkPhoenix`, `EtienneDrone`, `Sentry*`, and the `Gyrfalcon` air unit).
4. Clamp anything copied off a boss: cap `spawnCount`, `stunDuration`, radii and trigger `interval`, and allow-list which actions come across (leave `DrainLivesActionModel` and `SellTowersInRadiusActionModel` behind).
5. A generic stat buff must know which weapons are spawners. Detect summon weapons (projectile has `CreateTowerModel` or `CreateTypedTowerModel`), floor them at 2 seconds separately and skip them for pierce/damage.

**Il2Cpp null-check trap.** Some Il2Cpp types overload `operator ==` and the overload converts its operand to a native pointer via `Il2CppObjectBaseToPtrNotNull`, which throws `NullReferenceException` on `null`. So the null check is what crashes (confirmed on `PrefabReference`: every tower and bloon in one mod failed to register, and the stack pointed at `if (prefab == null) return;`). Use `is null` / `is not null` (pattern matching compiles to a direct reference comparison); `?.` and `??` are safe. Plain Model subclasses have been fine with `== null` across five mods; when in doubt `is null` costs nothing. Diagnose with the full stack trace: do not grep the log for your own mod name, that hides the `Il2CppInterop` frames that are the entire answer.

**Mod Helper API gotchas.**
- Art filenames differ per type: `ModTower.Icon` is `"[Name]-Icon"`, `ModTower.Portrait` is `"[Name]-Portrait"`, `ModUpgrade.Icon` is `"[ClassName]-Icon"`, and `ModBloon.Icon` is the bare class name (`GladosBloon.png`, not `...-Icon.png`; getting it wrong made a bloon render nothing). `ModParagonUpgrade` forces its own internal `Name`, so pin `Icon` and `Portrait` explicitly. Verify by dumping the getter's IL with Cecil.
- Sealed (cannot override): `ModHero.TowerSet`, `ModHeroLevel.Icon` (name PNGs `Level2-Icon.png` and so on). Obsolete: `ModHero.Abilities`, `ModBloon.PixelsPerUnit`.
- `ParagonMode` is `None` / `Base000` / `Base555` (no `Base`). A Paragon requires a full 5/5/5 tower: at 2/2/5 the vanilla upgrade screen threw `NullReferenceException` and drew no Paragon node.
- **Windows filenames are case-insensitive.** In a mod whose folder/class is `gladosbloon`, a file `GladosBloon.cs` is the same file as `gladosbloon.cs` and silently overwrites the mod's main class, so the whole mod stops loading. Name content files something distinct (`Bloon.cs`, `Tower.cs`).
- `ModifyBaseTowerModel` runs once per tier combination (dozens of times at 5/5/5), so gate diagnostic logging behind a `static bool loggedOnce`.
- A third-party mod that enumerates the seven vanilla boss types can spam `KeyNotFoundException` when it meets custom content; disable it when diagnosing anything boss-related.

**Harmony patching (when there is no field to flip).** Some rules are enforced in game code, not model data. Example: only one Tier 5 of a given upgrade can be owned. Find the method with Cecil (`TowerInventory.IsPathTierLocked(Tower tower, int path, int tier)` returns bool), read its exact signature (parameter names must match for Harmony to bind), and postfix it, only ever loosening the result for your own content:

```csharp
[HarmonyPatch(typeof(TowerInventory), nameof(TowerInventory.IsPathTierLocked))]
internal static class TowerInventory_IsPathTierLocked
{
    [HarmonyPostfix]
    private static void Postfix(Tower tower, ref bool __result)
    {
        if (!__result) return;   // never tighten, only loosen
        if (tower?.towerModel?.BaseId == ModContent.GetInstance<GlueDartMonkey>().Id)
            __result = false;
    }
}
```

MelonLoader runs `PatchAll` on the mod assembly at load, so the class existing in the project is enough. `ref bool __result` is Harmony's magic name for the return value; `__instance` gets the receiver. Guard on `BaseId` against your own tower's id so vanilla towers keep their restrictions, and early-return when the result is already false so you never re-enable something another mod locked.

**BTD6 case studies (all six mods; none of them were verified by long play).**
- **Glue Dart Monkey (first mod).** A Dart Monkey whose darts glue bloons: `SlowModel` (in `Il2CppAssets.Scripts.Models.Towers.Projectiles.Behaviors`) is what makes glue glue (`lifespan`, `multiplier`, `glueLevel`, `mutationId`).
- **SWARM Summoner Monkey.** Magic-set tower ($900) built on the Engineer, with 5/5/5 paths (beasts, buffs, sentries) and a $400,000 Paragon. Summoning needs no custom spawning code: the Engineer already fires a projectile with a `CreateTowerModel` behaviour whose `tower` field is a `TowerModel`, so `createTowerModel.tower = Game.instance.model.GetTowerWithName(<id>).Duplicate()` swaps what is summoned. Reach them with `towerModel.FindDescendants<CreateTowerModel>()`. Real spawnable ids on `TowerType`: `Sentry`, `SentryBoom`, `SentryCrushing`, `SentryCold`, `SentryEnergy`, `SentryParagon`, `Gyrfalcon`, also `TechBot`, `AmbushBot`, `ShootyTurret`, `Marine`, `Pontoon`. Two debugging wins: (1) the 0-0-0 Engineer has no sentry behaviour at all (it belongs to the top-path tier 1), so the machinery must be grafted from the 1-0-0 model or every summon helper silently does nothing:

```csharp
var withSentry = Game.instance.model.GetTowerModel(TowerType.EngineerMonkey, 1, 0, 0);
foreach (var attack in withSentry.GetAttackModels())
    if (attack.weapons.Any(IsSummonWeapon)) towerModel.AddBehavior(attack.Duplicate());
```

(2) there are two spawn behaviours with no shared interface: `CreateTowerModel` (single `tower`) and `CreateTypedTowerModel` (`crushingTower`, `boomTower`, `coldTower`, `energyTower`); write one `ForEachSummonSlot(model, Func<TowerModel,TowerModel>)` helper. Two beast summons froze the game and were replaced by phoenixes (see freeze safety).
- **Glue God hero.** A hero built on Quincy re-armed with the Glue Gunner's real `SlowModel`, 20 levels. Heroes are towers with a level ladder: `ModHero : ModTower` and `ModHeroLevel : ModUpgrade`, keyed by `Level` (level 1 is the base model, so classes start at level 2). An abstract shared base class (`GlueGodLevel`) is skipped by Mod Helper's content scan, useful for common helpers across all 19 level classes. Level icon PNGs must be named for the level class. Not implemented: real abilities and select-screen portraits.
- **GLaDOS Bloon (custom boss-like bloon).** Design history: (1) a Ceramic-type with a custom icon was invisible (wrong icon filename) and had no special behaviour; (2) a real boss as the base registered but boss bloons cannot be spawned from Sandbox at all (only by boss events); (3) final: based on a plain bloon type with 60,000 health and boss behaviours grafted on. Enumerate bloons at runtime (`Game.instance.model.bloons`, `GetBloon(id)`); boss ids are not constants anywhere, so scan for them. Bloon behaviours are a two-part trigger/action system: `TimeTriggerModel` (`interval`, `triggerImmediately`, `actionIds`) references separate action behaviours on the same bloon by id. Copy the actions first, record which ids exist, then copy the timers and strip any `actionId` whose action did not come across (a dangling reference). **The link field is `BloonBehaviorActionModel.actionId`, not `Model.name`** (`name` is empty; reading it produced an empty id set, every timer looked dangling, actions had nothing to fire them and no error was logged). When copying from more than one boss, re-prefix ids (`glados_{boss}_{actionId}`) and remap the timers. Useful actions: `SpawnBloonsActionModel`, `StunTowersInRadiusActionModel`, `DamageBloonsInRadiusActionModel`. Sandbox hiding is `BloonModel.dontShowInSandbox` / `dontShowInSandboxOnRelease`.
- **DDT Destroyer Monkey.** A deliberately absurd $100,000 Magic-set tower built on the Super Monkey (5/5/5, up to $1,000,000). A DDT is Camo + Lead + Black and fast, so three things must be true at once. The recipe: (1) strip damage immunities on every projectile, setting both fields:

```csharp
var damage = weapon.projectile.GetBehavior<DamageModel>();
damage.immuneBloonProperties = BloonProperties.None;
damage.immuneBloonPropertiesOriginal = BloonProperties.None;   // set BOTH
```

(2) camo in both halves: `projectile.SetHitCamo(true)` for hitting, and `FindDescendants<FilterInvisibleModel>()` with `isActive = false` for targeting; (3) enough damage. `BloonProperties` is a flags enum (`None=0, Lead=1, Black=2, White=4, Purple=8, Frozen=16, Immune=32, Glass=64`); `None` means immune to nothing. Re-assert the stripping at the end of every upgrade so a later model change cannot restore an immunity. `DamageModifierForTagModel` and friends are how vanilla does type bonus damage.
- **ANTIBLOONS (inversion).** Bloon towers that fight for you (Dart Monkeys wearing bloon models: a colour ladder from Blue $400 to Ceramic $7,800, all 0/0/0 so the ladder is the progression) and "anti-monkeys" (bloons wearing monkey models that stun towers every 5-6 seconds). A model swap is a `PrefabReference` assignment: no asset bundles. The two sides are not symmetrical: a bloon has `BloonModel.display` directly; a tower's visual lives on a `DisplayModel` behaviour (`towerModel.GetBehavior<DisplayModel>().display`, with `scale`, `positionOffset`, `layer`; in `Il2CppAssets.Scripts.Models.GenericBehaviors`), reached with `FindDescendants<DisplayModel>()`. When disguising a bloon also clear `BloonModel.damageDisplayStates` or it breaks the disguise on the first hit. Scale matters (~2.5x for a Red bloon on a tower, ~1x for a MOAB). Make a bloon attack towers by grafting `StunTowersInRadiusActionModel` + `TimeTriggerModel` off a vanilla boss (set the interval explicitly rather than scaling an inherited one). An abstract `BloonColorTower` base class with hook properties (`BloonId`, `BloonScale`, `ExtraPierce`, `ExtraDamage`, `RateScale`, `ExtraRange`) is skipped by Mod Helper's scan so only concrete colours register; use `BloonType.sBlue` and so on instead of raw id strings. A "tech" family adds a generic bottom-path upgrade that calls an `AmplifyTech` hook on the tower type so one upgrade does something different on every tower; its stat helper skips summon weapons for pierce and floors their rate, while the ladder's helper does not (never reuse it on a tower with summons).

#### Tooling notes from these projects

- **Text-to-3D on a small GPU.** Big open-source text-to-3D generators (TRELLIS, Hunyuan3D-2) need 6-16 GB of VRAM and cannot run on a 4 GB laptop GPU. Shap-E (openai/shap-e) is the one that fits (it can run on CPU) but its output is blobby, very high-poly and static; a hardware ceiling, not a bug. Poly Pizza was the only free model site that worked as a fallback. After getting an OBJ, the pipeline is OBJ -> Blockbench-style bridge -> Bedrock geometry -> wire into the mob (custom mobs are static unless vanilla animation controllers are wired in).
- **Input automation safety.** Never launch input-hijacking automation (auto clickers, macro bots) until you have confirmed it can be stopped. One auto-clicker had a "turbo" mode on by default that fired 200 clicks per batch, which made its own GUI Stop button unreachable. Read the script, identify the stop mechanism, prefer hotkey-controlled variants (hold-to-click or a toggle hotkey) and state the kill method (hotkey or `Stop-Process`) in the same message that launches it. Reading a script is not the same as verifying it is escapable.
