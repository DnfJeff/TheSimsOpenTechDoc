# Part IV: Graphics & Animation

> **How The Sims renders objects and animates characters**

This part covers the complete graphics pipeline: sprite compositing, 3D meshes, skeletal animation, and compressed animation data.

---

## Document Navigation

| Part                                           | Focus                        | Status          |
| ---------------------------------------------- | ---------------------------- | --------------- |
| [I. Foundations](Part_I_Foundations.md)        | Architecture & Concepts      |                 |
| [II. Containers](Part_II_Containers.md)        | FAR & IFF Formats            |                 |
| [III. Object System](Part_III_ObjectSystem.md) | OBJD, BHAV, VM               |                 |
| **IV. Graphics & Animation**                   | Sprites, Meshes, Animation   | 📍 You are here |
| [V. Game Content](Part_V_GameContent.md)       | Characters, Careers, Strings |                 |
| [VI. Tools & Workflows](Part_VI_Tools.md)      | IFFSnooper Guides            |                 |

---

## Key Terms

| Term         | Definition                                                  |
| ------------ | ----------------------------------------------------------- |
| **DGRP**     | Draw Group — sprite compositor for object rendering         |
| **SPR#**     | Sprite (Type A) — 8-bit indexed sprite format               |
| **SPR2**     | Sprite (Type B) — multi-channel sprite with depth/alpha     |
| **PALT**     | Palette — 256-color lookup table                            |
| **CMX**      | Character animation file (ASCII) — skeletons, suits, skills |
| **BCF**      | Binary CMX Format — compiled animation data                 |
| **SKN**      | Skin mesh file (ASCII) — 3D deformable mesh                 |
| **BMF**      | Binary Mesh Format — compiled mesh data                     |
| **CFP**      | Compressed Floating Point — animation frame data            |
| **Z-Buffer** | Depth channel determining sprite layering                   |
| **Alpha**    | Transparency channel (0=transparent, 255=opaque)            |

---

## Chapter 1: DGRP Draw Groups

### Overview

DGRP (Draw Group) combines multiple sprites into composite images, defining how objects appear from different directions and zoom levels.

| Property  | Value                              |
| --------- | ---------------------------------- |
| Type code | `DGRP`                             |
| Entries   | Always 12 (4 directions × 3 zooms) |
| Versions  | 20000–20004                        |

### Structure

```text
DGRP Layout
┌─────────────────────────┐
│ Header (version + data) │
├─────────────────────────┤
│ Image Entry 1           │
│ Image Entry 2           │
│ ... (12 entries)        │
└─────────────────────────┘
```

### Direction Flags

| Value | Direction   |
| ----- | ----------- |
| 0x01  | Right rear  |
| 0x02  | Right       |
| 0x04  | Right front |
| 0x08  | Front       |
| 0x10  | Left front  |
| 0x20  | Left        |
| 0x40  | Left rear   |
| 0x80  | Rear        |

### Zoom Factors

| Value | Scale        | Description   |
| ----- | ------------ | ------------- |
| 1     | Furthest     | 50% of zoom 2 |
| 2     | Intermediate | 50% of zoom 3 |
| 3     | Closest      | Full detail   |

### DGRP Image Entry

| Field       | V.0-V.1 | V.3-V.4 | Description          |
| ----------- | ------- | ------- | -------------------- |
| Count       | 2       | 4       | Sprite count         |
| Direction   | 1       | 4       | View direction flags |
| Zoom        | 1       | 4       | Zoom level (1-3)     |
| Sprite info | 12-16   | 24-32   | Per-sprite data      |

### Sprite Info Entry

| Field     | V.0-V.1 | V.3-V.4 | Description           |
| --------- | ------- | ------- | --------------------- |
| Tag       | 2       | —       | -3 (normal) or -5     |
| SPR ID    | 2       | 4       | Sprite resource ID    |
| SPR frame | 2       | 4       | Frame within sprite   |
| Flags     | 2       | 4       | Rendering flags       |
| Pixel X   | 2       | 4       | Screen X offset       |
| Pixel Y   | 2       | 4       | Screen Y offset       |
| Z offset  | —       | 4       | World Z (V.1+, float) |
| X offset  | —       | 4       | World X (V.4, float)  |
| Y offset  | —       | 4       | World Y (V.4, float)  |

### Sprite Flags

