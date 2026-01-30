# Part VI: Tools & Workflows

> **IFFSnooper guides and practical modding workflows**

This part covers practical tool usage for editing IFF files, sprites, objects, and other game resources.

---

## Document Navigation

| Part                                            | Focus                        | Status          |
| ----------------------------------------------- | ---------------------------- | --------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture & Concepts      |                 |
| [II. Containers](Part_II_Containers.md)         | FAR & IFF Formats            |                 |
| [III. Object System](Part_III_ObjectSystem.md)  | OBJD, BHAV, VM               |                 |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, Meshes, Animation   |                 |
| [V. Game Content](Part_V_GameContent.md)        | Characters, Careers, Strings |                 |
| **VI. Tools & Workflows**                       | IFFSnooper Guides            | 📍 You are here |

---

## Key Terms

| Term              | Definition                                    |
| ----------------- | --------------------------------------------- |
| **Alpha Channel** | Transparency mask (0=transparent, 255=opaque) |
| **GUID**          | Global Unique Identifier (4 bytes)            |
| **Magic Cookie**  | Middle bytes of GUID identifying creator      |
| **P-Sprite**      | Primary sprite (visible pixels)               |
| **Z-Buffer**      | Depth map determining layer ordering          |
| **Disassemble**   | Export IFF resources to separate files        |
| **Assemble**      | Recombine resources into IFF                  |

---

## Chapter 1: IFFSnooper Overview

### Capabilities

| Category            | Description                  |
| ------------------- | ---------------------------- |
| **Viewing**         | All IFF resource types       |
| **Full Editing**    | OBJD, CTSS, STR#, TTAs, CARR |
| **Partial Editing** | TTAB (flags and motives)     |
| **Export/Import**   | Sprites, resources           |
| **Read-Only**       | BHAV, SLOT, TREE, OBJf       |

### Resource Support Matrix

| Resource | View | Edit        | Export   |
| -------- | ---- | ----------- | -------- |
| OBJD     | ✓    | ✓ Full      | ✓        |
| CTSS     | ✓    | ✓ Full      | ✓        |
| STR#     | ✓    | ✓ Full      | ✓        |
| TTAs     | ✓    | ✓ Full      | ✓        |
| TTAB     | ✓    | Partial     | ✓        |
| SPR2     | ✓    | Import      | ✓ Images |
| PALT     | ✓    | Auto        | ✓        |
| CARR     | ✓    | ✓ (v1.1.5+) | ✓        |
| BHAV     | ✓    | —           | ✓        |
| SLOT     | ✓    | —           | ✓        |
| TREE     | ✓    | —           | ✓        |
| OBJf     | ✓    | —           | ✓        |

### Version Features

| Version | Key Features                     |
| ------- | -------------------------------- |
| 1.0.6   | Basic editing, Import → Assemble |
| 1.0.7   | Import → Resources path change   |
| 1.1.3   | GUID-dependent BHAV auto-update  |
| 1.1.5   | CARR editing, objt viewing       |
| 1.2.0   | Add Menu button, Autofirst flag  |

---

## Chapter 2: Sprite Anatomy

### Frame Composition

Each sprite frame consists of three channels:

```
┌─────────────────────────────────┐
│         Complete Frame          │
│                                 │
│  ┌───────────────────────────┐ │
│  │      P-Sprite (RGB)       │ │  ← Visible pixel colors
│  └───────────────────────────┘ │
│  ┌───────────────────────────┐ │
│  │      Z-Buffer (gray)      │ │  ← Depth (dark=near)
│  └───────────────────────────┘ │
│  ┌───────────────────────────┐ │
│  │      Alpha (gray)         │ │  ← Transparency
│  └───────────────────────────┘ │
└─────────────────────────────────┘
```

### Z-Buffer Values

| Value           | Depth            | Visual Result   |
| --------------- | ---------------- | --------------- |
| Dark (0-50)     | Closer to camera | Drawn in front  |
| Mid (100-150)   | Middle ground    | Normal layering |
| Light (200-255) | Further away     | Drawn behind    |

### Alpha Channel Values

