# VERSION 2.5 – HARNESS VISUALIZER ARCHITECTURE DOCUMENTATION

---

## 1. SYSTEM SUMMARY AND PURPOSE

1.1. This visualizer models the wiring harness of an electric motorcycle as a graph of **physical components** (containers, connectors) joined by **relationships** (fixed wires, pluggable couplings) and organized by **logical signals** (nets).

1.2. It allows visualizing the elements, moving them in edit mode while respecting the containment hierarchy, observing flexible wires, and automatically validating the consistency of genders, pins, signals, and connector facing.

### 1.3 Usage Philosophy and Scope

1.3.1. This application is not an electrical simulator nor a circuit design tool. It is an **interactive visual documentation platform** for technicians and mechanics, designed to represent the wiring "as if it were laid out on the workshop floor."

1.3.2. The tool allows both **consulting** an existing harness and **building and modifying** its representation from scratch. All entities (containers, connectors, wires, couplings) can be created, edited, and deleted directly from the graphical interface or by loading data files.

1.3.3. The relationship between the graphical view and the underlying data is **bidirectional**:
- Any change made in the drawing (dragging, resizing, creating connections) is automatically reflected in the data tables.
- Any modification in the data (property change, adding notes, adjusting pins) immediately updates the visual representation.

1.3.4. The main uses are:
- **Trace routes**: visually follow the path of a signal from a source pin to its destination, even if it passes through several connectors and wires.
- **Identify properties**: know at a glance the color, gauge, length, and type of each wire, as well as the pins involved in each connection.
- **Filter and isolate**: hide entire subsystems to focus on a specific circuit (e.g., see only the ECU and the button panel).
- **Document and annotate**: add notes with history to any element, keeping a record of modifications, maintenance observations, or design decisions.
- **Manage the complete harness**: create, modify, and delete components, wires, and connections, always maintaining consistency between the visual view and the project database.

1.3.5. In essence, it is a **dynamic and interactive blueprint** that acts as an intermediate layer between the technical schematic and the harness database, allowing documentation and consultation work to be carried out in an organic and visual way.

1.3.6. As of V2.0, the system incorporates **reusable catalogs**, a **dynamic validation schema**, and a **user and responsible person management system**.

1.3.7. As of V2.1, the architecture is simplified: **nets are moved to catalogs**, **`metadata.uiSettings` is removed**, **native SVG** is specified as the visualization technology.

1.3.8. As of V2.2, usability and simplification improvements are applied: **edit mode toggle in the header**, **sidebar with state memory**, **unified validation**, **removal of the model/instance consistency rule**, **removal of `sectionRef` in connectors**, **gauge as a recommended field**, **intermediate filter**, and **explicit documentation of the GitHub Pages flow**.

1.3.9. As of V2.3, the following are incorporated: **inference principle from catalogs** (automatic autocomplete), **`colorPalette` catalog** for SVG rendering, **entity-specific schema**, **simplification of the `people` catalog**, **removal of `shield` as a signal type**, **hierarchical depth limit (4 levels) with cycle detection**, **100% custom validation** (without external libraries), and **single-file architecture** (`index.html` + `db.json`).

1.3.10. As of V2.4, light mode is completely removed, keeping only dark mode with a color palette optimized to avoid visual conflicts with standard electrical wire colors.

1.3.11. As of V2.5, the architecture transitions from single-file to **multi-file modular structure** (`index.html` + `styles.css` + `js/app.js` + `js/data.js` + `js/render.js` + `js/interaction.js`). Validation logic is simplified by removing model-instance consistency checks. The development workflow now involves a coordinated team of specialized AI agents (Orchestrator, Dev, QA, Integrator).

### 1.4 Technician Workflow

1.4.1. The interface offers two complementary and synchronized views: the **2D graphical canvas (SVG)** and the **data tables**. The user can switch between them or work with both simultaneously. Any change made in one view instantly propagates to the other.

1.4.2. **Collapsible property panel**: clicking on any element of the canvas (wire, connector, container, M) opens a side panel that shows all its attributes. This panel allows direct editing of the fields. In read-only mode, the fields appear locked; in edit mode, they are enabled for modification.

1.4.3. **Intermediate filters (V2.2)**: the technician can visually isolate elements using a combination of simple filters that are applied simultaneously (AND):
- Text field for searching by ID, name, or designator.
- Dropdown to filter by entity type (All / Containers / Connectors / Wires / Mates).
- Dropdown to filter by net (All / list of nets from the catalog).
- Dropdown to filter by section (All / list of sections from the catalog).
- "Clear filters" button to reset.

Elements that do not match are dimmed in the visual view (reduced opacity) and hidden in the table view. The filter state is preserved in `localStorage`.

1.4.4. **Context persistence**: applied filters, canvas position, and zoom level are stored in the browser's `localStorage` so that the technician recovers exactly the view they were working on.

### 1.5 Inference Principle from Catalogs (V2.3)

1.5.1. The application follows the principle of **"infer before asking"**: if a field of an entity can be deduced from a referenced catalog, the application **automatically autocompletes it** when creating or editing the entity. The user can always overwrite the autocompleted value.

1.5.2. This principle applies in the following cases:

| Field to autocomplete | Inferred from | Condition |
|---|---|---|
| `wires.gaugeUnit` | `wireTypeRef.unit` | If `wireTypeRef` exists and has `unit` |
| `wires.color` | `nets.colorCode` of the assigned net | If the wire has `net` and the net has `colorCode` in `colorPalette`. If not, default `"black"`. |
| `connectors.pins` | `modelRef.pins` | If `modelRef` exists and has `pins` |
| `connectors.gender` | `modelRef.gender` | If `modelRef` exists and has `gender` |
| `mates.pinMapping` | Default value `"direct"` | When creating a new mate **from the UI**. Existing mates in the JSON keep their original value. |
| `wires.gaugeUnit` | `"mm2"` | If there is no `wireTypeRef` OR if `wireTypeRef.unit` is missing (global default) |

1.5.3. Autocomplete is triggered:
- When **creating** a new entity (inferable fields are filled in).
- When **changing a reference** (e.g., changing `wireTypeRef` updates `gaugeUnit` if it was not manually overwritten).
- The user can **overwrite** any autocompleted value. Once overwritten, the field is considered "manual" and is not automatically autocompleted again unless the user explicitly resets it.

1.5.4. This principle **does not apply** to fields that are the source of truth for the instance. For example, `connectors.pins` is autocompleted from `modelRef.pins`, but if the user changes it, the user's value prevails and is not reverted when changing the model.

1.5.5. Autocomplete is a **speed aid**, not a restriction. The user always has the final say on the value of any field.

**Design Memory – Section 1**
The original purpose of the MVP is maintained: to offer an interactive view of the harness with realistic physical constraints. The separation between components, relationships, and signals follows the model-view pattern. V2.3 introduces the inference principle from catalogs as a transversal philosophy of the application, prioritizing user agility without sacrificing control.

---

## 2. IDENTIFIER SYSTEM (ENTITY INDEX)

2.1. Each entity in the system receives a **unique ID** composed of a type letter plus a three-digit number. The following table acts as a general index of all modeled entities.

**Table 1 – Entity index and identification prefixes**

| Prefix | Entity                  | Numeric range       | Example |
|--------|--------------------------|----------------------|---------|
| T      | Container (system, enclosure, pcb) | 100 – 399 (subdivided) | T100, T200, T300 |
| C      | Connector                | 001 – 999            | C001    |
| W      | Fixed wire (wired)       | 001 – 999            | W001    |
| M      | Pluggable coupling (mated)| 001 – 999            | M001    |

> **Note (V2.1):** The entity `N` (net) has been removed from the entity index. Nets are no longer entities with their own ID (`N001`, `N002`), but entries in the catalog `metadata.catalogs.nets` with descriptive names (`GND`, `+12V`, `CAN_H`). Wires and mates reference them by their name/ID in the catalog.

2.2. The letter **`T`** groups all containers (system, enclosure, pcb) under the same prefix. The subdivision of the numeric range indicates the hierarchical level:
 2.2.1. `1xx` → root system.
 2.2.2. `2xx` → enclosure.
 2.2.3. `3xx` → printed circuit board (pcb).

2.3. This organization allows up to 99 containers of each type (e.g., T101, T201, T301…) and maintains conceptual coherence: all containers share the prefix, but their hierarchical position is deducible from the number.

2.4. The `type` field (e.g., `"system"`, `"enclosure"`, `"pcb"`) remains present in the data to explicitly define the subtype; the ID only acts as a unique identifier.

2.5. **Range convention (V2.2):** The numeric ranges (T: 100-399, C/W/M: 001-999) are an **organizational guide**, not a technical restriction. The validation pattern `^[TCWM]\d{3}$` accepts any three-digit number. Specific ranges are not validated in the schema or in the code.

### 2.6 Positioning Principle

2.6.1. **General rule**:
- Any component with `parent_id: null` has an absolute position relative to the canvas (attributes `x`, `y`).
- Any component with a non-null `parent_id` has a position relative to its parent (attributes `offsetX`, `offsetY` for containers; `offset` for fixed connectors; flying connectors do not store their own position).

2.6.2. In current practice, only `T100` (root system) has `parent_id: null`. Connectors like C004 and C005 have `parent_id: "T100"` (i.e., they have a parent; they are not `null`). The rule is stated generically to admit future root components without modifying the positioning logic.

2.6.3. Dimensions (`width`, `height`) are always stored in a `size` object separate from the position, both for containers and connectors. This conceptually differentiates location from size and facilitates code maintenance.

### 2.7 Data File Structure (`db.json`)

2.7.1. The project is stored in a single JSON file called **`db.json`** with two root sections: `metadata` and `data`.

2.7.2. **`metadata`**: Contains all the project context information: description, authorship, save traceability, validation rules, and reusable catalogs (including nets and the color palette).

2.7.3. **`data`**: Contains the actual physical harness instances: containers, connectors, wires, and couplings. This section is the source of truth of the model.

2.7.4. **Tree structure**:
```
db.json
├── metadata
│   ├── projectInfo        (object: name, description, date, model)
│   ├── lastSave           (ISO 8601 string: timestamp of last save)
│   ├── savedBy            (string: ref to catalogs.people, who saved)
│   ├── version            (number: save counter, increments by 1)
│   ├── schema             (object: validation rules per entity)
│   └── catalogs           (object: reusable catalogs)
│       ├── people
│       ├── sections
│       ├── connectorModels
│       ├── wireTypes
│       ├── nets
│       └── colorPalette   ← New in V2.3
└── data
    ├── containers         (array of T containers)
    ├── connectors         (array of C connectors)
    ├── wires              (array of W wires)
    └── mates              (array of M couplings)
```

2.7.5. The file **MUST NOT contain comments** (`//` or `/* */`). Standard JSON does not support them. If annotations are needed, use the `notes` field of each entity or a `"_comment"` field at the root level.

