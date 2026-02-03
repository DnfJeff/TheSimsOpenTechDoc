# Part III: Object System

> **How objects are defined, behave, and interact in The Sims**

This part covers everything that makes objects work: definitions, behaviors, interactions, routing, and the virtual machine that executes it all.

---

## Document Navigation

| Part                                            | Focus                        | Status          |
| ----------------------------------------------- | ---------------------------- | --------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture & Concepts      |                 |
| [II. Containers](Part_II_Containers.md)         | FAR & IFF Formats            |                 |
| **III. Object System**                          | OBJD, BHAV, VM               | 📍 You are here |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, Meshes, Animation   |                 |
| [V. Game Content](Part_V_GameContent.md)        | Characters, Careers, Strings |                 |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | IFFSnooper Guides            |                 |

---

## Key Terms

| Term          | Definition                                                     |
| ------------- | -------------------------------------------------------------- |
| **OBJD**      | Object Definition — master resource defining object properties |
| **OBJf**      | Object Functions — entry point table for behaviors             |
| **BHAV**      | Behavior — VM subroutine containing executable code            |
| **BCON**      | Behavioral Constants — tunable integer values                  |
| **TTAB**      | Tree Table — interaction definitions (pie menu)                |
| **GLOB**      | Global — reference to shared semi-global file                  |
| **SLOT**      | Slot — character routing and approach positions                |
| **Opcode**    | Instruction code specifying VM operation                       |
| **Primitive** | Built-in VM operation (opcodes 0-63)                           |

---

## Chapter 1: OBJD Object Definition

### Overview

OBJD is the master resource defining an object — nearly 100 fields control identity, appearance, behavior, pricing, and categorization.

| Property   | Value                           |
| ---------- | ------------------------------- |
| Type code  | `OBJD`                          |
| Version    | 138 (documented)                |
| Size       | 216 bytes (108 × 2-byte fields) |
| Byte order | Little-endian                   |

### Object Roles

| Configuration       | master_id | sub_index     | Usage                |
| ------------------- | --------- | ------------- | -------------------- |
| Single-tile         | `0`       | ignored       | Most objects         |
| Master (multi-tile) | non-zero  | `0xFFFF` (-1) | Object-wide settings |
| Slave (multi-tile)  | non-zero  | `(y<<8)\|x`   | Per-tile settings    |

⚠️ **Multi-tile objects:** Each tile needs its own OBJD. The master holds shared settings (price, categories); slaves hold tile-specific data (sprites, positions).

### Key Field Groups

#### Identity (Offsets 0-4)

| Offset | Field              | Description                  |
| ------ | ------------------ | ---------------------------- |
| 0-1    | version            | Format version (138)         |
| 2      | initial_stack_size | Stack allocation (4 typical) |
| 3      | base_graphic       | First DGRP resource ID       |
| 4      | num_graphics       | DGRP count for object states |

#### GUID (Offsets 14-15)

| Offset | Field | Description                       |
| ------ | ----- | --------------------------------- |
| 14-15  | guid  | 32-bit globally unique identifier |

⚠️ **GUID Uniqueness:** Every object across all installed content must have a unique GUID. Conflicts cause objects to disappear from the catalog.

#### Object Type (Offset 9)

| Code | Type     | Description            |
| ---- | -------- | ---------------------- |
| 0    | unknown  | Undetermined           |
| 2    | person   | Character/NPC          |
| 4    | normal   | Buyable object         |
| 7    | sim_type | System objects         |
| 8    | portal   | Stairs, doors, windows |
| 34   | food     | Food items             |

#### Pricing (Offsets 18, 34-38)

| Offset | Field                | Description                |
| ------ | -------------------- | -------------------------- |
| 18     | price                | Purchase price (Simoleans) |
| 34     | sale_price           | Current depreciated value  |
| 35     | initial_depreciation | First-day loss             |
| 36     | daily_depreciation   | Per-day loss               |
| 38     | depreciation_limit   | Minimum resale floor       |

#### Catalog Categories (Offsets 39-40)