| Value        | Effect            | Use Case                  |
| ------------ | ----------------- | ------------------------- |
| 0 (black)    | Fully transparent | Background                |
| 1-254 (gray) | Semi-transparent  | Anti-aliased edges, glass |
| 255 (white)  | Fully opaque      | Solid objects             |

### Frame Positioning

DGRP resources specify where sprites appear:

| DGRP Field  | Description                        |
| ----------- | ---------------------------------- |
| pixelx      | Horizontal offset from tile center |
| pixely      | Vertical offset from tile center   |
| SPR# ref    | Which sprite resource              |
| Frame index | Which frame within sprite          |

### Naming Convention

IFFSnooper exports sprites with this pattern:

```
{object}_{size}_{direction}_{channel}.bmp

Sizes:      large, medium, small
Directions: ne, se, nw, sw, n, s
Channels:   p (color), z (depth), a (alpha)
```

**Examples:**

- `chair_large_ne_p.bmp` — Large NE view, colors
- `chair_medium_sw_z.bmp` — Medium SW view, depth
- `lamp_small_se_a.bmp` — Small SE view, transparency

---

## Chapter 3: Creating IFF Files

### Workflow Overview

```
1. Source Object   ──▶ Find similar existing IFF
         │
2. Disassemble     ──▶ Export → Disassemble
         │
3. Edit Resources  ──▶ Modify OBJD, sprites, text
         │
4. Edit Sprites    ──▶ Export/modify/reimport images
         │
5. Reassemble      ──▶ Import → Assemble (or Resources)
         │
6. Test            ──▶ Load in game
```

### Export (Disassemble)

**Menu:** Export → Disassemble

**Output structure:**

```
output_folder/
├── IFF_header.bin
├── rsmp.bin
├── OBJD_001.bin
├── CTSS_001.bin
├── DGRP_001.bin
├── SPR2_001.bin
├── PALT_001.bin
├── STR#_001.bin
├── TTAB_001.bin
├── TTAs_001.bin
└── BMP__001.bin
```

### Import (Reassemble)

| Version | Menu Path          |
| ------- | ------------------ |
| 1.0.7+  | Import → Resources |
| 1.0.6-  | Import → Assemble  |

### Minimum Resources

| Object Type | Required                        |
| ----------- | ------------------------------- |
| Basic       | OBJD, DGRP, SPR2, PALT          |
| Interactive | + TTAB, TTAs                    |
| Catalog     | + CTSS, BMP\_                   |
| Multi-tile  | Multiple OBJD (master + slaves) |

### Common Pitfalls

| Problem          | Symptom              | Solution                 |
| ---------------- | -------------------- | ------------------------ |
| Missing PALT     | Crash or invisible   | Include palette          |
| Invalid GUID     | Object missing       | Generate unique GUID     |
| Orphaned SPR2    | Sprites don't show   | Fix DGRP references      |
| Broken TTAB      | Menus fail           | Verify structure         |
| Missing CTSS     | No catalog entry     | Add CTSS, set ID in OBJD |
| Wrong DGRP count | Graphics don't cycle | Match num_graphics       |

---

## Chapter 4: Editing OBJD

### Panel Layout

**Single-tile objects:** Flat parameter list

**Multi-tile objects:**

- Tile-specific parameters shown
- "Master" button navigates to master OBJD

### Key Parameters

| Section       | Parameters                                             |
| ------------- | ------------------------------------------------------ |
| **Identity**  | GUID, Magic Cookie, Master ID, Subindex                |
| **Graphics**  | DGRP base, DGRP count, Dynamic sprite base/count       |
| **Resources** | CTSS resource, STR# base, TTAs resource, SLOT resource |
| **Catalog**   | Price, Minimum resale, Room flag, Function flag        |
| **Expansion** | Downtown, Island, OldTown, StudioTown, MagicTown flags |

### Multi-Tile Structure

**Master OBJD:**

```
subindex = -1 (0xFFFF)
master_id = same as own ID
Contains: price, catalog flags, object-wide settings
```

**Slave OBJD:**

```
subindex = (Y_offset << 8) | X_offset
master_id = master's OBJD ID
Contains: tile-specific DGRP, sprite references
```

### Editing Workflow