2.7.6. **`metadata.version` field (V2.2):** It is a **save counter** (number), not a schema version. It increments by 1 each time the user exports/saves the file. It starts at 1. If the user needs to record the project version (e.g., "MotoStudents v2.0"), they can do so in `metadata.projectInfo.description` or in a custom field within `projectInfo`. There is no migration logic based on this field.

> **Historical note (V2.3):** In V2.2, the field `revision` was renamed to `version`. The functionality is identical: save counter that increments by 1.

2.7.7. **`metadata.uiSettings` does not exist** (removed in V2.1). All interface preferences (zoom, canvas position, filters, active view, container expansion state) are stored in the browser's `localStorage`. See section 11.9.

2.7.8. **Multi-file architecture (V2.5):** The application is delivered as a set of separate files for maintainability and team collaboration:
- `index.html`: HTML structure only (~100 lines). Loads CSS and JS modules.
- `styles.css`: All CSS styles (~300 lines).
- `js/app.js`: State management, initialization, utilities (~500 lines).
- `js/data.js`: Validation, fetch, import/export, autosave (~700 lines).
- `js/render.js`: SVG rendering (containers, connectors, wires, selection) (~600 lines).
- `js/interaction.js`: Drag/drop, zoom, pan, keyboard, modals, sidebar (~600 lines).
- `db.json`: External project data file.

No bundlers or frameworks are required. Deployment on GitHub Pages consists of uploading all files to the repository root.

### 2.8 Catalogs (`metadata.catalogs`)

2.8.1. Catalogs are collections of **generic models and reusable data**. They do not represent physical harness instances, but templates and references that enrich the entities in `data`.

2.8.2. **Fundamental principle**: catalogs are **optional and complementary**. If a JSON file does not contain `catalogs` or any of its sub-sections, the application functions correctly. The fields `modelRef`, `wireTypeRef` and `owner` in `data` are all optional.

2.8.3. **Precedence rule**: the instance fields in `data` are always the source of truth. The catalog only provides documentary information and values for autocomplete (see 1.5). No consistency between catalog and instance is validated.

2.8.4. **Sub-catalogs**:

**a) `people`** — Catalog of project people (simplified in V2.3).
```
"<person_id>": {
  "name": "string (mandatory)"
}
```
It is referenced from `metadata.savedBy` and from the `owner` field of any entity in `data`. The catalog only stores the name. Aliases, roles, or notes are not included: if a person fulfills a function or is responsible, it is indicated by the `owner` field in the corresponding entity. All `owner` fields are optional.

**Example:**
```json
"people": {
  "leo": { "name": "Leo" },
  "martin": { "name": "Martin" },
  "nico": { "name": "Nico" }
}
```

**b) `sections`** — Functional sections of the system (simplified in V2.3).
```
"<section_id>": {
  "name": "string (mandatory)"
}
```
It is referenced from the `sectionRef` field of **containers** (connectors inherit the section from their parent container, see 2.10.4). It allows grouping entities by subsystem (CCU, sensors, power, etc.). The catalog only stores the section name. The belonging of people to sections is deduced from the `owner` and `sectionRef` fields in the `data` entities.

**Example:**
```json
"sections": {
  "ccu": { "name": "Central Control Unit" },
  "sensors": { "name": "Safety Sensors" },
  "power": { "name": "Power Distribution" }
}
```

**c) `connectorModels`** — Generic connector models.
```
"<model_id>": {
  "manufacturer": "string (optional)",
  "partNumber": "string (optional)",
  "pins": "number (optional)",
  "gender": "'male' | 'female' (optional)",
  "type": "string (optional)",
  "datasheetUrl": "string / null (optional)",
  "specs": "object (optional, free form)"
}
```
It is referenced from `connectors.modelRef`. It provides documentary information (datasheet, technical specs) that is displayed in the detail panel in read-only mode. The `pins` and `gender` fields are used for autocomplete (see 1.5).

**Important note**: the same physical model with opposite genders (e.g., Molex 2P male and Molex 2P female) must be represented as **two separate entries** in the catalog, since they generally have different part numbers.

**d) `wireTypes`** — Generic wire types.
```
"<type_id>": {
  "unit": "'AWG' | 'mm2' (mandatory)",
  "shielded": "boolean (optional)",
  "insulationType": "string (optional)",
  "temperatureRating": "string (optional)",
  "standards": ["array of strings (optional)"]
}
```
It is referenced from `wires.wireTypeRef`. It provides documentary information about the wire type. **It does not contain the `gauge` value**; that value lives exclusively in the instance (`data.wires.gauge`). The `unit` field is used to autocomplete `gaugeUnit` in the instance (see 1.5).

**e) `nets`** — Standard electrical signals (V2.1).
```
"<net_id>": {
  "name": "string (mandatory)",
  "signalType": "'power' | 'ground' | 'data' | 'communication' | 'analog' (mandatory)",
  "voltage": "string (optional)",
  "standard": "string (optional)",
  "colorCode": "string (optional, ref to colorPalette)",
  "description": "string (optional)"
}
```
Nets are reusable standard signals (GND, +12V, +5V, CAN_H, CAN_L, etc.). They are referenced from `wires.net` and `mates.net` by their ID in the catalog. **`data.nets` does not exist** (removed in V2.1).

> **Note (V2.3):** The value `"shield"` has been removed from `signalType`. Shielding/braid is a physical property of the wire, not an electrical signal. It is documented via `wireTypes.shielded`. The `signalType` list is extensible: new types can be added to the catalog in future versions or by direct JSON editing.

**f) `colorPalette`** — Color palette for SVG rendering (V2.3).
```
"<color_name>": "string (hexadecimal)"
```
Maps descriptive color names to hexadecimal values for SVG rendering. The `color` fields in wires and `colorCode` in nets reference keys from this catalog.

**Example:**
```json
"colorPalette": {
  "black": "#000000",
  "red": "#dc2626",
  "blue": "#2563eb",
  "green": "#16a34a",
  "yellow": "#eab308",
  "orange": "#ea580c",
  "white": "#f5f5f5",
  "gray": "#6b7280",
  "brown": "#92400e",
  "violet": "#7c3aed",
  "Green/White": "#22c55e",
  "White/Green": "#e5e7eb"
}
```

**Usage rules:**
- If a `color` or `colorCode` field references a key that **does not exist** in `colorPalette`, the default gray color (`#6b7280`) is used and a **Warning** is emitted in the log panel.
- The user can customize colors by editing the catalog directly in the JSON.
- The palette is extensible: new entries can be freely added.

### 2.9 Validation Schema (`metadata.schema`)

2.9.1. The `schema` object defines **dynamic** validation rules that the application interprets automatically when loading or modifying the JSON. This allows changing validation rules without modifying the JavaScript code.

2.9.2. **Implementation (V2.5):** Validation is implemented in **100% custom JavaScript** within `js/data.js`. The `validateProject()` function handles schema rules (required, recommended, pattern, unique, ref) and business rules (sections 10.1–10.12, 10.14–10.16). **Model-instance consistency validation has been removed** to simplify logic; the instance is always the source of truth without cross-checking against catalog values.

2.9.3. **Unified validation (V2.2):** All validation runs from a **single `validateProject()` function** that:
1. Iterates over each `data` array (containers, connectors, wires, mates).
2. Applies the corresponding sub-schema to each entity (required, recommended).
3. Applies global rules (unique, pattern, ref).
4. Applies business rules (10.1 to 10.12 and 10.14 to 10.16).
5. Returns an array of objects `{ level: "error"|"warning"|"info", message: string, entityId: string }`.

This function is called:
- **When loading** the file: full validation.
- **When saving/exporting**: full validation.
- **In real time**: only the entity being edited in the `details` panel is validated (incremental validation, for performance).

2.9.4. **Entity-specific schema (V2.3):** The schema is structured by entity type, since each one has different mandatory and recommended fields:

```json
"schema": {
  "containers": {
    "required": ["id", "type", "position", "size"],
    "recommended": ["name", "owner", "sectionRef"]
  },
  "connectors": {
    "required": ["id", "type", "gender", "mountType", "edgeSide", "parent_id", "pins"],
    "recommended": ["name", "owner", "modelRef"]
  },
  "wires": {
    "required": ["id", "type", "from", "to", "net"],
    "recommended": ["name", "owner", "gauge", "color", "wireTypeRef"]
  },
  "mates": {
    "required": ["id", "type", "from", "to", "net"],
    "recommended": ["name", "owner", "pinMapping"]
  },
  "rules": {
    "id": { "unique": true, "pattern": "^[TCWM]\\d{3}$" },
    "containers.parent_id": { "ref": "containers" },
    "connectors.parent_id": { "ref": "containers" },
    "connectors.modelRef": { "ref": "connectorModels" },
    "connectors.owner": { "ref": "people" },
    "wires.from.connector": { "ref": "connectors" },
    "wires.to.connector": { "ref": "connectors" },
    "wires.net": { "ref": "nets" },
    "wires.wireTypeRef": { "ref": "wireTypes" },
    "wires.owner": { "ref": "people" },
    "mates.from.connector": { "ref": "connectors" },
    "mates.to.connector": { "ref": "connectors" },
    "mates.net": { "ref": "nets" },
    "mates.owner": { "ref": "people" }
  }
}
```

2.9.5. **Schema fields per entity**:
- `required`: Array of strings with names of fields that must exist in every entity **of that type**. If a required field is missing, an **Error** is generated.
- `recommended`: Array of strings with names of fields whose absence generates a **Warning** (not an error).

2.9.6. **Rule types** (in `rules`):
- `"unique": true` → The value must be unique in the entire array of that entity. **Custom validation**.
- `"pattern": "regex"` → The value must comply with the regular expression. **Custom validation**.
- `"ref": "catalog_name"` → The value must match an existing key in the specified catalog (within `metadata.catalogs`) or in the corresponding `data` array. **Custom validation**.

2.9.7. **Path conventions in `rules`**:
- The rule `"id"` without an entity prefix applies to **all** entities of all arrays in `data`.
- `"containers.parent_id"` applies only to the `parent_id` field within objects in the `data.containers` array.
- `"ref": "containers"` validates against IDs in `data.containers`.
- `"ref": "connectorModels"` validates against keys in `metadata.catalogs.connectorModels`.
- `"ref": "nets"` validates against keys in `metadata.catalogs.nets`.
- `"ref": "people"` validates against keys in `metadata.catalogs.people`.

2.9.8. **Separation criterion schema vs code**:
- **Schema (custom validation):** Validations of **referential integrity** and **format** (unique, pattern, ref, required, recommended).
- **Code (rules 10.1–10.12 and 10.14–10.16):** **Business logic** validations (opposite genders, kinematics, M composition, hierarchical compatibility, flying connectors without a partner, hierarchical cycles).

### 2.10 User and Responsible Person Management

