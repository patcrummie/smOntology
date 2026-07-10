# Manufacturing Ontology (MF) — Change Log & Design Decisions

This document tracks the evolution of the MF ontology across versions MF1 through MF6 and the MF_Rules companion file, summarising what changed in each version and the reasoning behind those decisions.

---

## MF1 — Initial Skeleton

### What was added

**Classes:** `Capacity`, `Container`, `Flow`, `Mass`, `MassFlow`, `Material`, `Measures`, `Quantity`, `Tank`, `TankType`, `UnitOfMeasurement`, `Vessel`, `Volume`

**Object Properties:** `hasQuantity`, `hasTankType`, `hasUnitOfMeasurement`, `hasVolume`

**Hierarchy:**
- `Tank` → `Vessel` → `Container`
- `Flow`, `Mass`, `MassFlow`, `Volume` → `Measures`
- `hasVolume` domain restricted to `Container`

### Design Decisions

The first version established the broad shape of the domain: a container hierarchy (`Container` → `Vessel` → `Tank`), a measurement hierarchy rooted at `Measures`, and `Material` as a standalone class. The initial intent was to link volume to containers via the `hasVolume` property.

**Identified problem — `hasVolume` on `Container`:** Restricting `hasVolume` to the domain `Container` caused a logical error. If a reasoner infers that anything with a volume is a `Container`, then a `Material` with a volume would incorrectly be classified as a container. The dev notes record this explicitly: *"hasVolume ties volume to container — not good enough. Volume need to be a class."* This insight drove the subsequent restructuring. `Volume` was already present as a subclass of `Measures`, but the property wiring was wrong.

---

## MF2 — Expanded Core; Material Tracking & Packaging

### What was added

**Classes:**
- `MaterialLot`, `MaterialQuantity` — to track discrete material batches and their measured quantities
- `PackageContainer` and subtypes: `Carton`, `Box`, `Drum`, `Cannister`, `Tanker`
- `PackagingUnit` — links a lot to a package container with a quantity
- `UOMAbbreviation` — structured abbreviation strings for units
- `Measures` renamed to `Measure`

**Object Properties:** `hasMaterialQuantity`, `countOf`, `quantityHasUnit`, `hasAbbreviation`, `hasPackagingQuantity`, `hasPackageContainerType`

**Data Properties:** `lotID`, `materialID`, `hasNumericValue`, `packagingUnitCount`, `abbreviationValue`

**Named Individuals:** `Kilogram`, `Tonne`, `Liter`, `CubicMeter` (as `UnitOfMeasurement` individuals); abbreviation individuals `abbr_kg`, `abbr_t`, `abbr_L`, `abbr_m3`

### Design Decisions

The ontology shifted from pure structure to practical data modelling. A key decision was introducing `MaterialLot` as the central tracking unit — a batch of material with an identity (`lotID`, `materialID`), a quantity, and packaging information. This separates *what the material is* (`Material`) from *a trackable instance of it* (`MaterialLot`).

**`MaterialQuantity` as a first-class class:** Rather than using a plain numeric data property to express "how much", `MaterialQuantity` becomes a class that links a numeric value (`hasNumericValue`) to a unit of measurement. This supports unit-aware reasoning and avoids bare numbers floating without context.

**Named individuals for units:** UOM instances (`Kilogram`, `Tonne`, etc.) are asserted directly as named individuals. This makes unit comparisons tractable for a reasoner and avoids treating units as opaque strings.

**`UOMAbbreviation` class:** Abbreviations (`kg`, `t`, etc.) are modelled as individuals of a dedicated class rather than plain string literals. This preserves the ability to link abbreviations to their parent units via object properties.

---

## MF3 — Tank–Material and Tank–Capacity Relationships

### What was added

**Object Properties:**
- `contains` — domain `Tank`, range `Material`
- `hasCapacity` — domain `Tank`, range `Capacity`

