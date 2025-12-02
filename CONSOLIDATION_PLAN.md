# CryEngine Template Consolidation Plan

This document outlines the plan to consolidate the 010 Editor binary templates into a unified, well-organized structure with a single entry point.

## Current State Summary

| Category | Files | Lines |
|----------|-------|-------|
| Traditional CryEngine (3.4/3.8) | 5 files | 1,080 |
| Star Citizen Ivo | 5 files | 1,556 |
| Shared/Base | 2 files | 715 |
| **Total** | **13 files** | **3,351** |

### Entry Points Today
- `Cryengine.bt` - Traditional formats (CryT, CrCh), plus redirect for #ivo
- `CryEngine-Ivo.bt` - Star Citizen #ivo files (CGF, SKIN, CAF, DBA)

### Key Conflicts Identified

1. **Enum Name Collisions**
   - `CHUNKTYPE` defined differently in both enum files
   - `DATASTREAMTYPE` vs `DatastreamType` - different value ranges entirely
   - Traditional uses `_ui`/`_us` suffixes; Ivo uses bare names

2. **Duplicate Structs**
   - `BBOX` - identical definition in both struct files
   - `MESHCHUNK`, `COMPILEDBONES`, `MTLNAME` - same names, different signatures

3. **Organizational Issues**
   - `Cryengine-Common.bt` mixes utilities, structs, and functions
   - `Cryengine-Helpers.bt` - "helpers" meaning unclear (mostly print functions)
   - Animation code scattered across multiple files

4. **Naming Inconsistencies**
   - Mix of `PascalCase`, `SCREAMING_SNAKE_CASE`, `camelCase`
   - Ivo-prefixed types (`IVOVERTUV`) vs non-prefixed (`MESHCHUNK`)
   - Print function naming: `PrintVector3()` vs `PrintAnimInfo()`

---

## Proposed Architecture

### New Directory Structure

```
010-Templates/
├── Cryengine.bt                    # Single unified entry point
├── core/
│   ├── BaseTypes.bt                # Primitive types (VECTOR3, QUATERNION, MATRIX, etc.)
│   ├── Enums.bt                    # Unified enum definitions with namespacing
│   └── Utilities.bt                # Shared utility functions (CryHalfToFloat, alignment)
├── chunks/
│   ├── Header.bt                   # HEADER, CHUNKTABLE, CHUNKTABLEENTRY
│   ├── Mesh.bt                     # MESHCHUNK, MESHSUBSETS, datastreams
│   ├── Bones.bt                    # COMPILEDBONES, COMPILEDBONE, BONEMAP
│   ├── Materials.bt                # MTLNAME, material structures
│   ├── Animation.bt                # Controllers, CAF, DBA structures
│   └── Node.bt                     # NODE, NODEMESHCOMBO structures
├── display/
│   └── PrintFunctions.bt           # All Print* functions for 010 Editor display
├── CLAUDE.md
├── CONSOLIDATION_PLAN.md
└── README.md
```

### Naming Convention Standard

Adopt C-style naming conventions for 010 Editor templates:

| Element | Convention | Example |
|---------|------------|---------|
| Structs | `PascalCase` | `MeshChunk`, `CompiledBone` |
| Enums | `PascalCase` | `ChunkType`, `DataStreamType` |
| Enum Values | `PascalCase` with format suffix | `Mesh_Cry34`, `Mesh_Ivo` |
| Functions | `PascalCase` | `PrintVector3`, `CryHalfToFloat` |
| Variables/Parameters | `camelCase` | `fileType`, `chunkVersion`, `numBones` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_BONE_COUNT` |
| Header Guards | `SCREAMING_SNAKE_CASE` + `_BT` | `CORE_BASETYPES_BT` |

---

## Enum Consolidation Strategy

### Problem

The same logical chunk types have different numeric values across formats:

```c
// Traditional CryEngine 3.4 (uint32)
Mesh_ui = 0xCCCC0000

// Traditional CryEngine 3.8 (uint16)
Mesh_us = 0x1000

// Star Citizen Ivo (uint32)
MeshChunk = 0x9293b9d8
```

### Solution: Format-Suffixed Enum Values

Create unified enums with format-specific suffixes:

```c
// core/Enums.bt - Unified Chunk Type Enum

