# Extending the D&D Rules System

## Architecture Overview

The rules system is split into **ruleset-specific data** and **shared logic**:

- `src/lib/dnd.ts` — all shared TypeScript types and interfaces (`Ruleset`, `ClassDef`, `Species`, `Spell`, etc.)
- `src/lib/dnd/rulesets.ts` — the registry; lists every available ruleset and provides `getRuleset(id)`
- `src/lib/dnd/srd51.ts` / `srd52.ts` — each ruleset's implementation: assembles data from submodules and implements the `Ruleset` interface (spell progression, slot calculations, etc.)
- `src/lib/dnd/srd51/core.ts` / `srd52/core.ts` — the raw data files: species, classes, backgrounds
- `src/lib/dnd/spells/srd51.ts` — spell data, shared across rulesets
- `src/lib/dnd/beasts/srd51.ts` / `beasts/srd52.ts` — beast data per ruleset
- `src/lib/dnd/itemTemplates/` — weapon and armor templates per ruleset

Each character row in the database has a `ruleset` column (`"srd51"` or `"srd52"`). `getRuleset(id)` returns the correct `Ruleset` object for all game logic.

---

## Adding Species, Classes, or Backgrounds to an Existing Ruleset

Edit the relevant core data file directly:

- SRD 5.1 (2014): `src/lib/dnd/srd51/core.ts`
- SRD 5.2 (2024): `src/lib/dnd/srd52/core.ts`

Each file exports three things:

```typescript
const SpeciesData: Species[] = [ ... ]
const ClassData: Record<ClassNameType, ClassDef> = { ... }
const BackgroundData: Record<string, Background> = { ... }

export default { species: SpeciesData, classes: ClassData, backgrounds: BackgroundData }
```

### Adding a Species

```typescript
// in srd51/core.ts — SpeciesData array
{
  name: "halfling",
  description: "...",
  size: "small",
  speed: 25,
  traits: [
    { name: "lucky", description: "When you roll a 1 on a d20, you can reroll..." },
    { name: "brave", description: "Advantage on saving throws against being frightened." },
  ],
  lineages: [
    {
      name: "lightfoot",
      description: "Lightfoot halflings can easily hide...",
      traits: [{ name: "naturally stealthy", description: "..." }],
    },
  ],
}
```

Species fields:
- `lineages` — optional sub-variants (like dragonborn ancestry). Each lineage can have its own `traits` and `abilityScoreModifiers`.
- `abilityScoreModifiers` — SRD 5.1 only; `{ dexterity: 2 }` format.
- `traits[].level` — optional; indicates the level at which the trait is gained.

### Adding a Class

`ClassNameType` is a fixed enum in `src/lib/dnd.ts`. Adding a new class name requires updating that enum first — it is intentionally restrictive to the 12 core D&D classes. To add homebrew classes, see **Adding a New Ruleset** below.

To modify an existing class's data:

```typescript
// in ClassData record
fighter: {
  name: "fighter",
  description: "...",
  hitDie: 10,
  primaryAbilities: ["strength", "dexterity"],
  savingThrows: ["strength", "constitution"],
  armorProficiencies: ["light", "medium", "heavy", "shields"],
  weaponProficiencies: ["simple", "martial"],
  toolProficiencies: [],
  skillChoices: { choose: 2, from: ["acrobatics", "athletics", ...] },
  traits: [
    { name: "second wind", description: "..." },
    { name: "action surge", description: "...", level: 2 },
  ],
  subclasses: [
    {
      name: "champion",
      description: "...",
      traits: [{ name: "improved critical", description: "..." }],
    },
  ],
  subclassLevel: 3,
  spellcasting: { enabled: false },
}
```

Spellcasting config for a caster class:

```typescript
spellcasting: {
  enabled: true,
  kind: "full",           // "full" | "half" | "third" | "pact"
  ability: "intelligence",
  changePrepared: "longrest",  // "longrest" | "levelup"
  subclasses: ["arcane trickster"], // optional: subclasses that grant spellcasting
}
```

### Adding a Background

```typescript
// in BackgroundData record — key is lowercase background name
acolyte: {
  name: "Acolyte",
  description: "...",
  skillProficiencies: ["insight", "religion"],
  additionalLanguages: 2,
  traits: [
    { name: "shelter of the faithful", description: "..." },
  ],
}
```

---

## Adding Spells

All spells live in `src/lib/dnd/spells/srd51.ts` (SRD 5.1) as a single exported array. There is currently no separate SRD 5.2 spell file; srd52 uses the same spell list.

