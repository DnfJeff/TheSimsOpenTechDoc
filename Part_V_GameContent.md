# Part V: Game Content

> **Characters, careers, strings, and localization data**

This part covers the high-level game content: character bundles, career definitions, string tables, and the language system.

---

## Document Navigation

| Part                                            | Focus                        | Status          |
| ----------------------------------------------- | ---------------------------- | --------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture & Concepts      |                 |
| [II. Containers](Part_II_Containers.md)         | FAR & IFF Formats            |                 |
| [III. Object System](Part_III_ObjectSystem.md)  | OBJD, BHAV, VM               |                 |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, Meshes, Animation   |                 |
| **V. Game Content**                             | Characters, Careers, Strings | 📍 You are here |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | IFFSnooper Guides            |                 |

---

## Key Terms

| Term                 | Definition                                         |
| -------------------- | -------------------------------------------------- |
| **Character Bundle** | IFF file containing all resources for a Sim or NPC |
| **CARR**             | Career resource — job track definitions            |
| **STR#**             | String Table — multi-language text storage         |
| **CTSS**             | Catalog Strings — object/character descriptions    |
| **FCNS**             | Function Constants — game-wide tuning values       |
| **Body Strings**     | STR# 200 — character appearance configuration      |
| **Language Code**    | Numeric identifier for localized text (0–20)       |

---

## Chapter 1: Character Bundles

### Overview

Character files are IFF bundles containing everything needed to define a Sim or NPC: identity, appearance, behaviors, and portraits.

### Resource Summary

| Type  | ID        | Purpose                         |
| ----- | --------- | ------------------------------- |
| OBJD  | 128       | Character definition (type = 2) |
| CTSS  | 2000      | Character surname               |
| STR#  | 200       | Body strings (appearance)       |
| STR#  | 256       | Person attributes               |
| BMP\_ | 2002–2007 | Character portraits             |
| SLOT  | 128       | Routing/approach positions      |
| GLOB  | 128       | Semi-global reference           |
| BHAV  | 4096+     | Character behaviors             |

### OBJD for Characters

Character OBJD resources use:

- **ID:** Always 128
- **Type:** 2 (person)
- **Name format:** `{filename}-{surname}` (e.g., "User00029-Bob")

### CTSS Surname Resource

| Entry | Content                       |
| ----- | ----------------------------- |
| 1     | Surname in specified language |
| 2     | Character description         |

Original characters have full translation sets; user-created characters typically have 2 entries in user's locale only.

### Body Strings (STR# 200)

30 strings define character appearance:

| Index | Field             | Example Values                                     |
| ----- | ----------------- | -------------------------------------------------- |
| 1     | Age               | `adult`, `child`                                   |
| 2     | Body mesh/texture | `b002mafat_01,BODY=B002MAFatlgt_slob`              |
| 3     | Head mesh/texture | `c010ma_baldbeard,HEAD-HEAD=C010MAlgt_baldbeard01` |
| 4–7   | Skeleton refs     | HEAD, Top, R_HAND, Palm                            |
| 13    | Gender            | `male`, `female`                                   |
| 14    | Age number        | `27` (adult), `9` (child)                          |
| 15    | Skin color        | `drk`, `lgt`, `med`                                |
| 16    | Nude mesh/texture | Swimwear/underwear                                 |
| 17    | Swim mesh/texture | Swimming outfit                                    |
| 18–21 | Hand meshes       | Left/right open/closed                             |

### Mesh/Texture Encoding

```text
Body:   mesh_name,BODY=texture_name
Head:   mesh_name,HEAD-HEAD=texture_name
Hand:   hand_mesh,HAND=skin_texture
```

**Example decode:**

```text
Input:  "b002mafat_01,BODY=B002MAFatlgt_slob"
Mesh:   b002mafat_01.cmx.bcf
Texture: B002MAFatlgt_slob.bmp
```

### Mesh Naming Conventions

**Body types:** `fat`, `fit`, `skn` (skinny)  
**Gender:** `m` (male), `f` (female)  
**Skin tones:** `drk`, `lgt`, `med`

**Hand mesh pattern:**

```text
H[m/f/u][L/R][O/C]
 │  │    │   └── Open/Closed
 │  │    └────── Left/Right
 │  └─────────── m=male, f=female, u=child
 └────────────── Hand prefix
```

### XXXX Filler Resources

- **Flags:** 0x004 (unique)
- **Purpose:** Padding from in-place edits
- **Action:** Safely ignore during parsing

---

## Chapter 2: Career System (CARR)

### CARR Overview

CARR resources define complete career tracks: job levels, requirements, scheduling, pay, uniforms, and motive effects.