2.10.1. **`metadata.savedBy`**: Records the `person_id` of the person who performed the last save of the file. It is automatically updated on export.

2.10.2. **`owner` field**: Available in all `data` entities (containers, connectors, wires, mates). Indicates the person responsible for that specific entity. It is optional and references `catalogs.people`.

2.10.3. **`sectionRef` field**: Available **only in containers** (V2.2). Links the container to a functional section defined in `catalogs.sections`. It allows filtering and grouping entities by subsystem.

2.10.4. **Section inheritance in connectors (V2.2):** Connectors **do not have their own `sectionRef` field**. They always inherit the section from their parent container. If the parent container has `sectionRef: "ccu"`, all its descendant connectors belong to the "ccu" section. If the parent container does not have `sectionRef`, it is inherited from the grandparent, and so on up to the root. If no ancestor has `sectionRef`, the connector has no assigned section. The inheritance chain is limited to **4 levels** of depth (see rule 10.16).

> **Note (V2.5):** Section inheritance uses the same 4-level depth limit as container hierarchy. Connectors never have their own `sectionRef`; they always inherit from the nearest ancestor container that has one.

2.10.5. The interface displays the current user name (from `catalogs.people`) and allows changing it from the configuration panel. When saving, `savedBy` is automatically updated with the active user.

**Design Memory – Section 2**
The single entity table facilitates quick consultation. The `type` field is kept because the letter `T` alone does not distinguish between system, box, or board. The decision to use hundreds as hierarchical levels allows for comfortable growth. The separation of `position` and `size` responds to the need to treat location and dimensions independently. In V2.3, the `people` catalog is simplified (only name), `owners` in `sections` is removed, `colorPalette` is added, and entity-specific schema is adopted with 100% custom validation.

---

## 3. CONTAINERS (SYSTEM, ENCLOSURE, PCB)

They are the **containers** that define the physical structure of the harness. They have no gender or pins; their function is to group connectors and limit their movement.

### 3.1 Common Attributes

**Table 2 – Container attributes**

| Field      | Type         | Description |
|------------|--------------|-------------|
| id         | string       | T100, T200, T300… |
| type       | string       | `"system"`, `"enclosure"`, `"pcb"` |
| name       | string       | Descriptive name (e.g., "Box 1") |
| parent_id  | string / null| ID of the parent container (`null` only for the root system) |
| designator | string       | Technical label (e.g., "BOX1", "PCB1") |
| position   | object       | **If `parent_id` is `null`**: `{ x, y }` in absolute pixels. **If `parent_id` is not `null`**: `{ offsetX, offsetY }` in pixels relative to parent. |
| size       | object       | `{ width, height }` in pixels |
| owner      | string / null| Ref to `catalogs.people`. Container responsible. Optional. |
| sectionRef | string / null| Ref to `catalogs.sections`. Functional section to which it belongs. Optional. |
| notes      | array        | Note history (see 3.1.1) |

#### 3.1.1 Note Structure

Each entry in the `notes` array is an object with the format:
- `date` (string, ISO 8601): date of note creation.
- `user` (string): identifier of the user who wrote it.
- `text` (string, maximum 500 characters): comment content.

The `notes` field is available in **all** entities (containers, connectors, wires, mated). It is conceived as a cumulative history of technical annotations.

### 3.2 Container Catalog

**Table 3 – List of containers in the example**

| ID   | Type       | Name            | Parent | Designator | Position (offsetX, offsetY) | Size (w, h) | Owner  | SectionRef | Notes |
|------|------------|-----------------|--------|------------|-----------------------------|-------------|--------|------------|-------|
| T100 | system     | Electric Motorcycle | null   | EMOTO-Z1   | x:50, y:50 *(absolute)*     | 2600, 1200  | leo    | –          | –     |
| T200 | enclosure  | Box 1           | T100   | BOX1       | 90, 110                     | 950, 1000   | martin | ccu        | –     |
| T201 | enclosure  | Box 2           | T100   | BOX2       | 1593, 122                   | 950, 1000   | nico   | sensors    | –     |
| T300 | pcb        | PCB 1           | T200   | PCB1       | 80, 100                     | 368, 837    | martin | ccu        | –     |
| T301 | pcb        | PCB 2           | T201   | PCB2       | 427, 117                    | 472, 817    | nico   | sensors    | –     |

**Design Memory – Section 3**
The term "containers" groups system, enclosure, and pcb under the same concept. The hierarchy of `parent_id` null only in the root system avoids orphaned components. The position values have been converted to relative (offsetX/offsetY) except for T100, which maintains absolute coordinates as the root. The separation of `position` and `size` clarifies the data structure.

### 3.3 Movement and Resizing

#### 3.3.1 Container Movement

3.3.1.1. When dragging a container, all its descendants move by the same vector **(dx, dy) in real time**, without delay.
3.3.1.2. The dragged container **does not modify its size** or any other property; only its position values change (`offsetX`/`offsetY`, or `x`/`y` if it is T100).
3.3.1.3. Fixed connectors (`mountType: "fixed"`) are automatically repositioned on the parent's edge in the same frame. Flying connectors (`mountType: "flying"`) are not kinematically anchored to their container: their position is recalculated each frame from the position of the fixed connector to which they are plugged. Therefore, a flying connector moves only if the fixed one moves (by direct drag or by movement of the container that contains the fixed one). If the flying connector's container moves but the fixed one does not, the flying connector stays in place, which may cause it to be outside its container's limits (see 10.13, log panel).

**Design Memory – 3.3.1**
Solidary real-time movement avoids flickering. Since all positions are relative, moving a container does not require updating the coordinates of its descendants; the rendering recalculates absolute positions from the offsets in each frame.

#### 3.3.2 Resizing

3.3.2.1. Containers can be resized by dragging **exclusively the bottom-right corner**.
3.3.2.2. During resizing:
 a. Fixed connectors (`mountType: "fixed"`) keep their distance to the top-left corner of the container, which is the fixed corner during stretching.
 b. Flying connectors (`mountType: "flying"`) are not affected by the resizing of their parent container; their position is recalculated from their fixed partner in the M.

**Design Memory – 3.3.2**
Restricting resizing to a single corner (bottom right) simplifies implementation and avoids the inverse resizing problems detected in previous versions.

---

## 4. CONNECTORS

Connectors are the electrical connection points. Each has a gender, a mount type, and belongs to a physical container.

### 4.1 Attributes

**Table 4 – Connector attributes**

| Field     | Type          | Description |
|-----------|---------------|-------------|
| id        | string        | C001, C002… |
| type      | string        | Always `"connector"` |
| name      | string        | Descriptive name |
| parent_id | string        | ID of the container where it is physically located |
| designator| string        | Technical label (e.g., "J1") |
| pins      | number        | Number of pins |
| gender    | string        | `"male"` or `"female"` (mandatory) |
| mountType | string        | **Mandatory.** `"fixed"` or `"flying"` |
| edgeSide  | string        | **Mandatory.** `"left"`, `"right"`, `"top"`, `"bottom"` |
| offset    | number / null | Distance from the reference end of the edge. **Only operational for `mountType: "fixed"`.** For `"flying"`, this field is **ignored** and can be `null` or any value; it has no effect on positioning and does not generate warnings. |
| size      | object        | `{ width, height }` in pixels |
| matedId   | string / null | ID of the M coupling to which it belongs, or `null` if it is free |
| modelRef  | string / null | Ref to `catalogs.connectorModels`. **Optional, informative.** |
| owner     | string / null | Ref to `catalogs.people`. Connector responsible. Optional. |
| notes     | array         | Note history (see 3.1.1) |

> **Note (V2.2):** Connectors **do not have a `sectionRef` field**. They always inherit the section from their parent container (see 2.10.4).

> **⚠️ Critical precedence note (V2.0):** The fields `name`, `pins`, and `gender` of the instance in `data` are the **absolute source of truth**. `modelRef` only enriches the detail panel view with datasheet, manufacturer, and technical specifications from the catalog, and provides values for autocomplete (see 1.5). No consistency between catalog and instance is validated.

4.1.1. `mountType` defines the kinematics of the connector:
 a. **Fixed (`"fixed"`)**: the connector is screwed or soldered to its parent container. Its position is calculated from `edgeSide` and `offset`. When moving the container, the connector moves with it.
 b. **Flying (`"flying"`)**: the connector is the end of a wire. It is not mechanically fixed to its container; its `parent_id` is declarative/organizational (indicates in which box the wire physically is, allows calculating the lowest common ancestor for wires, and validates hierarchical compatibility), but does not influence its position. This is calculated from the fixed connector to which it is plugged (its partner in the M). When moving the parent container, the flying connector **does not move** with it; it follows its fixed partner. If as a result it ends up outside its container's limits, the system logs a warning in the log panel (see 10.13, log panel), but does not block the action.

4.1.2. Definition of `offset` (only for `mountType: "fixed"`):
 a. For `"left"` or `"right"`: `offset` is the distance from the top edge of the parent container to the top edge of the connector.
 b. For `"top"` or `"bottom"`: `offset` is the distance from the left edge of the parent container to the left edge of the connector.

4.1.3. Absolute position calculation at runtime for **fixed** connectors:

| `edgeSide` | `x_global` | `y_global` |
|------------|------------|------------|
| `"left"`   | `parent.x` | `parent.y + offset` |
| `"right"`  | `parent.x + parent.width - connector.width` | `parent.y + offset` |
| `"top"`    | `parent.x + offset` | `parent.y` |
| `"bottom"` | `parent.x + offset` | `parent.y + parent.height - connector.height` |

Where `parent.width` and `parent.height` come from the `size` of the parent container, and `connector.width` and `connector.height` from the `size` of the connector.

**Important note:** `parent.x` and `parent.y` in the above formulas are the absolute coordinates of the parent container, calculated recursively from its `position` and those of its ancestors. They should not be confused with the `offsetX`/`offsetY` values stored in the parent.

4.1.4. Absolute position calculation for **flying** connectors:
 a. The fixed connector with which it shares `matedId` is located.
 b. The opposite orientation is applied: if the fixed is `"right"`, the flying is placed with its left edge touching the right edge of the fixed. If the fixed is `"left"`, the flying is placed with its right edge touching the left edge of the fixed. Analogous for `"top"` and `"bottom"`.
 c. The perpendicular coordinate is aligned by **pin 1**: pin 1 of both connectors is at the same height (or horizontal position, depending on orientation). Additional pins of the larger connector extend in the corresponding direction.

> **Note (V2.3):** Alignment by pin 1 assumes that both connectors have the same numbering orientation (pin 1 at the top/left end). If a connector has non-standard numbering, use `pinMapping: null` in the coupling and manually specify the pins in each wire/mate.

4.1.5. `matedId` links the connector with the M coupling that joins it to its partner. If the connector participates in an M, the ID of that M is stored here.