**Room Sort Flags (Offset 39):**

| Mask | Category      |
| ---- | ------------- |
| 0x01 | Kitchen       |
| 0x02 | Bedroom       |
| 0x04 | Bathroom      |
| 0x08 | Family room   |
| 0x10 | Outside       |
| 0x20 | Dining room   |
| 0x40 | Miscellaneous |
| 0x80 | Study         |

**Function Sort Flags (Offset 40):**

| Mask | Category    |
| ---- | ----------- |
| 0x01 | Seating     |
| 0x02 | Surfaces    |
| 0x04 | Appliances  |
| 0x08 | Electronics |
| 0x10 | Plumbing    |
| 0x20 | Decorative  |
| 0x40 | General     |
| 0x80 | Lighting    |

#### Motive Ratings (Offsets 82-88)

These values appear in the object's catalog description:

| Offset | Rating  |
| ------ | ------- |
| 82     | Hunger  |
| 83     | Comfort |
| 84     | Hygiene |
| 85     | Bladder |
| 86     | Energy  |
| 87     | Fun     |
| 88     | Room    |

#### Skill Flags (Offset 89)

| Mask | Skill      |
| ---- | ---------- |
| 0x01 | Cooking    |
| 0x02 | Mechanical |
| 0x04 | Logic      |
| 0x08 | Body       |
| 0x10 | Creativity |
| 0x20 | Charisma   |
| 0x40 | School     |

#### Build Mode Types (Offset 69)

| Code | Tool       |
| ---- | ---------- |
| 1    | Door       |
| 2    | Window     |
| 3    | Stair      |
| 4    | Plant      |
| 5    | Fireplace  |
| 6    | Wall/Fence |
| 7    | Pool       |

#### Expansion Sort Fields

| Offset | Field            | Expansion    |
| ------ | ---------------- | ------------ |
| 94     | function_subsort | Hot Date+    |
| 95     | downtown_sort    | Hot Date     |
| 97     | vacation_sort    | Vacation     |
| 99     | community_sort   | Unleashed    |
| 103    | studiotown_sort  | Superstar    |
| 104    | magictown_sort   | Makin' Magic |

### Complete Field Index

From Behavior.iff STR# 204:

| Index | Name                | Index | Name                 |
| ----- | ------------------- | ----- | -------------------- |
| 0-1   | version_1/2         | 39-40 | room/function_sort   |
| 2     | initial_stack_size  | 41    | catalog_strings_id   |
| 3-4   | base_graphic/num    | 42    | is_global_sim_object |
| 7     | tree_table_id       | 46    | wall_style           |
| 8     | interaction_group   | 49    | object_version       |
| 9     | type                | 61    | front_direction      |
| 10-11 | master_id/sub_index | 63    | mt_lead_object       |
| 14-15 | guid_1/2            | 64-65 | dynamic_sprites      |
| 16    | disabled            | 69    | build_mode_type      |
| 18    | price               | 70-73 | original/suit_guid   |
| 19    | body_strings_id     | 75    | thumbnail_graphic    |
| 20    | slots_id            | 81    | wall_style_sprite_id |
| 34-38 | depreciation fields | 82-88 | motive ratings       |

---

## Chapter 2: OBJf Object Functions

### OBJf Overview

OBJf provides a vtable-like entry point table for object behaviors, replacing deprecated OBJD "old tree ID" fields.

| Property      | Value          |
| ------------- | -------------- |
| Type code     | `OBJf`         |
| Typical count | 31 entry pairs |

### Structure

**Header (16 bytes):**

| Offset | Size | Field       | Value                  |
| ------ | ---- | ----------- | ---------------------- |
| 0      | 4    | Reserved    | 0                      |
| 4      | 4    | Version     | 0                      |
| 8      | 4    | Identifier  | `fJBO` (little-endian) |
| 12     | 4    | Entry count | Number of pairs        |

**Entry Pairs (4 bytes each):**

| Offset | Size | Field          |
| ------ | ---- | -------------- |
| 0      | 2    | Guard BHAV ID  |
| 2      | 2    | Action BHAV ID |