**Class change:**
- `Capacity` is now a subclass of `MaterialQuantity` (previously just a bare class)

### Design Decisions

MF3 addressed the open questions in the To Do notes: *"Add concepts Tank Contains Material"* and *"Tank has Capacity / Tank capacity is the maximum quantity of a material that can be contained by the tank."*

**`Capacity` as a subclass of `MaterialQuantity`:** This is a meaningful modelling choice. Capacity is not merely a measurement — it is the maximum quantity of material a vessel can hold. By making it a specialisation of `MaterialQuantity`, it inherits the unit + numeric value structure, enabling direct comparison between capacity and actual material quantities.

**Scope limited to `Tank`:** At this stage, `contains` and `hasCapacity` were domain-restricted to `Tank`. This was intentional conservatism — `Vessel` is more general and the implications of attaching these properties more broadly had not yet been reasoned through.

---

## MF4 — Naming Convention Overhaul; Capacity Semantics; Standard Quantity

### What was changed / added

**Naming convention:** All multi-word identifiers moved from camelCase to underscore-separated (`MaterialLot` → `Material_Lot`, `TankType` → `Tank_Type`, `hasVolume` → `Has_Volume`, etc.)

**Domain broadening:** `Contains` and `Has_Capacity` domains changed from `Tank` to `Vessel`, making capacity and containment properties available to all vessels, not just tanks.

**New object properties:** `Standard_Quantity`, `Content`

**`Capacity` OWL restriction added:**
```
Capacity subClassOf (Has_Unit_Of_Measurement allValuesFrom (Mass ⊔ Volume))
```
This formally constrains capacity units to mass or volume — flow rates and time are excluded.

**Abbreviation individuals renamed:** `abbr_kg` → `Kilogram_Abbreviation` etc., aligning with the underscore naming convention.

**`Content` property** added to `Package_Container` with range `Material`, enabling a package container to reference its material directly.

### Design Decisions

**Naming convention change:** The move to underscore-separated names reflects a preference for readability in OWL editors like Protégé, where underscores clearly delimit word boundaries in IRIs. CamelCase can create ambiguity in longer compound names.

**Broadening from `Tank` to `Vessel`:** Recognising that `Tank` is just one kind of `Vessel`, the `Contains` and `Has_Capacity` properties were lifted to the `Vessel` level. Any vessel — tank, drum, canister — can have a capacity and can contain material. Keeping the domain at `Tank` was unnecessarily restrictive.

**Formal `Capacity` restriction:** The `allValuesFrom (Mass ⊔ Volume)` axiom is a guard against nonsensical capacity values. It is not meaningful to express the capacity of a vessel in flow rate units (litres-per-minute) or time. The restriction encodes this domain knowledge as machine-checkable logic rather than relying on documentation alone.

**`Capacity` removed as subclass of `Material_Quantity`:** Reclassified as a subclass of `Measure` only. A capacity represents *potential*, not *actual content* — making it a `Material_Quantity` blurred that distinction. The comment in the file states this explicitly: *"Not a MaterialQuantity — it represents potential, not actual content."*

---

## MF5 — Vessel Content State Model

### What was added

**Classes:**
- `Vessel_Content` — abstract superclass representing the current state of a vessel
- `Filled_Content` — subclass of `Vessel_Content`; the vessel holds material
- `Empty_Content` — subclass of `Vessel_Content`; the vessel is empty

**OWL axioms on `Filled_Content`:**
```
Filled_Content subClassOf (Has_Material minCardinality 1)
Filled_Content subClassOf (Has_Content_Quantity minCardinality 1)
Filled_Content subClassOf (Has_Content_Quantity allValuesFrom
    (Quantity_Has_Unit allValuesFrom (Mass ⊔ Volume)))
```

**Disjointness:** `Filled_Content` and `Empty_Content` are declared `owl:AllDisjointClasses`

