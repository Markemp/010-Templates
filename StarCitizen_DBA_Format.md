# Star Citizen DBA Animation Library Format

## Overview

Star Citizen uses a custom animation format based on CryEngine, wrapped in an `#ivo` container. DBA (Database Animation) files contain one or more animations bundled together for efficient loading.

**Files analyzed:**
- `BEHR_LaserCannon_S2.dba` - 1 animation, 1 controller (264 bytes)
- `BEHR_LaserCannon_S3.dba` - 1 animation, 1 controller (264 bytes)
- `NOVP.dba` - 1 animation, 4 controllers (448 bytes)
- `SuckerPunch.dba` - 2 animations with multiple controllers (768 bytes)

---

## File Structure

```
#ivo Container
├── Header (16 bytes)
├── Chunk Table (2 × 16 bytes)
├── Chunk 0: DBA Data (0x194FBC50)
│   ├── Chunk size (4 bytes)
│   ├── #dba Block 1
│   │   ├── Header (12 bytes)
│   │   ├── Controller hashes (4 × numControllers)
│   │   ├── Controller entries (24 × numControllers)
│   │   └── Keyframe data
│   ├── #dba Block 2 (if multiple animations)
│   └── ...
└── Chunk 1: Animation Info (0xF7351608)
    ├── Number of animations (4 bytes)
    ├── AnimInfo entries (44 × numAnimations)
    └── Path strings (null-terminated)
```

---

## #ivo Container Header (16 bytes)

| Offset | Size | Type     | Description              | Example Value |
|--------|------|----------|--------------------------|---------------|
| 0x00   | 4    | char[4]  | Signature                | "#ivo"        |
| 0x04   | 2    | uint16   | Version                  | 0x0900        |
| 0x06   | 2    | uint16   | Reserved                 | 0x0000        |
| 0x08   | 4    | uint32   | Number of chunks         | 2             |
| 0x0C   | 4    | uint32   | Chunk table offset       | 0x10          |

## Chunk Table Entry (16 bytes each)

| Offset | Size | Type   | Description            |
|--------|------|--------|------------------------|
| 0x00   | 4    | uint32 | Chunk ID               |
| 0x04   | 2    | uint16 | Version                |
| 0x06   | 2    | uint16 | Reserved               |
| 0x08   | 4    | uint32 | Offset from file start |
| 0x0C   | 4    | uint32 | Reserved               |

### Known Chunk IDs

| Chunk ID     | Version | Description        |
|--------------|---------|-------------------|
| 0x194FBC50   | 0x0900  | DBA Data          |
| 0xF7351608   | 0x0901  | Animation Info    |

---

## Chunk 0: DBA Data (0x194FBC50)

### DBA Data Header

| Offset | Size | Type   | Description        |
|--------|------|--------|--------------------|
| 0x00   | 4    | uint32 | Total chunk size   |

After the chunk size, one or more `#dba` blocks follow sequentially.

### #dba Block Header (12 bytes)

| Offset | Size | Type     | Description                    |
|--------|------|----------|--------------------------------|
| 0x00   | 4    | char[4]  | Signature "#dba"               |
| 0x04   | 2    | uint16   | Number of controllers (bones)  |
| 0x06   | 2    | uint16   | Magic marker (0xAA55)          |
| 0x08   | 4    | uint32   | Block size                     |

### Controller Hash Array

Immediately after the header: `numControllers × 4 bytes`

Each hash is a CRC32 of the bone name (e.g., `housing_left_bottom_panel` → `0x16955904`).

### Controller Entry (24 bytes)

Same structure as CAF animation controllers:

| Offset | Size | Type   | Description                    |
|--------|------|--------|--------------------------------|
| 0x00   | 2    | uint16 | Number of rotation keys        |
| 0x02   | 2    | uint16 | Rotation format flags          |
| 0x04   | 4    | uint32 | Rotation time offset           |
| 0x08   | 4    | uint32 | Rotation data offset           |
| 0x0C   | 2    | uint16 | Number of position keys        |
| 0x0E   | 2    | uint16 | Position format flags          |
| 0x10   | 4    | uint32 | Position time offset           |
| 0x14   | 4    | uint32 | Position data offset           |

**Offsets are relative to the start of the controller entry.**

Controllers with no rotation data have `numRotKeys=0` and zero offsets for rotation fields.

### Format Flags

| Value  | Description                              |
|--------|------------------------------------------|
| 0x0000 | No track                                 |
| 0x8040 | Rotation only, ubyte time keys           |
| 0x8042 | Rotation only, uint16 time with header   |
| 0xC040 | Position, ubyte time keys                |
| 0xC240 | Position, uint16 time with header        |

**Low nibble:**
- 0x00 = ubyte time array
- 0x02 = uint16 time with 8-byte header (startTime, endTime, marker)

**High byte bits:**
- 0x80 = rotation track present
- 0x40 = position track present

### Keyframe Data

Rotation data is stored as uncompressed quaternions (16 bytes each: x, y, z, w as floats).