4.1.6. **Flying connectors without a partner (V2.1):** If a connector has `mountType: "flying"` and `matedId: null` (or `matedId` points to a non-existent M), rule 10.14 applies: an Error is generated in logs, it is omitted from the SVG visualization, and it is highlighted in the table. The user can edit it, but it will remain marked until it has a valid partner.

4.1.7. **Fixed connectors without a partner (V2.2):** If a connector has `mountType: "fixed"` and `matedId: null`, it is a **valid** situation (soldered but not connected connector, reserve for future expansion). An **Info** message is generated in the log panel: "Fixed connector [ID] has no partner (may be intentional)". The connector is drawn normally in the SVG and is not highlighted in the table.

4.1.8. **Pin distribution inside the connector (V2.3):**
- Pins are distributed **vertically** within the connector (for `edgeSide: "left"` or `"right"`) or **horizontally** (for `"top"` or `"bottom"`).
- Pin spacing: `connectorHeight / (pins + 1)` for vertical distribution, or `connectorWidth / (pins + 1)` for horizontal distribution.
- Pin 1 is at the position closest to the top edge (or left edge).
- Each pin is drawn as a `<circle>` with a 5px radius on the connector edge.
- The wire connection point is the center of the pin circle.

**Design Memory – 4.1**
The introduction of `mountType` in V1.9 resolves the kinematic conflict. Flying connectors do not store an operational `offset`; their position is always derived from their fixed partner. The `modelRef` and `owner` fields are added in V2.0 as optional additives. In V2.1, pin 1 alignment and behavior of flying connectors without a partner are specified. In V2.2, `sectionRef` is removed from connectors and the behavior of fixed connectors without a partner is defined. In V2.3, it is clarified that `offset` is ignored in flying connectors, the pin distribution is specified, and a note on numbering orientation is added.

### 4.2 Gender and Validation

4.2.1. Every connector has a mandatory gender (`male` or `female`).
4.2.2. **M** couplings require opposite genders. It does not matter which is `from` or `to`.
4.2.3. The system will verify gender consistency when loading or editing.

### 4.3 Complete Connector Catalog

**Table 5 – Connectors in the example**

| ID   | Name     | Parent | Designator | Pins | Gender | MountType | EdgeSide | Offset | Size (w,h) | matedId | ModelRef           | Owner  | Notes |
|------|----------|--------|------------|------|--------|-----------|----------|--------|------------|---------|--------------------| ------ |-------|
| C001 | Molex 2P | T300   | J1         | 2    | male   | fixed     | right    | 100    | 180, 115   | M001    | MOLEX_2P_MALE      | martin | –     |
| C002 | Molex 2P | T200   | J2         | 2    | female | flying    | left     | null   | 180, 115   | M001    | MOLEX_2P_FEMALE    | martin | –     |
| C003 | GX12     | T200   | J3         | 2    | female | fixed     | right    | 740    | 180, 115   | M002    | GX12_2P_FEMALE     | martin | –     |
| C004 | GX12     | T100   | J4         | 2    | male   | flying    | left     | null   | 180, 115   | M002    | GX12_2P_MALE       | leo    | –     |
| C005 | GX12 4P  | T100   | J5         | 4    | male   | flying    | right    | null   | 180, 115   | M003    | GX12_4P_MALE       | leo    | –     |
| C006 | GX12 4P  | T201   | J6         | 4    | female | fixed     | left     | 728    | 180, 115   | M003    | GX12_4P_FEMALE     | nico   | –     |
| C007 | Molex 5P | T201   | J7         | 5    | male   | flying    | left     | null   | 180, 115   | M004    | MOLEX_5P_MALE      | nico   | –     |
| C008 | Molex 5P | T301   | J8         | 5    | female | fixed     | right    | 71     | 180, 115   | M004    | MOLEX_5P_FEMALE    | nico   | –     |
| C009 | Molex 2P | T301   | J9         | 2    | female | fixed     | left     | 251    | 180, 115   | null    | MOLEX_2P_FEMALE    | nico   | See note |

**Note for C009**: `[{"date":"2026-07-12","user":"leo","text":"Reserve for auxiliary headlight"}]`

**Design Memory – 4.3**
Connectors C002, C004, C005, and C007 are flying: they are the ends of wires. They do not store an operational `offset`. Connectors C001, C003, C006, C008, and C009 are fixed. `modelRef` and `owner` are added in V2.0. Each gender of the same physical model has its own entry in `connectorModels`. C009 is a fixed connector without a partner (reserve), a valid situation that generates Info in logs.

### 4.4 Connector Positioning

4.4.1. Fixed connector: completely inside the container, with the pin face exactly on the edge indicated by `edgeSide`.
4.4.2. Flying connector: its position is calculated to face its fixed partner (see 4.1.4). It may end up outside the limits of its parent container; this generates a warning in the log panel (10.13, log panel) but is not blocked.
4.4.3. Example: `edgeSide: "right"` on a fixed → right edge of the connector touches the right edge of the container.

### 4.5 Connector Movement

4.5.1. **Fixed connectors**: can **slide along the edge** (changing their `offset`) or **change to another edge** of the same container if the cursor exceeds 30 px of perpendicular distance from the current edge.
4.5.2. **Flying connectors**: cannot be dragged directly. Their position is automatically updated when moving the fixed connector to which they are plugged.
4.5.3. **Rigid movement propagation:**
 4.5.3.1. Connectors joined by an **M** move solidarily. When moving a fixed connector (directly or by moving its container), the associated flying connector is automatically repositioned to maintain the coupling, ignoring its own container for position purposes.
 4.5.3.2. Connectors joined only by wires do not drag each other; the wire is redrawn.

---

## 5. FIXED WIRES (WIRED) — ENTITY TYPE W

They represent a real physical conductor (soldered or crimped wire). They do not transmit mechanical movement.

### 5.1 Attributes

**Table 6 – Wire attributes**

| Field      | Type   | Description |
|------------|--------|-------------|
| id         | string | W001, W002… |
| type       | string | Always `"wired"` |
| from       | object | `{ connector: "C002", pin: 1 }` |
| to         | object | `{ connector: "C003", pin: 1 }` |
| net        | string | ID of the net it carries (ref to `catalogs.nets`). Mandatory. |
| length     | number | Real length in mm. **Direct and highly editable field.** |
| gauge      | number / null | Numeric gauge value. **Recommended, not mandatory.** If missing, a Warning is generated in logs. |
| gaugeUnit  | string / null | `"AWG"` or `"mm2"`. **Optional.** If `wireTypeRef` exists, it is inferred from the catalog (see 1.5). If it does not exist, defaults to `"mm2"`. |
| color      | string | Line color (ref to `catalogs.colorPalette`). Default: `"black"`. If the wire has `net` and the net has `colorCode`, the color is autocompleted from the net (see 1.5). |
| thickness  | number | Thickness in px (optional) |
| wireTypeRef| string / null | Ref to `catalogs.wireTypes`. Optional, informative. |
| owner      | string / null | Ref to `catalogs.people`. Optional. |
| notes      | array  | Note history |

> **Note (V2.3):** The `gauge` field is **recommended but not mandatory**. The wire cross-section is not visually represented in the SVG drawing (visual thickness is controlled by `thickness`), so the absence of `gauge` does not prevent rendering. If `gauge` is missing, a **Warning** is generated in the log panel to remind the user to document it. In the example, all wires have `gauge` filled in as good documentation practice.

> **Note (V2.5):** The `gaugeUnit` field is **optional but recommended**. When creating a new wire via UI, it defaults to `"mm2"`. If `wireTypeRef` exists and has a `unit`, it autocompletes from catalog. User can overwrite. Missing `gaugeUnit` generates a Warning only if `gauge` is present but unit is ambiguous.

> **Note (V2.1):** The `gauge` field is a **number** that lives exclusively in the instance. The `wireTypes` catalog **does not contain** the gauge value; it only provides documentary information (insulation, standards, temperature) and the unit (`unit`).

5.1.1. The physical location is automatically deduced from the lowest common ancestor of the two end connectors.

### 5.2 Wire Table from the Example

**Table 7 – Wires of the harness**

| ID   | From (C, pin) | To (C, pin) | Net | Length  | Gauge | GaugeUnit | WireTypeRef        | Owner  |
|------|---------------|-------------|-----|---------|-------|-----------|--------------------|--------|
| W001 | C002, 1       | C003, 1     | GND | 150 mm  | 22    | AWG       | AWG22_SHIELDED     | martin |
| W002 | C004, 1       | C005, 1     | GND | 400 mm  | 22    | AWG       | AWG22_UNSHIELDED   | leo    |
| W003 | C006, 1       | C007, 2     | GND | 180 mm  | 22    | AWG       | AWG22_SHIELDED     | nico   |

### 5.3 Visualization and Behavior

5.3.1. Curved line (cubic Bézier in SVG) of the color specified in `color` (resolved via `catalogs.colorPalette`), always visible.
5.3.2. Always above boxes and connectors.
5.3.3. Redrawn as an adaptive curve when moving an end.
5.3.4. The line shortens/stretches freely; `length` is only informative.
5.3.5. **Wires with invalid ends (V2.2):** If either of the two end connectors does not have a valid position (e.g., flying connector without a partner, rule 10.14), the wire **is not drawn** in the SVG. The wire still appears in the data table, but not in the visual view. No additional error is generated for the wire; the error is already generated by the invalid connector.

5.3.6. **Bézier curve formula (V2.3):**
- Start point (`x1, y1`): center of the source pin (on the connector edge).
- End point (`x2, y2`): center of the destination pin.
- Control points: offset according to the `edgeSide` of the corresponding connector.
  - If the source connector has `edgeSide: "right"`, the first control point is offset in +X.
  - If the source connector has `edgeSide: "left"`, the first control point is offset in -X.
  - Analogous for the destination connector.
- Offset magnitude: `cx = Math.max(50, Math.abs(x2 - x1) * 0.4)`.
- SVG path: `M x1,y1 C x1+cx,y1 x2-cx,y2 x2,y2` (for left/right facing connectors).
- For top/bottom orientations, the offset is in Y instead of X.

---

## 6. PLUGGABLE COUPLINGS (MATED) — ENTITY TYPE M

Defines a male-female union between two connectors. Rigid union: the connectors move together, without an additional visual line.

### 6.1 Attributes

**Table 8 – Mated coupling attributes**

| Field     | Type   | Description |
|-----------|--------|-------------|
| id        | string | M001, M002… |
| type      | string | Always `"mated"` |
| from      | object | `{ connector: "C001", pin: 1 }` |
| to        | object | `{ connector: "C002", pin: 1 }` |
| net       | string | ID of the net it carries (ref to `catalogs.nets`) |
| pinMapping| string / null | `"direct"`, `"reversed"`, or `null`. If not specified, it is treated as `null`. |
| owner     | string / null | Ref to `catalogs.people`. Optional. |
| notes     | array  | Note history |