| Bit | Mask | Purpose                        |
| --- | ---- | ------------------------------ |
| 1   | 0x01 | Mirror horizontally            |
| 3   | 0x04 | Flicker/animate (fire, lights) |

### Multi-Tile Objects

Multi-tile objects use one DGRP per tile. Tile sprites align seamlessly across object surfaces.

---

## Chapter 2: Sprite Formats (SPR# and SPR2)

### SPR# (8-bit Sprites)

Legacy format for simple objects — walls, floors, thought balloons.

| Property    | Value          |
| ----------- | -------------- |
| Type code   | `SPR#`         |
| Color depth | 8-bit indexed  |
| Channels    | 1 (color only) |
| Versions    | 502–505        |

**Header (12+ bytes):**

| Offset | Size | Field              |
| ------ | ---- | ------------------ |
| 0      | 4    | Version            |
| 4      | 4    | Frame count        |
| 8      | 4    | Palette ID         |
| 12     | N×4  | Frame offset table |

**Byte Order Detection:**

- First two bytes = `0x00 0x00` → Big-endian
- Otherwise → Little-endian

**Row Encoding:**

| Code | Meaning                        |
| ---- | ------------------------------ |
| 0x09 | Fill N rows with background    |
| 0x04 | N bytes of compressed row data |

**Segment Codes:**

| Code | Data    | Effect                         |
| ---- | ------- | ------------------------------ |
| 1    | None    | Fill N pixels transparent      |
| 2    | 2 bytes | Fill N pixels with color (RLE) |
| 3    | N bytes | Copy N pixels verbatim         |

### SPR2 (Multi-channel Sprites)

Extended format supporting depth and transparency.

| Property  | Value         |
| --------- | ------------- |
| Type code | `SPR2`        |
| Version   | 1000 (always) |
| Channels  | Up to 3       |

**Channels:**

| Channel          | Purpose        | Values                    |
| ---------------- | -------------- | ------------------------- |
| Color            | Palette index  | 0–255                     |
| Depth (Z-buffer) | Layer ordering | 0=far, 255=near           |
| Alpha            | Transparency   | 0=transparent, 255=opaque |

**Header (12 bytes):**

| Offset | Size | Field          |
| ------ | ---- | -------------- |
| 0      | 4    | Version (1000) |
| 4      | 4    | Frame count    |
| 8      | 4    | Palette ID     |

**Frame Header (16 bytes):**

| Offset | Size | Field                    |
| ------ | ---- | ------------------------ |
| 0      | 2    | Width                    |
| 2      | 2    | Height                   |
| 4      | 2    | Flags (channel encoding) |
| 6      | 2    | Reserved (0)             |
| 8      | 2    | Palette ID (override)    |
| 10     | 2    | Transparent pixel index  |
| 12     | 2    | Y location               |
| 14     | 2    | X location               |

**Channel Flags:**

| Value | Channels              |
| ----- | --------------------- |
| 0x01  | Color only            |
| 0x03  | Color + Depth         |
| 0x07  | Color + Depth + Alpha |

**Segment Codes:**

| Code | Bytes/Pixel | Content               |
| ---- | ----------- | --------------------- |
| 1    | 2           | Depth + Color         |
| 2    | 3           | Depth + Color + Alpha |
| 3    | 0           | Transparent fill      |
| 6    | 1           | Color only            |

### SPR# vs SPR2 Comparison

| Feature      | SPR#        | SPR2          |
| ------------ | ----------- | ------------- |
| Channels     | 1           | 3             |
| Translucency | None        | Full alpha    |
| Z-buffer     | None        | Depth channel |
| Use case     | Simple/flat | 3D effects    |

---

## Chapter 3: PALT Color Palettes

### PALT Overview

PALT translates 8-bit sprite indices to 24-bit RGB colors.

| Property   | Value         |
| ---------- | ------------- |
| Type code  | `PALT`        |
| Entries    | 256 (always)  |
| Entry size | 3 bytes (RGB) |
| Total size | 784 bytes     |

### PALT Structure

**Header (16 bytes):**

| Offset | Size | Field             |
| ------ | ---- | ----------------- |
| 0      | 4    | Version/count (1) |
| 4      | 4    | Entry count (256) |
| 8      | 8    | Reserved (0)      |

**Color Table (768 bytes):**

256 RGB triplets starting at offset 16:

| Byte | Component     |
| ---- | ------------- |
| 0    | Red (0–255)   |
| 1    | Green (0–255) |
| 2    | Blue (0–255)  |

### Usage

```text
Sprite Rendering:
1. Read pixel value (0-255) from SPR#/SPR2
2. Look up PALT[pixel_value]
3. Get (R, G, B) color
4. Apply to screen
```

---

## Chapter 4: CMX/BCF Character Animation

### CMX/BCF Overview

CMX (text) and BCF (binary) define character animation: skeletons, suits, and skills.

| Format | Extension  | Type   | Purpose          |
| ------ | ---------- | ------ | ---------------- |
| CMX    | `.cmx`     | ASCII  | Source/editable  |
| BCF    | `.cmx.bcf` | Binary | Compiled/runtime |

CMX is authoritative; BCF is generated from CMX.

### Three Data Sections

| Section       | Purpose        | Content                            |
| ------------- | -------------- | ---------------------------------- |
| **Skeletons** | Body structure | Bone names, hierarchy, orientation |
| **Suits**     | Visual mapping | Skeleton bones → mesh coverage     |
| **Skills**    | Animation      | Bone motion sequences with timing  |

💡 Each BCF file contains exactly ONE non-zero section.

### Binary Data Types (BCF)

| Type    | Encoding                           |
| ------- | ---------------------------------- |
| Integer | Little-endian                      |
| Float   | IEEE 754, little-endian            |
| String  | Pascal-style (length byte + chars) |

### CMX Text Format

- Lines terminated by `<CR><LF>` (DOS)
- Numbers as decimal, whitespace-separated
- Strings on own line, EOL-terminated

### BCF File Structure

```text
Offset  Size  Description
0       4     Skeleton count
4       var   Skeleton data
var     4     Suit count
var     var   Suit data
var     4     Skill count
var     var   Skill data
```

### Skeleton Structure

| Field      | Size | Description            |
| ---------- | ---- | ---------------------- |
| Name       | var  | Skeleton name (string) |
| Bone count | 4    | Number of bones        |
| Bones      | var  | Bone data (repeated)   |

**Standard Skeletons:**

| Skeleton | Bones | Used For   |
| -------- | ----- | ---------- |
| Adult    | 29    | Adult Sims |
| Child    | 29    | Child Sims |
| Dog      | 37    | Pet dogs   |
| Cat      | 40    | Pet cats   |

**Bone Fields:**

| Field         | Size | Description             |
| ------------- | ---- | ----------------------- |
| Name          | var  | Bone identifier         |
| Parent        | var  | Parent bone name        |
| Position      | 12   | X, Y, Z floats          |
| Rotation      | 16   | Quaternion (W, X, Y, Z) |
| Can-translate | 4    | Boolean flag            |
| Can-rotate    | 4    | Boolean flag            |
| Wiggle value  | 4    | Animation variance      |

### Suit Structure

Associates skeleton bones with mesh/clothing:

| Field      | Size | Description          |
| ---------- | ---- | -------------------- |
| Name       | var  | Suit identifier      |
| Bone count | 4    | Number of bindings   |
| Bindings   | var  | Bone → skin mappings |

### Skill Structure (Animation)

| Field        | Size | Description            |
| ------------ | ---- | ---------------------- |
| Name         | var  | Animation identifier   |
| Duration     | 4    | Milliseconds           |
| Motion count | 4    | Number of bone motions |
| Motions      | var  | Motion data            |

**Motion Fields:**

| Field           | Size | Description          |
| --------------- | ---- | -------------------- |
| Bone name       | var  | Target bone          |
| Frame count     | 4    | Animation frames     |
| Duration        | 4    | Motion duration      |
| Position offset | 4    | Index into CFP array |
| Rotation offset | 4    | Index into CFP array |

### Time Properties and Events

Skills can trigger events during playback:

| Event           | Values  | Purpose          |
| --------------- | ------- | ---------------- |
| `xevt`          | 0–305   | VM callback      |
| `footstep`      | -2 to 2 | Sound intensity  |
| `anchor`        | 0–2     | Foot stickiness  |
| `interruptable` | 0–2     | Abort difficulty |
| `sound`         | `.vox`  | Sound emission   |

---

## Chapter 5: SKN/BMF Mesh Format

### SKN/BMF Overview

SKN (text) and BMF (binary) define deformable 3D meshes for character rendering.