Position data is stored as Vector3 (12 bytes each: x, y, z as floats).

**Note:** Some position data may be stored as SNORMs (normalized signed integers) rather than floats. This is still being investigated.

Time keys are stored as:
- ubyte array (1 byte per key) for format 0x40
- 8-byte header (uint16 startTime, uint16 endTime, uint32 marker) for format 0x42

---

## Chunk 1: Animation Info (0xF7351608)

### AnimInfo Header

| Offset | Size | Type   | Description            |
|--------|------|--------|------------------------|
| 0x00   | 4    | uint32 | Number of animations   |

### AnimInfo Entry (44 bytes each)

| Offset | Size | Type       | Description                    |
|--------|------|------------|--------------------------------|
| 0x00   | 4    | uint32     | Flags (usually 2)              |
| 0x04   | 2    | uint16     | Frames per second (typically 30) |
| 0x06   | 2    | uint16     | Number of controllers          |
| 0x08   | 4    | uint32     | Unknown1                       |
| 0x0C   | 4    | uint32     | Unknown2                       |
| 0x10   | 16   | Quaternion | Start rotation (x, y, z, w)    |
| 0x20   | 12   | Vector3    | Start position (x, y, z)       |

### Path Strings

After all AnimInfo entries, null-terminated path strings follow (one per animation):

```
animations/spaceships/weapons/behr/behr_lasercannon_s2/behr_lasercannon_s2_recoil.caf\0
```

---

## Example: BEHR_LaserCannon_S2.dba (264 bytes)

```
Offset  Data                                      Description
------  ----------------------------------------  -----------
0x00    23 69 76 6F 00 09 00 00 02 00 00 00 ...  #ivo header
0x10    50 BC 4F 19 00 09 00 00 30 00 00 00 ...  Chunk 0 @ 0x30
0x20    08 16 35 F7 01 09 00 00 80 00 00 00 ...  Chunk 1 @ 0x80

Chunk 0 (DBA Data):
0x30    4C 00 00 00                              Chunk size: 76 bytes
0x34    23 64 62 61                              "#dba"
0x38    01 00                                    1 controller
0x3A    55 AA                                    Magic marker
0x3C    4C 00 00 00                              Block size: 76

0x40    B7 37 61 9A                              Controller hash (CRC32)

0x44    00 00 00 00 00 00 00 00 00 00 00 00      No rotation (12 zeros)
0x50    04 00 40 C2 18 00 00 00 1C 00 00 00      Position: 4 keys, format 0xC240

0x5C    00 01 0D 10                              Position time keys (ubyte)
0x60    FF FF 7F 7F ...                          Time header + position data

Chunk 1 (AnimInfo):
0x80    01 00 00 00                              1 animation
0x84    02 00 00 00 1E 00 01 00 ...              AnimInfo entry (44 bytes)
0xB0    61 6E 69 6D 61 74 69 6F 6E 73 2F ...    Path string
```

---

## Example: SuckerPunch.dba (768 bytes)

This file has **2 animations** with multiple #dba blocks:

```
Chunk 0 (DBA Data):
├── #dba Block 0 (deploy animation)
│   ├── 4 controllers
│   ├── Controller hashes (bone CRC32s)
│   └── Keyframe data
└── #dba Block 1 (recoil animation)
    ├── 4 controllers
    └── Keyframe data

Chunk 1 (AnimInfo):
├── 2 AnimInfo entries
├── Path 1: "...suckerpunch_s1_deploy.caf"
└── Path 2: "...suckerpunch_s1_recoil.caf"
```

---

## Controller Hash Examples

The controller hashes are CRC32 values of bone names from the skeleton:

| Hash       | Bone Name                  |
|------------|---------------------------|
| 0x16955904 | housing_left_bottom_panel |
| 0x61926992 | housing_right_bottom_panel |
| 0x88F1CCA7 | housing_left_top_panel    |
| 0xF89B3828 | housing_right_top_panel   |

---

## Relationship to CAF Format

DBA and CAF files share the same controller entry structure (24 bytes). The main differences:

| Aspect           | CAF                    | DBA                         |
|------------------|------------------------|-----------------------------|
| Container        | Single animation       | Multiple animations         |
| Chunk IDs        | 0x47338DD8, 0xD8496CA8 | 0x194FBC50, 0xF7351608     |
| Block structure  | Single #caf block      | Multiple #dba blocks        |
| Path storage     | AnimInfo chunk         | Separate path strings       |

---

## 010 Editor Template

Use `CryengineUnified.bt` to parse DBA files. The template will:
1. Detect the #ivo container format
2. Parse the chunk table
3. Read all #dba blocks with controller data
4. Display rotation quaternions and position vectors
5. Parse AnimInfo entries and path strings

---

## Notes

- Block boundaries: Each #dba block ends after its controller headers; the next block starts immediately after
- Position format: Some position data may use SNORM encoding instead of floats (investigation ongoing)
- Start rotation/position in AnimInfo represents the animation's reference pose
