# Ivo Animation Format Specification

This document describes the `#ivo` animation file formats (CAF and DBA) used by Star Citizen, compared to the older CryTek formats.

**Last Updated:** 2024-12-19

---

## Overview

Star Citizen uses custom animation formats based on CryEngine, wrapped in an `#ivo` container:

| Format | Extension | Purpose | Block Signature |
|--------|-----------|---------|-----------------|
| CAF | `.caf` | Single animation | `#caf` |
| DBA | `.dba` | Animation library (multiple animations) | `#dba` |

Both formats share the same controller entry structure and keyframe data formats.

### Comparison with CryTek Format

| Aspect | CryTek CAF | Ivo CAF/DBA |
|--------|------------|-------------|
| Magic | `CryT` | `#ivo` |
| Chunk structure | N+1 chunks (SpeedInfo + N Controllers) | 2 chunks (AnimInfo + Data) |
| Bone identification | ControllerId per chunk | BoneHash array |
| Data layout | Inline per controller | Offset-based, consolidated |
| Rotation format | SmallTree48BitQuat (6 bytes) | Uncompressed (16 bytes) |

---

## #ivo Container Structure

### File Header (16 bytes)

| Offset | Size | Type | Description | Example |
|--------|------|------|-------------|---------|
| 0x00 | 4 | char[4] | Signature | "#ivo" |
| 0x04 | 2 | uint16 | Version | 0x0900 |
| 0x06 | 2 | uint16 | Reserved | 0x0000 |
| 0x08 | 4 | uint32 | Number of chunks | 2 |
| 0x0C | 4 | uint32 | Chunk table offset | 0x10 |

### Chunk Table Entry (16 bytes each)

| Offset | Size | Type | Description |
|--------|------|------|-------------|
| 0x00 | 4 | uint32 | Chunk ID |
| 0x04 | 2 | uint16 | Version |
| 0x06 | 2 | uint16 | Reserved |
| 0x08 | 8 | uint64 | Offset from file start |

### Known Chunk IDs

| Chunk ID | Version | Format | Description |
|----------|---------|--------|-------------|
| 0x4733C6ED | 0x0901 | CAF | Animation Info |
| 0xA9496CB5 | 0x0900 | CAF | CAF Data |
| 0x194FBC50 | 0x0900 | DBA | DBA Data |
| 0xF7351608 | 0x0901 | DBA | DBA Animation Info |

---

## CAF Format (Single Animation)

### File Structure

```
#ivo Container
├── Header (16 bytes)
├── Chunk Table (2 × 16 bytes)
├── Chunk 0: AnimInfo (0x4733C6ED)
└── Chunk 1: CAFData (0xA9496CB5)
    ├── #caf block header (12 bytes)
    ├── BoneHashes array (boneCount × 4 bytes)
    ├── Controller array (boneCount × 24 bytes)
    └── Keyframe data (rotation, time, position)
```

### AnimInfoChunk_Ivo (48 bytes)

```c
struct AnimInfoChunk_Ivo {
    uint32 flags;                 // Animation flags (0=normal, 2=pose?)
    uint16 framesPerSecond;       // FPS - typically 30
    uint16 numBones;              // Number of animated bones
    uint32 unknown2;              // Reserved
    uint32 endFrame;              // Last frame number (matches timeEnd in time keys)
    Quaternion startRotation;     // Reference pose rotation (x, y, z, w)
    Vector3 startPosition;        // Reference pose position (x, y, z)
    uint32 padding;               // Reserved
};
```

---

## DBA Format (Animation Library)

### File Structure

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

### DBA AnimInfo Entry (44 bytes)

| Offset | Size | Type | Description |
|--------|------|------|-------------|
| 0x00 | 4 | uint32 | Flags (usually 2) |
| 0x04 | 2 | uint16 | Frames per second (typically 30) |
| 0x06 | 2 | uint16 | Number of controllers |
| 0x08 | 4 | uint32 | Unknown1 |
| 0x0C | 4 | uint32 | Unknown2 |
| 0x10 | 16 | Quaternion | Start rotation (x, y, z, w) |
| 0x20 | 12 | Vector3 | Start position (x, y, z) |

---

## Shared Structures

### Animation Block Header (12 bytes)

Used by both `#caf` and `#dba` blocks:

```c
struct AnimBlockHeader_Ivo {
    char signature[4];    // "#caf" or "#dba"
    uint16 boneCount;     // Number of bones in this animation
    uint16 magic;         // 0xAA55 (DBA) or 0xFFFF (CAF)
    uint32 dataSize;      // Total block size INCLUDING this 12-byte header
};
```