### Guard vs Action Functions

| Function   | Purpose                   | Returns    |
| ---------- | ------------------------- | ---------- |
| **Guard**  | Check if action available | True/False |
| **Action** | Execute the interaction   | (executes) |

### Entry Point Index

| Index | Entry                    | Index | Entry            |
| ----- | ------------------------ | ----- | ---------------- |
| 0     | init                     | 16    | cook             |
| 1     | main                     | 17    | eat_food         |
| 2     | load                     | 18    | dispose          |
| 3     | cleanup                  | 19    | fire             |
| 4     | queue_skipped            | 20    | burn_down        |
| 5     | allow_intersection       | 21    | wash_hands       |
| 6     | wall_adjacency_changed   | 22    | wash_dishes      |
| 7     | room_changed             | 23    | use_toilet       |
| 8     | dynamic_multitile_update | 24    | use_tub          |
| 9     | placement                | 25    | sit              |
| 10    | pick_up                  | 26    | stand            |
| 11    | user_placement           | 27    | lie_in           |
| 12    | user_pickup              | 28    | place_on_surface |
| 13    | level_info_request       | 29    | pickup_from_slot |
| 14    | move_in                  | 30    | garden           |
| 15    | move_out                 |       |                  |

💡 **Precedence:** When OBJf is present, OBJD "old tree ID" fields are ignored.

---

## Chapter 3: BHAV Behaviors

### BHAV Overview

BHAV resources are the executable code of The Sims — VM subroutines that drive all gameplay operations.

| Property         | Value                 |
| ---------------- | --------------------- |
| Type code        | `BHAV`                |
| Header size      | 12 bytes (13 for TSO) |
| Instruction size | 12 bytes (fixed)      |
| Max instructions | 253                   |
| Byte order       | Little-endian         |

### Physical Structure

```text
BHAV Layout
┌─────────────────────────┐
│ Header (12 bytes)       │
├─────────────────────────┤
│ Instruction 0 (12B)     │
│ Instruction 1 (12B)     │
│ ...                     │
└─────────────────────────┘
```

### Header (12 bytes)

| Offset | Size | Field     | Description                    |
| ------ | ---- | --------- | ------------------------------ |
| 0-1    | 2    | Signature | Version marker (0x8000–0x8003) |
| 2-3    | 2    | Count     | Instruction count (1–253)      |
| 4-5    | 2    | Type      | Usually 0                      |
| 6-7    | 2    | Args      | Parameter count (0–4)          |
| 8      | 1    | Locals    | Local variable count (0–12)    |
| 9      | 1    | Flags     | Behavior flags                 |
| 10-11  | 2    | Reserved  | Always zero                    |

**Signature Versions:**

| Signature | Count | Notes        |
| --------- | ----- | ------------ |
| 0x8000    | 90    | Deprecated   |
| 0x8001    | 26    | Deprecated   |
| 0x8002    | 8,896 | **Standard** |
| 0x8003    | —     | TSO only     |

### Instruction Format (12 bytes)

| Offset | Size | Field    | Description                  |
| ------ | ---- | -------- | ---------------------------- |
| 0-1    | 2    | Opcode   | Primitive or subroutine call |
| 2      | 1    | Addr1    | Next instruction on success  |
| 3      | 1    | Addr2    | Next instruction on failure  |
| 4-11   | 8    | Operands | Instruction-specific data    |

### Flow Control Addresses

| Address | Name  | Effect                 |
| ------- | ----- | ---------------------- |
| 254     | TRUE  | Return true to caller  |
| 255     | FALSE | Return false to caller |
| 253     | ERROR | Terminate with error   |
| 0–252   | —     | Next instruction index |

💡 **Two-address flow:** Instructions specify both success and failure paths. Code doesn't need to execute sequentially.

### Operand Interpretation

The 8-byte operand field has flexible interpretation depending on opcode:

- Individual bytes as single-byte operands
- Aligned pairs as 2-byte little-endian integers
- Complex references to external data containers