| Property    | Value                           |
| ----------- | ------------------------------- |
| Type code   | `CARR`                          |
| Encoding    | Bit-packed fields               |
| Compression | Variable-width (0x01 indicator) |

### File Locations

| Game Version       | File        | Container           |
| ------------------ | ----------- | ------------------- |
| Base + House Party | Careers.iff | GameData/           |
| Hot Date+          | work.iff    | ExpansionShared.far |

### Physical Structure

```text
Offset  Content
0       0x00000000 (zero)
4       0x00000000 (zero)
8       'RRAC' (little-endian marker)
12      Career Info header
var     Job level 1
var     Job level 2
...     Job level N
```

### Career Tracks

| ID  | Career            | ID  | Career            |
| --- | ----------------- | --- | ----------------- |
| 1   | Business          | 6   | Military          |
| 2   | Entertainment     | 7   | Politics          |
| 3   | Law Enforcement   | 8   | Pro Athlete       |
| 4   | Life of Crime     | 9   | Science           |
| 5   | Medicine          | 10  | X-Treme           |
| -1  | School (children) | 12+ | Expansion careers |

💡 **ID 11:** Missing from sequence. Possibly reserved for "Unemployed" status.

### Job Level Data

**Skill Requirements (×100 scale):**

| Field      | Description                |
| ---------- | -------------------------- |
| Friends    | Friendship points required |
| Cooking    | Skill × 100                |
| Mechanical | Skill × 100                |
| Charisma   | Skill × 100                |
| Body       | Skill × 100                |
| Logic      | Skill × 100                |
| Creativity | Skill × 100                |

**Motive Decay (per hour at work):**

| Motive  | Typical Range | Notes                              |
| ------- | ------------- | ---------------------------------- |
| Hunger  | -3 to 0       | Depletes                           |
| Comfort | -5 to 1       | Usually negative                   |
| Hygiene | -3 to 1       | Usually negative                   |
| Bladder | -4 to 0       | Depletes steadily                  |
| Energy  | -7 to 0       | High for labor jobs                |
| Fun     | -5 to 1       | Work rarely fun                    |
| Social  | -2 to 4       | **Positive** for political careers |

⚠️ **Anomaly:** Hacker career's Security Consultant has positive energy decay (likely a bug).

**Schedule & Pay:**

| Field      | Range   | Description          |
| ---------- | ------- | -------------------- |
| Salary     | 0–5000+ | Daily pay            |
| Start time | 0–23    | Hours (0 = midnight) |
| End time   | 0–23    | Hours                |
| Car type   | 0–11    | Vehicle code         |

### Vehicle Type Codes

| Code | Vehicle  | Code | Vehicle   |
| ---- | -------- | ---- | --------- |
| 0    | Coupe    | 6    | Bentley   |
| 1    | Jeep     | 7    | Junker    |
| 2    | Cruiser  | 8    | Limo      |
| 3    | Sedan    | 9    | Truck     |
| 4    | SUV      | 10   | Circus    |
| 5    | Town car | 11   | Clown car |

### Uniform Specification

| Field               | Description             |
| ------------------- | ----------------------- |
| Job title           | Position name           |
| Male uniform mesh   | With substitution codes |
| Female uniform mesh | Null if same as male    |
| Uniform skin        | Bitmap texture          |

**Substitution codes:**

| Code | Values                     |
| ---- | -------------------------- |
| `$g` | `m` or `f` (gender)        |
| `$b` | `fat`, `fit`, `skn` (body) |
| `$c` | `drk`, `lgt`, `med` (skin) |

**Example:** `uniform_$g_$b_$c` → `uniform_m_fat_drk`

### Field Encoding

CARR uses bit-level variable-width compression:

```text
Zero value:    1 bit (0)
Non-zero:      PPP S NNNN...
               │   │ └── Value bits
               │   └──── Sign bit
               └──────── 3-bit width prefix
```

| Prefix | Width   |
| ------ | ------- |
| 100    | 5 bits  |
| 101    | 10 bits |
| 110    | 20 bits |
| 111    | 31 bits |

---

## Chapter 3: String Tables (STR#)

### STR# Overview

STR# resources store text with multi-language support. Multiple encoding formats exist.

| Property  | Value             |
| --------- | ----------------- |
| Type code | `STR#`            |
| Languages | Up to 20          |
| Formats   | 5 (auto-detected) |

### Format Detection