**New object properties:** `Has_Current_Content` (domain `Vessel`, range `Vessel_Content`), `Has_Material` (domain `Filled_Content`, range `Material`), `Has_Content_Quantity` (domain `Filled_Content`, range `Material_Quantity`)

**Named individual:** `:Empty` (singleton instance of `Empty_Content`)

### Design Decisions

MF4 used `Contains` (domain `Vessel`, range `Material`) to assert that a vessel holds a material. This approach has a gap: it cannot represent an *empty* vessel, only a vessel that contains something. To represent "this tank is currently empty" requires a different mechanism.

**`Vessel_Content` pattern:** Rather than attaching material directly to a vessel, the content state is reified into its own class. A vessel *has a current content* (`Has_Current_Content`), which is either a `Filled_Content` or an `Empty_Content`. This makes the empty state a first-class citizen rather than an absence of a triple.

**Singleton `:Empty`:** Because all empty vessels are in the same state (no material, no quantity), a single named individual `:Empty` can be shared across all empty vessels. This avoids creating redundant blank nodes and allows reasoning over "which vessels are empty" via a simple individual query.

**Mincardinality on `Filled_Content`:** A vessel described as filled must have at least one material and at least one quantity. These cardinality constraints allow the reasoner to flag incomplete assertions — if a `Filled_Content` individual has no `Has_Material` triple, it violates the restriction.

**Unit guard on quantity:** The `allValuesFrom` chain restricts content quantities to mass or volume units, consistent with the `Capacity` restriction in MF4. Flow rates do not make sense as a snapshot quantity of material in a vessel.

**Disjointness axiom:** Without `owl:AllDisjointClasses`, a reasoner could not infer that a vessel cannot be both filled and empty simultaneously. The disjointness makes this logically explicit.

---

## MF6 — Time as a Measure; Flow Rate UOM Individuals

### What was added

**Class:**
- `Time` — subclass of `Measure`; comment lists applicable units: Day, Hour, Minute, Second, Millisecond

**Named Individuals:**
- Flow rate units: `Cubic_Meter_Per_Hour`, `Liter_Per_Minute` (as `Unit_Of_Measurement`, `Flow`)
- Mass flow units: `Tonne_Per_Hour`, `Kilogram_Per_Minute` (as `Unit_Of_Measurement`, `Mass_Flow`)
- Time units: `Day`, `Hour`, `Minute`, `Second`, `Millisecond` (as `Unit_Of_Measurement`, `Time`)
- Corresponding abbreviation individuals for all of the above

### Design Decisions

**`Time` as a `Measure`:** The `Flow` and `Mass_Flow` classes were already present in the ontology, but without grounded unit individuals they could not be used in assertions. Flow rates are inherently time-based (tonnes *per hour*, litres *per minute*). Rather than embedding time implicitly in unit labels, `Time` is made an explicit subclass of `Measure`. This keeps the measurement hierarchy coherent: `Mass`, `Volume`, `Flow`, `Mass_Flow`, and `Time` are all peers under `Measure`, each with their own associated unit individuals.

**Population of flow and time unit individuals:** MF6 completes the unit vocabulary needed for process-level descriptions — flow meters, throughput rates, and durations can now be modelled using proper typed individuals rather than raw strings. This is consistent with the pattern established in MF2 for mass and volume units.

**No structural changes:** The class hierarchy and property structure are unchanged from MF5. MF6 is a vocabulary enrichment — extending what can be *said* rather than changing the rules for how things are *classified*.

---

## MF7 — Density; Formula-Registry Pattern; Physical Property Value; Data Quality Tags

### What was added

**Classes:**
- `Density` — subclass of `Measure`; comment lists applicable units: Kilogram Per Cubic Meter (base), Gram Per Milliliter, Tonne Per Cubic Meter
- `Physical_Property_Value` — new shared superclass of `Material_Quantity` and `Capacity`
- `Derivation_Formula` — pointer to a named formula in the external Calculation Engine
- `Value_Status` — data-quality classification for any `Physical_Property_Value`, with individuals `Valid`, `Questionable`, `Suspect`, `Bad`
- `Verification_Status` — identity-confirmation classification for `Filled_Content`, with individuals `Verified`, `Unverified`