### 6.2 Validations

6.2.1. The two connectors must have opposite genders.
6.2.2. If a connector has `matedId`, that ID must match the M that includes it. Conversely, if a connector appears in the `from` or `to` of an M, it must have that `matedId`. The system will reject any configuration that does not meet this bidirectional correspondence.
6.2.3. The system will verify consistency on load: each M must be referenced by its two connectors.
6.2.4. **Composition of M:** an M always connects a fixed connector (`mountType: "fixed"`) with a flying one (`mountType: "flying"`). M between two fixed or two flying are not allowed.
6.2.5. **Pin mapping:**
 a. If `pinMapping` is `"direct"`, the coupling respects the natural order: pin 1 with pin 1, pin 2 with pin 2, etc. The system will validate that the pins declared in `from` and `to` comply with this correspondence.
 b. If `pinMapping` is `"reversed"`, the order is reversed: pin 1 with pin N, pin 2 with N‑1, etc., where N is the number of pins of the smaller connector.
 c. If `pinMapping` is `null` or not present, no automatic mapping validation is applied.
 d. If the connectors have a different number of pins and `pinMapping` is `"direct"` or `"reversed"`, the system will issue an informative warning indicating how many pins of the larger connector are left unassigned in this coupling.

6.2.6. **`pinMapping` autocomplete (V2.3):** When creating a new mate from the interface, the `pinMapping` field is autocompleted with `"direct"` as a suggested value (inference principle, see 1.5). The user can change it to `"reversed"` or `null` as needed. The mapping validation is only activated if the value is explicitly `"direct"` or `"reversed"`. This autocomplete applies only to **new mates created from the interface**. Existing mates in the JSON keep their original value (e.g., M004 with `pinMapping: null`).

6.2.7. **Coupling model limitation (V2.3):** An M connects **exactly two connectors** (a `from` and a `to`). Atypical cases such as a connector of N pins connected to two connectors of N/2 pins must be represented as **two separate M**, each with `pinMapping: null` (without automatic validation, because the mapping is partial). This case is extremely rare in electric motorcycle harnesses and no complexity is added to the model to support it natively.

### 6.3 Mated Table from the Example

**Table 9 – Mated couplings**

| ID   | From (C, pin) | To (C, pin) | Net | pinMapping | Owner  |
|------|---------------|-------------|-----|------------|--------|
| M001 | C001, 1       | C002, 1     | GND | direct     | martin |
| M002 | C004, 1       | C003, 1     | GND | direct     | leo    |
| M003 | C005, 1       | C006, 1     | GND | direct     | nico   |
| M004 | C007, 2       | C008, 2     | GND | null       | nico   |

> **Note (V2.3):** In the base example, all wires and mates share the same net (`GND`), reflecting the case of the original MVP where all connections were part of a single circuit. If one wishes to show signal diversity (power, data, communication), a separate additional example with fictitious components should be created, without modifying the base example.

### 6.4 Visualization

6.4.1. No line; connectors face each other edge to edge.
6.4.2. The rigid union manifests in the solidary movement.
6.4.3. **Visual rigid block**: since an M coupling represents a real, inseparable physical union, in the graphical view both connectors must appear strictly facing each other and without separation. The user cannot separate them by dragging in edit mode; the flying connector always follows the fixed one. If the mathematical position leaves a gap, a warning is issued (see 10.10).

---

## 7. DYNAMIC LOCKING (`lockedWith`)

### 7.1 Runtime Generation

7.1.1. `lockedWith` is not stored; it is calculated from all existing M (all are active by definition).
7.1.2. Each M causes both connectors to be mutually included in `lockedWith`.
7.1.3. Wires do not contribute.

### 7.2 Rigid Chains

7.2.1. In the example:
 a. C001‑C002 (M001)
 b. C003‑C004 (M002)
 c. C005‑C006 (M003)
 d. C007‑C008 (M004)
7.2.2. Wires join these chains but do not make them mechanically solidary.

---

## 8. LOGICAL SIGNALS (NETS) — CATALOG

### 8.1 Nature of Nets (V2.1)

8.1.1. As of V2.1, nets **are not data entities** but **entries in the catalog** `metadata.catalogs.nets`. This reflects their real nature: electrical signals (GND, +12V, +5V, CAN_H, CAN_L, etc.) are reusable standards across projects, not data specific to a harness.

8.1.2. **`data.nets` does not exist.** Wires and mates reference nets by their ID in the catalog (e.g., `"net": "GND"`, `"net": "+12V"`).

8.1.3. **The entity type `N` does not exist** in the entity index (Table 1). Nets do not have an ID with the prefix `N001`; they use descriptive names as catalog keys.

### 8.2 Attributes of a Net in the Catalog

**Table 10 – Net attributes in `catalogs.nets`**

| Field      | Type          | Description |
|------------|---------------|-------------|
| name       | string        | Descriptive name (mandatory) |
| signalType | string        | Category: `"power"`, `"ground"`, `"data"`, `"communication"`, `"analog"` (mandatory). Extensible list. |
| voltage    | string / null | Nominal voltage (optional) |
| standard   | string / null | Applicable standard (e.g., "ISO 11898-2" for CAN) |
| colorCode  | string / null | Suggested color for visualization (ref to `catalogs.colorPalette`) |
| description| string / null | Additional description |

> **Note (V2.3):** The value `"shield"` has been removed from `signalType`. Shielding is a physical property of the wire (`wireTypes.shielded`), not a signal. The `signalType` list is extensible: new types can be added to the catalog in future versions or by direct JSON editing.

### 8.3 Net Catalog from the Example

**Table 11 – Nets defined in `catalogs.nets`**

| ID (key) | Name          | SignalType | Voltage | Standard    | ColorCode |
|----------|---------------|------------|---------|-------------|-----------|
| GND      | System Ground | ground     | 0V      | –           | black     |

> **Note (V2.3):** The base example uses a single net (`GND`) for all connections, maintaining consistency with the original MVP. The catalog can contain more nets (e.g., `+12V`, `+5V`, `CAN_H`, `CAN_L`) for use in real projects or additional examples, but the base example only uses `GND`.

### 8.4 Use of Nets

8.4.1. Check electrical continuity: the system examines wires and mateds with the same net and builds a graph of nodes (connector, pin).
8.4.2. Validate signal consistency: all wires and mates that share a net must be electrically compatible.
8.4.3. Visual highlighting when selecting a net: all wires and mates carrying that signal are highlighted.
8.4.4. Autocomplete: when creating a wire or mate, the user selects the net from the catalog (dropdown with all available nets). The wire's `color` field is autocompleted with the `colorCode` of the selected net (see 1.5).
8.4.5. Future: implicit chassis points.

### 8.5 Automatic Graph Construction

8.5.1. The system examines wires and mateds with the same net and builds a graph of nodes (connector, pin).
8.5.2. The route of the example for `GND`: C001.pin1 – C002.pin1 – C003.pin1 – C004.pin1 – C005.pin1 – C006.pin1 – C007.pin2 – C008.pin2.

> **Note (V2.3):** In the example, all segments share the net `GND`, reflecting a single circuit. In a real harness, each segment would carry its specific signal (power, data, ground, etc.). Signal diversity is shown in additional examples, not in the base example.

---

## 9. CONTAINMENT TREE (PHYSICAL HIERARCHY)

### 9.1 Hierarchical Diagram

```
T100 (Motorcycle) [owner: leo]
├── T200 (Box 1) [owner: martin, section: ccu]
│   ├── T300 (PCB 1) [owner: martin, section: ccu]
│   │   └── C001 (J1, male, fixed, right) → inherits section: ccu
│   ├── C002 (J2, female, flying, left) → inherits section: ccu
│   └── C003 (J3, female, fixed, right) → inherits section: ccu
├── T201 (Box 2) [owner: nico, section: sensors]
│   ├── T301 (PCB 2) [owner: nico, section: sensors]
│   │   ├── C008 (J8, female, fixed, right) → inherits section: sensors
│   │   └── C009 (J9, female, fixed, left) → inherits section: sensors
│   ├── C006 (J6, female, fixed, left) → inherits section: sensors
│   └── C007 (J7, male, flying, left) → inherits section: sensors
├── C004 (J4, male, flying, left) → inherits section: null (T100 has no sectionRef)
└── C005 (J5, male, flying, right) → inherits section: null
```

### 9.2 Effective Wire Location

9.2.1. W001: inside T200.
9.2.2. W002: in T100.
9.2.3. W003: in T201.

---

## 10. CONSISTENCY RULES AND VALIDATIONS

10.1. **Gender in M:** opposite genders mandatory.
10.2. **matedId integrity:** if a connector has `matedId`, that M must exist, contain the connector, and both connectors of the M must have the same `matedId`. Conversely, if a connector appears in the `from` or `to` of an M, it must have that `matedId`; the system will reject any configuration that does not meet this bidirectional correspondence.
10.3. **edgeSide and mountType mandatory:** every connector must have `edgeSide` and `mountType` defined.
10.4. **Composition of M:** an M connects a fixed with a flying.
10.5. **Valid pins:** referenced pins must exist in the connectors (the pin number must be ≤ `pins` of the connector).
10.6. **Hierarchical compatibility:** two connectors can only connect (via a W wire or an M coupling) if they share a common ancestor container in the containment tree. Otherwise, the system will reject the connection.
10.7. **Wire length:** `length` is informative, without restriction.
10.8. **Net continuity:** the net graph must be connected; signal conflicts are detected.
10.9. **Pin mapping (optional):** if `pinMapping` is `"direct"` or `"reversed"`, the `from` and `to` pins must comply with the declared order. If the connectors have a different number of pins, an informative warning will be issued about the unassigned pins.
10.10. **Pin facing in M:** the two connectors sharing a `matedId` must have opposite orientations. Additionally, the system will verify that the positions are aligned on the coordinate perpendicular to the edge. For left/right couplings, the difference between their global Y coordinates must not exceed 2 pixels. For top/bottom couplings, the difference between their global X coordinates must not exceed 2 pixels. If this threshold is exceeded, a misalignment warning will be issued.

### 10.11 Reference Validation (`ref`)

10.11.1. The system must verify that all fields with the `ref` rule in `metadata.schema` point to existing IDs in their corresponding destination:
- `parent_id` → existing ID in `data.containers`.
- `modelRef` → existing key in `metadata.catalogs.connectorModels`.
- `wireTypeRef` → existing key in `metadata.catalogs.wireTypes`.
- `sectionRef` → existing key in `metadata.catalogs.sections` (only in containers).
- `owner` and `savedBy` → existing key in `metadata.catalogs.people`.
- `net` → existing key in `metadata.catalogs.nets`.
- `from.connector` and `to.connector` → existing ID in `data.connectors`.
- `color` and `colorCode` → existing key in `metadata.catalogs.colorPalette`.