---

## Chapter 4: BCON & TRCN Constants

### BCON: Behavioral Constants

BCON stores tunable integer values that behavior code references instead of hard-coded literals.

| Property    | Value            |
| ----------- | ---------------- |
| Type code   | `BCON`           |
| Max entries | 65               |
| Entry size  | 2 bytes (signed) |

**Structure:**

| Offset | Size | Field                            |
| ------ | ---- | -------------------------------- |
| 0      | 1    | Count (N)                        |
| 1      | 1    | Flags (0x00 or 0x80)             |
| 2      | 2×N  | Constants (little-endian signed) |

### TRCN: Trait Constants (Labels)

TRCN provides human-readable labels and valid ranges for BCON constants.

| Property     | Value                           |
| ------------ | ------------------------------- |
| Type code    | `TRCN`                          |
| Relationship | Same resource ID as paired BCON |
| Required     | No (optional for tools)         |

**Header (16 bytes):**

| Offset | Size | Field                |
| ------ | ---- | -------------------- |
| 0      | 4    | Reserved (0)         |
| 4      | 4    | Version (0, 1, or 2) |
| 8      | 4    | Identifier (`TRCN`)  |
| 12     | 4    | Entry count          |

**Entry:**

| Field        | Size | Description                    |
| ------------ | ---- | ------------------------------ |
| Disable flag | 1    | 0 = enabled, 1 = disabled      |
| Unknown      | 7    | Purpose unclear                |
| Label        | var  | Null-terminated + 0xA3 padding |
| Comment      | var  | Null-terminated + 0xA3 padding |

**Version 1+ adds range validation:**

| Field         | Size | Description         |
| ------------- | ---- | ------------------- |
| Range enabled | 1    | 0 = no, 1 = yes     |
| Low range     | 2    | Minimum legal value |
| High range    | 2    | Maximum legal value |

---

## Chapter 5: TPRP Trait Properties

TPRP provides human-readable labels for BHAV parameters and local variables.

| Property     | Value                   |
| ------------ | ----------------------- |
| Type code    | `TPRP`                  |
| Relationship | Same ID as paired BHAV  |
| Required     | No (optional for tools) |

**Header (20 bytes):**

| Offset | Size | Field                |
| ------ | ---- | -------------------- |
| 0      | 4    | Reserved (0)         |
| 4      | 4    | Version (2, 3, or 4) |
| 8      | 4    | Identifier (`TPRP`)  |
| 12     | 4    | Param count (N1)     |
| 16     | 4    | Local count (N2)     |

**Label Data:**

- Parameter labels (N1): Null-terminated, 0xA3-padded
- Local labels (N2): Same format
- Parameter flags (N1 bytes): 0 = enabled, 1 = disabled

---

## Chapter 6: TTAB Interactions

### TTAB Overview

TTAB (Tree Table) defines object interactions — generating pie menus and controlling autonomy.

| Property  | Value                    |
| --------- | ------------------------ |
| Type code | `TTAB`                   |
| Versions  | 2–9 (V9 most common)     |
| Related   | TTAs (menu text strings) |

### Header

| Offset | Size | Field                    |
| ------ | ---- | ------------------------ |
| 0-1    | 2    | Interaction count        |
| 2-3    | 2    | Version (2–9)            |
| 4      | 1    | Compression code (V9: 1) |

### Core Fields Per Interaction

| Field              | Size | Description                |
| ------------------ | ---- | -------------------------- |
| Action function    | 2    | BHAV ID to execute         |
| Guard function     | 2    | Condition check BHAV ID    |
| Motive count       | 2-4  | Always 16                  |
| Flags              | 2-4  | Behavior control bits      |
| String table index | 4    | TTAs menu text (V5+)       |
| Attenuation code   | 4    | Distance fade (V7+)        |
| Autonomy           | 4    | Autonomy eligibility (V5+) |
| Join index         | 4    | Group interaction ID (V5+) |

### Flags Bitfield

