# Part I: Foundations

> **The Sims 1 Technical Documentation — Complete Reference**

Welcome to the comprehensive technical documentation for The Sims (2000). This community-driven knowledge base enables tool development, custom content creation, and deep game modification.

---

## Document Navigation

| Part                                            | Focus                        | You'll Learn                |
| ----------------------------------------------- | ---------------------------- | --------------------------- |
| **I. Foundations**                              | Architecture & Concepts      | 📍 You are here             |
| [II. Containers](Part_II_Containers.md)         | FAR & IFF Formats            | How files are packaged      |
| [III. Object System](Part_III_ObjectSystem.md)  | OBJD, BHAV, VM               | How objects work            |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, Meshes, Animation   | How things are displayed    |
| [V. Game Content](Part_V_GameContent.md)        | Characters, Careers, Strings | How game data is structured |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | IFFSnooper Guides            | How to edit the game        |

---

## Chapter 1: Documentation Overview

### Purpose

This documentation collects technical information on The Sims file formats and internal workings, enabling community members to create:

- **Custom objects** — Furniture, appliances, decorations
- **Characters & NPCs** — New Sims, service providers
- **Careers** — Job tracks with custom requirements
- **Tools** — Editors, converters, validators

### Audience

| Reader               | Recommended Path                           |
| -------------------- | ------------------------------------------ |
| **New modders**      | Part I → Part VI (Tools) → Parts as needed |
| **Tool developers**  | Part I → Part II → Part III                |
| **Content creators** | Part I → Part VI → Part IV (Graphics)      |
| **Researchers**      | All parts sequentially                     |

### Documentation Conventions

Throughout this documentation:

- **Hexadecimal** values use `0x` prefix: `0xFF`, `0x8002`
- **Binary** layouts show offset, size, and field name
- **Code blocks** contain structure definitions or examples
- **Tables** summarize field values and meanings
- ⚠️ **Warnings** highlight critical requirements
- 💡 **Tips** provide practical guidance

### Contributors & History

This documentation originated from the SimTech project (simtech.sourceforge.net, 2000-2010) through community reverse-engineering:

- Dave Baum — Animation formats, CFP compression research
- Greg Noel — IFF format, resource specifications
- Don Hopkins — VM documentation, behavior analysis
- Lee Reeves — Format discoveries
- Peter Gould — IFFSnooper tool and documentation

---

## Chapter 2: Game Architecture

### The Data Hierarchy

The Sims organizes all game data in a nested container structure:

```text
Game Installation
│
├── FAR Archives (.far)
│   │   Bundles of files for efficient storage
│   │
│   └── IFF Files (.iff)
│       │   Resource containers for game objects
│       │
│       └── Resources (OBJD, BHAV, SPR#, etc.)
│               Individual data units
│
├── Loose IFF Files (.iff, .flr, .wll)
│       Custom content in Downloads/ folder
│
└── Support Files
        Preferences, saves, configuration
```

### File Locations

| Path               | Contents                           | Modifiable      |
| ------------------ | ---------------------------------- | --------------- |
| `GameData/`        | Core game resources                | ⚠️ Backup first |
| `GameData/Global/` | Shared behaviors and constants     | ⚠️ Backup first |
| `Downloads/`       | User-installed custom content      | ✅ Yes          |
| `UserData/`        | Saves, preferences, generated data | ✅ Yes          |
| `ExpansionPack*/`  | Expansion-specific assets          | ⚠️ Backup first |

### Resource Loading Priority

The game loads resources in priority order, allowing custom content to override defaults:

```text
1. Downloads/     ← Highest (custom content wins)
2. ExpansionPack/ ← Expansion overrides base
3. GameData/      ← Lowest (base game)
```

This hierarchy enables modding without modifying original files.

---

## Chapter 3: Technical Conventions

### Byte Order (Endianness)

⚠️ **Critical:** The Sims uses DIFFERENT byte orders for different formats:

| Format              | Byte Order    | Multi-byte Example         |
| ------------------- | ------------- | -------------------------- |
| **IFF containers**  | Big-endian    | `0x1234` stored as `12 34` |
| **Everything else** | Little-endian | `0x1234` stored as `34 12` |

**"Everything else" includes:** FAR, CMX, BCF, CFP, SKN, BMF, BMP, and all IFF resource _contents_.

The IFF container format inherited big-endian from EA's legacy systems, but resource data inside IFF files uses little-endian.

### Numeric Types