10.11.2. If a reference points to a non-existent ID, an **Error** is generated in the log panel.
10.11.3. If the field is `null` or does not exist and is optional, no error is generated.
10.11.4. If a `color` or `colorCode` field references a key that does not exist in `colorPalette`, the default gray color (`#6b7280`) is used and a **Warning** is emitted.

### 10.12 Uniqueness Validation (`unique`)

10.12.1. Fields marked with `"unique": true` in `metadata.schema` must have different values in all entities of the same array.
10.12.2. If a duplicate value is detected, an **Error** is generated in the log panel indicating which entities share the same value.
10.12.3. The rule `"id"` without prefix applies to all entities of all arrays in `data`.

### 10.13 Log Panel (Active)

10.13.1. The log panel is a container located at the bottom of the screen, collapsible downwards via CSS transition (`translate-y-full`).

10.13.2. **Three severity levels:**
- **Error** (red `#ef4444`): Problems that prevent model coherence (broken references, violated uniqueness, same genders in an M, flying connectors without a partner, hierarchical cycles, etc.).
- **Warning** (yellow `#ffcc00`): Inconsistencies that do not break the model but deserve attention (flying connector outside parent container, M misalignment, missing recommended fields such as `gauge` or `owner`, non-existent color in colorPalette).
- **Info** (blue `#3b82f6`): Informative messages (successful load, export completed, unassigned pins in an M with different number of pins, fixed connector without a partner).

10.13.3. The panel automatically integrates the results of the unified `validateProject()` function (see 2.9.3), which includes:
- The custom `schema` validation (required, recommended, pattern, unique, ref).
- The business rules (sections 10.1 to 10.12 and 10.14 to 10.16). Section 10.13 describes the log panel itself, not a validation rule.
- Relevant user events (import, export, deletion).

10.13.4. The panel includes:
- Filter by message type (show/hide errors, warnings, info).
- Button to clear all logs.
- Message counter per level.
- Timestamp on each entry.

### 10.14 Flying Connectors Without a Partner (V2.1)

10.14.1. If a connector has `mountType: "flying"` and `matedId: null` (or `matedId` points to a non-existent M), an **Error** is generated in the log panel with the message: "Flying connector [ID] has no valid partner (null or non-existent matedId)".

10.14.2. **In the visual view (SVG):** the connector **is not drawn**. It is completely omitted from the SVG rendering. Wires connected to that connector are also not drawn (see 5.3.5).

10.14.3. **In the table view:** the connector row is **highlighted with a red border** (`border: 2px solid #ef4444`) and an error icon. The highlighting persists until the connector has a valid `matedId`.

10.14.4. The user **can edit** the connector from the `details` panel or the table (change `matedId`, `mountType`, etc.). Once the connector has a valid `matedId`, the error disappears from the log, the highlighting is removed, and the connector is drawn normally in the SVG.

10.14.5. This rule does not block file loading or prevent other operations. It is a persistent visual warning.

### 10.15 Offset in Flying Connectors (V2.3)

10.15.1. If a connector has `mountType: "flying"`, the `offset` field is **completely ignored**. It may be `null`, `0`, or any number. No warning or error is generated. This is intentional tolerance for legacy/manual data.

10.15.2. This rule is one of **tolerance**: it allows JSON files with inherited or manually edited data not to generate unnecessary noise in the logs for a field that does not apply to flying connectors.

### 10.16 Hierarchical Depth Limit and Cycle Detection (V2.3)

10.16.1. **Depth limit:** The `parent_id` chain in containers has a maximum limit of **4 levels** of depth (counting from 0). This covers the current model (system → enclosure → pcb, depth 2) with two levels of margin for expansion (e.g., system → enclosure → sub-enclosure → pcb, depth 3). If the limit is exceeded, an **Error** is issued: "Hierarchy too deep in [ID] (maximum 4 levels)".

10.16.2. **Cycle detection:** The `parent_id` chain must not contain cycles. A cycle occurs when a container is an ancestor of itself (e.g., T200.parent_id = T300 and T300.parent_id = T200). The system detects cycles using a set of visited IDs during the recursive traversal. If a cycle is detected, an **Error** is issued: "Hierarchical cycle detected in [ID]" and the position and inheritance calculation for the involved containers is stopped.

10.16.3. **`sectionRef` inheritance:** The section inheritance chain (see 2.10.4) respects the same 4-level limit. If exceeded, an Error is issued and inheritance stops.

10.16.4. **Absolute position calculation:** The recursive calculation of absolute positions (see 4.1.3) also respects the 4-level limit and cycle detection. If a cycle is detected or the limit is exceeded, the affected containers are positioned at (0, 0) as a fallback and an Error is issued.

---

## 11. GENERAL VISUALIZATION AND INTERFACE

### 11.1 Visualization Technology: Native SVG (V2.1)

11.1.1. The visual view is implemented with **native SVG** (not Canvas). All graphical elements are drawn as SVG elements manipulated directly from JavaScript using `document.createElementNS`.

11.1.2. **SVG elements used:**
- **Containers:** `<rect>` with rounded corners, semi-transparent fill, and `<text>` label.
- **Connectors:** `<rect>` with pins represented as small `<circle>` on the corresponding edge.
- **Wires:** `<path>` with cubic Bézier curves (command `C`). The stroke color is resolved via `catalogs.colorPalette`.
- **Couplings (M):** No line. Connectors are drawn edge to edge.
- **Pins:** `<circle>` with 5px radius, individually clickable.
- **Labels:** `<text>` with clear typography (minimum 14px on labels, 18-22px on container titles).

11.1.3. **Advantages of SVG over Canvas for this application:**
- Direct interaction with elements (click, hover, drag) without manual hit-testing.
- Perfect scaling with zoom (no pixelation).
- CSS styles directly applicable (colors, thicknesses, animations, glow filters).
- Easier debugging (inspect elements in the browser DOM).

11.1.4. **Fixed wires:** cubic Bézier curves in SVG, of the color specified in the `color` field (resolved via `colorPalette`), always visible. They are dynamically redrawn when moving connectors.

11.1.5. **M couplings:** no line; connectors face each other and are locked as a single rigid set.

11.1.6. **Connectors:** inside the container (fixed) or following their partner (flying); pins on the edge. Flying connectors without a valid partner (rule 10.14) **are not drawn**. Fixed connectors without a partner (rule 4.1.7) **are drawn** normally.

11.1.7. **Render order (Z-index, V2.3):** From bottom to top:
1. SVG background (black rectangle).
2. Containers (by hierarchy: system → enclosure → pcb).
3. Connectors (fixed and flying).
4. M couplings (no line, only facing connectors).
5. W wires (Bézier curves).
6. Wire labels (net, length).
7. Pins (small circles, clickable).
8. Resize handles (only in edit mode).
9. Selection overlay (fluorescent yellow border + glow).

11.1.8. **Zoom and Pan (V2.3):**
- A `<g id="viewport">` element wrapping all SVG content is used.
- **Zoom:** Mouse wheel. Modifies `transform: scale(z)` of the viewport, centered on the cursor position. Range: 0.2 to 3.0.
- **Pan:** Drag with **middle mouse button** (wheel). Modifies `transform: translate(px, py)` of the viewport. Alternative: `Shift` + left button.
- **Select:** Simple left click on an element.
- **Move:** Left click hold + drag on a draggable element (container or fixed connector).
- **Deselect:** Left click on empty background, or `Esc` key.
- Zoom and pan state is saved in `localStorage`.

### 11.2 Minimalist Header (V2.5)

11.2.1. The header contains exactly four zones:
- **Left**: App name (**ArnesViz**) + open file name.
- **Center**: View tabs (**Table** | **Visual**).
- **Right**: **Edit Mode Toggle** (always visible switch) + **Config Button** (⚙).

The Edit Mode Toggle allows activating/deactivating edit mode without opening the sidebar. Shortcut: `Ctrl+Shift+E`.

11.2.2. The header has a dark neumorphic aesthetic (see 11.8) and minimum fixed height to not take space away from the work area.

### 11.3 Single Reusable Sidebar (V2.2)

11.3.1. There is a **single side panel** located on the right edge of the screen. This panel has **two mutually exclusive states**: `config` and `details`.

11.3.2. **Opening and closing:**
- Pressing the **Config** button in the header opens (or changes) the panel to the `config` state.
- Clicking on an entity on the canvas or table opens (or changes) the panel to the `details` state.
- The panel includes a **close button (✕)** in its upper right corner. Pressing it closes the panel.
- **There is NO floating side button (◀) to reopen the panel.** The panel is reopened only by pressing Config or by selecting an entity.

11.3.3. **State memory (V2.2):**
- When closing the panel, the app **remembers the last state** (last entity selected in `details`, or `config`).
- When reopening the panel (by pressing Config or selecting an entity), the corresponding content is shown.
- The last state is saved in memory (not in localStorage), so it is lost when reloading the page.

11.3.4. **Transition between states:**
- When changing from `config` to `details` (or vice versa), the panel content is replaced with a smooth transition (`0.25s ease`).
- The `Esc` key closes the panel regardless of its current state.

11.3.5. The panel width is fixed (approximately 380px) and occupies all the available height below the header.

### 11.4 `details` State (Property Panel)

11.4.1. It is activated by selecting an entity (T, C, W, M) in the visual view or in the table.
11.4.2. It shows **all fields** of the selected entity organized in sections.
11.4.3. In **read-only mode**, all fields appear as non-editable text.
11.4.4. In **edit mode**, the fields become editable fields (inputs, selects, textareas). Changes are synchronized bidirectionally with the tables and the canvas.
11.4.5. If the entity has `modelRef` or `wireTypeRef`, an additional section is shown in **read-only** with the catalog information: manufacturer, part number, datasheet (link), specifications.
11.4.6. If the entity has `owner`, a dropdown selector with the people from `catalogs.people` is displayed.
11.4.7. The `notes` history is shown at the bottom of the panel, with the option to add new notes (which are automatically sealed with the current date and user).
11.4.8. **Incremental validation (V2.2):** While the user edits fields in the `details` panel, only the **current entity** is validated in real time. Errors and warnings are shown inline (next to the field) and updated in the log panel. The whole project is not revalidated on every change.
11.4.9. **Autocomplete (V2.3):** When creating a new entity or when changing a reference (e.g., `modelRef`, `wireTypeRef`, `net`), the inferable fields are autocompleted automatically according to the inference principle (see 1.5). The user can overwrite any autocompleted value.
11.4.10. **Action buttons (V2.3):**
- **Delete:** Only in edit mode. Opens a confirmation modal. If the entity has references (e.g., connector used in wires), it shows a warning with a list of affected entities.
- **Duplicate:** Creates a copy of the entity with a new auto-assigned ID.

### 11.5 `config` State (Configuration Panel)

11.5.1. It is activated by pressing the **Config** (⚙) button in the header.
11.5.2. It contains the following sections, organized from top to bottom:

