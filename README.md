# Manufacturing Ontology (MF)

An OWL ontology describing manufacturing concepts — containers and vessels, materials and their quantities, units of measurement, and the physical-property relationships between them — so that an agent can reason about manufacturing equipment and infer properties that aren't directly measured.

## Scope

This ontology describes **concepts**, not instances. It defines what a `Tank` or `Material_Lot` or `Density` *is* and how these concepts relate to one another — it does not enumerate specific equipment, sensors, or measured values. Those live in a separate knowledge graph, which asserts instance data using the classes and properties defined here (e.g. "Tank-104 `Has_Capacity` 5000 kg", "Tank-104's level sensor reading has `Has_Status` `Suspect`"). The ontology's job is to give an agent enough conceptual structure to reason correctly about *any* asset described that way — including deriving a value that wasn't directly measured, via the formula-registry pattern described below.

## Current version

**`MF7.owl`** is the current ontology. **`smOntology_v1_00.owl`** is an identical, version-tagged copy of MF7 intended as a stable reference point for consumers who want to pin to a specific release rather than track the latest `MFn.owl`.

`MF_Changelog.md` is the authoritative history of every version — what was added and, more importantly, *why* — and should be read alongside any non-trivial change to understand prior design decisions before revisiting them.

## Key concepts in MF7

- **Container hierarchy** — `Container` → `Vessel` → `Tank`; `Package_Container` and its subtypes (`Carton`, `Box`, `Drum`, `Cannister`, `Tanker`).
- **Vessel content state** — `Vessel_Content` (`Filled_Content` / `Empty_Content`) reifies "what a vessel currently holds" as a first-class object rather than a bare triple, so an empty vessel is representable.
- **Measure hierarchy** — `Measure` and its subclasses `Mass`, `Volume`, `Density`, `Flow`, `Mass_Flow`, `Time`, `Capacity`, each with grounded `Unit_Of_Measurement` individuals (e.g. `Kilogram`, `Cubic_Meter`, `Gram_Per_Milliliter`) carrying SI abbreviations and a `Conversion_Factor_To_Base` for unit normalisation.
- **`Physical_Property_Value`** — shared superclass of `Material_Quantity` (an actual/current amount) and `Capacity` (a potential/limit amount): any single numeric value with a unit, a data-quality status, and (if computed rather than measured) a link to the formula that produced it.
- **Data quality tags** — `Value_Status` (`Valid` / `Questionable` / `Suspect` / `Bad`) for numeric values, and `Verification_Status` (`Verified` / `Unverified`) for material-identity assertions on `Filled_Content`. Both are generic tags an agent (or the knowledge graph) can apply to any observed property — the ontology does not assert these per asset.
- **`Derivation_Formula`** — a pointer to a named formula in an external Calculation Engine, describing which `Measure` types a formula relates (`Has_Parameter_Measure`, `Has_Result_Measure`) and the lookup key (`Formula_Name`). Only universal physical relationships live here (e.g. `Density_From_Mass_Volume` — true for every material, unconditionally); asset-specific formulas (e.g. a particular tank's level-to-volume geometry) are knowledge-graph instance data using the same pattern, not ontology individuals.

## Files

| File | Purpose |
|---|---|
| `MF7.owl` | Current ontology (see version history below) |
| `smOntology_v1_00.owl` | Version-tagged copy of MF7, for consumers pinning to a stable release |
| `MF1.owl` – `MF6.owl` | Prior versions, retained for history — see `MF_Changelog.md` |
| `MF_Rules1.ttl` | SHACL validation layer for **MF6 only** (unit-conversion reference data + capacity/overflow checks). Superseded from MF7 onward — see "Formula evaluation" below — and kept only as a historical artifact; nothing extends it. |
| `MF_Changelog.md` | Full version history and the design reasoning behind each change |
| `IndustrialStandard-ODP-ISA88_v1.4.2_ISA88.owl` | Imported ISA-88 reference ontology |
| `catalog-v001.xml` | Protégé/OWL API catalog mapping the ISA-88 import to its local file |
| `To do.txt`, `Ontology Dev notes.txt` | Working notes from early development |

## Formula evaluation

MF7 does not embed arithmetic (no SWRL, no SHACL-SPARQL rules). Instead, `Derivation_Formula` individuals point by name to formulas registered in a separate **Calculation Engine** (`calc_tool` — a SymPy-based formula store with a REST API: `GET /api/formulas/{name}` for parameters, `POST /api/calculate` to evaluate). An agent that needs a value not directly measured:

1. Finds the `Derivation_Formula` whose `Has_Result_Measure` matches the missing value and whose `Has_Parameter_Measure`s match what's already known.
2. Looks up its `Formula_Name` in the Calculation Engine for exact parameter metadata.
3. Normalises its known values to base units via `Conversion_Factor_To_Base`.
4. Calls the Calculation Engine and asserts the result back as a new `Physical_Property_Value`, linked to the formula via `Was_Derived_By`.

This is why there is no `MF_Rules2.ttl` and won't be one going forward: everything the ontology needs to express is standard OWL.

## Loading the ontology

Open `MF7.owl` in Protégé (or load with any OWL API / rdflib-based tool). The ISA-88 import resolves via `catalog-v001.xml`, so keep `IndustrialStandard-ODP-ISA88_v1.4.2_ISA88.owl` in the same directory.