| Type             | Size | Range           | Usage                  |
| ---------------- | ---- | --------------- | ---------------------- |
| `u8` / `byte`    | 1    | 0–255           | Flags, small counts    |
| `i8` / `sbyte`   | 1    | -128–127        | Signed offsets         |
| `u16` / `word`   | 2    | 0–65,535        | Resource IDs, counts   |
| `i16` / `sword`  | 2    | -32,768–32,767  | Coordinates, values    |
| `u32` / `dword`  | 4    | 0–4,294,967,295 | Offsets, GUIDs         |
| `i32` / `sdword` | 4    | ±2.1 billion    | Large values           |
| `f32` / `float`  | 4    | IEEE 754        | Coordinates, animation |

### String Encodings

| Type              | Structure                    | Max Length | Used In                  |
| ----------------- | ---------------------------- | ---------- | ------------------------ |
| **Pascal string** | Length byte + characters     | 255        | BCF, BMF, most resources |
| **C string**      | Characters + null terminator | Unlimited  | IFF names, some strings  |
| **Padded string** | String + `0xA3` padding      | Varies     | CARR, GLOB, to alignment |

**Padding byte `0xA3`:** Used throughout to pad strings to even boundaries.

### Common Patterns

**Resource Identification:**

```text
┌────────────────────────────────────────┐
│ Every IFF resource has:                │
│ • Type code  (4 chars: "OBJD")         │
│ • Resource ID (0–65535, unique/type)   │
│ • Name (optional, 64 chars max)        │
└────────────────────────────────────────┘
```

**Version Fields:**
Many resources begin with a version number enabling format evolution:

- Check version before parsing
- Newer versions add fields at end
- Tools should handle multiple versions

---

## Chapter 4: Format Overview

### Container Formats