```typescript
// in spells/srd51.ts — spells array
{
  id: "srd_fireball",          // unique, snake_case
  name: "Fireball",
  level: 3,                    // 0 = cantrip
  school: "evocation",
  classes: ["sorcerer", "wizard"],
  briefDescription: "20-foot-radius explosion dealing fire damage",
  description: "A bright streak flashes from your pointing finger...",
  atHigherLevelsText: "The damage increases by 1d6 for each slot level above 3rd.",
  castingTime: { type: "action" },
  range: { type: "distance", feet: 150 },
  components: {
    verbal: true,
    somatic: true,
    material: { description: "a tiny ball of bat guano and sulfur" },
  },
  duration: { type: "instantaneous" },
  target: { type: "area", area: { shape: "sphere", radius: 20, origin: "point" } },
  resolution: { kind: "save", ability: "dexterity", onSuccess: "half" },
  damage: [{ type: "fire", dice: [6, 6, 6, 6, 6, 6, 6, 6] }], // 8d6
  source: "SRD",
}
```

If you add a spell file for a new ruleset, add it to `src/lib/dnd/spells.ts`:

```typescript
import * as _spellsSrd51 from './spells/srd51'
import * as _spellsMyRuleset from './spells/myRuleset'  // add this
export const spells: Spell[] = [_spellsSrd51, _spellsMyRuleset].flatMap((m) => m.spells ?? [])
```

---

## Adding Beasts

Beast data lives in `src/lib/dnd/beasts/srd51.ts` and `beasts/srd52.ts`. Add entries to the appropriate array:

```typescript
{
  id: "srd51_wolf",           // unique; prefix with ruleset
  name: "Wolf",
  size: "medium",
  cr: 0.25,                   // use decimals: 0.125=1/8, 0.25=1/4, 0.5=1/2
  xp: 50,
  ac: 13,
  acType: "natural armor",
  hitPoints: 11,
  speed: { walk: 40 },
  abilities: {
    strength: 12, dexterity: 15, constitution: 12,
    intelligence: 3, wisdom: 12, charisma: 6,
  },
  skills: { perception: 3, stealth: 4 },
  senses: "passive Perception 13",
  traits: [
    { name: "keen hearing and smell", description: "Advantage on Perception checks using hearing or smell." },
    { name: "pack tactics", description: "Advantage on attack rolls if an ally is adjacent to the target." },
  ],
  actions: [
    {
      name: "Bite",
      attackType: "melee",
      attackBonus: 4,
      reach: 5,
      damage: { type: "piercing", dice: [6, 6], flat: 2 },
      description: "On hit: DC 11 Strength save or target is knocked prone.",
    },
  ],
  source: "srd51",
}
```

`getBeasts()` and `getBeastsFilteredByCR()` in `beasts.ts` are the access points used throughout the app.

---

## Adding Item Templates (Weapons & Armor)

Item templates pre-populate the character inventory creation form. They live in `src/lib/dnd/itemTemplates/`:

- `srd51Weapons.ts` / `srd52Weapons.ts` — weapon templates
- `srd51Armor.ts` / `srd52Armor.ts` — armor templates

```typescript
// Weapon example
{
  name: "Longsword",
  category: "weapon",
  weapon_type: "melee",
  damage: [
    { num_dice: 1, die_value: 8, type: "slashing" },
    { num_dice: 1, die_value: 10, type: "slashing", versatile: true }, // two-handed
  ],
  martial: true,
  finesse: false,
}

// Armor example
{
  name: "Chain Mail",
  category: "armor",
  armor_type: "heavy",
  armor_class: 16,
  armor_class_dex: false,
  min_strength: 13,
  stealth_disadvantage: true,
}

// Shield example
{
  name: "Shield",
  category: "shield",
  armor_modifier: 2,
}
```

---

## Adding a New Ruleset

1. **Create the data file** `src/lib/dnd/myRuleset/core.ts` following the same structure as `srd51/core.ts`.

2. **Create the ruleset implementation** `src/lib/dnd/myRuleset.ts` — implement the `Ruleset` interface from `src/lib/dnd.ts`. Copy `srd51.ts` as a starting point; you must implement `maxCantripsKnown`, `maxSpellsPrepared`, and `getSlotsFor`.

3. **Register it** in `src/lib/dnd/rulesets.ts`:
   ```typescript
   import myRuleset, { MY_RULESET_ID } from './myRuleset'

   export const RulesetIdSchema = z.enum([SRD51_ID, SRD52_ID, MY_RULESET_ID])
   export const RULESETS: Ruleset[] = [srd51, srd52, myRuleset]

   export function getRuleset(ruleset: RulesetId): Ruleset {
     if (ruleset === MY_RULESET_ID) return myRuleset
     if (ruleset === SRD52_ID) return srd52
     return srd51
   }
   ```

4. **Add a migration** if you need a new `ruleset` enum value in the database:
   ```bash
   mise run dbmate new add_myRuleset_to_characters
   ```

> **Important:** Do not use `import.meta.glob` to load data files — it is not supported when running with `bun run` directly (only in the bundler). Always use explicit static imports as shown in `srd51.ts`.