**a) Current user:**
- Dropdown selector with the people from `catalogs.people`.
- On change, `metadata.savedBy` is updated.

**b) View and Rendering:**
- Visible columns selector (for the table view).
- Zoom options (numeric input + +/− buttons).
- Toggle to show/hide connector names, designators, wire lengths, etc.

**c) Data Management:**
- **Import JSON** button (shortcut `Ctrl+O`).
- **Export JSON** button (shortcut `Ctrl+S`).
- **Delete all** button.

**d) Catalog Management (read-only in V2.5):**
- List of people (`catalogs.people`): view.
- List of sections (`catalogs.sections`): view.
- List of connector models (`connectorModels`): view.
- List of wire types (`catalogs.wireTypes`): view.
- List of nets (`catalogs.nets`): view.
- List of colors (`catalogs.colorPalette`): view.

> **Note (V2.2):** Editing catalogs from the UI is postponed for a future version (see section 12.13). For now, catalogs are edited directly in the JSON file. The interface only allows viewing them.

11.5.3. **Import Flow:**
1. If `data` contains entities (non-empty arrays) **and there are unsaved changes**, a **confirmation modal** opens:
   - Title: "Unsaved changes"
   - Message: "There are unsaved changes. Do you want to export the current file before importing?"
   - Options: **[Export and continue]** | **[Discard and continue]** | **[Cancel]**
2. If the user chooses "Export and continue", an export of the current JSON is automatically executed before proceeding.
3. If the user chooses "Cancel", the import is aborted.
4. If `data` is empty **or there are no unsaved changes**, it imports directly without asking.

11.5.4. **Incompatible file detection (V2.2):**
1. When importing a JSON file, the system checks for the presence of obsolete fields: `data.nets`, `metadata.uiSettings`, `signalTypeRef`, `sectionRef` in connectors, `revision` in metadata, `shield` in signalType of nets.
2. If any of these fields are detected, the import is **rejected** and a modal is shown:
   - Title: "Incompatible format"
   - Message: "This file seems to be from a previous version. It cannot be imported directly. Manually update the file to the V2.4 format."
   - Options: **[Understood]**
3. No automatic migration is attempted.

11.5.5. **Delete All Flow:**
1. A **first modal** opens: "Are you sure you want to delete all current data?"
2. If confirmed, a **second modal** opens: "Do you want to export a backup copy before deleting?"
3. Options: **[Export and delete]** | **[Delete without saving]** | **[Cancel]**
4. Only after this double confirmation are all data cleared and the view reset.

11.5.6. **Protection of catalogs with references (V2.2):**
1. If the user tries to delete a catalog entry (person, model, wire type, net, section, color) that is being referenced by some entity in `data`, the system **prevents deletion**.
2. A modal is shown: "Cannot delete. This element is being used by [N] entities: [list of IDs]."
3. Options: **[Understood]**
4. If the entry has no references, it is deleted without additional confirmation.

### 11.6 Intermediate Filters (V2.2)

11.6.1. The interface includes a **filter panel** accessible from the toolbar (above the work area) or from the `config` panel.

11.6.2. **Filter controls:**
- **Text field**: search by ID, name, or designator. Partial match (substring, case-insensitive).
- **Dropdown "Type"**: All / Containers (T) / Connectors (C) / Wires (W) / Mates (M).
- **Dropdown "Net"**: All / (list of nets from the `catalogs.nets` catalog).
- **Dropdown "Section"**: All / (list of sections from the `catalogs.sections` catalog).
- **"Clear filters" button**: resets all filters to their default value.

11.6.3. **Filter combination:** Filters are combined with **AND** logic. An element must meet **all** active filters to be displayed.

11.6.4. **Effect on the visual view (SVG):**
- Elements that **match** the filters are displayed with normal opacity.
- Elements that **do not match** are dimmed (opacity `0.15`) but not completely hidden, to maintain visual context.
- Wires connected to dimmed connectors are also dimmed.

11.6.5. **Effect on the table view:**
- Rows that **match** the filters are displayed normally.
- Rows that **do not match** are **hidden** (not displayed).
- A counter is displayed: "Showing X of Y entities".

11.6.6. **Persistence:** The filter state is saved in `localStorage` under the key `arnesviz.filters`. It is restored when reloading the page.

11.6.7. **Shortcut:** The `F` key activates/deactivates focus on the search text field.

### 11.7 Keyboard Shortcuts

| Shortcut      | Action                                    |
|---------------|-------------------------------------------|
| `Ctrl+Shift+E`| Toggle edit mode (toggle in header)       |
| `Ctrl+S`      | Export JSON                              |
| `Ctrl+O`      | Import JSON                              |
| `Esc`         | Close side panel / Deselect all          |
| `F`           | Activate/deactivate search filter        |

### 11.8 Visual Aesthetics (V2.4)

11.8.1. **Dark Mode (only available theme):**
- General background: `#0a0a0f` (very dark black with a bluish tint).
- Cards / panels: `#141420` to `#1e1e2e` (very dark gray with a violet/blue tint).
- Subtle borders: `#2a2a3a`.
- Main text: `#f0f0f0` (soft white).
- Secondary text: `#9ca3af`.
- Primary accents: **Dark violet** (`#6366f1`, `#4f46e5`) for interactive elements and selections.
- Secondary accents: **Fluorescent yellow** (`#fbbf24`) or **Fluorescent gold** (`#d97706`) for highlights and prominent elements.
- Error: `#ef4444` (red).
- Warning: `#ffcc00` (yellow).
- Info: `#3b82f6` (blue).

> **Note (V2.4):** The interface color palette has been designed to avoid visual conflicts with standard electrical wire colors (black, red, blue, green, yellow, white, brown, orange). Dark violet and fluorescent gold rarely appear in real wires, allowing clear distinction between UI elements and harness elements.

11.8.2. **Subtle Neumorphic effect:**
- Double shadows on raised elements: `box-shadow: 6px 6px 12px rgba(0,0,0,0.6), -6px -6px 12px rgba(40,50,80,0.15);`
- Sunken elements (inputs): `box-shadow: inset 4px 4px 8px rgba(0,0,0,0.5), inset -4px -4px 8px rgba(40,50,80,0.1);`
- Rounded borders: `border-radius: 1rem` to `1.5rem`.

11.8.3. **Transitions:** `transition: all 0.25s ease` on hover, selection, opening/closing panels.

11.8.4. **Visual feedback:** Hover over interactive elements (subtle brightness/shadow change). Active selection (fluorescent yellow border + glow). Element being dragged (reduced opacity + deeper shadow).

### 11.9 Persistence: JSON vs localStorage (V2.1)

11.9.1. **Strict separation of responsibilities:**

| Storage              | Content | Persistence |
|----------------------|---------|-------------|
| **`db.json`** (file) | Project data: `metadata` (projectInfo, schema, catalogs, savedBy, lastSave, version) and `data` (containers, connectors, wires, mates). | Saved on export. Versioned in Git. |
| **`localStorage`** (browser) | User preferences: current zoom, canvas position (pan), applied filters, last used view (table/visual), container expansion state. | Persists between sessions of the same browser. Not exported. |

11.9.2. **`metadata.uiSettings` does not exist** in the JSON. All UI preferences are local to the user and are saved in `localStorage`.

11.9.3. **Suggested localStorage keys:**
- `arnesviz.zoom`: number
- `arnesviz.pan`: `{ x, y }`
- `arnesviz.filters`: object with filter state
- `arnesviz.activeView`: `"table"` | `"visual"`
- `arnesviz.expandedContainers`: object with expanded container IDs
- `arnesviz.lastProject`: blob or reference to the last loaded JSON (to restore session)
- `arnesviz.autosave`: backup copy of the current state (see 11.10.5)

11.9.4. **localStorage error handling (V2.2):**
- All write operations to `localStorage` are wrapped in `try/catch`.
- If writing fails (e.g., storage quota exceeded), a **Warning** is displayed in the log panel: "Preferences could not be saved. Local storage may be full."
- The application continues to function normally; only preference persistence is lost.
- **Note for future versions:** For very large projects that exceed the `localStorage` quota (~5-10 MB), it is suggested to migrate to `IndexedDB` (limit ~50 MB+). See section 12.14.

### 11.10 Close and Save Management (V2.1)

11.10.1. **Deployment context (V2.2):** The **ArnesViz** application is hosted on **GitHub Pages** (or similar) and is a **static** application. It reads a **`db.json`** file from the root of the repository on startup.

> **⚠️ Important (V2.2):** GitHub Pages is a **static** hosting service. It does not allow direct writing to the repository from the browser. To save changes to the `db.json` file, the user must:
> 1. Press **Export JSON** in the app (the updated file is downloaded).
> 2. Manually upload the file to the GitHub repository (via commit + push, or the GitHub web interface).
>
> Autosave in `localStorage` is a **local** safety mechanism to recover work in case of unexpected browser closure. **It does not replace** the Git commit or the manual upload of the file.

11.10.2. **Startup flow:**
1. On page load, the app attempts to fetch `db.json` from the root of the repository.
2. If it exists and is valid, the project is loaded.
3. If it does not exist or is invalid, a welcome screen is displayed with buttons: **"Import JSON"** and **"Create new project"**.

11.10.3. **Close / navigate away flow:**
1. If there are **no unsaved changes** in the project data (`isDirty === false`), the app allows closing or navigating without asking.
2. If there are **unsaved changes** (`isDirty === true`), a modal is displayed: "There are unsaved changes. Do you want to export the current file?"
   - Options: **[Export]** | **[Discard and close]** | **[Cancel]**
3. If the user chooses "Export", the updated JSON is downloaded and then closing is allowed.
4. If the user chooses "Discard and close", it closes without saving.
5. If the user chooses "Cancel", it returns to the app.

11.10.4. **Change detection:** The app maintains an `isDirty` flag that is activated with any modification to the project data (containers, connectors, wires, mates, catalogs, metadata). It is deactivated on export or when importing a new file.

11.10.5. **Autosave in localStorage:** As a safety mechanism, the app periodically (every 30 seconds or on each significant change) saves a copy of the current state in `localStorage` under the key `arnesviz.autosave`. If the app closes unexpectedly (browser crash), on restart it offers to restore from the autosave.

### 11.11 Entity Creation and Deletion (V2.3)

11.11.1. **Create entity:**
- **"+ Add"** button in the table toolbar (visible in edit mode).
- Pressing it opens the `details` panel with an empty form of the selected entity type (T, C, W, M).
- The ID is automatically assigned: the next available number is searched for the corresponding prefix (e.g., if C001-C009 exist, the new one will be C010).
- Inferable fields are autocompleted according to the inference principle (see 1.5).
- On saving, the entity is added to `data` and the table and SVG are re-rendered.

