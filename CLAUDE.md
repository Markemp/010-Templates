# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains 010 Editor binary templates (.bt files) for analyzing CryEngine, Lumberyard, and Open 3D Engine (O3DE) game asset files. These templates parse binary file formats used by games like MechWarrior Online, ArcheAge, Hunt: Showdown, and Star Citizen.

010 Editor: https://www.sweetscape.com/010editor/

## Template Architecture

### Entry Points

Three main entry point templates exist for different file formats:

- **Cryengine.bt** - Universal entry point that auto-detects file type via magic bytes:
  - `CryT` = CryEngine 3.4 format (MWO, ArcheAge)
  - `CrCh` = CryEngine 3.8 format (Hunt: Showdown)
  - `#ivo` = Star Citizen custom format (redirects to CryEngine-Ivo.bt)

- **CryEngine-Ivo.bt** - Dedicated parser for Star Citizen #ivo files with different chunk structure

- **CryEngine-CryXmlB.bt** / **Cryengine-pbxml.bt** - Binary XML format parsers

### Shared Module Structure

Templates use `#include` directives and header guards (`#ifndef/#define/#endif`) for modularity:

| File | Purpose |
|------|---------|
| `Cryengine-BaseTypes.bt` | Primitive types (VECTOR3, QUATERNION, MATRIX3x4, etc.) with custom read functions for normalized types (snorm, unorm) |
| `Cryengine-Enums.bt` | Chunk types (CHUNKTYPE_UI, CHUNKTYPE_US), data stream types, vertex formats, compression formats |
| `Cryengine-Structs.bt` | Main data structures (HEADER, CHUNKTABLE, MESHCHUNK, COMPILEDBONE, CONTROLLER_829/905, etc.) |
| `Cryengine-Helpers.bt` | Print/comment functions for displaying parsed data in 010 Editor |
| `Cryengine-Common.bt` | Shared utilities like CryHalfToFloat conversion |

Star Citizen #ivo files have parallel modules:
- `CryengineIvo-Enums.bt` - Different chunk types (0x-prefixed identifiers from reverse engineering)
- `CryengineIvo-Structs.bt` - Ivo-specific structures (SKINMESH, NODEMESHCOMBOCHUNK, etc.)
- `CryengineIvo-Helpers.bt` - Ivo-specific print functions

### Key Patterns

1. **Version-aware parsing**: Structures accept `FILETYPE` or version parameters to handle format differences:
   ```c
   struct HEADER(FILETYPE fileType) {
       if (fileType == CRYENGINE_3_4) { ... }
       else if (fileType == CRYENGINE_3_8) { ... }
   }
   ```

2. **Chunk-based format**: Files contain a header with chunk table, then chunks at specified offsets. Main parsing loop iterates chunk table and switches on chunk type.

3. **Big/Little Endian handling**: Some formats (version 0x80000800) require `BigEndian()`/`LittleEndian()` toggling.

4. **Comment functions**: Every major struct has a corresponding `Print*` function for 010 Editor display (e.g., `PrintVector3`, `PrintBoneName`).

## Development Notes

- When adding new chunk types, update both the enum and add a case in the main switch statement
- Ivo datastream types are identified by magic uint32 values (e.g., `0x91329AE9` = IvoVertsUvs)
- Alignment helpers exist: `AlignTo4()`, `AlignTo8ByteBoundary()`
- Use `<comment=FunctionName>` and `<bgcolor=color>` attributes for 010 Editor visualization

## Planned Work (from README)

- Consolidate the 3 entry points into a single root template
- Split structs and common functions into more logical structure
