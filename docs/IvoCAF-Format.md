# Ivo CAF Animation Format Analysis

This document describes the `#ivo` CAF animation file format used by Star Citizen, compared to the older CryTek CAF format.

**Last Updated:** 2024-12-19
**Reference Files:**
- New format: `D:\depot\SC4.1\Data\Animations\Characters\Creatures\aloprat\pose_ai_aloprat_stand_idle_01.caf`
- Old format: `D:\depot\ArmoredWarfare\animations\animals\birds\chicken\walk.caf`

---

## Format Overview

| Aspect | CryTek CAF | Ivo CAF |
|--------|------------|---------|
| Magic | `CryT` | `#ivo` |
| Chunk structure | N+1 chunks (SpeedInfo + N Controllers) | 2 chunks (AnimInfo + CAFData) |
| Bone identification | ControllerId per chunk | BoneHash array |
| Data layout | Inline per controller | Offset-based, consolidated |
| Rotation format | SmallTree48BitQuat (6 bytes) | Uncompressed (16 bytes) |

---

## Ivo CAF File Structure

```
┌─────────────────────────────────────────────────────────────┐
│ #ivo File Header (16 bytes)                                 │
│   0x00: "#ivo" signature                                    │
│   0x04: version (uint32)                                    │
│   0x08: numChunks (uint32)                                  │
│   0x0C: chunkTableOffset (int32)                            │
├─────────────────────────────────────────────────────────────┤
│ Chunk Table (numChunks × 16 bytes each)                     │
│   ChunkType_Ivo (uint32)                                    │
│   version (uint32)                                          │
│   offset (uint64)                                           │
├─────────────────────────────────────────────────────────────┤
│ Chunk 0: AnimInfoChunk_Ivo (0x4733C6ED)                     │
├─────────────────────────────────────────────────────────────┤
│ Chunk 1: CAFData (0xA9496CB5)                               │
│   ├── #caf block header (12 bytes)                          │
│   ├── BoneHashes array (boneCount × 4 bytes)                │
│   ├── Controller array (boneCount × 24 bytes)               │
│   └── Keyframe data (rotation, time, position)              │
└─────────────────────────────────────────────────────────────┘
```

---

## AnimInfoChunk_Ivo (0x4733C6ED)

Version: 0x0901

```c
struct AnimInfoChunk_Ivo {
    uint32 flags;                     // Animation flags (0=normal, 2=pose?)
    uint16 framesPerSecond;           // FPS - typically 30
    uint16 numBones;                  // Number of animated bones
    uint32 unknown2;                  // Reserved/padding
    uint32 numPositionTracks;         // Bones with position animation
    Quaternion startRotation;         // Reference pose rotation (x, y, z, w)
    Vector3 startPosition;            // Reference pose position (x, y, z)
    uint32 padding;                   // Reserved
};  // Total: 48 bytes
```

**Field identification:**
- `framesPerSecond` confirmed as FPS by comparing:
  - Idle pose: 2 keys/bone, field=30
  - Walk cycle: 30-31 keys/bone, field=30
  - Both show 30, so it's FPS not keyframe count

**Example values (idle pose):**
- flags: 2
- framesPerSecond: 30
- numBones: 28
- numPositionTracks: 1
- startRotation: (-0.707, 0, 0, 0.707) - 90° around X
- startPosition: (0, 0, 0)

**Example values (walk cycle):**
- flags: 0
- framesPerSecond: 30
- numBones: 31
- numPositionTracks: 30
- Total rotation keys: 956

---

## CAFData Chunk (0xA9496CB5)

Version: 0x0900

### #caf Block Header (12 bytes)

```c
struct AnimBlockHeader_Ivo {
    char signature[4];    // "#caf"
    ubyte boneCount;      // Number of bones in this animation
    ubyte padding;
    uint16 magic;         // 0xFFFF observed
    uint32 dataSize;      // Total block size INCLUDING this 12-byte header
};
```

**Important:** `dataSize` includes the 12-byte header itself, so actual data content is `dataSize - 12` bytes.

### BoneHashes Array

Immediately follows the header. Contains `boneCount` CRC32 hashes identifying which bones are animated.

```c
uint32 boneHashes[boneCount];
```

**Known bone hash mappings (from aloprat skeleton):**

| Index | Hash | Bone Name |
|-------|------|-----------|
| [2] | 307473952 (0x1253AE20) | L_UpperLeg |
| [23] | 3738240529 (0xDED10611) | Hips |
| [25] | 4223024711 (0xFBB63E47) | World |

Note: Not all skeleton bones appear in the hash array - only animated bones are included.

### Controller Array

Each controller entry is **24 bytes**:

```c
struct AnimControllerEntry_Ivo {
    // Rotation track (12 bytes)
    ubyte numRotKeys;          // Number of rotation keyframes
    ubyte rotPadding;
    uint16 rotFormatFlags;     // 0x8040 = standard rotation
    uint32 rotDataOffset;      // Offset to rotation data (relative to keyframe section)
    uint32 rotTimeOffset;      // Offset to rotation times

    // Position track (12 bytes)
    ubyte numPosKeys;          // Number of position keyframes (0 if none)
    ubyte posPadding;
    uint16 posFormatFlags;     // 0xC040 = has position, 0x0000 = none
    uint32 posDataOffset;      // Offset to position data
    uint32 posTimeOffset;      // Offset to position times
};
```

**Format Flags:**

Rotation flags:
- `0x8040` - Rotation track, ubyte time keys
- `0x8042` - Rotation track, uint16 time with header

Position flags (high byte determines data format):
- `0x0000` - No position track
- `0xC040` - Float Vector3, ubyte time keys
- `0xC042` - Float Vector3, uint16 time with header
- `0xC140` - SNORM full (all channels), ubyte time
- `0xC142` - SNORM full (all channels), uint16 time
- `0xC240` - SNORM packed (active channels only), ubyte time
- `0xC242` - SNORM packed (active channels only), uint16 time

**Offset behavior:**
- `rotDataOffset` values **decrease** across controllers (0x2A0 → 0x18)
- `rotTimeOffset` values **increase** across controllers (0x2A4 → 0x3E0)
- This suggests rotation data is stored in reverse controller order

### Keyframe Data Section

Starts after the controller array. Structure:

```
┌─────────────────────────────────────────┐
│ Header (4 bytes) - value 0x00000100     │
├─────────────────────────────────────────┤
│ Rotation Quaternion Data                │
│   Uncompressed 16-byte quaternions      │
│   Stored in REVERSE controller order    │
├─────────────────────────────────────────┤
│ Time Data                               │
│   Key times for rotation/position       │
├─────────────────────────────────────────┤
│ Position Vector Data                    │
│   Only for bones with position tracks   │
└─────────────────────────────────────────┘
```

**Offset calculation:**
```
absolute_position = keyframe_section_start + offset
```

Where `keyframe_section_start` is the position immediately after the controller array.

---

## Position Data Formats

Position data format is determined by the high byte of the position format flags.

### 0xC0xx - Float Vector3 (No Header)

Direct float Vector3 values, 12 bytes per key. No header required.

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

---

## CryTek CAF Format (for comparison)

### SpeedInfo Chunk (0xAAFC0002)

Contains animation metadata:
- Speed, distance, slope
- Turn speed, asset turn
- Start/end frames

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

**Common time formats:**
- `2` = eByte (1 byte per time key)

---

## Key Differences Summary

| Feature | CryTek | Ivo |
|---------|--------|-----|
| Bone identification | ControllerId in each chunk | Separate BoneHash array |
| Data storage | Inline in each controller chunk | Consolidated keyframe block with offsets |
| Compression | SmallTree quaternions (6-8 bytes) | Uncompressed quaternions (16 bytes) |
| Number of chunks | N+1 (SpeedInfo + controllers) | 2 (AnimInfo + CAFData) |
| Chunk overhead | 16-byte header per bone | Single header for all data |

---

## Implementation Status

The following structures are implemented in `chunks/Animation.bt`:

| Structure | Status | Notes |
|-----------|--------|-------|
| `AnimInfoChunk_Ivo` | ✅ Complete | All fields documented |
| `AnimBlockHeader_Ivo` | ✅ Complete | 12-byte #caf/#dba header |
| `AnimControllerEntry_Ivo` | ✅ Complete | 24-byte dual-track structure |
| `CAFChunk_Ivo` | ✅ Complete | Parses bone hashes, controllers, rotation data |
| `DBAAnimationBlock_Ivo` | ✅ Complete | Same structure as CAF |
| `UncompressedQuat_Ivo` | ✅ Complete | 16-byte quaternion |

**Keyframe data parsing:**
- Rotation quaternions are parsed as `UncompressedQuat_Ivo` array
- Position data supports three formats: float Vector3, SNORM full, SNORM packed
- Time data parsed as ubyte array or uint16 with header based on format flags

---

## Open Questions

1. **4-byte keyframe header (0x00000100)**: What does this value represent? Possibly a format version or flags.
2. **Compression options**: Does the Ivo format support compressed quaternions?

---

## Revision History

| Date | Changes |
|------|---------|
| 2024-12-08 | Initial analysis from pose_ai_aloprat_stand_idle_01.caf |
| 2024-12-08 | Implemented AnimControllerEntry_Ivo with dual-track structure (rot+pos) |
| 2024-12-08 | Updated CAFChunk_Ivo to parse rotation quaternions |
| 2024-12-08 | Fixed dataSize interpretation (includes 12-byte header) |
| 2024-12-08 | Confirmed framesPerSecond field by comparing idle (2 keys) vs walk (31 keys) |
| 2024-12-19 | Fixed AnimInfoChunk_Ivo: boundMin/scale/precision → startRotation + startPosition |
| 2024-12-19 | Added position format documentation (0xC0 float, 0xC1/0xC2 SNORM) |
| 2024-12-19 | Documented SNORM packed format with FLT_MAX channel mask |
