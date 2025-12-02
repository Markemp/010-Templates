# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains 010 Editor binary templates (.bt files) for analyzing CryEngine, Lumberyard, and Open 3D Engine (O3DE) game asset files. These templates parse binary file formats used by games like MechWarrior Online, ArcheAge, Hunt: Showdown, and Star Citizen.

010 Editor: https://www.sweetscape.com/010editor/

## Template Architecture

### Unified Entry Point

Use `CryengineUnified.bt` as the main entry point. It auto-detects file format via magic bytes:

| Magic Bytes | Format | Suffix | Example Games |
|-------------|--------|--------|---------------|
| `CryT` | CryTek | `_CryTek` | MechWarrior Online, ArcheAge |
| `CrCh` | CrChF | `_CrChF` | Hunt: Showdown |
| `#ivo` | Ivo | `_Ivo` | Star Citizen |

### Directory Structure

```
010-Templates/
├── CryengineUnified.bt      # Main entry point
├── core/                    # Foundation modules
│   ├── BaseTypes.bt         # Primitive types (Vector3, Matrix3x4, Quaternion, etc.)
│   ├── Enums.bt             # Unified enums with format suffixes
│   └── Utilities.bt         # Shared utilities (CryHalfToFloat, alignment)
├── chunks/                  # Domain-specific chunk parsers
│   ├── Header.bt            # File headers, chunk tables
│   ├── Mesh.bt              # Mesh, datastreams, vertices
│   ├── Bones.bt             # Skeleton, compiled bones
│   ├── Materials.bt         # Materials, physics, helpers
│   ├── Animation.bt         # Controllers, CAF/DBA
│   └── Node.bt              # Node hierarchy
├── display/
│   └── PrintFunctions.bt    # 010 Editor comment functions
└── [legacy files]           # Deprecated, kept for compatibility
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Structs | PascalCase | `MeshChunk_Cry`, `CompiledBones_Ivo` |
| Enums | PascalCase | `ChunkType_CryTek`, `DataStreamType_Ivo` |
| Enum Values | PascalCase + suffix | `Mesh_CryTek`, `MeshInfo_Ivo` |
| Functions | PascalCase | `PrintVector3`, `CryHalfToFloat` |
| Variables | camelCase | `numChunks`, `bytesPerElement` |
| Header Guards | SCREAMING_SNAKE + `_BT` | `CHUNKS_MESH_BT` |

### Format Suffixes

- `_CryTek` - CryTek format only (CryT header, uint32 chunk types)
- `_CrChF` - CrChF format only (CrCh header, uint16 chunk types)
- `_Ivo` - Ivo format only (#ivo header, hash-based chunk types)
- `_Cry` - Shared between CryTek and CrChF formats

## Key Patterns

### 1. Format-Parameterized Structs

Structures that vary by format accept a `FileFormat` parameter:

```c
struct Header(FileFormat format) {
    switch (format) {
        case CryTek:
            // CryTek-specific layout
            break;
        case CrChF:
            // CrChF-specific layout
            break;
        case Ivo:
            // Ivo-specific layout
            break;
    }
}
```

### 2. Separate Enums Per Format

Due to value collisions, chunk types are in separate enums:

```c
enum <uint32> ChunkType_CryTek {
    Mesh_CryTek = 0xCCCC0000,
    // ...
};

enum <uint16> ChunkType_CrChF {
    Mesh_CrChF = 0x1000,
    // ...
};

enum <uint32> ChunkType_Ivo {
    MeshInfo_Ivo = 0x92914444,
    // ...
};
```

### 3. Big/Little Endian Handling

Some CryTek format files (version 0x80000800) require endian toggling:

```c
if (chunkHeader.version == 0x80000800) {
    BigEndian();
}
// ... parse data ...
if (chunkHeader.version == 0x80000800) {
    LittleEndian();
}
```

### 4. Comment Functions

Every major struct has a corresponding Print* function for 010 Editor display:

```c
string PrintVector3(Vector3 &v) {
    string result;
    SPrintf(result, "%f, %f, %f", v.x, v.y, v.z);
    return result;
}
```

## Development Notes

### Adding New Chunk Types

1. Add enum value to `core/Enums.bt` with format suffix
2. Create struct in appropriate `chunks/*.bt` file
3. Add case to switch in `CryengineUnified.bt`

### Datastream Types

- CryTek/CrChF use sequential values (0x0-0xF)
- Ivo uses hash values (e.g., `0x91329AE9` = VertsUvs)
- Check `IsIvoDataStreamType()` helper function

### Alignment Helpers

- `AlignTo4()` - Align to 4-byte boundary
- `AlignTo8()` - Align to 8-byte boundary (common in Ivo)
- `AlignToN(n)` - Align to n-byte boundary

### Legacy Files

These are deprecated but kept for compatibility:
- `Cryengine.bt` - Old CryTek/CrChF entry point
- `CryEngine-Ivo.bt` - Old Ivo entry point
- `Cryengine-*.bt`, `CryengineIvo-*.bt` - Old module files
