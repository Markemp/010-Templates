# Star Citizen DBA Animation Library Format

## Overview

Star Citizen uses a custom animation format based on CryEngine, wrapped in an `#ivo` container format. DBA (Database Animation) files contain multiple animations bundled together for efficient loading.

**File analyzed:** `ursa.dba` - Contains 22 animations for the RSI Ursa Rover vehicle

## File Structure

```
#ivo Container
├── Header (16 bytes)
├── Chunk Table (16 bytes × numChunks)
├── Chunk 0: Animation Data (#dba blocks)
│   ├── Total chunk size (4 bytes)
│   ├── #dba Block 1
│   ├── #dba Block 2
│   └── ... (one per animation)
└── Chunk 1: Animation Metadata
    ├── Header (8 bytes)
    ├── Animation Entries (44 bytes each, last is 40)
    └── String Table (null-terminated paths)
```

---

## #ivo Container Header

| Offset | Size | Type     | Description              | Example Value |
|--------|------|----------|--------------------------|---------------|
| 0x00   | 4    | char[4]  | Signature                | "#ivo"        |
| 0x04   | 2    | uint16   | Version (little endian)  | 0x0900        |
| 0x06   | 2    | uint16   | Reserved                 | 0x0000        |
| 0x08   | 4    | uint32   | Number of chunks         | 2             |
| 0x0C   | 4    | uint32   | Header size              | 0x10          |

## Chunk Table Entry (16 bytes each)

| Offset | Size | Type   | Description        |
|--------|------|--------|--------------------|
| 0x00   | 4    | uint32 | Chunk ID           |
| 0x04   | 2    | uint16 | Version            |
| 0x06   | 2    | uint16 | Reserved           |
| 0x08   | 4    | uint32 | Offset from file start |
| 0x0C   | 4    | uint32 | Reserved           |

### Known Chunk IDs