| Format  | Extension                              | Purpose                         | Documented In                                                                 |
| ------- | -------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------- |
| **FAR** | `.far`                                 | Archive bundling multiple files | [Part II, Ch. 1](Part_II_Containers.md#chapter-1-far-archive-format)          |
| **IFF** | `.iff`, `.flr`, `.wll`, `.spf`, `.stx` | Resource container              | [Part II, Ch. 2](Part_II_Containers.md#chapter-2-iff-interchange-file-format) |

### External File Formats

| Format  | Extension  | Purpose                     | Documented In                                                              |
| ------- | ---------- | --------------------------- | -------------------------------------------------------------------------- |
| **CMX** | `.cmx`     | Skeleton/animation (text)   | [Part IV, Ch. 4](Part_IV_Graphics.md#chapter-4-cmxbcf-character-animation) |
| **BCF** | `.cmx.bcf` | Skeleton/animation (binary) | [Part IV, Ch. 4](Part_IV_Graphics.md#chapter-4-cmxbcf-character-animation) |
| **SKN** | `.skn`     | 3D mesh definition (text)   | [Part IV, Ch. 5](Part_IV_Graphics.md#chapter-5-sknbmf-mesh-format)         |
| **BMF** | `.bmf`     | 3D mesh definition (binary) | [Part IV, Ch. 5](Part_IV_Graphics.md#chapter-5-sknbmf-mesh-format)         |
| **CFP** | `.cfp`     | Compressed animation frames | [Part IV, Ch. 6](Part_IV_Graphics.md#chapter-6-cfp-animation-data)         |
| **BMP** | `.bmp`     | Windows bitmap image        | [Part II, Ch. 4](Part_II_Containers.md#chapter-4-bmp-bitmap-format)        |

### IFF Resource Types

**Object System** — [Part III](Part_III_ObjectSystem.md)

| Code   | Name                 | Purpose                        |
| ------ | -------------------- | ------------------------------ |
| `OBJD` | Object Definition    | Properties, pricing, behaviors |
| `OBJf` | Object Functions     | Entry point table (vtable)     |
| `BHAV` | Behavior             | VM subroutine code             |
| `BCON` | Behavioral Constants | Tunable integer values         |
| `TRCN` | Trait Constants      | Labels for BCON values         |
| `TPRP` | Trait Properties     | Labels for BHAV parameters     |
| `TTAB` | Tree Table           | Interaction definitions        |
| `TTAs` | Tree Table Strings   | Pie menu text                  |
| `GLOB` | Global Reference     | Semi-global file link          |
| `SLOT` | Slot                 | Character routing positions    |

**Graphics System** — [Part IV](Part_IV_Graphics.md)

| Code   | Name            | Purpose               |
| ------ | --------------- | --------------------- |
| `DGRP` | Draw Group      | Sprite compositor     |
| `SPR#` | Sprite (Type A) | 8-bit indexed sprites |
| `SPR2` | Sprite (Type B) | Multi-channel sprites |
| `PALT` | Palette         | 256-color table       |
| `BMP_` | Bitmap          | Embedded BMP image    |
| `FBMP` | Floor Bitmap    | Floor textures        |

**Content Data** — [Part V](Part_V_GameContent.md)

| Code   | Name               | Purpose                  |
| ------ | ------------------ | ------------------------ |
| `STR#` | String Table       | Multi-language text      |
| `CTSS` | Catalog Strings    | Object descriptions      |
| `CARR` | Career             | Career track definitions |
| `FCNS` | Function Constants | Game-wide tuning         |
| `FWAV` | File WAV           | Audio reference          |

**Structure** — [Part II](Part_II_Containers.md)

| Code   | Name         | Purpose               |
| ------ | ------------ | --------------------- |
| `rsmp` | Resource Map | Index for fast lookup |

---

## Chapter 5: Modding Quick Start

### Creating Custom Content

**Objects (furniture, appliances):**

1. Find similar existing object
2. Clone with IFFSnooper (File → Clone)
3. Edit OBJD (properties, price)
4. Replace sprites (export → edit → import)
5. Modify CTSS (catalog description)
6. Test in-game

**Characters (NPCs):**

1. Start from existing character IFF
2. Edit body strings (STR# 200)
3. Create/modify meshes (SKN)
4. Assign textures
5. Test in-game

**Careers:**

1. Edit CARR resource
2. Modify job levels, requirements
3. Update STR# strings for localization
4. Test progression in-game

### Essential Tools

| Tool                   | Purpose                 | Required     |
| ---------------------- | ----------------------- | ------------ |
| **IFFSnooper**         | View/edit IFF resources | ✅ Essential |
| **Hex editor**         | Binary inspection       | ✅ Essential |
| **Image editor**       | Sprite/texture editing  | ✅ Essential |
| **Transmogrifier**     | Advanced object editing | Recommended  |
| **SimAntics debugger** | Behavior development    | Advanced     |

### Critical Rules

⚠️ **GUID Uniqueness:** Every object must have a unique GUID. Conflicts cause objects to disappear.

⚠️ **Backup First:** Always backup original files before modification.

⚠️ **Test Thoroughly:** Test all modifications in-game before distribution.

💡 **Downloads Folder:** Place custom content in `Downloads/` — never modify `GameData/` directly.

---

## Quick Reference Tables

### File Size Limits

| Format               | Maximum | Notes                    |
| -------------------- | ------- | ------------------------ |
| FAR archives         | 4 GB    | 32-bit offset addressing |
| IFF files            | 4 GB    | 32-bit size fields       |
| Individual resources | ~16 MB  | Practical limit          |

### Platform Specifications

| Property           | Value                 |
| ------------------ | --------------------- |
| Original platforms | Windows 98/ME/2000/XP |
| Architecture       | 32-bit x86            |
| Graphics API       | DirectX 7+            |
| Rendering          | Software and hardware |

### Expansion Pack Order

| #   | Expansion    | New Features           |
| --- | ------------ | ---------------------- |
| 1   | Livin' Large | NPCs, careers, objects |
| 2   | House Party  | Party system           |
| 3   | Hot Date     | Downtown, dating       |
| 4   | Vacation     | Vacation Island        |
| 5   | Unleashed    | Pets, Old Town         |
| 6   | Superstar    | Studio Town, fame      |
| 7   | Makin' Magic | Magic Town, spells     |

---

## Cross-Reference: Document Map

```text
Part I: FOUNDATIONS (You are here)
│
├── Part II: CONTAINERS
│   ├── FAR Archives
│   ├── IFF Format
│   ├── Resource Map (rsmp)
│   └── BMP Reference
│
├── Part III: OBJECT SYSTEM
│   ├── Object Definition (OBJD)
│   ├── Behaviors (BHAV)
│   ├── Constants (BCON, TRCN)
│   ├── Interactions (TTAB)
│   ├── Routing (SLOT)
│   └── Virtual Machine
│
├── Part IV: GRAPHICS & ANIMATION
│   ├── Sprites (DGRP, SPR#, SPR2, PALT)
│   ├── Character Meshes (SKN/BMF)
│   ├── Skeletons (CMX/BCF)
│   └── Animation Data (CFP)
│
├── Part V: GAME CONTENT
│   ├── Character Bundles
│   ├── Career System (CARR)
│   ├── String Tables (STR#)
│   └── Language Reference
│
└── Part VI: TOOLS & WORKFLOWS
    ├── IFFSnooper Guide
    ├── Editing Tutorials
    └── Workflow Reference
```

---

\_Part I Complete — Continue to [Part II: Containers](Part_II_Containers.md)
