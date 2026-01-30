# The Sims (2000) — Technical Documentation

Comprehensive technical reference for The Sims file formats, resource structures, and modding workflows.

---

## 📖 Documentation Structure

| Part                                            | Title                   | Content                                  |
| ----------------------------------------------- | ----------------------- | ---------------------------------------- |
| [I. Foundations](Part_I_Foundations.md)         | Architecture & Concepts | Overview, conventions, getting started   |
| [II. Containers](Part_II_Containers.md)         | File Formats            | FAR archives, IFF structure, preferences |
| [III. Object System](Part_III_ObjectSystem.md)  | Objects & Behaviors     | OBJD, BHAV, VM, interactions             |
| [IV. Graphics & Animation](Part_IV_Graphics.md) | Rendering               | Sprites, meshes, animation system        |
| [V. Game Content](Part_V_GameContent.md)        | Data Structures         | Characters, careers, strings             |
| [VI. Tools & Workflows](Part_VI_Tools.md)       | Practical Guides        | IFFSnooper tutorials                     |

---

## 🎯 Quick Navigation

### By Task

| I want to...                | Go to...                                                                    |
| --------------------------- | --------------------------------------------------------------------------- |
| Understand the architecture | [Part I Ch.2](Part_I_Foundations.md#chapter-2-game-architecture)            |
| Parse a FAR archive         | [Part II Ch.1](Part_II_Containers.md#chapter-1-far-archive-format)          |
| Parse an IFF file           | [Part II Ch.2](Part_II_Containers.md#chapter-2-iff-interchange-file-format) |
| Understand object structure | [Part III Ch.1](Part_III_ObjectSystem.md#chapter-1-objd-object-definition)  |
| Learn BHAV bytecode         | [Part III Ch.3](Part_III_ObjectSystem.md#chapter-3-bhav-behaviors)          |
| Work with sprites           | [Part IV Ch.2](Part_IV_Graphics.md#chapter-2-sprite-formats-spr-and-spr2)   |
| Parse meshes                | [Part IV Ch.5](Part_IV_Graphics.md#chapter-5-sknbmf-mesh-format)            |
| Understand animations       | [Part IV Ch.4](Part_IV_Graphics.md#chapter-4-cmxbcf-character-animation)    |
| Edit characters             | [Part V Ch.1](Part_V_GameContent.md#chapter-1-character-bundles)            |
| Edit careers                | [Part V Ch.2](Part_V_GameContent.md#chapter-2-career-system-carr)           |
| Use IFFSnooper              | [Part VI](Part_VI_Tools.md)                                                 |

### By Format

| Format    | Location                                                                    |
| --------- | --------------------------------------------------------------------------- |
| FAR       | [Part II Ch.1](Part_II_Containers.md#chapter-1-far-archive-format)          |
| IFF       | [Part II Ch.2](Part_II_Containers.md#chapter-2-iff-interchange-file-format) |
| OBJD      | [Part III Ch.1](Part_III_ObjectSystem.md#chapter-1-objd-object-definition)  |
| BHAV      | [Part III Ch.3](Part_III_ObjectSystem.md#chapter-3-bhav-behaviors)          |
| BCON/TRCN | [Part III Ch.4](Part_III_ObjectSystem.md#chapter-4-bcon--trcn-constants)    |
| TTAB      | [Part III Ch.6](Part_III_ObjectSystem.md#chapter-6-ttab-interactions)       |
| SLOT      | [Part III Ch.8](Part_III_ObjectSystem.md#chapter-8-slot-routing-positions)  |
| DGRP      | [Part IV Ch.1](Part_IV_Graphics.md#chapter-1-dgrp-draw-groups)              |
| SPR#/SPR2 | [Part IV Ch.2](Part_IV_Graphics.md#chapter-2-sprite-formats-spr-and-spr2)   |
| PALT      | [Part IV Ch.3](Part_IV_Graphics.md#chapter-3-palt-color-palettes)           |
| CMX/BCF   | [Part IV Ch.4](Part_IV_Graphics.md#chapter-4-cmxbcf-character-animation)    |
| SKN/BMF   | [Part IV Ch.5](Part_IV_Graphics.md#chapter-5-sknbmf-mesh-format)            |
| CFP       | [Part IV Ch.6](Part_IV_Graphics.md#chapter-6-cfp-animation-data)            |
| CARR      | [Part V Ch.2](Part_V_GameContent.md#chapter-2-career-system-carr)           |
| STR#      | [Part V Ch.3](Part_V_GameContent.md#chapter-3-string-tables-str)            |
| FCNS      | [Part V Ch.4](Part_V_GameContent.md#chapter-4-fcns-function-constants)      |

---

## 📊 Overview

**Game:** The Sims (Maxis/EA, 2000)  
**Primary Container:** IFF (Interchange File Format)  
**Archive Format:** FAR (File ARchive)  
**Byte Order:** Big-endian headers, little-endian content

### Key Concepts

- **Objects** are defined by OBJD resources and use BHAV bytecode for behaviors
- **Graphics** use isometric sprites (SPR2) positioned by DGRP resources
- **Characters** are defined by CMX/BCF (skeletons/animations) and SKN/BMF (meshes)
- **Careers** are encoded in CARR resources with bit-packed variable-width fields

---

## 📝 Conventions

| Convention  | Meaning                     |
| ----------- | --------------------------- |
| `0x` prefix | Hexadecimal                 |
| BE          | Big-endian                  |
| LE          | Little-endian               |
| Fields      | Listed offset → size → name |

---


_Documentation revised 2026_
