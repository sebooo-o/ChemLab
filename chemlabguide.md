# ChemLab JSON Guide

This guide defines the exact JSON format that `chemlab_json_structure_trainer.html` loads. It is written so that an AI can read it and produce a valid file for any set of molecules.

**How to use it:** paste this whole file into an AI chat, then ask something like *"Make me a ChemLab JSON with 12 common drugs, grouped into 3 levels."* Save the reply as `something.json` and load it with **Load JSON** (or drag it onto the page).

---

## 1. Instructions for the AI (read this first)

1. Output **only valid JSON**: double quotes, no comments, no trailing commas, no text before or after. UTF-8 characters such as `°`, `−`, `₂` are fine.
2. Use the structure in section 2. Each molecule must have a `name` and either a `smiles` string **or** both `atoms` and `bonds`.
3. **Prefer `smiles`.** Use `atoms`/`bonds` only for things SMILES can't express well (see section 5).
4. **Never include hydrogens in SMILES** unless they are inside brackets (e.g. `[NH4+]`). Hydrogens are filled in automatically.
5. Each molecule must be **one connected structure**. Do **not** use `.` (dot) in SMILES. Salts and mixtures cannot be built by the learner, so write the ion or molecule of interest alone.
6. Use the **Kekulé form** (alternating `=`) for any aromatic ring that is not plain benzene or pyridine-type (see section 4.3).
7. Double-check every SMILES against the checklist in section 8 before answering.
8. Keep molecules reasonably small: **up to ~30 heavy atoms** works well. The learner must draw each one by hand.

---

## 2. File structure

```json
{
  "title": "Dataset title",
  "description": "One or two sentences about the set.",
  "levels": { "1": "Level 1: Basics", "2": "Level 2: Functional groups" },
  "molecules": [ { }, { } ]
}
```

| Field | Required | Notes |
|---|---|---|
| `title` | no | Shown in the header. Default: "Chemical Structures". |
| `description` | no | Shown in the Dataset card. |
| `levels` | no | Maps each level key (as a **string**) to a label for the Builder's level buttons. A level with no entry is labelled "Level n". |
| `molecules` | **yes** | Non-empty array. A bare array `[ {...}, {...} ]` is also accepted instead of the object. |

---

## 3. Molecule fields

```json
{
  "id": "acetic-acid",
  "name": "Acetic acid",
  "abbr": "AcOH",
  "formula": "C₂H₄O₂",
  "category": "Carboxylic acids",
  "level": 2,
  "smiles": "CC(=O)O",
  "hint": "A methyl group attached to a –COOH group.",
  "facts": { "Boiling point": "118 °C", "pKa": "4.76" },
  "synonyms": ["ethanoic acid", "vinegar acid"]
}
```

| Field | Required | Type | Behaviour |
|---|---|---|---|
| `name` | **yes** | string | Shown as the target in the Builder; quiz answer. Must be unique within the file. |
| `smiles` | one of | string | Structure as SMILES (section 4). |
| `atoms` + `bonds` | one of | arrays | Explicit structure (section 5). Ignored if `smiles` is present. |
| `id` | no | string | Default: lowercased name with spaces replaced by `-`. Duplicates get a numeric suffix. Use lowercase letters, digits, hyphens. |
| `abbr` | no | string | Short label or common abbreviation (`EtOH`, `Ac2O`, `Py`). Also accepted as a quiz answer. Alias: `symbol`. |
| `formula` | no | string | If omitted, it is calculated automatically in Hill order with subscripts (e.g. `C₂H₆O`). **Recommended to omit it** so it can never disagree with the structure. |
| `category` | no | string | Groups molecules in the Atlas filter buttons. Default `"Uncategorized"`. Reuse the exact same spelling across molecules and keep to about 3–8 categories. |
| `level` | no | number or string | Difficulty tier for the Builder (default `1`). Sorted numerically. Should match keys in `levels`. |
| `hint` | no | string | Shown only when the learner clicks **Hint**. It may describe the structure ("benzene with an –OH"). |
| `facts` | no | object of string → string/number | Free-form properties shown as chips in the Builder banner and the Atlas. |
| `synonyms` | no | array of strings | Extra accepted quiz answers and search terms. |
| `explicitH` | no | boolean | Explicit-atoms mode only (section 5). |

