# Food App — Spec (v1)

A personal recipe bank: store recipes, scale them up or down, and search by the ingredients you have. Built on the same framework as Iron-Log (github.com/geni4a/Iron-Log).

## 1. Goals

- **Recipe bank:** ingredients (amount, unit, form, prep note) and step-by-step instructions.
- **Scaling:** halve, quarter, double, scale by servings, or scale from the amount of one ingredient you have.
- **Ingredient search:** enter what you have, get recipes ranked by % match, with missing items and possible substitutes listed.
- **Ingredient dictionary:** one standard entry per ingredient, arranged in a kind hierarchy, with forms kept as a separate attribute. Fixes naming and matching problems.
- **Cook history:** log when you cooked something, at what scale, and how it went.

**Not in v1:** importing from recipe URLs (needs a backend), sync across devices, nutrition, meal planning, shopping list (planned for v2; the data model already supports it).

## 2. Platform (same as Iron-Log)

- A single `index.html` web app that installs on a phone (`manifest.json`) and works offline (`sw.js` keeps a cached copy).
- Data is stored on the device in IndexedDB, using Iron-Log's helpers (`getAll`, `getOne`, `putOne`, `deleteOne`, …).
- Hash-based navigation with a bottom tab bar.
- Backup: export everything to one JSON file and import it back.
- First launch loads a starter ingredient dictionary and some sample recipes (like Iron-Log's `maybeSeed`).

## 3. Data model (IndexedDB stores)

### `ingredients` — the dictionary
```
{
  key: "habanero",                 // unique id (lowercase-slug)
  display: "Habanero",
  parent: "chili-pepper" | null,   // kind hierarchy (strict tree)
  related: ["scotch-bonnet"],      // optional cross-links, partial credit
  aliases: ["habanero pepper"],
  forms: ["fresh", "dried"],       // forms this ingredient can take
  form_overrides: { "fresh>dried": 0.8 },   // optional, beats the default form table
  form_ratios:   { "fresh>dried": 0.33 },   // optional, for "use 1 tsp dried instead"
  staple: false,                   // staples weigh 0 in search
  aisle: "produce",                // for the v2 shopping list
  density_g_per_cup: null          // optional, for future volume↔weight conversion
}
```

**Rules:**
- **Kind hierarchy:** a strict tree with one parent each, 2–3 levels deep. Example: chili pepper → jalapeño → chipotle.
- **Ambiguous names point to several entries.** "pepper" is an alias of black pepper, chili pepper and bell pepper, so entering it asks which one you mean.
- **Kind vs. form:**
  - If swapping it changes *what the ingredient is* (habanero vs. jalapeño), it goes in the **tree**.
  - If it changes *how it's prepared or preserved* (fresh, canned, dried, frozen, ground, paste), it's a **form**.
- **Parts** (juice, zest, yolk, white) are stored as forms with a form score of 0 between them. A whole lemon covers both juice and zest.

### `recipes`
```
{
  id, title, base_servings, tags: [], prep_min, cook_min,
  source, notes, created, last_cooked,
  ingredients: [
    { key: "tomato", form: "canned" | "any", qty: 400, unit: "g",
      note: "crushed", role: "main" | "supporting" | "garnish",
      optional: false, section: "sauce", scale: "linear" | "taste" | "none" }
  ],
  steps: ["Simmer {0} with {2} for 20 min…"]   // {n} = ingredient index, scales with the recipe
}
```

### `history`
`{ id, recipe_id, date, scale, rating (1–5), notes }`, with an index on `recipe_id`.

### `settings`
Unit preference, the staples list, the default form-score table, and similar.

## 4. Scaling

- **Computed when displayed, never saved.** The stored recipe always stays at its base size.
- **How you choose the size:**
  - Factor chips: ¼ · ⅓ · ½ · 1 · 2 · 3.
  - "Serves N".
  - "I have X of [ingredient]": the factor is X ÷ the recipe's amount of that ingredient.
- **Display:**
  - Fractions instead of decimals (⅛, ¼, ⅓, ½, ⅔, ¾).
  - Switch to a sensible unit (0.125 cup → 2 tbsp; 24 tsp → ½ cup; 1000 g → 1 kg).
  - Whole items (eggs, cloves, cans) are rounded, with a note when the rounding matters ("1.5 eggs → 1 egg + 1 yolk, or round to 2").
- **Inline amounts in steps:** `{n}` tokens in the steps are replaced with the scaled amount.
- **Warnings:** lines marked `scale: "taste"` (salt, spice) or `"none"` (pan size, bay leaf), and baking/leavening, get a ⚠ note when the factor ≠ 1. Cook times are not scaled.
- **Units:** metric and US units are both supported, and conversion stays within the same kind of measure (volume↔volume, weight↔weight). Converting volume↔weight needs `density_g_per_cup` and waits until later.

## 5. Ingredient search

**Input:** a list of ingredients you have, each with an optional form (no form = matches any form).

**Per-ingredient match score** = `kind_score × form_score`, taking the best of everything you have.

| kind_score | When |
|---|---|
| 1.0 | You have the same ingredient or a more specific one (recipe needs chili pepper, you have habanero) |
| sibling_score (default 0.5) | You have a sibling (recipe needs habanero, you have scotch bonnet). Each parent sets its own `sibling_score` — 0.85 for pasta shapes, 0.25 for herbs. Cousins fade 20% per extra level. |
| 0.5 | You have a `related` item (lemon ↔ lime) |
| 0.4 / 0.2 | You have the parent / a grandparent (recipe needs habanero, you have "chili pepper") |
| 0 | Different branch |

| form_score (defaults; per-ingredient `form_overrides` win) | Value |
|---|---|
| Same form, or either side is "any" | 1.0 |
| fresh ↔ frozen | 0.9 |
| fresh ↔ canned, fresh ↔ dried | 0.7 |
| whole ↔ ground | 0.8 |
| whole/fresh ↔ paste | 0.2 |
| Part ↔ different part (juice ↔ zest) | 0 |

**Recipe score (weighted coverage):**
```
coverage = Σ weight(line) × match(line)  /  Σ weight(line)
weights:  main 3 · supporting 2 · garnish 1 · staple 0 · optional 0
```

**Results:**
- Each result shows: the % match, what's **missing**, and any **substitutes** ("using chipotle for habanero", "canned instead of fresh — works").
- If form ratios exist, show a conversion hint ("use 1 tsp dried").
- **Filters:** max missing items (default: any — a stricter default hid everything until you'd entered lots of ingredients), tags, total time.
- **Sort:** coverage, then the recipe cooked longest ago first (Iron-Log's novelty tie-break).
- **Optional second score:** "uses up my stuff" = matched items ÷ the number of items you entered.

## 6. Entering recipes

- **Paste box:** paste ingredient lines, e.g. "2 cups flour, sifted". A parser pulls out amount, unit, name, form and note into a table you can edit.
- **Unknown names:** if a name isn't in the dictionary (including aliases), the app asks to *link to an existing ingredient* or *create a new one under [parent]*. The dictionary grows as you add recipes.
- **Steps:** pasted as numbered lines. Ingredient mentions can be turned into `{n}` tokens with one tap.

## 7. Screens (tab bar)

1. **Recipes:** list with text search and tag filters, then recipe view (scale chips, ingredients, steps, "Cooked it").
2. **Add:** recipe form with the paste parser.
3. **Search:** ingredient search with chips, ranked results, missing items and substitutes.
4. **Ingredients:** dictionary tree; add, edit, re-parent, set forms and staple flag, merge two entries. Merging updates every recipe that uses them.
5. **History:** cook log; backup export and import under settings.

## 8. Build order

1. Basic structure: copied from Iron-Log, stores renamed, seed data.
2. Ingredients dictionary: tree, forms, aliases, merge.
3. Recipes: add (with paste parser), view, edit.
4. Scaling.
5. Ingredient search.
6. History and backup.
7. v2: shopping list (combine the chosen recipes' ingredients, grouped by aisle), saved pantry, volume↔weight conversion.

## 9. Open questions (defaults used until decided)

- **Sync across devices?** Default: no, data stays on the device and moves by JSON backup.
- **Units:** default is to show recipes in the units they were entered in, with an optional metric/US toggle.
- **Search input:** default is to type each time; a saved pantry comes in v2.
- **Starting data:** default is a starter dictionary of ~100 ingredients and ~10 sample recipes. Replace with your own if you have them.

## 10. Build notes (v1, Nov 2026)

v1 is built: `index.html`, `manifest.json`, `sw.js`. Changes from the plan above:
- Each parent ingredient has a `sibling_score` (see the matching table).
- Cans/jars with a size note ("1 (400 g) can") show the equivalent weight when scaled ("that's about 200 g").
- Awkward cup amounts show as "½ cup + 2 tbsp" rather than ⅝ cup.
- Unitless amounts in steps fix the noun's plural ("crack in 1 egg").
- Starter data: 159 ingredients and 10 sample recipes.