```
1. Open IFF       ──▶ File → Open
2. Select OBJD    ──▶ Click in resource list
3. View params    ──▶ Tab panel displays values
4. Edit           ──▶ Click "Edit" or double-click
5. Modify         ──▶ Change values as needed
6. Save           ──▶ Click "Save" in EditWindow
7. Export/Import  ──▶ Reassemble IFF
8. Test           ──▶ Verify in game
```

---

## Chapter 5: GUID & Magic Cookie

### GUID Format

```
GUID: aa bb cc dd (4 bytes, hexadecimal)
      └─┴─┴─┴─── Must be unique across ALL objects
```

### GUID Conflicts

**Problem:** Multiple objects with same GUID  
**Result:** Only one appears in-game  
**Solution:** Clone or manually edit GUID

### Clone Operations

| Operation       | Function                        |
| --------------- | ------------------------------- |
| File → Clone    | Copy with new GUID              |
| File → Subclone | Partial clone (subset of tiles) |

💡 Version 1.1.3+ automatically updates GUID-dependent BHAV code during clone.

### Magic Cookie

**Location:** Middle 2 bytes of GUID (`bb cc` in `aa bb cc dd`)

**Purpose:** Identifies object creator

**Setting:**

1. Edit → Set Cookie
2. Enter your value
3. Future GUIDs will include it

**Large values:** Use modulo 65536

```
Cookie 139072 = 139072 - (2 × 65536) = 8000
```

### Multi-Tile Requirements

| Component     | GUID Rule             |
| ------------- | --------------------- |
| Master OBJD   | Unique GUID           |
| Slave OBJD #1 | Different unique GUID |
| Slave OBJD #2 | Different unique GUID |
| ...           | Each must be unique   |

⚠️ Non-unique slave GUIDs = incomplete object

---

## Chapter 6: Text Editing

### String Resource Types

| Type | Content                   | Safety    |
| ---- | ------------------------- | --------- |
| CTSS | Catalog title/description | ✓ Safe    |
| STR# | General strings           | ⚠️ Varies |
| TTAs | Pie menu text             | ✓ Safe    |

### CTSS Fields

| Field               | Purpose                   |
| ------------------- | ------------------------- |
| Catalog Title       | Object name (Buy Mode)    |
| Catalog Description | Descriptive text          |
| Title Comment       | Internal note (not shown) |

### STR# Safety

| Context         | Safety     | Notes                 |
| --------------- | ---------- | --------------------- |
| Walls/Floors    | ✓ Safe     | No dependencies       |
| Body Strings    | ⚠️ Caution | Must match skin files |
| Animation Names | ⚠️ Caution | Must match animations |
| Dialog Text     | ✓ Safe     | Display only          |

### Multi-Language Support