| Bit | Meaning                      |
| --- | ---------------------------- |
| 0   | Accessible to visitors       |
| 1   | Joinable (group interaction) |
| 3   | Allow consecutive repeats    |
| 4   | NOT accessible to child      |
| 6   | NOT accessible to adult      |
| 9   | Enable for cats (Unleashed)  |
| 10  | Enable for dogs (Unleashed)  |

### Motive Entries (16 per interaction)

| Index | Motive  | Index | Motive  |
| ----- | ------- | ----- | ------- |
| 3     | Mood    | 9     | Bladder |
| 5     | Energy  | 13    | Room    |
| 6     | Comfort | 14    | Social  |
| 7     | Hunger  | 15    | Fun     |
| 8     | Hygiene |       |         |

**Motive Entry Fields (V7+):**

| Field          | Size | Description           |
| -------------- | ---- | --------------------- |
| Minimum        | 2    | Threshold value       |
| Delta          | 2    | Change amount claimed |
| Character type | 2    | Personality modifier  |

💡 **Objects lie:** Objects often claim motive benefits they don't actually deliver.

### Attenuation Settings

| Setting | Effect                 |
| ------- | ---------------------- |
| None    | Always visible in menu |
| Low     | Moderate distance fade |
| Medium  | Medium distance fade   |
| High    | Quick distance fade    |

---

## Chapter 7: GLOB Semi-Globals

### GLOB Overview

GLOB references a shared semi-global IFF file containing common resources.

| Property  | Value            |
| --------- | ---------------- |
| Type code | `GLOB`           |
| Limit     | One per IFF file |
| Common ID | 128              |

### GLOB Structure

| Offset | Size | Description                            |
| ------ | ---- | -------------------------------------- |
| 0      | var  | Semi-global filename (various formats) |
| var    | opt  | Filler (0xA3 padding)                  |

### String Encoding Formats

| Format | Detection              | Structure                     |
| ------ | ---------------------- | ----------------------------- |
| 1      | Resource size = string | Raw string, no terminator     |
| 2      | First byte ≥ 48        | Null-terminated + padding     |
| 3      | First byte < 48        | Count byte + string + padding |

**Examples:**

- "CarGlobals" → carglobals.iff
- "CybernauticsGlobals" → cybernauticsglobals.iff

The reference is case-insensitive.

---

## Chapter 8: SLOT Routing Positions

### SLOT Overview

SLOT defines character routing and approach positions for objects — where Sims stand, sit, or navigate to when using an object.

| Property  | Value             |
| --------- | ----------------- |
| Type code | `SLOT`            |
| Versions  | 4, 6, 7, 8, 9, 10 |
| Unit      | 1/16 of a tile    |

### Header (16 bytes)

| Offset | Size | Field                             |
| ------ | ---- | --------------------------------- |
| 0      | 4    | Reserved (0)                      |
| 4      | 4    | Version                           |
| 8      | 4    | Identifier (`TOLS` little-endian) |
| 12     | 4    | Slot count                        |

### Slot Types

| Type | Frequency | Purpose                        |
| ---- | --------- | ------------------------------ |
| 0    | 24.7%     | Support slots (non-routing)    |
| 1    | 1.6%      | Specialized (rare, elevated)   |
| 3    | 73.5%     | Routing (character navigation) |

### Core Slot Descriptor

| Offset | Size | Field          | Description             |
| ------ | ---- | -------------- | ----------------------- |
| 0      | 2    | type           | 0, 1, or 3              |
| 2      | 4    | xoffset        | X position (float)      |
| 6      | 4    | yoffset        | Y position (float)      |
| 10     | 4    | altoffset      | Z/height (float)        |
| 14     | 4    | standing       | Standing preference     |
| 18     | 4    | sitting        | Sitting preference      |
| 22     | 4    | ground         | Lying preference        |
| 26     | 4    | rsflags        | Routing/facing flags    |
| 30     | 4    | snaptargetslot | Target slot (-1 = none) |

**Additional fields by version:**