| Chunk ID     | Version | Description                    |
|--------------|---------|--------------------------------|
| 0x194FBC50   | 0x0900  | Animation Data (contains #dba) |
| 0xF7351608   | 0x0901  | Animation Metadata             |

---

## Chunk 0: Animation Data

### Chunk 0 Header

| Offset | Size | Type   | Description                    |
|--------|------|--------|--------------------------------|
| 0x00   | 4    | uint32 | Total chunk data size (bytes)  |

After this 4-byte size field, the animation blocks follow consecutively.

### #dba Animation Block

Each animation in the DBA is stored as a `#dba` block:

| Offset | Size | Type     | Description              |
|--------|------|----------|--------------------------|
| 0x00   | 4    | char[4]  | Signature "#dba"         |
| 0x04   | 1    | uint8    | Bone count               |
| 0x05   | 1    | uint8    | Padding (always 0)       |
| 0x06   | 2    | uint16   | Magic marker (0xAA55)    |
| 0x08   | 4    | uint32   | Animation data size      |
| 0x0C   | var  | uint32[] | Bone hash array          |
| var    | var  | struct[] | Controller entries       |
| var    | var  | bytes    | Keyframe data            |

**Bone Hash Array:** `boneCount × 4 bytes` (CRC32 hashes of bone names)

**Controller Entries:** `boneCount × 24 bytes`

### Controller Entry (24 bytes)

| Offset | Size | Type   | Description               |
|--------|------|--------|---------------------------|
| 0x00   | 1    | uint8  | Number of keyframes       |
| 0x01   | 1    | uint8  | Padding                   |
| 0x02   | 2    | uint16 | Format flags              |
| 0x04   | 4    | uint32 | Rotation data offset      |
| 0x08   | 4    | uint32 | Rotation time offset      |
| 0x0C   | 4    | uint32 | Position data offset      |
| 0x10   | 4    | uint32 | Position time offset      |
| 0x14   | 4    | uint32 | Reserved                  |

### Format Flags

The format flags field (2 bytes) encodes compression information:

| Bit Pattern | Meaning                    |
|-------------|----------------------------|
| 0x8042      | Rotation only, SmallTree48 |
| 0xC242      | Has position, SmallTree48  |
| 0x80xx      | Rotation only flag         |
| 0xC0xx      | Has position flag          |

**Rotation Compression Formats** (low byte):
- 0x42 = SmallTree48BitQuat (6 bytes per key)
- 0x43 = SmallTree64BitQuat (8 bytes per key)
- 0x00 = NoCompressQuat (16 bytes per key)

**Position Compression Formats:**
- Similar to CAF controller formats

### Keyframe Data Layout

For each bone controller, keyframe data is stored at the specified offsets:

1. **Rotation values:** `numKeys × compressionSize` bytes
2. **Rotation times:** `numKeys × timeFormat` bytes
3. **Position values:** (if present) `numKeys × posFormat` bytes
4. **Position times:** (if present) `numKeys × timeFormat` bytes

---

## Chunk 1: Animation Metadata

### Metadata Header

| Offset | Size | Type   | Description        |
|--------|------|--------|--------------------|
| 0x00   | 4    | uint32 | Animation count    |
| 0x04   | 4    | uint32 | Reserved           |

### Animation Entry (44 bytes, last entry is 40 bytes)

| Offset | Size | Type    | Description               |
|--------|------|---------|---------------------------|
| 0x00   | 2    | uint16  | Number of keys/frames     |
| 0x02   | 2    | uint16  | Bone count                |
| 0x04   | 4    | uint32  | Flags                     |
| 0x08   | 4    | uint32  | Path length               |
| 0x0C   | 12   | Vec3    | Start position (x,y,z)    |
| 0x18   | 16   | Quat    | Start rotation (x,y,z,w)  |
| 0x28   | 4    | uint32  | Extra field (padding)*    |

*The last entry in the array does not have the 4-byte extra field.

### String Table

Immediately after the animation entries, null-terminated animation paths are stored consecutively:

```
animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_enter.caf\0
animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_exit.caf\0
...
```

---

## Example: ursa.dba Analysis

**File size:** 2,682,360 bytes

**Animations (22 total):**
| Index | Bones | Keys | Path |
|-------|-------|------|------|
| 0     | 112   | 30   | rsi_ursa_rover_gunner_enter.caf |
| 1     | 112   | 30   | rsi_ursa_rover_gunner_exit.caf |
| 2     | 107   | 30   | rsi_ursa_rover_gunner_idle.caf |
| 3     | 107   | 30   | rsi_ursa_rover_gunner_mobiglas_enter.caf |
| ...   | ...   | ...  | ... |
| 21    | 116   | 30   | rsi_ursa_rover_pilot_steer_fullrange.caf |

---

## Differences from Standard CryEngine/Lumberyard

1. **Container format:** Uses `#ivo` instead of standard CryEngine chunk headers
2. **Inline bone hashes:** DBA blocks include bone CRC32 hashes directly
3. **Simplified controller:** 24-byte controller entries vs Lumberyard's variable structures
4. **Magic marker:** `0xAA55` marker after bone count

---

## CAF vs DBA Chunk IDs

Star Citizen uses different chunk IDs for CAF (single animation) and DBA (animation library) files:

| File Type | Chunk ID     | Description           |
|-----------|--------------|----------------------|
| **DBA**   | 0x194FBC50   | Animation Data       |
| **DBA**   | 0xF7351608   | Animation Metadata   |
| **CAF**   | 0x47338DD8   | Animation Data       |
| **CAF**   | 0xD8496CA8   | Animation Info       |

---

## 010 Editor Templates

Two 010 Editor templates are provided:

1. **StarCitizen_DBA.bt** - Specialized template for DBA files
2. **StarCitizen_IVO.bt** - Universal template supporting both CAF and DBA

To use:
1. Open the .dba or .caf file in 010 Editor
2. Run the template (Templates > Run Template)
3. Navigate the parsed structure in the Template Results panel

---

## Related Formats

- **Star Citizen CAF:** Single animation files using same `#ivo` container
- **Lumberyard DBA:** Standard CryEngine v0905 animation library format
- **Lumberyard CAF:** Standard CryEngine animation format (v0829, v0831)

---

## Appendix: Full Animation List (ursa.dba)

| # | Bones | Animation Path |
|---|-------|----------------|
| 0 | 112 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_enter.caf |
| 1 | 112 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_exit.caf |
| 2 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_idle.caf |
| 3 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_mobiglas_enter.caf |
| 4 | 108 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_mobiglas_exit.caf |
| 5 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_gunner_mobiglas_idle.caf |
| 6 | 111 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_enter.caf |
| 7 | 112 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_exit.caf |
| 8 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_forward.caf |
| 9 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_forward_high.caf |
| 10 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_leftbank.caf |
| 11 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_leftbank_high.caf |
| 12 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_reverse.caf |
| 13 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_reverse_high.caf |
| 14 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_rightbank.caf |
| 15 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_gforce_rightbank_high.caf |
| 16 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_idle.caf |
| 17 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_idledriving.caf |
| 18 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_mobiglas_enter.caf |
| 19 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_mobiglas_exit.caf |
| 20 | 107 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_mobiglas_idle.caf |
| 21 | 116 | animations/characters/human/male_v7/vehicles/rsi/ursa/rsi_ursa_rover_pilot_steer_fullrange.caf |