**Object Properties:**
- `Has_Parameter_Measure`, `Has_Result_Measure` (domain `Derivation_Formula`, range `Measure`)
- `Was_Derived_By` (domain `Physical_Property_Value`, range `Derivation_Formula`) — presence/absence of this triple is the measured-vs-derived signal, rather than a separate boolean
- `Has_Status` (domain `Physical_Property_Value`, range `Value_Status`, functional)
- `Has_Verification_Status` (domain `Filled_Content`, range `Verification_Status`, functional)
- `Has_Quantity` (present since MF1 but unused) promoted to a real super-property: `Has_Material_Quantity`, `Has_Content_Quantity`, `Has_Packaging_Quantity`, and `Standard_Quantity` are now all `rdfs:subPropertyOf :Has_Quantity`

**Data Properties:**
- `Formula_Name` (domain `Derivation_Formula`, range `xsd:string`) — matches a formula's `name` in the calc_tool Calculation Engine database
- `Conversion_Factor_To_Base` — migrated in from MF_Rules1.ttl (see below), unchanged in meaning
- `Has_Numeric_Value` — domain widened from `Material_Quantity` to `Physical_Property_Value`

**Class axiom changes:**
- `Material_Quantity` and `Capacity` are now both `rdfs:subClassOf :Physical_Property_Value`

**Named Individuals:**
- Density units: `Kilogram_Per_Cubic_Meter`, `Gram_Per_Milliliter`, `Tonne_Per_Cubic_Meter` (as `Unit_Of_Measurement`, `Density`), plus abbreviations and `Conversion_Factor_To_Base` values (1, 1000, 1000)
- `Density_From_Mass_Volume`, `Mass_From_Volume_Density`, `Volume_From_Mass_Density` (`Derivation_Formula` individuals), each with a `Formula_Name` and its `Has_Parameter_Measure`/`Has_Result_Measure` pair
- `Valid`, `Questionable`, `Suspect`, `Bad` (`Value_Status` individuals, declared `owl:AllDifferent`)
- `Verified`, `Unverified` (`Verification_Status` individuals, declared `owl:AllDifferent`)
- Every pre-existing `Unit_Of_Measurement` individual now carries its `Conversion_Factor_To_Base` inline (previously asserted only in MF_Rules1.ttl)

### Design Decisions

**Density derivation is a lookup, not embedded arithmetic.** The trigger for this version was wanting Mass, Volume, and Density to be mutually derivable (any two determine the third). Early drafts of this design considered doing that derivation inside the ontology itself — either as SHACL rules that materialise the missing value as new triples, or as a SHACL-SPARQL consistency check in the style of MF6's `Vessel_Capacity_Shape`. Both were dropped once it became clear a separate Calculation Engine (`calc_tool`) already exists for exactly this purpose: named, versioned, tested formulas evaluated through a REST API (`GET /api/formulas/{name}`, `POST /api/calculate`). The ontology's job was redefined accordingly: describe the *relationship* between Measure types and record the *lookup key*, not perform the arithmetic. `Derivation_Formula` is that pointer — generic across any Measure relationship, not density-specific (the same pattern could later describe `Flow = Volume / Time`, which today has no formal relationship recorded anywhere).

**Universal formulas vs. asset-specific formulas.** `Density_From_Mass_Volume` and its two counterparts are asserted as ontology-level individuals because the relationship they encode is a universal physical law — true for every material, unconditionally — the same category as the `Kilogram`/`Tonne` unit individuals already in the ontology. This is explicitly *not* the pattern for asset-specific formulas: `calc_tool`'s own database already holds formulas like `Box Tank Level Volume`, which depends on a particular tank's geometry and therefore has no business being pre-enumerated in a conceptual ontology. The ontology describes concepts an agent can reason with about *any* tank; which formula a *specific* tank uses is knowledge-graph instance data, asserted using the same `Derivation_Formula` shape but not shipped inside MF7.owl.