| Version | Bytes | Added Field    |
| ------- | ----- | -------------- |
| 4       | 34    | snaptargetslot |
| 6       | 54    | flags          |
| 7       | 58    | gradient       |
| 8       | 62    | height         |
| 9       | 66    | facing         |
| 10      | 70    | resolution     |

### rsflags Bit Definitions

| Bit | Value | Meaning              |
| --- | ----- | -------------------- |
| 0   | 1     | N (0°)               |
| 1   | 2     | NE (45°)             |
| 2   | 4     | E (90°)              |
| 3   | 8     | SE (135°)            |
| 4   | 16    | S (180°)             |
| 5   | 32    | SW (225°)            |
| 6   | 64    | W (270°)             |
| 7   | 128   | NW (315°)            |
| 8   | 256   | Allow any rotation   |
| 9   | 512   | Absolute coordinates |
| 11  | 2048  | Ignore rooms         |
| 12  | 4096  | Snap to direction    |
| 13  | 8192  | Random scoring       |

### Preference Values

| Value     | Meaning           |
| --------- | ----------------- |
| 0         | Disabled          |
| 1         | Weak preference   |
| 10        | Moderate          |
| 100, 1000 | Strong preference |

**Example:**

```text
standing=1, sitting=1000, ground=0
→ Character strongly prefers sitting
→ Will not lie down
→ Standing is low priority
```

### Coordinate System

| Axis | Direction       | Unit      |
| ---- | --------------- | --------- |
| X    | Positive = left | 1/16 tile |
| Y    | Positive = back | 1/16 tile |
| Z    | Height          | 1/16 tile |

Reference point is object center (0, 0, 0).

---

## Chapter 9: Virtual Machine

### VM Overview

The SimAntics Virtual Machine executes BHAV subroutines to drive all gameplay — a multi-threaded engine managing every object's behavior.

| Property      | Value                           |
| ------------- | ------------------------------- |
| Architecture  | Multi-threaded, cooperative     |
| Tick rate     | 15 ticks = 1 simulated minute   |
| Scheduling    | Per-object threads, yield-based |
| Documentation | ~10% of full system             |

### Threading Model

| Property     | Description                        |
| ------------ | ---------------------------------- |
| Threading    | Each object has its own thread     |
| Cooperation  | Threads yield voluntarily per tick |
| Display rate | 15 Hz update                       |

### Subroutine Address Ranges

| Range     | Hex         | Type                    |
| --------- | ----------- | ----------------------- |
| 0–63      | 0x0000–003F | Primitives (built-in)   |
| 256–4095  | 0x0100–0FFF | Global (Global.iff)     |
| 4096–8191 | 0x1000–1FFF | Local (object file)     |
| 8192+     | 0x2000+     | Semi-global (GLOB file) |

### Data Scopes

**Per-Invocation (Call Stack):**

- Literals, parameters, locals, array index

**Per-Thread (Object Context):**

- My object ID, stack object, neighbor ID

**Per-Object (Instance):**

- Temps, definition, attributes, motives, slots

**Per-File (IFF Scope):**

- Constants, subroutines, strings, semi-global ref

**Global (Shared):**

- Global.iff constants and subroutines

### Key Primitives (Opcodes 0-63)

**Core:**

| Op  | Hex  | Name             | Purpose                         |
| --- | ---- | ---------------- | ------------------------------- |
| 0   | 0000 | Yield            | Sleep/yield thread              |
| 1   | 0001 | Generic Sim Call | VM sub-operation                |
| 2   | 0002 | Expression       | Arithmetic/comparison (**55%**) |
| 3   | 0003 | Find Best Action | Select optimal action           |
| 4   | 0004 | Grab             | Acquire object                  |
| 5   | 0005 | Drop             | Release object                  |
| 8   | 0008 | Random           | Random number                   |

**Animation & Display:**

| Op  | Hex  | Name           | Purpose               |
| --- | ---- | -------------- | --------------------- |
| 6   | 0006 | Change Article | Change suit/accessory |
| 7   | 0007 | Update         | Refresh display       |
| 22  | 0016 | Look Towards   | Change facing         |
| 41  | 0029 | Set Balloon    | Thought/speech bubble |
| 44  | 002C | Animate        | Play animation        |