enum <uint32> ChunkType {
    // CryEngine 3.4 format (suffix: _Cry34)
    Mesh_Cry34              = 0xCCCC0000,
    Node_Cry34              = 0xCCCC000B,
    Controller_Cry34        = 0xCCCC000D,
    CompiledBones_Cry34     = 0xCCCC1001,

    // CryEngine 3.8 format (suffix: _Cry38) - stored as uint16, cast to uint32
    Mesh_Cry38              = 0x1000,
    Node_Cry38              = 0x100B,
    Controller_Cry38        = 0x100D,
    CompiledBones_Cry38     = 0x1001,

    // Star Citizen Ivo format (suffix: _Ivo)
    MeshInfo_Ivo            = 0x92914444,
    MeshChunk_Ivo           = 0x9293B9D8,
    CompiledBones_Ivo       = 0xC201973C,
    SkinMesh_Ivo            = 0x78B6E3C7,
    NodeMeshCombo_Ivo       = 0x8F3F6E06,
    AnimInfo_Ivo            = 0xB490E23E,
    AnimController_Ivo      = 0x0E3D6C48,
};
```

### DataStream Type Strategy

Similar approach for datastream types:

```c
enum <uint32> DataStreamType {
    // Traditional sequential values (suffix: _Cry)
    Vertices_Cry        = 0x00,
    Normals_Cry         = 0x01,
    Uvs_Cry             = 0x02,
    Colors_Cry          = 0x03,
    Tangents_Cry        = 0x06,
    BoneMap_Cry         = 0x08,
    Indices_Cry         = 0x05,
    VertsUvs_Cry        = 0x0F,

    // Ivo magic hash values (suffix: _Ivo)
    VertsUvs_Ivo        = 0x91329AE9,
    Indices_Ivo         = 0xEECDC168,
    Normals_Ivo         = 0xB872A3E8,
    Normals2_Ivo        = 0x38A581FE,
    Tangents_Ivo        = 0x33A931F2,
    BoneMap_Ivo         = 0x677C7B23,
    Colors_Ivo          = 0x22B38ED6,
};
```

---

## Struct Consolidation Strategy

### Approach 1: Parameterized Structs (Preferred)

Where structs differ by format, use a `FileFormat` parameter:

```c
enum FileFormat {
    CryEngine34,
    CryEngine38,
    StarCitizenIvo
};

struct Header(FileFormat format) {
    switch (format) {
        case CryEngine34:
        case CryEngine38:
            char signature[4];
            uint32 version;
            uint32 numChunks;
            uint32 chunkTableOffset;
            break;
        case StarCitizenIvo:
            char signature[4];
            uint32 version;
            uint32 numChunks;
            uint32 chunkTableOffset;
            // Additional Ivo-specific fields
            uint64 extendedOffset;
            break;
    }
};
```

### Approach 2: Format-Specific Struct Names

Where structures are fundamentally different, use suffixed names:

```c
// Traditional mesh chunk with complex versioning
struct MeshChunk_Cry(FileFormat format, int chunkVersion) { ... }