**`Physical_Property_Value` closes a gap MF_Rules1 had flagged and left open.** `Has_Numeric_Value`'s domain being `Material_Quantity` meant asserting it on a `Capacity` individual (as MF_Rules1's `Vessel_Capacity_Shape` did) technically entailed the `Capacity` individual was also a `Material_Quantity` — contradicting the MF4 decision that capacity is potential, not actual, content. Introducing `Physical_Property_Value` as a shared superclass and widening `Has_Numeric_Value`'s domain onto it fixes this properly: `Material_Quantity` and `Capacity` both satisfy the domain without being equated to each other.

**Status and provenance are generic tags, not per-asset modelling.** `Value_Status` and `Was_Derived_By` apply to `Physical_Property_Value` as a class — any measured-or-derived numeric quantity, on any equipment, can carry them. The ontology does not enumerate which specific tank has which specific sensor or which specific quantity is currently `Suspect`; that is knowledge-graph data. The same reasoning applies to `Verification_Status`: it tags whether a `Filled_Content`'s material identity has been confirmed, deliberately scoped narrower than `Value_Status` because identity-confidence ("is this really Material X?") and value-confidence ("is this measurement any good?") are different quality dimensions with different causes. A third category considered — status for static configuration facts like "this tank is jacketed" — was explicitly ruled out of ontology scope: that is general knowledge-graph data-quality/QA territory (checked at load time), not something an agent reasons about dynamically.

**`Has_Quantity` given a real purpose.** It existed since MF1 as a declared-but-unused property. Making it the common super-property of the four existing quantity-linking properties lets an agent collect every `Physical_Property_Value` attached to a subject (a `Material_Lot`, `Filled_Content`, `Packaging_Unit`, or `Package_Container`) through a single property path — the natural first step before deciding, for a given subject, which two of Mass/Volume/Density are already known and which `Derivation_Formula` (if any) could supply the third.

**No more separate rules file.** MF_Rules1.ttl existed because SHACL-SPARQL could do arithmetic and closed-world validation that plain OWL couldn't. Now that formula evaluation is delegated to the external Calculation Engine by reference (`Formula_Name`) rather than encoded as SHACL-SPARQL, that reason no longer applies — everything MF7 needs is expressible in standard OWL. `Conversion_Factor_To_Base` (the one piece of MF_Rules1 that was really just reference data, not validation logic) is folded directly into MF7.owl. The two SHACL shapes that *were* validation logic — `Unit_Of_Measurement_Shape` (every unit needs a conversion factor) and `Vessel_Capacity_Shape` (content shouldn't exceed capacity) — are retired rather than carried forward; the capability they provided becomes an agent-driven query against the knowledge graph (and, where arithmetic is needed, a call to the Calculation Engine) rather than an embedded ontology constraint. MF_Rules1.ttl is left in place, unmodified, as the historical validation layer for MF6 — it is not deleted, just not extended.

---

## Summary Table