| Format | Extension | Type   |
| ------ | --------- | ------ |
| SKN    | `.skn`    | ASCII  |
| BMF    | `.bmf`    | Binary |

### File Sections

```text
SKN/BMF Structure:
1. FILENAMES    - Mesh and texture names
2. BONE         - Skeleton bone list
3. FACES        - Triangle vertex indices
4. BONEBINDINGS - Vertex → bone assignments
5. TEXTUREVERTICES - UV mappings
6. BLENDDATA    - Vertex blending info
7. VERTICES     - Position and normal data
```

### Binary Types (BMF)

| Type    | Encoding                |
| ------- | ----------------------- |
| Integer | Little-endian           |
| Float   | IEEE 754, little-endian |
| String  | Pascal-style            |

### Filenames Section

```text
mesh_name      ; Mesh identifier
texture_name   ; Bitmap texture
```

### Bone Section

List of bone names from skeleton (order significant for indices).

### Faces Section

| Field     | Size    | Description         |
| --------- | ------- | ------------------- |
| Count     | 4       | Number of triangles |
| Face data | 12 each | 3 vertex indices    |

Vertex order is counter-clockwise (determines face normal).

### Bone Bindings

| Field         | Size | Description            |
| ------------- | ---- | ---------------------- |
| Bone index    | 4    | Reference to bone      |
| First vertex  | 4    | Raw vertex start       |
| Vertex count  | 4    | Raw vertices bound     |
| First blended | 4    | Blended vertex start   |
| Blended count | 4    | Blended vertices bound |

### Texture Vertices (UV)

| Field | Size | Description                      |
| ----- | ---- | -------------------------------- |
| U     | 4    | Horizontal texture coord (float) |
| V     | 4    | Vertical texture coord (float)   |

1:1 correspondence with raw vertices.

### Blend Data

| Field         | Size | Description              |
| ------------- | ---- | ------------------------ |
| Weight        | 4    | Blend factor (float)     |
| Vertex offset | 4    | Reference offset (float) |

Enables smooth transitions across bone boundaries.

### Vertices

| Field      | Size | Description              |
| ---------- | ---- | ------------------------ |
| X, Y, Z    | 12   | Position (3 floats)      |
| NX, NY, NZ | 12   | Normal vector (3 floats) |

**Invariant:** Total vertices = texture vertices + blended vertices

### Rendering Pipeline

```text
1. Load SKN mesh definition
2. Load texture from FILENAMES
3. For each animation frame:
   a. Load skeleton positions (CMX/BCF)
   b. Transform bones relative to root
   c. Apply bone matrix to bound vertices
   d. Apply blend adjustments
   e. Render triangles with UV mapping
   f. Sample texture; shade by normals
```

---

## Chapter 6: CFP Animation Data

### CFP Overview

CFP stores compressed floating-point animation data — positions and rotations for each frame.

| Property    | Value                         |
| ----------- | ----------------------------- |
| Extension   | `.cfp`                        |
| Compression | ~7:1 ratio                    |
| Type        | Lossy (3+ significant digits) |

### Data Organization

```text
CFP Layout (per bone motion):
┌─────────────────────────┐
│ Offset X (all frames)   │
│ Offset Y (all frames)   │
│ Offset Z (all frames)   │
├─────────────────────────┤
│ Rotation W (all frames) │
│ Rotation X (all frames) │
│ Rotation Y (all frames) │
│ Rotation Z (all frames) │
└─────────────────────────┘
```

Grouping same components together enables efficient processing.

### Compression Encodings

**1. Uncompressed Float (0xFF marker):**

| Offset | Size | Description                |
| ------ | ---- | -------------------------- |
| 0      | 1    | 0xFF marker                |
| 1      | 4    | IEEE float (little-endian) |

**2. Repeat Previous (0xFE marker):**

| Offset | Size | Description                  |
| ------ | ---- | ---------------------------- |
| 0      | 1    | 0xFE marker                  |
| 1      | 2    | Repeat count (little-endian) |

Repeats previous value N+1 times. Used for stationary positions.

**3. Delta Encoding (0x00–0x79, 0x85–0xFC):**

```text
new_value = previous_value + delta_table[code]
```

| Code Range | Purpose         |
| ---------- | --------------- |
| 0–119      | Negative deltas |
| 133–252    | Positive deltas |