11.11.2. **Delete entity:**
- **"Delete"** button in the `details` panel (only in edit mode).
- Opens a confirmation modal: "Delete [type] [ID]? This action cannot be undone."
- If the entity has references (e.g., a connector used in wires or mates), the modal shows: "Warning: [ID] is referenced by [list of IDs]. Deleting it will cause reference errors."
- Options: **[Delete anyway]** | **[Cancel]**.
- On deletion, it is removed from `data`, the table and SVG are re-rendered, and the project is revalidated.

11.11.3. **Duplicate entity:**
- **"Duplicate"** button in the `details` panel (only in edit mode).
- Creates an exact copy of the entity with a new auto-assigned ID.
- Internal references (e.g., `from.connector`, `to.connector`) are kept pointing to the same original entities.
- The user must manually adjust references if necessary.

### 11.12 Data Table (V2.3)

11.12.1. The table is implemented with **native HTML** (`<table>`, `<tr>`, `<td>`) and vanilla JavaScript. No external libraries are used.

11.12.2. **Structure:**
- Top toolbar: entity type selector (T / C / W / M), "+ Add" button, quick search field.
- Table body: one row per entity, with columns according to the visible column configuration.
- Default columns: `id`, `name`, `type`, `owner`.

11.12.3. **Inline editing:**
- In edit mode, double-clicking on a cell transforms it into an editable input.
- `Enter` saves the change. `Esc` cancels.
- Changes are synchronized with the SVG and the `details` panel.

11.12.4. **Error highlighting:**
- Rows of entities with validation errors are highlighted with a red border (`border: 2px solid #ef4444`).
- Rows of entities with warnings are highlighted with a yellow border (`border: 2px solid #ffcc00`).
- On hovering over the row, a tooltip with the error/warning message is displayed.

11.12.5. **Bidirectional synchronization:**
- Changes in the table update the SVG and the `details` panel.
- Changes in the SVG (drag, selection) update the table and the `details` panel.
- Changes in the `details` panel update the table and the SVG.

### 11.13 Responsive (V2.3)

11.13.1. The side panel is **overlay** (floats over the content, does not push it).
11.13.2. On screens < 768px, the side panel occupies **100% of the screen width**.
11.13.3. The header remains fixed. The work area occupies the rest of the height.
11.13.4. Filters collapse into a button (funnel icon) on small screens. Pressing it deploys a floating panel with the filter controls.

### 11.14 AI Team Workflow (V2.5)

Development is coordinated by four specialized AI agents:
- **Orchestrator** (Qwen Max): Coordinates tasks, makes architectural decisions, tracks M1-M4 progress.
- **Dev** (Qwen Coder): Generates and modifies code in specific JS modules.
- **QA** (Qwen Max): Reviews code for bugs, edge cases, V2.5 compliance.
- **Integrator** (Qwen Coder): Applies precise diffs/patches to existing code without breaking functionality.

All agents operate in **thinking mode** (not fast mode) to ensure deep reasoning and adherence to spec.

---

## 12. NOTES FOR FUTURE VERSIONS

12.1. Boot mode configuration (read/edit) in preferences.
12.2. Automatic wire routing with obstacle avoidance.
12.3. Visual connector size adaptable to pins or designator.
12.4. Support twisted pairs (`pair` field in nets in the catalog).
12.5. Chassis points as implicit ground nodes.
12.6. Integration with Kanban systems for task tracking based on notes.
12.7. If required, reintroduction of states in M (planned, disconnected, obsolete) with a more robust model.
12.8. **Configurable misalignment threshold:** the 2-pixel threshold for offset alignment validation (10.10) can be adjusted in the configuration panel to adapt to different canvas resolutions or zoom levels.
12.9. Undo/Redo system with `Ctrl+Z` / `Ctrl+Y`.
12.10. Export to additional formats (PDF, SVG image, BOM).
12.11. Integration with CAD tools (KiCad, Altium) for netlist import.
12.12. Real-time multi-user synchronization (collaboration).
12.13. **Full catalog editing from UI (V2.5):** In V2.5, catalogs remain read-only in the interface. Future versions will implement full CRUD UI for catalog entries with reference protection.
12.14. **Migration to IndexedDB (V2.2):** Autosave and preferences are saved in `localStorage`, which has a limit of ~5-10 MB. For very large projects, it is suggested to migrate to `IndexedDB` (limit ~50 MB+) in a future version.
12.15. **File migration tool (V2.2):** Currently, files from previous versions (V1.9.1, V2.0, V2.1, V2.2, V2.3) are rejected upon import. In a future version, an automatic migration tool could be implemented to convert old files to the current format.
12.16. **Multi-file optimization:** Current module split (app/data/render/interaction) may be refined based on bundle size analysis and loading performance metrics.

> **Note:** The old point 12.9 of V1.9.1 (Event/logging system) was promoted to active implementation in section 10.13 of V2.0.

---

## 13. VERSION HISTORY

**Version 1.0** – Initial documentation with ID system, container hierarchy, connectors with expectedPair, dynamic lockedWith calculation, nets as labels.

**Version 1.1** – Structural improvements: single entity table, "Containers" section.

**Version 1.2** – Edit mode with toggle (Ctrl+Shift+E), removal of explicit connector states, addition of `notes` and `hidden` fields.

**Version 1.3** – Major refactoring: replacement of `expectedPair` with `matedId`, historical note structure, configuration panel, duplicate numbering fix.

**Version 1.4** – Addition of subsection 1.3 "Usage Philosophy and Scope".

**Version 1.5** – Coherence error corrections and addition of `pinMapping` in M.

**Version 1.6** – Model simplification: removal of `hidden` field in connectors and states in M.

**Version 1.7** – Positioning system refactoring: relative coordinates for all components except those with `parent_id: null`, removal of free connector (`edgeSide` mandatory), replacement of `x`/`y` with `offset` in connectors, new validations and filter persistence.

**Version 1.8** – Context improvements and data refinement: separation of `position` and `size`, new pin facing validation in M and visual rigid block rule, new subsection 1.4 "Technician Workflow", property panel, quick filters.

**Version 1.8.1** – Minor corrections: removed reference to automatic migration, unified term "size", fixed connector example to comply with facing rule, removed obsolete point about unanchored components.

**Version 1.8.2** – Terminology cleanup: removed all references to "anchored connectors", removed "Free" definition, simplified hierarchical diagram.

**Version 1.9** – Kinematic conflict resolution and resizing simplification: introduction of `mountType` (`"fixed"` / `"flying"`), flying connectors without own `offset`, resizing limited to bottom-right corner, correction of `matedId` validation (bidirectional) and offset alignment.

**Version 1.9.1** – Consistency and clarity fixes: restored rule 10.6, improved wording of 3.3.1.3 and 4.5.3.1, added clarification on declarative `parent_id` in flying connectors, added note 12.10 on configurable threshold.

**Version 2.0** – Formal JSON file structure (`metadata` + `data`), reusable catalogs, dynamic validation schema, user and responsible person management, optional additive fields, minimalist interface with header and single reusable sidebar, active log panel with 3 levels, import and delete flows with confirmation, keyboard shortcuts, dark mode neumorphic aesthetics.

**Version 2.1** – Nets moved to catalogs, gauge simplified, flying connectors without a partner (rule 10.14), native SVG specified, `metadata.uiSettings` removed, JSON vs localStorage separation, intelligent close flow, `sectionRef` inheritance, pin 1 alignment.

**Version 2.2** – Edit mode toggle in header, sidebar with state memory, model/instance consistency rule removed, `sectionRef` removed from connectors, `metadata.revision` renamed to `version`, gauge as recommended field, fixed connectors without a partner (Info in logs), wires with invalid ends omitted in SVG, unified validation, intermediate filters, catalog protection, incompatible file detection, GitHub Pages documentation, localStorage error handling.

**Version 2.3** – Inference principle from catalogs (automatic autocomplete, section 1.5), `colorPalette` catalog for SVG rendering, entity-specific schema (required/recommended per type), simplification of `people` catalog (only name), removal of `owners` in `sections`, removal of `shield` from `signalType`, optional `gaugeUnit` with catalog inference (default `"mm2"`), `pinMapping` autocompleted as `"direct"` in UI (only for new mates), M limitation documented (two connectors per coupling), `offset` ignored in flying (no warning), hierarchical depth limit of 4 levels with cycle detection (rule 10.16), 100% custom validation (without AJV or external libraries), single-file architecture (`index.html` + `db.json`), middle mouse button pan mechanic, Bézier curve formula documented, pin distribution documented, render order (Z-index) documented, entity creation/deletion/duplication documented, native HTML table documented, responsive with overlay panel, base example restored (all M/W share net `GND`), cross-reference corrections (3.3.1.3, 4.1.1.b, 4.4.2, 10.13.3), historical note on revision→version rename, default color changed to `"black"`.

**Version 2.4** – Complete removal of light mode, keeping only dark mode. New color palette optimized to avoid visual conflicts with standard electrical wire colors: background `#0a0a0f`, panels `#141420`-`#1e1e2e`, dark violet accents (`#6366f1`, `#4f46e5`) and fluorescent yellow/gold (`#fbbf24`, `#d97706`). Theme selector removed from configuration panel (section 11.5.2). `arnesviz.theme` removed from localStorage keys (section 11.9.3). Documentation updated in section 11.8 with explanatory note on color choice.

**Version 2.5** – Transition to multi-file architecture (`index.html` + `styles.css` + 4 JS modules). Removal of model-instance consistency validation. Simplified gaugeUnit handling with mm2 default. AI team workflow formalized (Orchestrator/Dev/QA/Integrator in thinking mode). Header simplified to 4 zones. Catalog management remains read-only. Offset tolerance clarified for flying connectors. Section inheritance depth limit enforced. Documentation updated for modular structure.

---

## APPENDIX A – COMPLETE EXAMPLE JSON (V2.4)

**Note:** A complete example JSON file (`db-example-v2.4.json`) is provided as a separate file. It contains 5 containers, 9 connectors, 3 wires, and 4 mates with all catalogs and schema definitions.
---

## APPENDIX B – SUMMARY OF CHANGES V2.4 → V2.5

| Change | V2.4 | V2.5 |
| :--- | :--- | :--- |
| **Architecture** | Single-file (`index.html`) | Multi-file (HTML + CSS + 4 JS modules) |
| **Validation** | Model-instance consistency checked | Consistency check removed; instance is truth |
| **gaugeUnit** | Optional, inferred from catalog | Optional, defaults to "mm2", inferred if available |
| **Header** | 5 zones | 4 zones (Edit toggle always visible) |
| **Catalog UI** | Read-only | Read-only (postponed to future) |
| **Dev Workflow** | Single AI | 4-agent team (Orchestrator/Dev/QA/Integrator) |
| **AI Mode** | Not specified | Thinking mode for all agents |
| **Flying offset** | Ignored, no warning | Explicitly documented as tolerated |

---

**END OF DOCUMENTATION – VERSION 2.5**