// Ivo mesh chunk - simpler, fixed structure
struct MeshChunk_Ivo { ... }
```

### Struct Migration Table

| Current (Traditional) | Current (Ivo) | Consolidated |
|----------------------|---------------|--------------|
| `HEADER(FILETYPE)` | `HEADER` | `Header(FileFormat)` |
| `CHUNKTABLEENTRY(FILETYPE)` | `CHUNKTABLEENTRY` | `ChunkTableEntry(FileFormat)` |
| `MESHCHUNK(FILETYPE, ver)` | `MESHCHUNK` | `MeshChunk_Cry(FileFormat, ver)` + `MeshChunk_Ivo` |
| `COMPILEDBONES(n, ver, type)` | `COMPILEDBONES(ver)` | `CompiledBones(FileFormat, ver)` |
| `MTLNAME(type, entry)` | `MTLNAME` | `MtlName_Cry(type, entry)` + `MtlName_Ivo` |
| `BBOX` (duplicate) | `BBOX` (duplicate) | `BBox` (single definition) |
| N/A | `SKINMESH` | `SkinMesh` (Ivo-only) |
| N/A | `NODEMESHCOMBOCHUNK` | `NodeMeshComboChunk` (Ivo-only) |
| `CONTROLLER_829`, `CONTROLLER_905` | N/A | `Controller829`, `Controller905` |
| N/A | `ANIM_INFO_CHUNK` | `AnimInfoChunk` (Ivo-only) |

---

## Implementation Phases

### Phase 1: Core Foundation

**Goal**: Create the new directory structure and core modules without breaking existing functionality.

**Tasks**:
1. Create `core/`, `chunks/`, `display/` directories
2. Create `core/BaseTypes.bt` - copy from `Cryengine-BaseTypes.bt`, apply naming conventions
3. Create `core/Enums.bt` - merge enums with format suffixes
4. Create `core/Utilities.bt` - consolidate utility functions
5. Update header guards to new naming scheme

**Files Created**:
- `core/BaseTypes.bt`
- `core/Enums.bt`
- `core/Utilities.bt`

### Phase 2: Chunk Modules

**Goal**: Reorganize structs into domain-specific modules.

**Tasks**:
1. Create `chunks/Header.bt` - unified Header, ChunkTable, ChunkTableEntry
2. Create `chunks/Mesh.bt` - all mesh-related structs
3. Create `chunks/Bones.bt` - all bone/skeleton structs
4. Create `chunks/Materials.bt` - material structures
5. Create `chunks/Animation.bt` - merge animation code
6. Create `chunks/Node.bt` - node structures

**Files Created**:
- `chunks/Header.bt`
- `chunks/Mesh.bt`
- `chunks/Bones.bt`
- `chunks/Materials.bt`
- `chunks/Animation.bt`
- `chunks/Node.bt`

### Phase 3: Display Functions

**Goal**: Consolidate all print functions into a single module.

**Tasks**:
1. Create `display/PrintFunctions.bt`
2. Merge all `Print*` functions from both helpers files
3. Remove duplicates (e.g., `PrintVector3` exists in both)
4. Standardize function naming

**Files Created**:
- `display/PrintFunctions.bt`

### Phase 4: Unified Entry Point

**Goal**: Create single `Cryengine.bt` that handles all formats.

**Tasks**:
1. Rewrite `Cryengine.bt` with format detection and unified parsing
2. Implement main switch/case using consolidated enums
3. Remove `CryEngine-Ivo.bt` as separate entry point
4. Test with all supported file types

**Structure**:
```c
// Cryengine.bt - Unified Entry Point
#include "core/BaseTypes.bt"
#include "core/Enums.bt"
#include "core/Utilities.bt"
#include "chunks/Header.bt"
#include "chunks/Mesh.bt"
#include "chunks/Bones.bt"
#include "chunks/Materials.bt"
#include "chunks/Animation.bt"
#include "chunks/Node.bt"
#include "display/PrintFunctions.bt"

local FileFormat detectedFormat;

// Read magic signature
local char sig[4];
ReadBytes(sig, 0, 4);

// Detect format
if (sig == "#ivo") {
    detectedFormat = StarCitizenIvo;
} else if (sig == "CryT") {
    detectedFormat = CryEngine34;
} else if (sig == "CrCh") {
    detectedFormat = CryEngine38;
}

// Parse header
Header header(detectedFormat);

// Parse chunk table
ChunkTable chunkTable(detectedFormat, header.numChunks);