### Controller Hash Array

Immediately follows the block header. Contains `boneCount` CRC32 hashes identifying which bones are animated:

```c
uint32 boneHashes[boneCount];
```

### Controller Entry (24 bytes)

```c
struct AnimControllerEntry_Ivo {
    // Rotation track (12 bytes)
    uint16 numRotKeys;        // Number of rotation keyframes
    uint16 rotFormatFlags;    // Format flags
    uint32 rotTimeOffset;     // Offset to rotation time keys
    uint32 rotDataOffset;     // Offset to rotation data

    // Position track (12 bytes)
    uint16 numPosKeys;        // Number of position keyframes (0 if none)
    uint16 posFormatFlags;    // Format flags
    uint32 posTimeOffset;     // Offset to position time keys
    uint32 posDataOffset;     // Offset to position data
};
```

**Note:** Offsets are relative to the start of the controller entry.

---

## Format Flags

### Rotation Flags

| Value | Description |
|-------|-------------|
| 0x0000 | No rotation track |
| 0x8040 | Rotation, ubyte time keys |
| 0x8042 | Rotation, uint16 time with header |

### Position Flags

| Value | Description |
|-------|-------------|
| 0x0000 | No position track |
| 0xC040 | Float Vector3, ubyte time |
| 0xC042 | Float Vector3, uint16 time |
| 0xC140 | SNORM full (all channels), ubyte time |
| 0xC142 | SNORM full (all channels), uint16 time |
| 0xC240 | SNORM packed (active channels only), ubyte time |
| 0xC242 | SNORM packed (active channels only), uint16 time |

### Flag Breakdown

**Low nibble (time format):**
- 0x40 = ubyte time array (1 byte per key)
- 0x42 = uint16 time with 8-byte header (startTime, endTime, marker)

**High byte (data format):**
- 0x80 = rotation track present
- 0xC0 = position as float Vector3 (no header)
- 0xC1 = position as SNORM with header (all channels)
- 0xC2 = position as SNORM with header (packed active channels only)

---

## Keyframe Data

### Rotation Data

Stored as uncompressed quaternions (16 bytes each: x, y, z, w as floats).

### Time Keys

- **0x40 format:** ubyte array (1 byte per key)
- **0x42 format:** 8-byte header followed by data
  ```c
  struct TimeHeader {
      uint16 startTime;
      uint16 endTime;
      uint32 marker;    // 0x7FFFFFFF, 0x5FFFFFFF, etc.
  };
  ```

---

## Position Data Formats

Position data format is determined by the high byte of the position format flags.

### 0xC0xx - Float Vector3 (No Header)

Direct float Vector3 values, 12 bytes per key:

```
Position[0]: float x, float y, float z  (12 bytes)
Position[1]: float x, float y, float z  (12 bytes)
...
```

### 0xC1xx - SNORM with Header (Full Channels)

24-byte header followed by full SNORM Vector3 values (6 bytes per key):

```
Header (24 bytes):
├── channelMask: Vector3 (12 bytes) - FLT_MAX = channel not animated
└── scale: Vector3 (12 bytes) - Scale factors for decompression

Position[0]: int16 x, int16 y, int16 z  (6 bytes)
Position[1]: int16 x, int16 y, int16 z  (6 bytes)
...
```

### 0xC2xx - SNORM with Header (Packed Active Channels)

24-byte header followed by packed SNORM values for active channels only:

```
Header (24 bytes):
├── channelMask: Vector3 (12 bytes) - FLT_MAX (0x7F7FFFFF) = channel not animated
└── scale: Vector3 (12 bytes) - Scale factors for decompression

Position data: only stores int16 values for channels where channelMask != FLT_MAX
```

**Example:** If only Y is animated (X and Z have FLT_MAX in channelMask):
- Each position key is 2 bytes (single int16 for Y)
- X and Z values are assumed to be 0

**Decompression formula:**
```
actual_position[channel] = (snorm_value / 32767.0) * scale[channel]
```

### Channel Mask Examples

| channelMask Value | Meaning |
|-------------------|---------|
| (0, 0, 0) | All channels animated |
| (FLT_MAX, 0.05, FLT_MAX) | Only Y animated, scale=0.05 |
| (FLT_MAX, FLT_MAX, 0.1) | Only Z animated, scale=0.1 |

This is commonly seen in weapon recoil animations where only one axis moves.

---

## Controller Hash Examples

The controller hashes are CRC32 values of bone names from the skeleton:

| Hash | Bone Name |
|------|-----------|
| 0x16955904 | housing_left_bottom_panel |
| 0x61926992 | housing_right_bottom_panel |
| 0x88F1CCA7 | housing_left_top_panel |
| 0xF89B3828 | housing_right_top_panel |
| 0x1253AE20 | L_UpperLeg |
| 0xDED10611 | Hips |
| 0xFBB63E47 | World |

---

## Examples

### CAF File (aloprat idle pose)
D:\depot\SC4.1\Data\Animations\Characters\Creatures\aloprat\ai_aloprat_stand_idle_01.caf
```
- flags: 2
- framesPerSecond: 30
- numBones: 30
- endFrame: 128
- startRotation: (0, 0, 0, 1)
- startPosition: (0, 0, 0)
```

### DBA File (BEHR_LaserCannon_S2.dba - 264 bytes)

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
```

### DBA File with Multiple Animations (SuckerPunch.dba)

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

## CryTek CAF Format (Legacy Reference)

### SpeedInfo Chunk (0xAAFC0002)

Contains animation metadata: speed, distance, slope, turn speed, start/end frames.

### Controller829 Chunk (0xCCCC000D)

One chunk per animated bone:

```c
struct Controller829_Cry {
    ChunkHeader chunkHeader;   // 16 bytes embedded
    uint32 controllerId;       // Bone CRC32 hash
    uint16 numRotationKeys;
    uint16 numPositionKeys;
    ubyte rotationFormat;      // ECompressionFormat
    ubyte rotationTimeFormat;  // EKeyTimesFormat
    ubyte positionFormat;
    ubyte positionKeyInfo;
    ubyte positionTimeFormat;
    ubyte tracksAligned;
    uint16 padding;
    // Inline rotation data
    // Inline rotation times
    // Inline position data (if any)
    // Inline position times (if any)
};
```

**Common rotation formats:**
- `5` = SmallTree48BitQuat (6 bytes per quaternion)
- `6` = SmallTree64BitQuat (8 bytes per quaternion)
- `8` = SmallTree64BitExtQuat (8 bytes per quaternion)

---

## 010 Editor Template

Use `CryengineUnified.bt` to parse CAF and DBA files. The template will:
1. Detect the #ivo container format
2. Parse the chunk table
3. Read animation blocks with controller data
4. Display rotation quaternions and position vectors
5. Parse AnimInfo entries and path strings

---

## Implementation Status

| Structure | Status | Notes |
|-----------|--------|-------|
| `AnimInfoChunk_Ivo` | Complete | CAF animation info |
| `DBAAnimInfoEntry_Ivo` | Complete | DBA animation info (44 bytes) |
| `AnimBlockHeader_Ivo` | Complete | 12-byte #caf/#dba header |
| `AnimControllerEntry_Ivo` | Complete | 24-byte dual-track structure |
| `CAFChunk_Ivo` | Complete | Single animation parsing |
| `DBABlock_Ivo` | Complete | DBA block parsing |
| `DbaDataChunk_Ivo` | Complete | Multiple #dba blocks |
| `PosHeader_C2_Ivo` | Complete | SNORM position header |
| `PackedSNORMPos_Ivo` | Complete | Packed active channels |

---

## Open Questions

1. **4-byte keyframe header (0x00000100)**: What does this value represent? Possibly a format version or flags.
2. **Compression options**: Does the Ivo format support compressed quaternions?

---

## Notes

- Block boundaries: Each #dba block ends after its keyframe data; the next block starts immediately after
- Position format: Supports float Vector3 (0xC0), SNORM full (0xC1), and SNORM packed (0xC2)
- SNORM packed format uses FLT_MAX (0x7F7FFFFF) as sentinel for inactive channels
- Start rotation/position in AnimInfo represents the animation's reference pose
- Weapon recoil animations typically use 0xC2 format with only one active axis

---

## Revision History

| Date | Changes |
|------|---------|
| 2024-12-08 | Initial CAF analysis |
| 2024-12-08 | Implemented AnimControllerEntry_Ivo with dual-track structure |
| 2024-12-08 | Confirmed framesPerSecond field |
| 2024-12-19 | Fixed AnimInfoChunk_Ivo: startRotation + startPosition |
| 2024-12-19 | Added position format documentation (0xC0, 0xC1, 0xC2 SNORM) |
| 2024-12-19 | Documented SNORM packed format with FLT_MAX channel mask |
| 2024-12-19 | Consolidated CAF and DBA documentation into single file |
