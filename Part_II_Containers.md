# Part II: Containers & File Structures

> **How The Sims packages and organizes data files**

This part covers the container formats that hold all game data: FAR archives bundle multiple files, IFF containers hold resources, and the resource map enables fast lookup.

---

## Document Navigation

| Part                                            | Focus                        | Status          |
| ----------------------------------------------- | ---------------------------- | --------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture & Concepts      | ← Start here    |
| **II. Containers**                              | FAR & IFF Formats            | 📍 You are here |
| [III. Object System](Part_III_ObjectSystem.md)  | OBJD, BHAV, VM               |                 |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Sprites, Meshes, Animation   |                 |
| [V. Game Content](Part_V_GameContent.md)        | Characters, Careers, Strings |                 |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | IFFSnooper Guides            |                 |

---

## Key Terms

| Term         | Definition                                                       |
| ------------ | ---------------------------------------------------------------- |
| **FAR**      | File Archive — bundles multiple files into single `.far` package |
| **IFF**      | Interchange File Format — container for game resources           |
| **rsmp**     | Resource Map — index enabling fast resource lookup               |
| **Resource** | Individual data unit within IFF (OBJD, BHAV, SPR#, etc.)         |

---

## Chapter 1: FAR Archive Format

### Overview

FAR archives bundle multiple files into a single container for efficient storage and distribution. The format is simple: header → concatenated files → manifest.

| Property    | Value                 |
| ----------- | --------------------- |
| Extension   | `.far`                |
| Byte order  | Little-endian         |
| Compression | None (raw storage)    |
| Max size    | 4 GB (32-bit offsets) |

### File Structure

```
FAR Archive Layout
┌─────────────────────────┐  Offset 0
│ Header (16 bytes)       │
├─────────────────────────┤  Offset 16
│ Archived File 1         │
│ Archived File 2         │
│ ... (concatenated)      │
├─────────────────────────┤  (manifest offset)
│ Manifest                │
└─────────────────────────┘  EOF
```

### Header (16 bytes)

| Offset | Size | Field           | Value                      |
| ------ | ---- | --------------- | -------------------------- |
| 0      | 8    | Signature       | `FAR!byAZ` (ASCII literal) |
| 8      | 4    | Version         | `1`                        |
| 12     | 4    | Manifest offset | Byte position of manifest  |

### Manifest Structure

```
Offset  Size   Field
0       4      File count (N)
4       var    Entry 1
...     var    Entry 2..N
```

### Manifest Entry

| Offset | Size | Field           | Description                     |
| ------ | ---- | --------------- | ------------------------------- |
| 0      | 4    | File length 1   | Size in bytes                   |
| 4      | 4    | File length 2   | Duplicate (keep identical)      |
| 8      | 4    | File offset     | Byte offset from FAR start      |
| 12     | 4    | Filename length | Bytes in filename               |
| 16     | var  | Filename        | UTF-8/ASCII, no null terminator |

**Entry size:** `16 + filename_length` bytes

💡 **Duplicate Length Fields:** Both length fields are always identical. They may have been reserved for future compressed/uncompressed size support. Keep them identical for compatibility.

### Implementation Notes

- Archived files are concatenated sequentially without padding
- All observed files are multiples of 4 bytes (possible alignment requirement)
- Entry order matches archived file order
- No compression — FAR stores raw file data

### Example: Reading a FAR Archive

```
1. Read header (16 bytes)
2. Verify signature = "FAR!byAZ"
3. Seek to manifest_offset
4. Read file_count
5. For each entry:
   a. Read file_length, file_offset, filename
   b. Seek to file_offset
   c. Read file_length bytes
```

---

## Chapter 2: IFF Interchange File Format

### Overview

IFF (Interchange File Format) is the primary container for all game object data. Each IFF file contains multiple resources — the actual data units that define objects, behaviors, sprites, and more.

| Property         | Value                                        |
| ---------------- | -------------------------------------------- |
| Extensions       | `.iff`, `.flr`, `.wll`, `.spf`, `.stx`       |
| Byte order       | **Big-endian** (header and resource headers) |
| Resource content | Little-endian (inside resources)             |
| Max size         | 4 GB                                         |

⚠️ **Critical:** IFF uses big-endian for the container format, but resource _contents_ use little-endian. This is a common source of parsing errors.

### IFF Extension Variants

| Extension | Specialization | Typical Contents              |
| --------- | -------------- | ----------------------------- |
| `.iff`    | Standard       | Objects, characters, controls |
| `.flr`    | Floors         | Floor sprites, catalog data   |
| `.wll`    | Walls          | Wall sprites, catalog data    |
| `.spf`    | Sprites        | Graphics separated from logic |
| `.stx`    | Strings        | Localization text             |

### File Structure

```
IFF File Layout
┌─────────────────────────┐  Offset 0
│ Header (64 bytes)       │
├─────────────────────────┤  Offset 64
│ Resource 1              │
│ Resource 2              │
│ ... (concatenated)      │
└─────────────────────────┘  EOF
```

### IFF Header (64 bytes)

| Offset | Size | Field       | Description                         |
| ------ | ---- | ----------- | ----------------------------------- |
| 0      | 60   | Signature   | Two validation strings (see below)  |
| 60     | 4    | rsmp offset | Byte offset to resource map, or `0` |

**Signature (60 bytes):**

```
IFF FILE 2.5:TYPE FOLLOWED BY SIZE\0 JAMIE DOORNBOS & MAXIS 1
```

The game validates all 60 bytes. Invalid signature = file rejected.

**rsmp Offset:**

- Non-zero: Points to rsmp resource for fast lookup
- Zero: No resource map (parse sequentially)

### Resource Structure

Each resource in an IFF file has a 76-byte header followed by content:

```
Resource Layout
┌─────────────────────────┐
│ Header (76 bytes)       │
├─────────────────────────┤
│ Content (size - 76)     │
└─────────────────────────┘
```

### Resource Header (76 bytes)

| Offset | Size | Field     | Description                              |
| ------ | ---- | --------- | ---------------------------------------- |
| 0      | 4    | Type code | 4-char identifier (`OBJD`, `BHAV`, etc.) |
| 4      | 4    | Size      | Total bytes (header + content)           |
| 8      | 2    | ID        | Unique per type within file (0–65535)    |
| 10     | 2    | Flags     | `0x0000` or `0x0010` (usage unclear)     |
| 12     | 64   | Name      | Human-readable, null-padded (optional)   |

**Type codes** are 4 ASCII characters: `STR#`, `OBJD`, `BHAV`, `SPR2`, etc.

**Resource ID** must be unique per type within a file. You can have OBJD #128 and STR# #128 in the same file.

### Parsing IFF Files

**With rsmp (fast lookup):**

```
1. Read 64-byte header
2. Seek to rsmp_offset
3. Parse resource map
4. Jump directly to needed resources
```

**Without rsmp (sequential):**

```
1. Read 64-byte header
2. At offset 64, read resource header
3. Skip (size - 76) bytes to next resource
4. Repeat until EOF
```

### Common Resource Types

| Type   | Name              | Documented In                                                                |
| ------ | ----------------- | ---------------------------------------------------------------------------- |
| `OBJD` | Object Definition | [Part III, Ch. 1](Part_III_ObjectSystem.md#chapter-1-objd-object-definition) |
| `BHAV` | Behavior          | [Part III, Ch. 3](Part_III_ObjectSystem.md#chapter-3-bhav-behaviors)         |
| `STR#` | String Table      | [Part V, Ch. 3](Part_V_GameContent.md#chapter-3-string-tables-str)           |
| `SPR2` | Sprite            | [Part IV, Ch. 2](Part_IV_Graphics.md#chapter-2-sprite-formats-spr-and-spr2)  |
| `DGRP` | Draw Group        | [Part IV, Ch. 1](Part_IV_Graphics.md#chapter-1-dgrp-draw-groups)             |
| `rsmp` | Resource Map      | [Chapter 3](#chapter-3-rsmp-resource-map)                                    |

---

## Chapter 3: rsmp Resource Map

### Overview

The rsmp resource provides a table of contents for fast resource lookup within an IFF file. It indexes all other resources by type and ID.

| Property  | Value                           |
| --------- | ------------------------------- |
| Type code | `rsmp`                          |
| Required  | No (IFF works without it)       |
| Benefit   | Fast random access to resources |

### Header (20 bytes)

| Offset | Size | Field      | Value                            |
| ------ | ---- | ---------- | -------------------------------- |
| 0      | 4    | Reserved   | `0`                              |
| 4      | 4    | Version    | `0` (original) or `1` (TSO)      |
| 8      | 4    | Identifier | `rsmp` (little-endian)           |
| 12     | 4    | Size       | Total rsmp size, or `0`          |
| 16     | 4    | Type count | Number of resource types indexed |

### Type List Structure

For each resource type in the file:

**Type Header (8 bytes):**

| Offset | Size | Field                            |
| ------ | ---- | -------------------------------- |
| 0      | 4    | Type code (`OBJD`, `STR#`, etc.) |
| 4      | 4    | Entry count                      |

**List Entry (Version 0):**

| Offset | Size | Field                  |
| ------ | ---- | ---------------------- |
| 0      | 4    | File offset            |
| 4      | 2    | Resource ID            |
| 6      | 2    | Flags                  |
| 8      | var  | Name (null-terminated) |

**List Entry (Version 1 — TSO):**

| Offset | Size | Field                  |
| ------ | ---- | ---------------------- |
| 0      | 4    | File offset            |
| 4      | 4    | Resource ID (extended) |
| 8      | 2    | Flags                  |
| 10     | var  | Name (Pascal string)   |

### Excluded Resources

The rsmp does not index:

- Itself (would be circular)
- `XXXX` filler pseudo-resources

### Usage Pattern

```
Fast Lookup:
1. Parse rsmp once at file open
2. Build type → ID → offset map
3. To read OBJD #128:
   - Look up offset in map
   - Seek directly to offset
   - Read resource

Sequential (no rsmp):
1. Read each resource header
2. Check if type/ID matches
3. Continue until found or EOF
```

---

## Chapter 4: BMP Bitmap Format

### Overview

BMP (Windows Bitmap) is the standard format for texture and sprite source images. This is not Sims-specific, but included as reference since many tools export/import BMP.

| Property   | Value         |
| ---------- | ------------- |
| Extension  | `.bmp`        |
| Byte order | Little-endian |
| Standard   | Windows DIB   |

### File Structure

```
BMP File Layout
┌─────────────────────────┐
│ BITMAPFILEHEADER (14B)  │
├─────────────────────────┤
│ BITMAPINFOHEADER (40B)  │
├─────────────────────────┤
│ Color Table (optional)  │
├─────────────────────────┤
│ Pixel Data              │
└─────────────────────────┘
```

### Color Depths

| Bits/Pixel | Colors             | Color Table       |
| ---------- | ------------------ | ----------------- |
| 1          | 2 (monochrome)     | Yes (2 entries)   |
| 4          | 16                 | Yes (16 entries)  |
| 8          | 256                | Yes (256 entries) |
| 24         | 16.7M (true color) | No                |

### Pixel Data Organization

| Property       | Value                          |
| -------------- | ------------------------------ |
| Scan direction | Left to right                  |
| Row order      | **Bottom to top**              |
| Row padding    | Zero-padded to 32-bit boundary |
| First byte     | Lower-left pixel               |
| Last byte      | Upper-right pixel              |

⚠️ **Bottom-up storage:** BMP stores rows from bottom to top. The first pixel data is the lower-left corner.

### Compression Types

| Constant  | Value | Description               |
| --------- | ----- | ------------------------- |
| `BI_RGB`  | 0     | Uncompressed              |
| `BI_RLE8` | 1     | 8-bit run-length encoding |
| `BI_RLE4` | 2     | 4-bit run-length encoding |

---

## Chapter 5: Preferences File

### Overview

A minimal 4-byte binary file storing initial game startup configuration.

| Property | Value                                                         |
| -------- | ------------------------------------------------------------- |
| Filename | Language-dependent ("The Sims Prefs", "Die Sims Prefs", etc.) |
| Location | System preferences folder                                     |
| Size     | 4 bytes                                                       |

### Structure (4 bytes)

| Offset | Field       | Values                                                                       |
| ------ | ----------- | ---------------------------------------------------------------------------- |
| 0      | Show dialog | `0` = skip, `1` = show preferences at startup                                |
| 1      | Resolution  | `0` = 800×600, `1` = 1024×768                                                |
| 2      | Language    | See [Part V, Language Codes](Part_V_GameContent.md#language-codes-reference) |
| 3      | Unknown     | No observed effect                                                           |

💡 **Extended Settings:** Runtime options are stored separately in `UserData/Options.iff`.

---

## Cross-Reference Diagram

```
FILE CONTAINER HIERARCHY

┌─────────────────────────────────────────────────────────────┐
│                     FAR ARCHIVE                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Header: "FAR!byAZ" │ Version │ Manifest Offset      │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │                                                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────┐   │    │
│  │  │  IFF File   │  │  IFF File   │  │  IFF File │   │    │
│  │  │  (object)   │  │  (object)   │  │  (global) │   │    │
│  │  └─────────────┘  └─────────────┘  └───────────┘   │    │
│  │                                                     │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │ Manifest: filename → offset → length                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      IFF FILE                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Header (64 bytes): Signature │ rsmp Offset          │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │    │
│  │  │ Resource │  │ Resource │  │ Resource │   ...   │    │
│  │  │  OBJD    │  │  BHAV    │  │  SPR2    │         │    │
│  │  │  #128    │  │  #4096   │  │  #200    │         │    │
│  │  └──────────┘  └──────────┘  └──────────┘         │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐  │    │
│  │  │ rsmp: Type → ID → Offset (fast lookup)       │  │    │
│  │  └──────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      IFF RESOURCE                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Header (76 bytes)                                   │    │
│  │  Type: "OBJD"  Size: 292  ID: 128  Name: "Chair"   │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │ Content (Size - 76 bytes)                          │    │
│  │  [Resource-specific binary data]                   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Format   | Purpose          | Byte Order           | Key Feature              |
| -------- | ---------------- | -------------------- | ------------------------ |
| **FAR**  | Bundle files     | Little-endian        | Manifest at end          |
| **IFF**  | Bundle resources | Big-endian (headers) | 76-byte resource headers |
| **rsmp** | Index resources  | Little-endian        | Fast lookup table        |
| **BMP**  | Bitmap images    | Little-endian        | Standard Windows format  |

---

_Part II Complete — Continue to [Part III: Object System](Part_III_ObjectSystem.md)_