| Version | Key Additions | Major Design Decision |
|---------|--------------|----------------------|
| MF1 | Core skeleton: Container/Vessel/Tank, Measure hierarchy, Material | `hasVolume` on Container identified as flawed |
| MF2 | MaterialLot, MaterialQuantity, PackageContainer types, UOM individuals | Reified quantities; named individuals for units |
| MF3 | `contains`, `hasCapacity` on Tank; Capacity → MaterialQuantity | Capacity is a specialisation of quantity (potential content) |
| MF4 | Underscore naming; domains lifted to Vessel; Capacity restriction | Capacity ≠ MaterialQuantity (potential vs. actual); formal unit guard |
| MF5 | Vessel_Content / Filled_Content / Empty_Content; `:Empty` singleton | Reified content state to allow empty vessel representation |
| MF6 | Time class; Flow, MassFlow, Time unit individuals | Explicit time measure to ground flow rate units |
| MF_Rules1 | SHACL shapes graph; `Conversion_Factor_To_Base` + factors for all units; `Vessel_Capacity_Shape` | SWRL retired in favour of SHACL; unit normalisation made explicit and machine-checkable |
| MF7 | Density; `Derivation_Formula` pointer pattern; `Physical_Property_Value`; `Value_Status`/`Verification_Status` | Arithmetic delegated to an external Calculation Engine by name lookup, not embedded in the ontology; no separate rules file needed or maintained from this version forward |

---

## MF_Rules1 — SWRL Retired, SHACL Adopted

### What changed

**Removed from MF6.owl:** the SWRL capacity-constraint sketch (comment-only, never a materialised axiom) and the unused `swrl:`/`swrlb:` prefixes. A pointer comment now directs readers to MF_Rules1.ttl.

**New file `MF_Rules1.ttl`:**
- `Conversion_Factor_To_Base` — data property (domain `Unit_Of_Measurement`, range `xsd:decimal`) giving each unit's multiplier relative to a canonical base unit per Measure subclass: `Kilogram` (Mass), `Cubic_Meter` (Volume), `Cubic_Meter_Per_Hour` (Flow), `Tonne_Per_Hour` (Mass_Flow), `Second` (Time) — each factor 1, with the rest of MF6's unit individuals assigned factors relative to these.
- `Unit_Of_Measurement_Shape` — SHACL shape requiring every `Unit_Of_Measurement` individual to carry exactly one conversion factor; catches units with no registered conversion data (e.g. an LLM-invented unit) as a validation failure.
- `Vessel_Capacity_Shape` — a SHACL-SPARQL constraint (`sh:sparql`) reproducing the retired SWRL rule: flags a `Vessel` whose current content quantity exceeds its capacity, normalising both to their Measure's base unit via `Conversion_Factor_To_Base` before comparing.

### Design Decisions

**Why SHACL over SWRL:** SWRL infers new facts under OWL's open-world assumption and has uneven reasoner support for arithmetic built-ins (`swrlb:multiply`/`divide`). SHACL validates under a closed-world assumption and produces structured validation reports (failing node, path, message) — a better fit for an agent that needs to check candidate facts against domain constraints and explain rejections, e.g. to guard an LLM against asserting invalid facts. SHACL-SPARQL constraints also do the required arithmetic as ordinary SPARQL `FILTER`/`BIND` expressions, which are portable across engines, unlike SWRL built-ins.

**Why conversion factors live in MF_Rules1.ttl, not MF6.owl:** Keeps the semantic model (MF6) and the validation/normalisation policy (MF_Rules1) on separate lifecycles, consistent with the "OWL for T-Box, SHACL for S-Box" split. Operationally this means MF_Rules1.ttl must be loaded into the *data* graph (not only as the shapes graph) when validating, since its conversion-factor assertions are reference data the SPARQL constraints query against.

**Known gap carried over, not fixed:** `Has_Numeric_Value`'s `rdfs:domain` is `Material_Quantity`, not `Measure`/`Capacity`. `Vessel_Capacity_Shape` still asserts it on `Capacity` individuals (mirroring the original SWRL sketch), which technically entails those individuals are also `Material_Quantity` — contradicting the MF4 decision that Capacity is not a Material_Quantity. Flagged for a future fix (broaden the domain, or give `Capacity` its own numeric-value property).

**Verified with pySHACL** against synthetic vessels covering same-unit overflow, within-capacity, cross-unit overflow, and cross-unit within-capacity cases (the last of which a naive unconverted comparison would misclassify) — all four resolved correctly, plus an unregistered unit correctly failing `Unit_Of_Measurement_Shape`.