| First 2 bytes (BE) | Format | Description            |
| ------------------ | ------ | ---------------------- |
| ≥ 0                | 0      | Count = value          |
| 0xFFFF             | FFFF   | Null-terminated        |
| 0xFEFF             | FEFF   | Paired null-terminated |
| 0xFDFF             | FDFF   | Language-coded pairs   |
| 0xFCFF             | FCFF   | Dynamic length (TSO+)  |

### Format 0: Simple Counted

```text
Header:  2 bytes (count, big-endian)
Entries: Length byte + characters (Pascal)
```

**Limit:** 255 bytes per string

### Format FFFF: Null-Terminated

```text
Header:  0xFFFF + count (4 bytes)
Entries: String + 0x00 terminator
```

**Advantage:** Unlimited string length

### Format FEFF: Paired Strings

```text
Header:  0xFEFF + count (4 bytes)
Entries: String1 + 0x00 + String2 + 0x00
```

**Purpose:** Data string + annotation/comment

### Format FDFF: Language-Coded

```text
Header:  0xFDFF + count (4 bytes)
Entries: Language code (1B) + String1 + 0x00 + String2 + 0x00
Trailer: 0xA3 padding bytes
```

### Format FCFF: Dynamic Length (TSO+)

```text
Header:  0xFCFF + set count (3 bytes)
Sets:    20 language sets (count + entries)
Entries: Code + Length1 + String1 + Length2 + String2
```

**Dynamic length encoding:**

- < 128: Single byte
- ≥ 128: Two bytes `(0x80 | low7, high8)`

### Format Comparison

| Feature       | 0    | FFFF | FEFF | FDFF | FCFF |
| ------------- | ---- | ---- | ---- | ---- | ---- |
| Max string    | 255B | ∞    | ∞    | ∞    | 32KB |
| Multi-lingual | No   | No   | No   | Yes  | Yes  |
| Dual strings  | No   | No   | Yes  | Yes  | Yes  |

---

## Chapter 4: FCNS Function Constants

### FCNS Overview

FCNS contains game-wide tuning constants: starting funds, motive decay ratios, skill rates.

| Property  | Value       |
| --------- | ----------- |
| Type code | `FCNS`      |
| Location  | Global.iff  |
| Instances | 4 resources |

### Header (16 bytes)

| Offset | Size | Field            |
| ------ | ---- | ---------------- |
| 0      | 4    | Reserved (0)     |
| 4      | 4    | Version (1 or 2) |
| 8      | 4    | `FCNS` marker    |
| 12     | 4    | Entry count      |

### Entry Structure

| Field             | Type   | Example              |
| ----------------- | ------ | -------------------- |
| Short description | String | "starting_simoleans" |
| Value             | Float  | 20000.0              |
| Long description  | String | Explanation          |

### Known Parameters (Speculative)

- Household starting funds
- Advertisement volume attenuation
- Motive decay relationships
- Skill gain rates
- Need satisfaction ratios

---

## Chapter 5: Language Codes Reference

### Canonical Language Code Table

This is the authoritative reference for language codes used throughout The Sims.

| Code | Language         | Code | Language            |
| ---- | ---------------- | ---- | ------------------- |
| 0    | (Default/unused) | 11   | Finnish             |
| 1    | **English (US)** | 12   | Hebrew              |
| 2    | English (UK)     | 13   | Russian             |
| 3    | French           | 14   | Portuguese          |
| 4    | German           | 15   | Japanese            |
| 5    | Italian          | 16   | Polish              |
| 6    | Spanish          | 17   | Simplified Chinese  |
| 7    | Dutch            | 18   | Traditional Chinese |
| 8    | Danish           | 19   | Thai                |
| 9    | Swedish          | 20   | Korean              |
| 10   | Norwegian        |      |                     |

### Usage Notes

- **Code 0:** Reserved, not used in STR# resources
- **Code 1 (US English):** Fallback for missing translations
- **Maximum translations:** Most objects have 14 languages
- **Custom content:** Typically only includes user's locale

### Language Fallback

When a string is missing in requested language:

```text
1. Try requested language code
2. Fall back to code 1 (US English)
3. If still missing, display empty or placeholder
```

---

## Chapter 6: Bundle Organization

### Bundle Types

| Bundle           | Primary Resources           | Purpose          |
| ---------------- | --------------------------- | ---------------- |
| **Object**       | OBJD, DGRP, SPR#, BHAV      | Catalog items    |
| **Character**    | OBJD, STR# 200, CTSS, BMP\_ | Sim characters   |
| **Career**       | CARR, STR#                  | Job tracks       |
| **Neighborhood** | (Unknown)                   | Families, houses |
| **Walls/Floors** | SPR#, STR#, pricing         | Build mode       |

### Object Bundle Pattern

