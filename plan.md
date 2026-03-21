# Schema-Driven UI for Fieldactivity

## Goal

Replace the hand-maintained `ui_structure.json` form definitions with dynamic UI generation driven directly by the [management-event JSON schema](https://www.fieldobservatory.org/wp-content/themes/observatory-wp-theme/assets/json/management-event.schema.json). The schema becomes the single source of truth for form fields, labels, validation rules, choices, and multilingual translations.

---

## Table of Contents

1. [Background & Motivation](#1-background--motivation)
2. [Current Architecture](#2-current-architecture)
3. [The Management-Event Schema](#3-the-management-event-schema)
4. [Target Architecture](#4-target-architecture)
5. [Implementation Plan](#5-implementation-plan)
6. [File-by-File Changes](#6-file-by-file-changes)
7. [Schema-to-Widget Mapping](#7-schema-to-widget-mapping)
8. [Handling Complex Schema Patterns](#8-handling-complex-schema-patterns)
9. [Data Storage & Compliance](#9-data-storage--compliance)
10. [Testing Strategy](#10-testing-strategy)
11. [Migration & Rollout](#11-migration--rollout)
12. [Stretch Goals](#12-stretch-goals)

---

## 1. Background & Motivation

Fieldactivity is an R Shiny app (built with Golem) that lets farmers and researchers record field management events — sowing, fertilizer application, tillage, harvest, etc. Events are stored as JSON files per site/block.

Currently, every form field is manually defined in two places:

- **`inst/extdata/ui_structure.json`** (~1200 lines) — widget types, code names, choices, conditions, validation rules, sub-element hierarchies
- **`inst/extdata/display_names.csv`** (~300 rows) — multilingual labels mapped to code names

When the Field Observatory consortium updates the data schema (adds fields, changes options, adds event types), someone must manually update both files and the R code. This is error-prone, slow, and means the app's data model drifts from the canonical schema.

The management-event schema already contains everything needed to generate the form: property names, types, titles in 3 languages, choices with `oneOf`, validation constraints (`minimum`, `maximum`, `required`), array definitions for multi-row inputs, UI hints (`x-ui`), and `$ref`/`$defs` for shared definitions.

**The goal: read the schema, render the form, collect data in schema-compliant format. No intermediate structure file for form fields.**

---

## 2. Current Architecture

### File Roles

| File | Role |
|---|---|
| `R/app_ui.R` | Main UI layout — frontpage, event list area, form panel shell |
| `R/app_server.R` | Main server — auth, site loading, event CRUD, language observers |
| `R/fct_ui.R` | Reads `ui_structure.json`, recursively creates Shiny widgets. Builds `structure_lookup_list` — a flat map of code_name → widget descriptor used everywhere. |
| `R/fct_language.R` | Reads `display_names.csv`. Provides `get_disp_name(code_name, language)` for label lookup. |
| `R/fct_event_list.R` | Turns event lists into data frames for display |
| `R/fct_files.R` | JSON file read/write per site/block |
| `R/mod_form.R` | Form module: renders the event entry form, collects data, validates, handles save/cancel/delete |
| `R/mod_form_fct_evaluate_js.R` | Evaluates JS condition strings to determine widget visibility |
| `R/mod_table.R` | Dynamic multi-row table module for array data (planting_list, harvest_list) |
| `R/mod_event_list.R` | Event list display with filtering |
| `R/mod_fileInput.R` | File upload widgets (e.g., images) |
| `R/mod_download.R` | CSV/JSON export |
| `R/utils_global.R` | Global constants, site loading from FOsites.csv, language config |
| `R/utils_validation.R` | Shinyvalidate rules |

### Data Flow

1. `ui_structure.json` is loaded at package load time by `fct_ui.R`
2. `fct_ui.R` exposes `activity_options` (the nested form field tree) and `structure_lookup_list` (flat code_name → descriptor map)
3. `mod_form.R` calls `create_ui(activity_options, ns)` to build the form widgets
4. On language change, both `app_server.R` and `mod_form.R` iterate `structure_lookup_list` and call Shiny's update functions with new labels from `display_names.csv`
5. `mod_form.R` uses `structure_lookup_list` for validation setup, determining relevant (visible) variables, and collecting form data
6. On save, `app_server.R` receives form data and writes to JSON via `fct_files.R`

### How `ui_structure.json` Works

The file defines a nested tree. Each node is either:
- A **widget descriptor**: has `code_name`, `type` (selectInput, numericInput, etc.), `label`, `choices`, `required`, `min`, `max`, etc.
- A **container**: has `condition` (a JavaScript expression for `conditionalPanel`), plus child nodes in `sub_elements`

The condition strings control visibility — e.g., planting fields are wrapped in a condition that checks whether the event type selector equals `"planting"`.

`fct_ui.R` recursively walks this tree to build Shiny UI. The `build_structure_lookup_list()` function flattens it into a name→descriptor map used everywhere for updates, validation, and data collection.

### How `display_names.csv` Works

Each row has: `category`, `code_name`, `disp_name_eng`, `disp_name_fin`.

- **category** groups entries: `variable_name` for field labels, `*_choice` for select options, etc.
- **code_name** matches `code_name` in `ui_structure.json`
- Two language columns only (English and Finnish). No Swedish.

### How `mod_form.R` Works

1. **Renders** the widget tree from `ui_structure.json` via `create_ui(activity_options, ns)`
2. **Sets up validators** per widget based on `structure_lookup_list` attributes (required, min, max, step, maxlength)
3. **Computes `relevant_variables()`** — which fields are currently visible, by evaluating the JS condition strings against current input state
4. **On save**: iterates relevant variables, reads values from `input[[code_name]]` or from table modules, assembles the event as a named list
5. **On set_values**: fills widgets from an existing event
6. **On language change**: re-labels all widgets using `get_disp_name()`

### How `mod_table.R` Works

For multi-row data (multiple crops in planting, multiple harvest items), `mod_table.R` creates a dynamic table. Each column is a variable. Rows can be added/removed. The table structure is defined in `ui_structure.json` via `type: "dataTable"` with `columns` and `rows` configuration. The module handles per-cell widget rendering, value collection, and validation.

---

## 3. The Management-Event Schema

**Location**: https://www.fieldobservatory.org/wp-content/themes/observatory-wp-theme/assets/json/management-event.schema.json

**Format**: JSON Schema draft-07, ~192KB

### Top-Level Structure

The root object has:
- **`properties`**: Common fields that apply to ALL event types (`mgmt_operations_event`, `date`, `mgmt_event_short_notes`)
- **`oneOf`**: An array of 15 event type definitions, each with its own properties, required fields, and titles
- **`$defs`**: Reusable definitions (crop species list, fertilizer application methods, mulch types)
- **`required`**: Top-level required fields (`mgmt_operations_event`, `date`)
- **`@context`**: Linked data context defining the semantic meaning of title fields (using Dublin Core)

### Key Schema Patterns

#### 3.1 Common Properties

Properties in the root `properties` object apply to ALL event types:
- `mgmt_operations_event` — the event type selector, marked with `"x-ui": { "discriminator": true }`
- `date` — event date (`format: "date"`)
- `mgmt_event_short_notes` — short description visible in event list

#### 3.2 Event Types via `oneOf`

Each entry in the root `oneOf` array defines an event type. Each has:
- Its own `properties` (fields specific to that event type)
- A `mgmt_operations_event` property with a `const` value identifying the type (e.g., `"const": "planting"`)
- A `required` array listing mandatory fields for that event type
- `title`, `title_fi`, `title_sv` for the event type name
- Optional `description`, `description_fi` for help text

The 15 event types are: planting, fertilizer, tillage, harvest, chemicals, grazing, weeding, irrigation, mowing, observation, bed_prep, inorg_mulch, Inorg_mul_rem, measurement, other.

#### 3.3 Nested `oneOf` (Subtypes)

Some events have subtypes selected by a secondary discriminator:

- **Fertilizer** has 3 subtypes: mineral, soil amendment, organic material — selected by `fertilizer_type` (which has `x-ui.discriminator: true`). Each subtype brings additional fields.
- **Observation** has 8 subtypes: soil, vegetation, water, animals, pests, disturbance, management, other — selected by `observation_type`.

This nesting goes one level deep (no further nesting within subtypes).

#### 3.4 Array Properties (Multi-Row Data)

Some properties are `type: "array"` with `items.type: "object"`:

| Array Property | Parent Event | Item Properties |
|---|---|---|
| `planting_list` | planting | `planted_crop`, `planting_material_weight`, `planting_depth`, `planting_material_source` |
| `harvest_list` | harvest | `harvest_crop`, `harvest_moisture`, `harvest_method`, `harvest_operat_component`, `canopy_height_harvest`, `harvest_cut_height`, `plant_density_harvest`, `harvest_residue_placement`, `harvest_yield_harvest_dw`, `harv_yield_harv_f_wt`, `yield_C_at_harvest` |
| `chemical_applic_material_list` | chemicals | `chemical_applic_material` |

These need to render as dynamic tables with add/remove row capability — exactly what `mod_table.R` already provides.

#### 3.5 `$ref` and `allOf` for Shared Definitions

Crop selectors use `allOf` to combine a title with a reference to a shared definition:
- First element provides `title`, `title_fi`, `title_sv`
- Second element provides `$ref: "#/$defs/crop_ident_ICASA"`

The `$ref` resolves to a `$defs` entry with a `oneOf` array of species options (each with `title`, `title_fi`, `title_sv`, and `const` — the ICASA code).

Three shared definitions exist:
- `crop_ident_ICASA` — ~60 crop species
- `fertilizer_applic_method` — ~10 application methods
- `mulch_type` — ~5 mulch types

#### 3.6 `x-ui` Hints

The schema uses a custom `x-ui` extension for UI metadata:

| Hint | Meaning | Example |
|---|---|---|
| `discriminator: true` | This field selects which event type or subtype is active | `mgmt_operations_event`, `fertilizer_type` |
| `form-type: "textAreaInput"` | Override default widget type | `mgmt_event_long_notes` |
| `form-placeholder` / `form-placeholder_fi` | Placeholder text per language | Various notes fields |
| `unit: "kg/ha"` | Display unit (informational, already embedded in title) | Fertilizer amounts |
| `unitless_title` / `unitless_title_fi` | Title without the unit suffix, for table column headers | All fields with units |
| `total_of_list` / `total_of_property` | This field should auto-sum a column from a table | `harvest_yield_harvest_dw_total` sums `harvest_yield_harvest_dw` from `harvest_list` |

#### 3.7 Multilingual Titles

Every property and choice option provides:
- `title` — English (the default/fallback)
- `title_fi` — Finnish
- `title_sv` — Swedish (where available)

Some properties also have `title2`, `title2_fi` (alternate phrasing, e.g., "sowing" vs "planting").

---

## 4. Target Architecture

### What changes

**Before**: `ui_structure.json` → `fct_ui.R` → Shiny widgets
**After**: `management-event.schema.json` → `fct_schema.R` + `fct_schema_ui.R` → Shiny widgets

The schema is parsed once at startup into two in-memory structures:
1. **Event registry** — keyed by event type const value, contains each event's properties, required fields, nested subtypes, and array definitions
2. **Property registry** — flat lookup keyed by property name, contains widget type, titles, choices, validation rules, parent event, and array membership

These registries replace `structure_lookup_list` for all form-related operations.

### What stays in `ui_structure.json`

Only app-level chrome that isn't in the management-event schema:
- Frontpage elements (titles, help texts, filter labels)
- Event list module configuration (activity/block/year filters)
- Action buttons (add_event, clone_event, save, cancel, delete)
- Site and block selectors (choices from FOsites.csv, not the schema)
- Download buttons
- User display field

### What stays in `display_names.csv`

Only labels for app chrome — the same items that remain in `ui_structure.json`, plus event type choice display names (for the event list filter). A new `disp_name_swe` column is added for Swedish.

### What gets removed

All form field definitions from `ui_structure.json`: every widget descriptor for every event type's fields — planting options, fertilizer options, tillage options, harvest options, chemicals, grazing, irrigation, mowing, observation, bed prep, mulch, measurement, other. And their corresponding rows in `display_names.csv`.

---

## 5. Implementation Plan

### Phase 1: Schema Interpreter — `R/fct_schema.R`

This is NOT a JSON Schema parser — `jsonlite::fromJSON()` already handles that. This module takes the parsed R list and interprets it for UI generation.

Responsibilities:

1. **Loading** the schema from `inst/extdata/management-event.schema.json` via `jsonlite::fromJSON()` (bundled with the package)

2. **Resolving `$ref` references** — a small utility that follows `$ref` paths (e.g., `"#/$defs/crop_ident_ICASA"`) within the already-parsed list and inlines the target content. For `allOf` combinations, merge: take titles from one element, type/choices from the resolved reference. This is a straightforward list traversal, not a schema parser.

3. **Building the event registry** — a named list keyed by event const value (e.g., `"planting"`, `"fertilizer"`). Each entry contains:
   - Multilingual titles for the event type
   - A list of property descriptors for that event type
   - The `required` array
   - If applicable, nested subtype information: the discriminator field name, and for each subtype, its const value, titles, and additional property descriptors

4. **Building the property registry** — a flat lookup keyed by schema property name. Each entry contains:
   - The property name (used as Shiny input ID)
   - The determined widget type (see mapping table in Section 7)
   - Multilingual titles (en, fi, sv)
   - For select-type fields: the list of choices with multilingual labels and const values
   - Validation attributes: required, minimum, maximum
   - UI hints from `x-ui`: placeholder text, form-type overrides, unit, total-of configuration
   - Context: which event type(s) this property belongs to, whether it's inside an array

5. **Determining widget type** from schema metadata — see the full mapping in Section 7

### Phase 2: Schema UI Renderer — `R/fct_schema_ui.R`

A new module that takes the parsed schema registries and produces Shiny UI:

1. **Main entry point** — called from `mod_form_ui()` in place of `create_ui(activity_options, ns)`. Creates:
   - The event type selector (from top-level `oneOf` titles)
   - A conditional panel per event type, shown when that type is selected
   - Inside each panel: widgets for that event type's properties

2. **Widget creation function** — takes a single schema property descriptor and returns the appropriate Shiny input widget (selectInput, numericInput, textInput, textAreaInput, dateInput). Uses the property name as the widget ID (stable across languages).

3. **Nested subtype handling** — for events with nested `oneOf` (fertilizer, observation): renders the subtype discriminator as a selectInput, then a nested conditional panel per subtype with its unique fields.

4. **Array-of-objects handling** — for properties like `planting_list`: integrates with `mod_table.R` by generating the table column configuration from the schema's `items.properties`. This provides column names, widget types, choices, and labels to the existing table module.

5. **Label and choice extraction helpers** — get the title in the requested language with fallback chain (requested → English → property name), and extract selectInput choices as named vectors from `oneOf` arrays.

### Phase 3: Form Module Integration — `R/mod_form.R`

Modify the existing form module:

1. **UI function**: Replace `create_ui(activity_options, ns)` with the new schema renderer call. Keep block selector (from FOsites.csv), keep save/cancel/delete buttons.

2. **Validation**: Instead of iterating `display_names.csv` variable_name entries and looking up rules in `structure_lookup_list`, iterate the schema property registry. The schema directly provides `minimum`, `maximum`, and `required`. Validators are conditionally active based on whether the property is relevant to the currently selected event type.

3. **Relevant variables calculation**: Replace the current approach (evaluating JS condition strings against `activity_options`) with a simpler lookup:
   - Read the selected event type from the event type input
   - Look up that event type in the event registry → get its property list
   - If the event has subtypes, read the subtype discriminator → get additional properties
   - Return the union of common properties + event properties + subtype properties

4. **Data collection on save**: Iterate relevant properties, read values from `input[[property_name]]` or from table modules for array properties. Inject const values (event type, subtype) automatically. Add `$schema` field to output.

5. **Value population on edit**: When loading an existing event, fill widgets using schema property names directly.

6. **Language switching**: Iterate the schema property registry instead of `structure_lookup_list`. For each relevant property, update labels and choices using schema multilingual data.

### Phase 4: Server & Global Integration

1. **`utils_global.R`**: Load and parse the schema at package load time. Add Swedish to the languages vector. Create a language code mapping between the CSV convention (`disp_name_eng`, `disp_name_fin`) and ISO codes (`en`, `fi`, `sv`).

2. **`app_server.R`**: Update the form-field language observer to use schema labels. Event list and app chrome language switching continues using `display_names.csv`.

3. **`fct_ui.R`**: Remove all form field entries from `structure_lookup_list`. Keep only app chrome widget descriptors. The `create_widget` and `create_ui` functions still serve app chrome rendering.

4. **`mod_table.R`**: Accept table column configuration generated from the schema (in addition to the current `ui_structure.json`-based config). Provide a pathway for the schema UI renderer to create table modules with schema-derived column definitions.

### Phase 5: Data Storage Compliance

1. **Property names**: Events saved by the app use the exact property names from the schema. Most already match. Document the few differences (see Appendix C).

2. **`$schema` field**: Add the schema URL to every saved event.

3. **Array storage**: Store array properties (planting_list, harvest_list) as proper JSON arrays of objects, not as flattened parallel vectors.

4. **Number types**: Store numbers as numbers, not strings.

5. **Backward-compatible reading**: When loading old events, detect the legacy format (no `$schema` field, flattened arrays) and normalize on read.

6. **Optional validation**: Use `jsonvalidate` package to validate events against the schema before writing. Log warnings for non-compliant data rather than blocking saves.

---

## 6. File-by-File Changes

| File | Change Type | Description |
|---|---|---|
| `R/fct_schema.R` | **NEW** | Schema interpreter: load via jsonlite, resolve `$ref`s, build event registry and property registry |
| `R/fct_schema_ui.R` | **NEW** | Schema → Shiny UI renderer: widgets, conditional panels, array tables |
| `R/mod_form.R` | **MODIFY** | Use schema renderer for form body; schema-based validation and data collection |
| `R/fct_ui.R` | **MODIFY** | Remove form field definitions. Keep only app chrome rendering. Slim down `structure_lookup_list`. |
| `R/app_server.R` | **MODIFY** | Language observer for form fields uses schema labels |
| `R/utils_global.R` | **MODIFY** | Load schema at startup. Add Swedish language. Language code mapping. |
| `R/fct_language.R` | **MODIFY** | Add `disp_name_swe` column support for app chrome labels |
| `R/fct_files.R` | **MODIFY** | Add `$schema` to saved events. Backward-compatible reading. Optional schema validation. |
| `R/mod_table.R` | **MODIFY** | Accept table column config from schema (in addition to `ui_structure.json`) |
| `inst/extdata/management-event.schema.json` | **NEW** | Bundled copy of the schema |
| `inst/extdata/ui_structure.json` | **REDUCE** | Remove all form field definitions, keep only app chrome |
| `inst/extdata/display_names.csv` | **REDUCE + EXTEND** | Remove form field labels (schema provides them), add `disp_name_swe` column for remaining entries |
| `DESCRIPTION` | **MODIFY** | Add `jsonlite` to Imports if not already present. Optionally add `jsonvalidate`. |

---

## 7. Schema-to-Widget Mapping

| Schema Pattern | Resulting Widget | Examples |
|---|---|---|
| `type: "string"` + `oneOf` with `const` values | selectInput | `tillage_practice`, `harvest_method`, `chemical_type` |
| `type: "string"` + `allOf` containing a `$ref` to a `$defs` entry with `oneOf` | selectInput | `planted_crop` (refs `crop_ident_ICASA`), `fertilizer_applic_method` |
| `type: "string"` + `x-ui.discriminator: true` | selectInput (controls conditional panels) | `mgmt_operations_event`, `fertilizer_type`, `observation_type` |
| `type: "string"` + `format: "date"` | dateInput | `date`, `end_date` |
| `type: "string"` + `x-ui.form-type: "textAreaInput"` | textAreaInput | `mgmt_event_long_notes` |
| `type: "string"` + `x-ui.form-type: "textInput"` | textInput | `planting_material_source` |
| `type: "string"` (plain, no special markers) | textInput | `chemical_product_name`, `fertilizer_product_name` |
| `type: "number"` | numericInput | `N_in_applied_fertilizer`, `tillage_operations_depth` |
| `type: "number"` + `minimum` / `maximum` | numericInput with min/max | `harvest_moisture` (0-100), `org_matter_moisture_conc` (0-100) |
| `type: "array"` + `items.type: "object"` | Dynamic table via `mod_table` | `planting_list`, `harvest_list`, `chemical_applic_material_list` |
| `x-ui.total_of_list` + `total_of_property` | numericInput (auto-calculated from table column sum) | `harvest_yield_harvest_dw_total`, `harv_yield_harv_f_wt_total` |
| Property with `const` value (inside event type definitions) | Not rendered — value injected automatically on save | `mgmt_operations_event` const per event type |

### Widget ID Convention

Use the schema property name directly as the Shiny input ID (within the form module namespace). This means:
- IDs are stable across languages (unlike the GSoC PR approach that used `make.names(label)`)
- IDs match the keys used in saved JSON events
- IDs match the schema property names exactly
- For properties inside arrays, the table module handles its own internal namespacing

---

## 8. Handling Complex Schema Patterns

### 8.1 Top-Level `oneOf` — Event Types

The root `oneOf` has 15 event types. Each contains a `mgmt_operations_event` property with a `const` value identifying it.

**Approach**: Render one selectInput for event type selection. Build its choices from the `oneOf` entries' titles and const values. For each event type, wrap its properties in a conditional panel that checks whether the selector matches that type's const value.

### 8.2 Nested `oneOf` — Subtypes

Fertilizer and observation events have secondary `oneOf` arrays with a discriminator.

**Approach**: Identical pattern, nested one level deeper. Identify the discriminator via `x-ui.discriminator`. Render it as a selectInput. Each subtype's unique properties go in a nested conditional panel. The subtype's const values for the discriminator are extracted from the nested `oneOf` entries.

Since this only goes one level deep (no subtypes of subtypes), a clean two-level handler is sufficient — no need for unbounded recursion.

### 8.3 Arrays of Objects — Dynamic Tables

`planting_list`, `harvest_list`, and `chemical_applic_material_list` need multi-row input.

**Approach**: For each array property, extract the item schema (`items.properties`) and transform it into the table column configuration that `mod_table.R` expects. Each property inside `items.properties` becomes a column in the table, with its widget type, label, choices, and validation rules determined by the same mapping used for regular properties.

The existing `mod_table.R` handles:
- Dynamic add/remove rows
- Per-cell widget rendering
- Value collection as structured data
- Per-cell validation

The main integration work is providing a way for `mod_table.R` to accept column definitions from the schema, in addition to (or instead of) the current `ui_structure.json`-based path.

Array properties also have `minItems: 1` in some cases — this means "at least one row required," which translates to a validation rule on the table.

### 8.4 `$ref` and `allOf` Resolution

All `$ref` resolution happens in the parser phase (Phase 1). By the time the UI renderer or form module sees a property, references are fully resolved.

For `allOf` combinations (e.g., planted_crop = title info + $ref to crop list), the parser merges them: titles come from the first element, type and choices from the resolved reference. The result is a single flat property descriptor.

### 8.5 Auto-Sum Fields (`total_of_list` / `total_of_property`)

Fields like `harvest_yield_harvest_dw_total` should automatically show the sum of a column from a table.

**Approach**: Render as a regular numericInput. In the form server, set up a reactive observer that watches the table module's values, sums the specified column, and updates this field. The `x-ui.total_of_list` tells us which table, and `x-ui.total_of_property` tells us which column.

This is already partially implemented for harvest totals in the current app. The schema makes it explicit and generalizable.

### 8.6 `const` Properties

Inside each event type definition, `mgmt_operations_event` has a `const` value (e.g., `"planting"`). Similarly, subtype discriminators have `const` values. These are not user-editable.

**Approach**: The parser flags these. The UI renderer skips them. On save, the data collection logic injects the appropriate const value based on which event type (and subtype) is selected.

### 8.7 Common Properties vs Event-Specific Properties

The root `properties` (date, mgmt_operations_event, mgmt_event_short_notes) are common to all events. Each event type's `properties` are specific to that type, though some property names repeat across types (e.g., `mgmt_event_long_notes` appears in most event types with different titles).

**Approach**: Common properties are rendered once, outside conditional panels. Event-specific properties go inside the per-event-type conditional panel. For `mgmt_event_long_notes`, the widget is rendered per event type (inside the conditional panel) with that event type's specific title, rather than once globally.

---

## 9. Data Storage & Compliance

### Current Storage Format

Events are stored as JSON arrays in files like `{site}/{block}/events.json`. Each event is a flat named list. Array data is stored as parallel vectors (e.g., `planted_crop: ["WH", "BA"]`, `planting_material_weight: [150, 200]`).

`block` is stored inside the event even though it's implied by the file path. Numbers are sometimes stored as strings. There is no `$schema` field.

### Target Storage Format

Events should match the schema structure:
- `$schema` field present, pointing to the schema URL
- Array properties stored as actual arrays of objects (not flattened parallel vectors)
- Numbers stored as numbers
- `block` remains in the event for compatibility (the app needs it)
- Property names match schema exactly

### Backward Compatibility

When reading existing events:
1. Check for `$schema` field — if absent, treat as legacy format
2. For legacy events with flattened arrays, reconstruct the nested array-of-objects structure on read
3. Optionally migrate files in-place (write back in new format after reading)

This ensures old data remains accessible while new events are saved in the compliant format.

---

## 10. Testing Strategy

### Unit Tests — Schema Interpreter

- All 15 event types are present in the event registry
- `$ref` references are fully resolved (e.g., `planted_crop` has choices from `crop_ident_ICASA`)
- Nested `oneOf` for fertilizer produces 3 subtypes with correct discriminator
- Nested `oneOf` for observation produces 8 subtypes
- Multilingual titles are extracted correctly for all three languages
- Widget type determination is correct for all property patterns (string, number, date, select, textarea, array)
- Properties with `const` are flagged as non-renderable
- Validation attributes (required, minimum, maximum) are extracted
- `x-ui` hints are preserved and accessible

### Unit Tests — Schema UI Renderer

- Each event type produces non-empty UI output
- Conditional panels are keyed on the correct input ID and const value
- selectInput choices match schema `oneOf` entries
- numericInput fields have correct min/max
- Array properties produce table module UI
- Language parameter changes the labels throughout

### Integration Tests

- Render the full form, select each event type, verify the correct fields appear
- Switch between all three languages, verify all labels update without errors
- Fill out a complete event for each type, save, verify the output JSON matches schema structure
- Load an existing event (both legacy and new format), verify the form populates correctly
- Verify auto-sum fields update when table values change
- Validate saved JSON against the schema using `jsonvalidate`

### Regression Tests

- For each event type: compare the set of rendered variable names against the set currently produced by `ui_structure.json` — no fields should be missing (though names may differ slightly; see Appendix C)
- For key selectInputs: compare the set of choices against the current app's choices — ensure no options are lost

---

## 11. Migration & Rollout

### Step 1: Schema parser (no app changes)

Create `fct_schema.R`, add the bundled schema file, write tests. This has zero impact on the running app. Verify the interpreter handles the full schema correctly.

### Step 2: Schema UI renderer (parallel to existing)

Create `fct_schema_ui.R`. Test it in isolation — a standalone test script that renders the schema form in a minimal Shiny app, verifying all event types render properly. Still no impact on the running app.

### Step 3: Form module integration (the switch)

Replace the `create_ui(activity_options, ns)` call in `mod_form_ui` with the schema renderer. Update validation, relevant-variable calculation, data collection, and language switching in `mod_form_server`. This is the critical change — thorough testing required.

### Step 4: Slim down static files

Remove form field definitions from `ui_structure.json` and their corresponding rows from `display_names.csv`. Keep app chrome only. Add `disp_name_swe` column.

### Step 5: Data storage updates

Add `$schema` field to saved events. Implement proper array-of-objects storage. Add backward-compatible reading of legacy format. Test with real site data.

### Step 6: Swedish language

Add Swedish to the language selector. Verify all schema-driven labels show Swedish correctly. Add Swedish translations for app chrome labels in `display_names.csv`.

### Step 7: Schema fetching (optional, stretch)

Add option to fetch latest schema from URL on startup, with fallback to bundled version.

---

## 12. Stretch Goals

- **Runtime schema updates**: Fetch schema from the Field Observatory URL on app startup, cache locally, re-parse. Fall back to bundled version if network is unavailable.
- **Schema versioning**: Track which schema version each saved event was created with (using the `$schema` field).
- **Batch data migration**: A utility function that reads all existing event files for all sites and rewrites them in schema-compliant format.
- **Schema diff detection**: On startup, compare fetched schema with bundled version, log any changes (new fields, removed options, etc.).
- **Export validation**: When exporting CSV/JSON, validate all events against the schema and flag non-compliant ones in the export.
- **Form preview mode**: A read-only rendering of an event using schema labels, for viewing without editing.

---

## Appendix A: Schema `$defs`

| Definition | Usage | Content |
|---|---|---|
| `crop_ident_ICASA` | `planted_crop`, `harvest_crop`, `mowed_crop` | ~60 crop species, each with `title`, `title_fi`, `title_sv`, and ICASA code as `const` |
| `fertilizer_applic_method` | `fertilizer_applic_method` | ~10 methods (broadcast, banded, foliar, etc.) with multilingual titles |
| `mulch_type` | `mulch_type`, `mulch_type_remove` | ~5 mulch materials with multilingual titles |

## Appendix B: Event Type Summary

| Event Type | Const Value | Has Nested oneOf | Has Arrays | Approx. Property Count |
|---|---|---|---|---|
| Sowing | `planting` | No | `planting_list` | 3 |
| Fertilizer | `fertilizer` | Yes (3 subtypes) | No | ~20 + subtype-specific |
| Tillage | `tillage` | No | No | 4 |
| Harvest | `harvest` | No | `harvest_list` | 6 + per-item fields |
| Chemicals | `chemicals` | No | `chemical_applic_material_list` | ~10 |
| Grazing | `grazing` | No | No | ~12 |
| Weeding | `weeding` | No | No | 1 |
| Irrigation | `irrigation` | No | No | 4 |
| Mowing | `mowing` | No | No | 5 |
| Observation | `observation` | Yes (8 subtypes) | Varies by subtype | Varies |
| Bed Prep | `bed_prep` | No | No | 1 |
| Mulch Placement | `inorg_mulch` | No | No | 5 |
| Mulch Removal | `Inorg_mul_rem` | No | No | 2 |
| Measurement | `measurement` | No | No | 2 |
| Other | `other` | No | No | 1 |

## Appendix C: Property Name Differences — Current App vs Schema

Most property names already align between the current `ui_structure.json` code_names and the schema property names. Known differences:

| Current App `code_name` | Schema Property Name | Notes |
|---|---|---|
| `mgmt_event_notes` | `mgmt_event_short_notes` | Schema adds "short" qualifier. Used for the event list description. |
| `planting_notes` | `mgmt_event_long_notes` | Schema unifies all per-event-type notes fields under one name. |
| `harvest_comments` | `mgmt_event_long_notes` | Same — the title changes per event type but the property name is shared. |
| `fertilizer_comments` | `mgmt_event_long_notes` | Same. |
| `tillage_notes` | `mgmt_event_long_notes` | Same. |
| `chemical_notes` | `mgmt_event_long_notes` | Same. |

The schema's approach is cleaner: one `mgmt_event_long_notes` property per event type, with event-type-specific titles (e.g., "sowing notes" for planting, "harvest comments" for harvest). The current app has separate code_names per event type for what is conceptually the same field.

This difference needs to be handled in backward-compatible reading: when loading legacy events with `planting_notes`, map it to `mgmt_event_long_notes`.