IFFSnooper supports all language codes (see [Part V Chapter 5](Part_V_GameContent.md#chapter-5-language-codes-reference) for canonical list).

**Tip:** Custom content typically only needs user's locale (code 1 for US English).

### Body Strings Setup

For character editing:

1. IFFSnooper Settings → Set GameData/Skins path
2. Body string fields show dropdown menus
3. BMP preview available for skins

---

## Chapter 7: Sprite Editing

### SPR2 Display

IFFSnooper shows:

- P-sprite (visible colors)
- Z-buffer (depth map)
- Alpha channel (transparency)
- All frames in resource
- Large preview of front frame

### Common Frame Sets

| Frames | Typical Use                     |
| ------ | ------------------------------- |
| 3      | Simple objects                  |
| 6      | Large/Medium/Small (front/back) |
| 12     | Extended orientations           |

### Export Sprites

**Menu:** Export → Sprite Images

**Output:**

```
ObjectName_sprites/
├── sprite0000/
│   ├── name_large_ne_p.bmp
│   ├── name_large_ne_z.bmp
│   └── name_large_ne_a.bmp
├── sprite0001/
│   └── ...
```

### Export View Options

| Setting     | Output              |
| ----------- | ------------------- |
| Shadow View | PICT (no extension) |
| TMog View   | BMP files           |

### Complete Sprite Workflow

```
1. Export sprites    ──▶ Export → Sprite Images
         │
2. Edit externally   ──▶ Photoshop, GIMP, etc.
         │
3. Preserve names    ──▶ Keep _p/_z/_a suffix
         │
4. Reimport         ──▶ Import → Sprite Image
         │
5. Verify preview   ──▶ Check display window
         │
6. Encode           ──▶ Export All (creates SPR2+PALT)
         │
7. Add to folder    ──▶ Copy to disassembled resources
         │
8. Reassemble       ──▶ Import → Assemble
```

### Memory Limit

**Constraint:** ~20 imported folders at once  
**Workaround:** Import/encode in batches for large sets

### Common Sprite Tasks

| Task          | Approach                        |
| ------------- | ------------------------------- |
| Recolor       | Edit P-sprite, preserve Z/Alpha |
| Add detail    | Higher quality artwork          |
| Anti-aliasing | Use Alpha values 1-254          |
| Animation     | Create sequential frames        |

---

## Chapter 8: Menu Editing (TTAB)

### Attenuation Settings

| Value  | Effect         |
| ------ | -------------- |
| None   | Always visible |
| Low    | Moderate fade  |
| Medium | Standard fade  |
| High   | Quick fade     |

### Autonomy

| Value | Effect                      |
| ----- | --------------------------- |
| 50%   | Occasional autonomous use   |
| 100%  | Always available when valid |

### Menu Flags

| Flag              | Purpose               |
| ----------------- | --------------------- |
| Visitor Enabled   | Visitors can use      |
| Joinable          | Group activity        |
| Run Immediately   | No queue wait         |
| Allow Consecutive | Repeat action         |
| Child Enabled     | Children can use      |
| Adult Enabled     | Adults can use        |
| Allow Anywhere    | Not location-specific |
| Autofirst         | Priority (v1.2.0+)    |
| Cat Enabled       | Pets (Unleashed+)     |
| Dog Enabled       | Pets (Unleashed+)     |

### Motive Effects

| Motive  | Positive   | Negative       |
| ------- | ---------- | -------------- |
| Hunger  | Satisfies  | Increases need |
| Comfort | Improves   | Worsens        |
| Hygiene | Cleans     | Dirties        |
| Bladder | Relieves   | Increases need |
| Energy  | Rests      | Tires          |
| Fun     | Entertains | Bores          |
| Room    | Improves   | Worsens        |

### Example Configurations

**Eating (solo):**

```
Visitor: Yes    Joinable: No
Child: Yes      Adult: Yes
Autonomy: 100%  Attenuation: Low
Motives: Hunger +80, Energy -20
```

**Kiss (social):**

```
Visitor: Yes    Joinable: Yes
Child: No       Adult: Yes
Autonomy: 50%   Attenuation: High
Motives: Fun +30, Comfort +20
```

### Add Menu Button (v1.2.0+)

Creates:

- New TTAB entry
- Corresponding TTAs entry
- Auto-assigned string index
- BHAV linkage (if provided)

### TTAB Limitations

Cannot edit:

- String table index (auto-linked)
- Joining index (auto-calculated)
- BHAV references (requires OBJf)

---

## Chapter 9: Other Resources

### Hex Data Viewer

Display options:

- Single-byte (8-bit)
- Double-byte (16-bit)
- Decimal representation

Export:

- Export Hex: Raw hex text
- Export Log: Navigation history

### CARR Editing (v1.1.5+)

| Editable   | Description     |
| ---------- | --------------- |
| Track name | Career title    |
| Job titles | Position names  |
| Work hours | Start/end times |
| Salaries   | Pay per level   |
| Car codes  | Vehicle type    |
| Uniform    | CMX/BMP refs    |

### Read-Only Resources

| Resource | Content                    |
| -------- | -------------------------- |
| BCON     | Constants (+ TRCN labels)  |
| BHAV     | VM bytecode                |
| OBJf     | Extended object definition |
| objt     | Object tuning              |
| SLOT     | Routing information        |
| TREE     | Behavior tree              |
| TRPR     | Trait properties           |

### Resource Operations

**Renaming:** Right-click → Rename (all types)

**Renumbering:** Via OBJD tab (auto-syncs OBJf)

---

## Chapter 10: Complete Workflows

### Object Clone Workflow

```
1. Open Source    ──▶ File → Open (find similar object)
         │
2. Clone          ──▶ File → Clone
         │               └── New GUID generated
         │               └── BHAV refs updated (1.1.3+)
         │
3. Edit OBJD      ──▶ Change name, price, categories
         │
4. Edit Text      ──▶ Update CTSS (catalog strings)
         │
5. Edit Sprites   ──▶ Export → edit → reimport
         │
6. Edit Menus     ──▶ Adjust TTAB if needed
         │
7. Reassemble     ──▶ Export/Import cycle
         │
8. Test           ──▶ Place in Downloads, load game
```

### New Sprite Workflow

```
1. Create Art     ──▶ 3D render or 2D paint
         │
2. Create Frames  ──▶ Multiple orientations + zooms
         │
3. Create Z-buf   ──▶ Depth map (darker = nearer)
         │
4. Create Alpha   ──▶ Transparency (white = opaque)
         │
5. Name Files     ──▶ Follow naming convention
         │
6. Import         ──▶ Import → Sprite Image
         │
7. Encode         ──▶ Creates SPR2 + PALT
         │
8. Integrate      ──▶ Add to resource folder
         │
9. Update DGRP    ──▶ Reference new SPR2 frames
```

### Character Edit Workflow

```
1. Open Character ──▶ File → Open (.iff from Characters/)
         │
2. Edit OBJD      ──▶ Basic properties
         │
3. Edit CTSS      ──▶ Name, description
         │
4. Edit Body STR# ──▶ Appearance strings (#200)
         │               └── Configure Skins path first
         │
5. Edit Portraits ──▶ BMP_ resources
         │
6. Save/Test      ──▶ Back up original first!
```

### Career Edit Workflow

```
1. Open Career    ──▶ Careers.iff or work.iff
         │
2. Select CARR    ──▶ Choose career track
         │
3. Edit Details   ──▶ Titles, hours, salaries
         │
4. Edit Uniforms  ──▶ CMX/BMP references
         │
5. Save           ──▶ Back up original first!
```

---

## External Tools Reference

For tasks beyond IFFSnooper capabilities:

| Tool                  | Purpose                | Use For            |
| --------------------- | ---------------------- | ------------------ |
| Transmogrifier (TMog) | Complete object editor | Complex objects    |
| Niotso/FreeSO tools   | BHAV editing           | Behavior code      |
| 7DS tools             | Animation              | Sprite sequences   |
| SimEnhancer           | Character editing      | Sims customization |
| GuidChecker           | GUID verification      | Conflict detection |
| Hex editors           | Raw binary             | Advanced debugging |

---

## Cross-Reference Diagram

```
IFFSNOOPER COMPLETE WORKFLOW

┌─────────────────────────────────────────────────────────────────┐
│                     FILE OPERATIONS                              │
│                                                                  │
│   File → Open ────────────► View Resources                      │
│         │                        │                               │
│         │                        ▼                               │
│   File → Clone ──────────► New GUID + BHAV update               │
│         │                        │                               │
│         │                        ▼                               │
│   Export → Disassemble ──► Resource Folder                      │
│                                  │                               │
│                                  ▼                               │
│                           Edit Resources                         │
│                                  │                               │
│                                  ▼                               │
│   Import → Assemble ◄───── Modified Folder                      │
│         │                                                        │
│         ▼                                                        │
│   Final IFF ─────────────► Test in Game                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   RESOURCE EDITORS                               │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    OBJD     │  │    Text     │  │   Sprite    │             │
│  │   Editor    │  │   Editor    │  │   Editor    │             │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤             │
│  │ GUID        │  │ CTSS        │  │ Export      │             │
│  │ Price       │  │ STR#        │  │ Edit P/Z/A  │             │
│  │ Flags       │  │ TTAs        │  │ Import      │             │
│  │ Multi-tile  │  │ Multi-lang  │  │ Encode      │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Menu     │  │   Career    │  │    Hex      │             │
│  │   Editor    │  │   Editor    │  │   Viewer    │             │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤             │
│  │ TTAB flags  │  │ Job titles  │  │ BHAV (view) │             │
│  │ Motives     │  │ Salaries    │  │ SLOT (view) │             │
│  │ Autonomy    │  │ Uniforms    │  │ OBJf (view) │             │
│  │ TTAs text   │  │ Car codes   │  │ Raw export  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   SPRITE PIPELINE                                │
│                                                                  │
│   Export → Sprite Images                                        │
│            │                                                     │
│            ▼                                                     │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  sprite0000/                                              │  │
│   │  ├── object_large_ne_p.bmp  ◄── Visible colors           │  │
│   │  ├── object_large_ne_z.bmp  ◄── Depth (dark = near)      │  │
│   │  └── object_large_ne_a.bmp  ◄── Transparency             │  │
│   └──────────────────────────────────────────────────────────┘  │
│            │                                                     │
│            ▼  Edit in Photoshop/GIMP                            │
│            │                                                     │
│            ▼                                                     │
│   Import → Sprite Image                                         │
│            │                                                     │
│            ▼                                                     │
│   Export All  ─────────────► Creates SPR2 + PALT                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

VALIDATION CHECKLIST

  □ Unique GUIDs for all OBJD
  □ DGRP IDs match OBJD base_graphic
  □ SPR2 IDs match DGRP references
  □ PALT exists for each color table
  □ CTSS present with correct ID in OBJD
  □ TTAB/TTAs for interactive objects
  □ STR# for animation names if animated
```

---

_Part VI Complete — End of documentation_

---

## Documentation Complete

This concludes the six-part technical documentation for The Sims (2000).

### Quick Reference

| Part                                            | Content                                 |
| ----------------------------------------------- | --------------------------------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture, concepts, getting started |
| [II. Containers](Part_II_Containers.md)         | FAR archives, IFF structure             |
| [III. Object System](Part_III_ObjectSystem.md)  | OBJD, BHAV, VM, interactions            |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, meshes, animation              |
| [V. Game Content](Part_V_GameContent.md)        | Characters, careers, strings            |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | IFFSnooper guides                       |

### Cross-Reference Index

| Topic             | Location                                                                   |
| ----------------- | -------------------------------------------------------------------------- |
| BHAV format       | [Part III Ch.3](Part_III_ObjectSystem.md#chapter-3-bhav-behaviors)         |
| Career tracks     | [Part V Ch.2](Part_V_GameContent.md#chapter-2-career-system-carr)          |
| CFP compression   | [Part IV Ch.6](Part_IV_Graphics.md#chapter-6-cfp-animation-data)           |
| Character bundles | [Part V Ch.1](Part_V_GameContent.md#chapter-1-character-bundles)           |
| CMX/BCF format    | [Part IV Ch.4](Part_IV_Graphics.md#chapter-4-cmxbcf-character-animation)   |
| DGRP format       | [Part IV Ch.1](Part_IV_Graphics.md#chapter-1-dgrp-draw-groups)             |
| FAR archives      | [Part II Ch.1](Part_II_Containers.md#chapter-1-far-archive-format)         |
| GUID editing      | [Part VI Ch.5](Part_VI_Tools.md#chapter-5-guid--magic-cookie)              |
| IFF format        | [Part II Ch.2](Part_II_Containers.md#chapter-2-iff-interchange-format)     |
| Language codes    | [Part V Ch.5](Part_V_GameContent.md#chapter-5-language-codes-reference)    |
| OBJD format       | [Part III Ch.1](Part_III_ObjectSystem.md#chapter-1-objd-object-definition) |
| SKN/BMF meshes    | [Part IV Ch.5](Part_IV_Graphics.md#chapter-5-sknbmf-mesh-format)           |
| SLOT format       | [Part III Ch.8](Part_III_ObjectSystem.md#chapter-8-slot-routing-positions) |
| SPR2 format       | [Part IV Ch.2](Part_IV_Graphics.md#chapter-2-sprite-formats-spr-and-spr2)  |
| Sprite editing    | [Part VI Ch.7](Part_VI_Tools.md#chapter-7-sprite-editing)                  |
| String tables     | [Part V Ch.3](Part_V_GameContent.md#chapter-3-string-tables-str)           |
| TTAB menus        | [Part III Ch.6](Part_III_ObjectSystem.md#chapter-6-ttab-interactions)      |
| Virtual Machine   | [Part III Ch.9](Part_III_ObjectSystem.md#chapter-9-virtual-machine)        |