```text
Object IFF
├── OBJD (definition, maybe multiple for multi-tile)
├── CTSS (catalog description)
├── DGRP (sprite groups)
├── SPR2 (sprite images)
├── PALT (color palette)
├── STR# (animation names)
├── TTAB (interactions)
├── TTAs (menu text)
├── BHAV (behaviors)
├── BCON (constants)
├── OBJf (entry points)
├── GLOB (semi-global ref)
└── BMP_ (catalog thumbnail)
```

### Character Bundle Pattern

```text
Character IFF
├── OBJD #128 (type=2, person)
├── CTSS #2000 (surname)
├── STR# #200 (body strings)
├── STR# #256 (attributes)
├── BMP_ #2002-2007 (portraits)
├── SLOT #128 (routing)
├── GLOB #128 (semi-global)
└── BHAV #4096+ (behaviors)
```

### Career Bundle Pattern

```text
Career IFF
├── CARR (career data × N tracks)
└── STR# (localized strings × 2 per track)
```

### Resource Dependencies

```text
OBJD (object definition)
├── → STR# (object name via catalog_strings_id)
├── → BHAV (behaviors via OBJf)
│      └── → BCON (constants)
├── → DGRP (graphics via base_graphic)
│      └── → SPR# / SPR2 (sprites)
│             └── → PALT (palette)
├── → GLOB (semi-global reference)
└── → FWAV (audio references)
```

---

## Cross-Reference Diagram

```text
GAME CONTENT HIERARCHY

┌─────────────────────────────────────────────────────────────┐
│                    CHARACTER BUNDLE                          │
│                                                             │
│  ┌─────────────┐                                           │
│  │  OBJD #128  │ ─── Type: 2 (person)                      │
│  │  Character  │                                           │
│  └──────┬──────┘                                           │
│         │                                                   │
│    ┌────┴────┬──────────┬──────────┬──────────┐           │
│    ▼         ▼          ▼          ▼          ▼           │
│ ┌──────┐ ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │
│ │ CTSS │ │STR#  │  │ BMP_ │  │ SLOT │  │ BHAV │          │
│ │#2000 │ │ #200 │  │Portr.│  │ #128 │  │4096+ │          │
│ │Name  │ │ Body │  │      │  │Route │  │      │          │
│ └──────┘ └──────┘  └──────┘  └──────┘  └──────┘          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     CAREER SYSTEM                            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    CARR Resource                      │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  Track Name: "Business"                        │  │  │
│  │  │  Job Levels: 10                                │  │  │
│  │  │                                                │  │  │
│  │  │  Level 1: Mailroom Clerk                       │  │  │
│  │  │   • Salary: §120    • Hours: 9-3               │  │  │
│  │  │   • Skills: None    • Friends: 0               │  │  │
│  │  │   • Vehicle: Junker • Uniform: mailroom_$g_$b  │  │  │
│  │  │                                                │  │  │
│  │  │  Level 10: CEO                                 │  │  │
│  │  │   • Salary: §3500   • Hours: 9-4               │  │  │
│  │  │   • Skills: 6/4/4/4 • Friends: 14              │  │  │
│  │  │   • Vehicle: Limo   • Uniform: ceo_$g_$b       │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  STR# Resources (per career × 2)                     │  │
│  │   • Performance strings (promotion messages)         │  │
│  │   • Grade strings (school)                          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

STRING TABLE FORMATS

┌─────────────────────────────────────────────────────────────┐
│  Detection: First 2 bytes (big-endian)                      │
│                                                             │
│  ≥ 0x0000  ──▶ Format 0 (Simple counted)                   │
│  0xFFFF   ──▶ Format FFFF (Null-terminated)                │
│  0xFEFF   ──▶ Format FEFF (Paired)                         │
│  0xFDFF   ──▶ Format FDFF (Language-coded)                 │
│  0xFCFF   ──▶ Format FCFF (Dynamic, TSO+)                  │
└─────────────────────────────────────────────────────────────┘

LANGUAGE CODE USAGE

┌─────────────────────────────────────────────────────────────┐
│  Resources using language codes:                            │
│                                                             │
│  • STR# (string tables)                                    │
│  • CTSS (catalog strings)                                  │
│  • TTAs (menu text)                                        │
│  • CARR (career embedded strings)                          │
│  • Preferences file (byte 2)                               │
│                                                             │
│  Fallback chain:                                           │
│  Requested → Code 1 (US English) → Empty                   │
└─────────────────────────────────────────────────────────────┘
```

---

_Part V Complete — Continue to [Part VI: Tools & Workflows](Part_VI_Tools.md)_