// Main chunk parsing loop
local int i;
for (i = 0; i < header.numChunks; i++) {
    FSeek(chunkTable.entries[i].offset);

    switch (chunkTable.entries[i].chunkType) {
        // Format-aware chunk parsing
        case Mesh_Cry34:
        case Mesh_Cry38:
            MeshChunk_Cry mesh(detectedFormat, chunkTable.entries[i].version);
            break;
        case MeshChunk_Ivo:
            MeshChunk_Ivo mesh;
            break;
        // ... additional cases
    }
}
```

### Phase 5: Cleanup and Documentation

**Goal**: Remove old files, update documentation.

**Tasks**:
1. Delete deprecated files:
   - `CryEngine-Ivo.bt`
   - `Cryengine-Enums.bt`
   - `CryengineIvo-Enums.bt`
   - `Cryengine-Structs.bt`
   - `CryengineIvo-Structs.bt`
   - `Cryengine-Helpers.bt`
   - `CryengineIvo-Helpers.bt`
   - `Cryengine-Common.bt`
   - `CryEngineIvo-Animation.bt`
2. Update `README.md` for GitHub presentation
3. Update `CLAUDE.md` with new architecture
4. Create migration notes for existing users

---

## Testing Strategy

### Test Files Required

| Format | File Types | Example Games |
|--------|------------|---------------|
| CryEngine 3.4 | `.cgf`, `.chr`, `.skin`, `.caf` | MechWarrior Online, ArcheAge |
| CryEngine 3.8 | `.cgf`, `.chr`, `.skin` | Hunt: Showdown |
| Star Citizen Ivo | `.cgf`, `.skin`, `.caf`, `.dba` | Star Citizen |

### Validation Checklist

- [ ] CryT magic files parse correctly
- [ ] CrCh magic files parse correctly
- [ ] #ivo CGF files parse correctly
- [ ] #ivo SKIN files parse correctly
- [ ] #ivo CAF animation files parse correctly
- [ ] #ivo DBA animation database files parse correctly
- [ ] All chunk types display proper comments in 010 Editor
- [ ] No enum value collisions at runtime
- [ ] All Print* functions display correctly

---

## Risk Mitigation

### Backup Strategy
- Create `legacy/` branch before starting Phase 1
- Keep original files until Phase 5 is validated

### Rollback Plan
- Each phase should be a separate commit
- Tag releases at end of each phase
- Keep `legacy/` branch for emergency rollback

### Known Challenges

1. **010 Editor Include Path**: May need to adjust include paths for subdirectories
2. **Enum Switch Limitations**: 010 Editor may have limits on enum values in switch statements
3. **Backwards Compatibility**: Users with existing scripts referencing old struct names

---

## README.md Update Plan

The README should be updated to:

1. **Professional Header**: Project name, badges, brief description
2. **Quick Start**: How to use the template in 010 Editor
3. **Supported Formats**: Clear table of supported games/formats
4. **Architecture Overview**: Brief explanation with diagram
5. **Contributing**: Guidelines for adding new chunk types
6. **License**: Clear license information

### Proposed README Structure

```markdown
# CryEngine Binary Templates for 010 Editor

Parse CryEngine, Lumberyard, and O3DE game asset files.

## Supported Formats
- CryEngine 3.4 (MechWarrior Online, ArcheAge)
- CryEngine 3.8 (Hunt: Showdown)
- Star Citizen (#ivo format)

## Quick Start
1. Open 010 Editor
2. File > Open Template > Cryengine.bt
3. Open any supported binary file

## File Format Detection
| Magic Bytes | Format |
|------------|--------|
| `CryT` | CryEngine 3.4 |
| `CrCh` | CryEngine 3.8 |
| `#ivo` | Star Citizen |

## Architecture
[Brief architecture description with file layout]

## Contributing
[How to add new chunk types]

## License
[License info]
```

---

## Success Criteria

1. **Single Entry Point**: One `Cryengine.bt` handles all formats
2. **Clean Organization**: Logical file structure by domain
3. **No Enum Conflicts**: All enum values uniquely identifiable
4. **Consistent Naming**: All code follows naming conventions
5. **Full Compatibility**: All existing file types still parse correctly
6. **Documentation**: README and CLAUDE.md fully updated

---

## Appendix: File Mapping

### Old to New File Mapping

| Old File | New Location | Notes |
|----------|--------------|-------|
| `Cryengine.bt` | `Cryengine.bt` | Rewritten as unified entry |
| `CryEngine-Ivo.bt` | Merged into `Cryengine.bt` | Deleted |
| `Cryengine-BaseTypes.bt` | `core/BaseTypes.bt` | Renamed, conventions applied |
| `Cryengine-Enums.bt` | `core/Enums.bt` | Merged with Ivo enums |
| `CryengineIvo-Enums.bt` | `core/Enums.bt` | Merged |
| `Cryengine-Structs.bt` | `chunks/*.bt` | Split by domain |
| `CryengineIvo-Structs.bt` | `chunks/*.bt` | Split by domain |
| `Cryengine-Helpers.bt` | `display/PrintFunctions.bt` | Merged |
| `CryengineIvo-Helpers.bt` | `display/PrintFunctions.bt` | Merged |
| `Cryengine-Common.bt` | `core/Utilities.bt` | Renamed |
| `CryEngineIvo-Animation.bt` | `chunks/Animation.bt` | Merged |

---

*Last Updated: December 2024*