**Navigation:**

| Op  | Hex  | Name              | Purpose            |
| --- | ---- | ----------------- | ------------------ |
| 11  | 000B | Distance To       | Calculate distance |
| 12  | 000C | Direction To      | Calculate heading  |
| 45  | 002D | Goto Routing Slot | Move to position   |
| 46  | 002E | Snap              | Align to position  |

**Object Management:**

| Op  | Hex  | Name            | Purpose            |
| --- | ---- | --------------- | ------------------ |
| 18  | 0012 | Remove Instance | Delete object      |
| 42  | 002A | Create Object   | Instantiate object |
| 43  | 002B | Drop Onto       | Place on surface   |

### Instruction Statistics

| Category   | Count  | Percentage |
| ---------- | ------ | ---------- |
| Expression | 55,088 | **57%**    |
| Animate    | 4,694  | 5%         |
| Play Sound | 4,050  | 4%         |
| Routing    | 749    | 1%         |
| Other      | 31,006 | 33%        |
| **Total**  | 95,487 | 100%       |

Expression (arithmetic/comparison) dominates — over half of all instructions.

### Global Subroutines (Samples)

| ID  | Name              | Purpose             |
| --- | ----------------- | ------------------- |
| 256 | Set graphic       | Display change      |
| 261 | SetComfort        | Increase comfort    |
| 266 | SetEnergy         | Increase energy     |
| 280 | Idle              | Standard waiting    |
| 370 | Animated idle for | Timed pause         |
| 419 | Wander            | Autonomous movement |

---

## Cross-Reference Diagram

```text
OBJECT SYSTEM ARCHITECTURE

┌─────────────────────────────────────────────────────────────┐
│                         OBJD                                 │
│              (Object Definition - Master)                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ GUID │ Price │ Categories │ Graphics │ Type        │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          │                                   │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │   OBJf   │    │   TTAB   │    │   SLOT   │            │
│    │  Entry   │    │ Interact │    │ Routing  │            │
│    │  Points  │    │  Menus   │    │ Positions│            │
│    └────┬─────┘    └────┬─────┘    └──────────┘            │
│         │               │                                   │
│         ▼               ▼                                   │
│    ┌──────────┐    ┌──────────┐                            │
│    │   BHAV   │    │   TTAs   │                            │
│    │ Behavior │    │Menu Text │                            │
│    │   Code   │    │          │                            │
│    └────┬─────┘    └──────────┘                            │
│         │                                                   │
│         ▼                                                   │
│    ┌──────────┐    ┌──────────┐                            │
│    │   BCON   │◀───│   TRCN   │                            │
│    │Constants │    │  Labels  │                            │
│    └──────────┘    └──────────┘                            │
│                                                             │
│    ┌──────────┐                                            │
│    │   GLOB   │───▶ Semi-global IFF (shared resources)     │
│    └──────────┘                                            │
└─────────────────────────────────────────────────────────────┘

VIRTUAL MACHINE EXECUTION

┌─────────────────────────────────────────────────────────────┐
│                    VM SCHEDULER (15 Hz)                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Thread  │  │ Thread  │  │ Thread  │  │ Thread  │  ...   │
│  │ (Obj 1) │  │ (Obj 2) │  │ (Obj 3) │  │ (Obj N) │        │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │            │            │            │              │
│       └────────────┴────────────┴────────────┘              │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              SUBROUTINE ADDRESS SPACE                │    │
│  │  0x0000-003F: Primitives (64 built-in opcodes)      │    │
│  │  0x0100-0FFF: Global subroutines (Global.iff)       │    │
│  │  0x1000-1FFF: Local subroutines (object file)       │    │
│  │  0x2000+    : Semi-global subroutines (GLOB ref)    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

_Part III Complete — Continue to [Part IV: Graphics & Animation](Part_IV_Graphics.md)_