**Important content rules**
- The Builder banner shows `name`, `abbr`, the formula, the `category` and all `facts` **before** the learner draws. So **never put the structure, SMILES, or substructure descriptions in `facts`, `abbr` or `category`**. Put structural description in `hint` only.
- `facts` values should be short (under ~25 characters). Only include values you are confident are correct.
- For a useful quiz, include **at least 8 molecules** (the quiz offers 4 choices and needs distractors).
- Typical sizes: 10–40 molecules per file, in 2–5 levels of 3–10 molecules each.

---

## 4. SMILES: exactly what is supported

### 4.1 Atoms
| Kind | Syntax | Examples |
|---|---|---|
| Organic subset (implicit H) | `B C N O P S F Cl Br I` | `C`, `Cl` |
| Aromatic (lowercase) | `b c n o p s` | `c1ccccc1` |
| Anything else, charged atoms, or fixed H count | bracket atom `[...]` | `[Na+]`, `[O-]`, `[NH4+]`, `[nH]`, `[Se]`, `[Fe]` |

Bracket atom format: `[` element, optional `H` or `H<n>`, optional charge `]`.
- Charge: `+`, `-`, `+2`, `-2`, `++`, `--`.
- A bracket atom has **exactly** the hydrogens you write. `[O-]` has 0 H, `[OH-]` has 1, `[NH4+]` has 4, `[CH3]` is a methyl radical group with 3.
- Isotope numbers (`[13C]`) are ignored; chirality marks (`@`, `@@`, `/`, `\`) are accepted but ignored. Don't rely on stereochemistry; ChemLab does not check it.
- Valid elements: any real element symbol from H to U in common use. Implicit-hydrogen filling is only defined for **H, B, C, N, O, F, Cl, Br, I, Si, P, S, Se**. Metals and other elements get **no** implicit hydrogens, so write them in brackets.

### 4.2 Bonds
| Symbol | Meaning |
|---|---|
| (none) | single bond (or aromatic between two aromatic atoms) |
| `-` | single |
| `=` | double |
| `#` | triple |
| `:` | aromatic |
| `( )` | branch |
| digits `1`-`9`, `%10`+ | ring closure |

Never use `.` (disconnected parts).

### 4.3 Aromatic rings: when to use lowercase and when to use Kekulé form
ChemLab converts aromatic SMILES to alternating single/double bonds with a simple matcher. It works for:

- benzene-type six-membered rings: `c1ccccc1`, `Cc1ccccc1`, `Oc1ccccc1`
- fused benzene-type rings: naphthalene `c1ccc2ccccc2c1`
- pyridine-type rings: `c1ccncc1`
- five-membered rings whose N–H is written as `[nH]`: pyrrole `c1cc[nH]c1`, imidazole `c1c[nH]cn1`, indole `c1ccc2[nH]ccc2c1`
- furan `c1ccoc1`, thiophene `c1ccsc1`

It does **not** work (the ring would silently end up with only single bonds) when a ring nitrogen carries a substituent instead of an H, e.g. N-methyl pyrrole-type nitrogens as in caffeine, or in other fused heterocycles with several ring nitrogens. **In any doubt, write the Kekulé form** (explicit `=`), which is always safe:

| Molecule | Use |
|---|---|
| Benzene | `c1ccccc1` or `C1=CC=CC=C1` |
| Caffeine | `CN1C=NC2=C1C(=O)N(C)C(=O)N2C` |
| N-methylpyrrole | `CN1C=CC=C1` |

Quick test: in aromatic SMILES, every lowercase atom must be able to take part in an alternating pattern. If you are unsure, use Kekulé.

### 4.4 Worked SMILES reference
| Molecule | SMILES |
|---|---|
| Methane | `C` |
| Ethene | `C=C` |
| Ethyne | `C#C` |
| Ethanol | `CCO` |
| Acetone | `CC(C)=O` |
| Acetic acid | `CC(=O)O` |
| Acetamide | `CC(N)=O` |
| Acetonitrile | `CC#N` |
| Cyclohexane | `C1CCCCC1` |
| Aspirin | `CC(=O)Oc1ccccc1C(=O)O` |
| Nitromethane | `C[N+](=O)[O-]` |
| Ammonium ion | `[NH4+]` |
| Acetate ion | `CC(=O)[O-]` |
| Sulfur hexafluoride | `FS(F)(F)(F)(F)F` |
| Naphthalene | `c1ccc2ccccc2c1` |
| Imidazole | `c1c[nH]cn1` |
| Caffeine | `CN1C=NC2=C1C(=O)N(C)C(=O)N2C` |
| Ibuprofen | `CC(C)Cc1ccc(cc1)C(C)C(=O)O` |

---

## 5. Explicit atoms and bonds (alternative to SMILES)

Use this for structures SMILES handles poorly, or when you want specific 2D coordinates.

```json
{
  "name": "Carbon dioxide",
  "abbr": "CO2",
  "category": "Inorganic",
  "atoms": [
    { "id": "o1", "el": "O", "x": -1.16, "y": 0 },
    { "id": "c",  "el": "C", "x": 0,     "y": 0 },
    { "id": "o2", "el": "O", "x": 1.16,  "y": 0 }
  ],
  "bonds": [ ["o1", "c", 2], ["c", "o2", 2] ]
}
```

**Atoms**: `{ "id", "el", "x"?, "y"?, "charge"?, "h"? }`
- `id`: any unique string or number within this molecule.
- `el`: element symbol with correct capitalisation (`"Cl"`, not `"CL"`). Aliases `element` and `sym` also work.
- `x`, `y`: optional. If given for at least two atoms they are rescaled automatically, so any unit is fine (Å, pixels, ...). **y points down** (screen coordinates). Atoms without coordinates are placed automatically. If you are not sure of good coordinates, **omit them all**.
- `charge`: integer formal charge, default 0.
- `h`: optional number of implicit hydrogens on that atom. Overrides automatic filling.

**Bonds**: either `[idA, idB, order]` arrays or `{ "a": idA, "b": idB, "order": n }` (also `from`/`to`).
- `order` must be `1`, `2` or `3`. **Never use 1.5.** Write aromatic rings in Kekulé form (alternating 2 and 1).

**Hydrogen rule (easy to get wrong):**
- If the `atoms` list contains **no** hydrogen atoms (and `explicitH` is not `true`), all missing hydrogens are filled in from standard valences.
- If it contains **any** `"H"` atom, **nothing** is auto-filled for the whole molecule. Then you must list every hydrogen explicitly, or set `h` on the atoms.
- `"explicitH": true` also switches auto-filling off.

---

## 6. What the app does with your data (so you can predict it)

- **Builder:** the learner draws the molecule from an empty canvas. The answer is correct if the **heavy-atom skeleton and bond orders** match your structure. Hydrogens are optional for the learner (missing ones are accepted), but extra bonds beyond an atom's valence are rejected. Formal charges and stereochemistry are **not** checked.
- **Mastery:** a molecule counts as mastered after 3 correct answers in a row.
- **Quiz:** shows your structure and asks for the name. Accepted answers (case-insensitive, exact): `name`, `abbr`, `id`, `formula`, any `synonyms`.
- **Atlas:** one card per molecule showing `category`, `hint`, `facts`, formula.
- **Hide-H toggle:** with H atoms hidden, hydrogens fold into the parent atom label (`OH`, `NH₂`, `CH₃`).

---

## 7. Complete example file

```json
{
  "title": "Everyday Medicines",
  "description": "Common over-the-counter drugs, from simple to more complex.",
  "levels": {
    "1": "Level 1: Simple",
    "2": "Level 2: Aromatic"
  },
  "molecules": [
    {
      "name": "Ethanol",
      "abbr": "EtOH",
      "category": "Solvents",
      "level": 1,
      "smiles": "CCO",
      "hint": "Ethane with one hydrogen replaced by –OH.",
      "facts": { "Boiling point": "78 °C" }
    },
    {
      "name": "Acetone",
      "category": "Solvents",
      "level": 1,
      "smiles": "CC(C)=O",
      "hint": "A carbonyl (C=O) flanked by two methyl groups.",
      "facts": { "Boiling point": "56 °C" },
      "synonyms": ["propanone"]
    },
    {
      "name": "Aspirin",
      "abbr": "ASA",
      "category": "Analgesics",
      "level": 2,
      "smiles": "CC(=O)Oc1ccccc1C(=O)O",
      "hint": "Benzene ring with an acetyl ester and a carboxylic acid on neighbouring carbons.",
      "facts": { "Melting point": "135 °C" },
      "synonyms": ["acetylsalicylic acid"]
    },
    {
      "name": "Paracetamol",
      "abbr": "APAP",
      "category": "Analgesics",
      "level": 2,
      "smiles": "CC(=O)Nc1ccc(O)cc1",
      "hint": "Para-substituted benzene: an –OH on one side, an acetamide on the other.",
      "facts": { "Melting point": "169 °C" },
      "synonyms": ["acetaminophen"]
    },
    {
      "name": "Caffeine",
      "category": "Stimulants",
      "level": 2,
      "smiles": "CN1C=NC2=C1C(=O)N(C)C(=O)N2C",
      "hint": "Fused five- and six-membered rings with three N–CH₃ groups and two C=O groups.",
      "facts": { "Melting point": "235 °C" }
    }
  ]
}
```

---

## 8. Final checklist for the AI before replying

- [ ] The reply is a single valid JSON document (no code-fence commentary, no comments, no trailing commas).
- [ ] Every molecule has a unique `name` and exactly one structure source (`smiles` or `atoms`+`bonds`).
- [ ] No `.` in any SMILES; no explicit H atoms outside brackets.
- [ ] Every opened ring digit is closed; every `(` has a `)`.
- [ ] Atom valences are plausible: C has 4 bonds, N 3, O 2 (unless bracketed and charged, e.g. `[N+]`, `[O-]`).
- [ ] Charged atoms are in brackets with the correct H count (`[NH4+]`, `[O-]`).
- [ ] Heterocycles with substituted ring nitrogens, and anything doubtful, are written in Kekulé form.
- [ ] Explicit `bonds` use only orders 1, 2, 3 and refer to existing atom `id`s.
- [ ] `level` values match the keys in `levels` (as strings), and `category` spellings are consistent.
- [ ] `facts`, `abbr`, `category` do not reveal the structure; only `hint` may.
- [ ] Facts are accurate and short, or omitted when unsure.
- [ ] `formula` is omitted (preferred), or is correct for the structure.

---

## 9. Ready-to-paste prompt

> Read the ChemLab JSON Guide above. Generate a ChemLab JSON file with **[N] molecules** about **[TOPIC]**, split into **[2–4] levels** from simple to hard. Use SMILES for every structure, Kekulé form for tricky heterocycles, omit `formula`, include a short `hint` and 1–2 accurate `facts` for each molecule, and add common `synonyms`. Reply with the JSON only.

---

## 10. Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Could not load … JSON" | Invalid JSON syntax (trailing comma, comments, single quotes). |
| "Molecule #n (Name): unexpected …" | A character in `smiles` isn't supported (e.g. `.`, an explicit `H`, a wrongly cased element). |
| "… unclosed ring" / "missing ]" | Ring digit or bracket not closed. |
| "unknown element" | Wrong capitalisation or invalid symbol (`CL`, `Xx`). |
| Aromatic ring drawn with only single bonds | Aromatic SMILES that can't be alternated; rewrite in Kekulé form. |
| Wrong number of hydrogens (explicit mode) | Mixed explicit `H` atoms and auto-fill; use all explicit H, or none. |
| Learner can't match the target | Check the structure for typos, and remember that charges and stereochemistry are not checked. |