**Table Symmetry:** `table[i] ≈ -table[252-i]`

### Delta Formula (Dave Baum)

$$f(x) = 3.9676 \times 10^{-10} \times (x-126)^3 \times |x-126|$$

### Creating New Animations

**Simple Strategy (larger files, but compatible):**

1. Use 0xFF for all float values
2. Apply 0xFE repeats where values don't change
3. Skip delta encoding (incomplete research)

**Trade-off:** Several times optimal size, but functional.

### Validation

Rotation quaternions satisfy: $w^2 + x^2 + y^2 + z^2 = 1$

If one component is missing, calculate: $w = \sqrt{1 - x^2 - y^2 - z^2}$

---

## Cross-Reference Diagram

```text
GRAPHICS PIPELINE

┌─────────────────────────────────────────────────────────────┐
│                    OBJECT RENDERING                          │
│                                                             │
│  OBJD ──▶ base_graphic ──▶ DGRP ──▶ SPR2 ──▶ PALT         │
│                              │                              │
│                              ▼                              │
│                    ┌─────────────────┐                      │
│                    │ Screen Display  │                      │
│                    │ • Position      │                      │
│                    │ • Z-ordering    │                      │
│                    │ • Transparency  │                      │
│                    └─────────────────┘                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  CHARACTER RENDERING                         │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ CMX/BCF │───▶│ SKN/BMF │───▶│   CFP   │                 │
│  │Skeleton │    │  Mesh   │    │Animation│                 │
│  └─────────┘    └─────────┘    └─────────┘                 │
│       │              │              │                       │
│       ▼              ▼              ▼                       │
│  ┌──────────────────────────────────────────┐              │
│  │          CHARACTER DISPLAY               │              │
│  │  1. Load skeleton (bone hierarchy)       │              │
│  │  2. Bind mesh vertices to bones          │              │
│  │  3. For each frame:                      │              │
│  │     a. Read positions/rotations from CFP │              │
│  │     b. Transform bone matrices           │              │
│  │     c. Deform mesh vertices              │              │
│  │     d. Apply texture, render triangles   │              │
│  └──────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘

SPRITE COMPOSITION

┌─────────────────────────────────────────────────────────────┐
│                         DGRP                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 12 Entries (4 directions × 3 zoom levels)           │    │
│  │                                                     │    │
│  │  Direction: NE  │  Direction: SE  │ ...            │    │
│  │  ├── Zoom 1     │  ├── Zoom 1     │                │    │
│  │  ├── Zoom 2     │  ├── Zoom 2     │                │    │
│  │  └── Zoom 3     │  └── Zoom 3     │                │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    SPR2 Frame                        │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │    │
│  │  │   P-Sprite  │  │  Z-Buffer   │  │    Alpha    │  │    │
│  │  │   (Color)   │  │   (Depth)   │  │  (Transp.)  │  │    │
│  │  │ PALT index  │  │ 0=far       │  │ 0=clear     │  │    │
│  │  │  → RGB      │  │ 255=near    │  │ 255=solid   │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

FILE FORMAT RELATIONSHIPS

    ASCII (Editable)              Binary (Runtime)
    ┌─────────┐                   ┌─────────┐
    │   CMX   │ ──── compile ────▶│   BCF   │
    │skeleton │                   │compiled │
    └─────────┘                   └─────────┘

    ┌─────────┐                   ┌─────────┐
    │   SKN   │ ──── compile ────▶│   BMF   │
    │  mesh   │                   │compiled │
    └─────────┘                   └─────────┘

                                  ┌─────────┐
                                  │   CFP   │
                                  │animation│
                                  │ frames  │
                                  └─────────┘
```

---

## Summary

| Component   | Format        | Purpose                               |
| ----------- | ------------- | ------------------------------------- |
| **DGRP**    | IFF resource  | Composite sprites by direction/zoom   |
| **SPR#**    | IFF resource  | 8-bit indexed sprites                 |
| **SPR2**    | IFF resource  | Multi-channel (color/depth/alpha)     |
| **PALT**    | IFF resource  | 256-color lookup table                |
| **CMX/BCF** | External file | Skeleton, suit, animation definitions |
| **SKN/BMF** | External file | Deformable 3D mesh                    |
| **CFP**     | External file | Compressed animation frames           |

---

_Part IV Complete — Continue to [Part V: Game Content](Part_V_GameContent.md)_
