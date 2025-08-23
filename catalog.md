# 2i2c Hub Configurator — **Catalog Format Specification (v2)**

This document defines the **machine-readable catalog format** used by the 2i2c Hub Configurator and related tooling (e.g., the Catalog Builder). It includes a formal specification plus copy-pasteable examples you can adapt for future catalogs.

---

## Table of Contents

1. [Purpose & Scope](#purpose--scope)
2. [Versioning & Compatibility](#versioning--compatibility)
3. [Top-Level Structure](#top-level-structure)
4. [Item Structure](#item-structure)
5. [Per-Tier Configuration](#per-tier-configuration)

   * [Kind: `single`](#kind-single)
   * [Kind: `flag`](#kind-flag)
   * [Kind: `select`](#kind-select)
6. [Dependency Language (`requires`)](#dependency-language-requires)

   * [Leaf Operators](#leaf-operators)
   * [Combinators](#combinators)
   * [Evaluation Semantics](#evaluation-semantics)
7. [Ordering Rules (Groups, Items, Options)](#ordering-rules-groups-items-options)
8. [Validation Rules & Lints](#validation-rules--lints)
9. [Design Guidelines & Best Practices](#design-guidelines--best-practices)
10. [Examples](#examples)

    * [A. Minimal Valid Catalog](#a-minimal-valid-catalog)
    * [B. Typical Catalog With Help & Dependencies](#b-typical-catalog-with-help--dependencies)
    * [C. Dependency Showcase](#c-dependency-showcase)

---

## Purpose & Scope

A **catalog** defines all **configuration items** the Hub Configurator should display across the three product tiers:

* `Essential`
* `Advanced`
* `Premier`

Each item defines **how it’s configured per tier**: fixed value (`single`), availability toggle (`flag`), or a dropdown (`select`). Items may include **helper text** and **dependencies** (`requires`) that constrain when the item (or a specific option) is valid, based on other selections in the **same tier**.

---

## Versioning & Compatibility

* **Schema name:** `2i2c Hub Catalog (v2)`
* **Top-level fields:** `name`, `version`, `items` (array)
* **Legacy formats:** Not supported in v2.

> **Version field** is a free string (e.g., `"2025.08.23"` or SemVer). Treat it as a human-readable release marker.

---

## Top-Level Structure

```ts
type Catalog = {
  name: string;              // Human-friendly catalog name
  version: string;           // Catalog version string
  items: Item[];             // Ordered list of items
};
```

* `name` and `version` are **required**.
* `items` is an **ordered** array (order affects UI grouping and display).

---

## Item Structure

```ts
type Item = {
  id: string;                // Unique, stable key (e.g., "group1:feature2")
  group: string;             // UI section label (e.g., "Infrastructure")
  label: string;             // User-facing item name
  help?: string;             // Item-level helper text (markdown-safe plain text)
  perTier: {
    Essential: TierConfig;
    Advanced:  TierConfig;
    Premier:   TierConfig;
  };
};
```

**Requirements**

* `id` must be **unique** across the catalog.

  * Recommended naming: `group:key` (ASCII, no spaces), keep stable across releases.
* All three tiers **must** be present under `perTier`.

---

## Per-Tier Configuration

```ts
type TierConfig =
  | SingleConfig
  | FlagConfig
  | SelectConfig;

type BaseTier = {
  help?: string;             // Tier-level help (appears under the control)
  requires?: RequiresClause; // When this tier's control is considered valid/active
};
```

### Kind: `single`

A fixed, read-only text value.

```ts
type SingleConfig = BaseTier & {
  kind: "single";
  text: string;              // The fixed value
};
```

### Kind: `flag`

A read-only boolean availability indicator for that tier.

```ts
type FlagConfig = BaseTier & {
  kind: "flag";
  available: boolean;        // true = included in tier; false = not included
};
```

### Kind: `select`

A dropdown control with options and a default.

```ts
type SelectConfig = BaseTier & {
  kind: "select";
  options: SelectOption[];   // Non-empty array
  default?: string;          // Must match one option.value; if omitted UI uses first option
};

type SelectOption = {
  value: string | number;    // Unique within this options list
  label?: string;            // User-facing option label (defaults to string(value))
  help?: string;             // Option-level helper text
  requires?: RequiresClause; // Option-level dependency
};
```

**Rules**

* `options` must be **non-empty** and have **unique** `value`s (stringified uniqueness).
* If `default` is present, it **must match** an option’s `value`.

---

## Dependency Language (`requires`)

Any tier config (item-level) or option (option-level) may include a `requires` clause.

### Leaf Operators

A **leaf** clause targets another item’s value in the **same tier**:

```ts
type LeafClause =
  | { id: string; op: "truthy" | "falsy" }
  | { id: string; op: "eq"  | "neq" ; value: Scalar }
  | { id: string; op: "in"  | "nin" ; value: Scalar[] }
  | { id: string; op: "gt"  | "gte" | "lt" | "lte"; value: number };
type Scalar = string | number | boolean;
```

* `truthy` / `falsy` ignore `value`.
* `in` / `nin` require an **array**.
* `gt/gte/lt/lte` require a **number**; evaluation will coerce the other item’s current value to a number (`Number(value)` must be finite).

> The **left-hand side** is the current value of the referenced item **within the same tier**:
>
> * `single` → the `text` (string)
> * `flag`   → the `available` (boolean)
> * `select` → the selected option `value` (string or number)

### Combinators

Combine clauses using boolean logic:

```ts
type RequiresClause =
  | LeafClause
  | { allOf: RequiresClause[] }   // All must pass
  | { anyOf: RequiresClause[] }   // At least one passes
  | { not: RequiresClause };      // Negation
```

### Evaluation Semantics

* Dependencies are evaluated **per tier** using the current selections in that tier.
* **UI behavior**: the builder/validator flags violations; the configurator keeps options visible but indicates unmet dependencies.
* Avoid circular dependencies; they’re not forbidden but can create confusing states.
* **Default viability check**: validators may simulate each tier’s defaults and warn if any default selection violates its own `requires`.

---

## Ordering Rules (Groups, Items, Options)

* **Group order**: derived from the first occurrence of each `group` label while scanning `items` top-to-bottom.
* **Item order**: the order of `items[]` is preserved in the UI.
* **Option order**: the order of `options[]` is preserved in the UI.

Use a catalog editor (or the provided builder) to reorder groups, items, and options.

---

## Validation Rules & Lints

**Errors (must pass)**

* `name` and `version` are strings; `items` is an array.
* Each item has unique `id` and `perTier` with all three tiers present.
* Tier `kind` ∈ `{single, flag, select}`.
* `flag.available` is boolean.
* `single.text` is string.
* `select.options` is non-empty; option `value`s are unique.
* If present, `select.default` matches an option `value`.
* All `requires` clauses reference **existing item ids**, use **valid operators**, and have correctly typed `value`s.

**Warnings (should pass)**

* Missing `help` fields (item/tier/option) reduce clarity.
* Missing `select.default` (UI will use first option).
* Default viability: selected defaults that fail their own `requires`.

---

## Design Guidelines & Best Practices

* **Stable IDs**: treat `id` as an API. Changing it is a breaking change for URLs and downstream tools.
* **Concise groups**: reuse a small number of `group` labels; it becomes the UI’s left column headings.
* **Helpful help**: one-sentence summaries are ideal; keep option-level `help` specific.
* **Dependencies**: prefer item-level `requires` for broad gating; use option-level for finer constraints.
* **Numeric comparisons**: store numeric-like options as numbers if you plan to use `gt/gte/lt/lte`.
* **No secrets**: catalogs are meant to be public; don’t embed credentials or URLs with tokens.

---

## Examples

### A. Minimal Valid Catalog

```json
{
  "name": "Minimal",
  "version": "1.0.0",
  "items": [
    {
      "id": "infra:provider",
      "group": "Infrastructure",
      "label": "Cloud provider",
      "perTier": {
        "Essential": { "kind": "single", "text": "AWS" },
        "Advanced":  {
          "kind": "select",
          "options": [
            { "value": "aws", "label": "AWS" },
            { "value": "gcp", "label": "Google Cloud" }
          ],
          "default": "aws"
        },
        "Premier":   { "kind": "flag", "available": true }
      }
    }
  ]
}
```

---

### B. Typical Catalog With Help & Dependencies

```json
{
  "name": "Example with Help",
  "version": "2025.08.23",
  "items": [
    {
      "id": "compute:type",
      "group": "Compute",
      "label": "Compute type",
      "help": "Choose CPU-only or GPU-enabled nodes.",
      "perTier": {
        "Essential": {
          "kind": "select",
          "help": "Entry-level options.",
          "options": [
            { "value": "cpu", "label": "CPU nodes", "help": "General purpose compute." },
            { "value": "gpu", "label": "GPU nodes", "help": "Acceleration for ML workloads." }
          ],
          "default": "cpu"
        },
        "Advanced": {
          "kind": "select",
          "help": "More flexibility for advanced users.",
          "options": [
            { "value": "cpu", "label": "CPU nodes", "help": "General purpose compute." },
            { "value": "gpu", "label": "GPU nodes", "help": "Acceleration for ML workloads." }
          ],
          "default": "cpu"
        },
        "Premier": {
          "kind": "select",
          "help": "Full catalog available.",
          "options": [
            { "value": "cpu", "label": "CPU nodes", "help": "General purpose compute." },
            { "value": "gpu", "label": "GPU nodes", "help": "Acceleration for ML workloads." }
          ],
          "default": "gpu"
        }
      }
    },
    {
      "id": "compute:gpuClass",
      "group": "Compute",
      "label": "GPU class",
      "help": "Choose a GPU accelerator (only relevant when Compute type is GPU).",
      "perTier": {
        "Essential": {
          "kind": "select",
          "requires": { "id": "compute:type", "op": "eq", "value": "gpu" },
          "options": [
            { "value": "t4",   "label": "NVIDIA T4",  "help": "Entry-level GPU." },
            { "value": "a100", "label": "NVIDIA A100","help": "High-performance GPU." }
          ],
          "default": "t4"
        },
        "Advanced": {
          "kind": "select",
          "requires": { "id": "compute:type", "op": "eq", "value": "gpu" },
          "options": [
            { "value": "t4",   "label": "NVIDIA T4",  "help": "Entry-level GPU." },
            { "value": "a100", "label": "NVIDIA A100","help": "High-performance GPU." }
          ],
          "default": "t4"
        },
        "Premier": {
          "kind": "select",
          "requires": { "id": "compute:type", "op": "eq", "value": "gpu" },
          "options": [
            { "value": "t4",   "label": "NVIDIA T4",  "help": "Entry-level GPU." },
            { "value": "a100", "label": "NVIDIA A100","help": "High-performance GPU." }
          ],
          "default": "a100"
        }
      }
    }
  ]
}
```

---

### C. Dependency Showcase

This example demonstrates all dependency operators and compositions.

```json
{
  "name": "Dependency Showcase",
  "version": "2.0.0",
  "items": [
    {
      "id": "g1:mode",
      "group": "Group 1",
      "label": "Mode",
      "perTier": {
        "Essential": {
          "kind": "select",
          "options": [
            { "value": "opt1", "label": "Option 1" },
            { "value": "opt2", "label": "Option 2" }
          ],
          "default": "opt1"
        },
        "Advanced": {
          "kind": "select",
          "options": [
            { "value": "opt1", "label": "Option 1" },
            { "value": "opt2", "label": "Option 2" },
            { "value": "opt3", "label": "Option 3" }
          ],
          "default": "opt2"
        },
        "Premier": {
          "kind": "select",
          "options": [
            { "value": "opt1", "label": "Option 1" },
            { "value": "opt2", "label": "Option 2" },
            {
              "value": "opt3",
              "label": "Option 3 (requires Switch or Scale ≥ 3)",
              "requires": { "anyOf": [
                { "id": "g1:switch", "op": "truthy" },
                { "id": "g2:scale",  "op": "gte", "value": 3 }
              ]}
            }
          ],
          "default": "opt3"
        }
      }
    },

    {
      "id": "g1:switch",
      "group": "Group 1",
      "label": "Enable Switch",
      "perTier": {
        "Essential": { "kind": "flag", "available": false },
        "Advanced":  { "kind": "flag", "available": true },
        "Premier":   { "kind": "flag", "available": true }
      }
    },

    {
      "id": "g2:scale",
      "group": "Group 2",
      "label": "Scale",
      "perTier": {
        "Essential": {
          "kind": "select",
          "options": [
            { "value": 1, "label": "Scale 1" },
            { "value": 3, "label": "Scale 3" }
          ],
          "default": 1
        },
        "Advanced": {
          "kind": "select",
          "options": [
            { "value": 1, "label": "Scale 1" },
            { "value": 3, "label": "Scale 3" },
            { "value": 5, "label": "Scale 5" }
          ],
          "default": 3
        },
        "Premier": {
          "kind": "select",
          "options": [
            { "value": 1, "label": "Scale 1" },
            { "value": 3, "label": "Scale 3" },
            { "value": 5, "label": "Scale 5" }
          ],
          "default": 5
        }
      }
    },

    {
      "id": "g3:theme",
      "group": "Group 3",
      "label": "Theme",
      "perTier": {
        "Essential": {
          "kind": "select",
          "options": [
            { "value": "default",  "label": "Default" },
            { "value": "contrast", "label": "High Contrast", "requires": { "not": { "id": "g1:mode", "op": "eq", "value": "opt3" } } }
          ],
          "default": "default"
        },
        "Advanced": {
          "kind": "select",
          "options": [
            { "value": "default",  "label": "Default" },
            {
              "value": "custom",
              "label": "Custom",
              "requires": {
                "allOf": [
                  { "id": "g1:switch", "op": "truthy" },
                  { "id": "g2:scale",  "op": "gte", "value": 3 }
                ]
              }
            }
          ],
          "default": "default"
        },
        "Premier": {
          "kind": "select",
          "options": [
            { "value": "default",  "label": "Default" },
            {
              "value": "custom",
              "label": "Custom",
              "requires": {
                "anyOf": [
                  { "id": "g1:switch", "op": "truthy" },
                  { "id": "g1:mode",   "op": "in", "value": ["opt2","opt3"] }
                ]
              }
            }
          ],
          "default": "custom"
        }
      }
    }
  ]
}
```

---

### Quick Authoring Checklist

* [ ] Top level has `name`, `version`, `items`.
* [ ] Every `item.id` is unique and stable.
* [ ] Each item provides **all three** tiers in `perTier`.
* [ ] `select` items have non-empty `options` with unique `value`s and a valid `default`.
* [ ] All `requires` clauses reference existing `id`s and use valid operators.
* [ ] Defaults don’t violate their own `requires` (run the validator).
* [ ] Helpful `help` text is present where useful (item-, tier-, and option-level).

---

### Notes for Tooling

* The Catalog Builder supports **import from URL/file**, **schema validation**, **dependency validation**, **reordering** (groups, items, options), and **export**.
* The Hub Configurator treats this catalog as read-only truth and renders two columns:

  * Left: description (group, label, help)
  * Right: control (single/flag/select) with option-level help and dependency hints.

---

**End of Specification.**
