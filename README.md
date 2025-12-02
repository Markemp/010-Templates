# CryEngine Binary Templates for 010 Editor

Binary templates for [010 Editor](https://www.sweetscape.com/010editor/) to parse CryEngine, Lumberyard, and Open 3D Engine game asset files.

## Supported Formats

| Format | Magic Header | Games |
|--------|--------------|-------|
| **CryTek** | `CryT` | MechWarrior Online, ArcheAge |
| **CrChF** | `CrCh` | Hunt: Showdown |
| **Ivo** | `#ivo` | Star Citizen |

### Supported File Types

- `.cgf` - Geometry files
- `.chr` - Character files
- `.skin` - Skinned mesh files
- `.caf` - Animation files
- `.dba` - Animation database files

## Quick Start

1. Open [010 Editor](https://www.sweetscape.com/010editor/)
2. Open a supported binary file (`.cgf`, `.chr`, `.skin`, `.caf`, `.dba`)
3. Run Template > Open Template > `CryengineUnified.bt`

The template auto-detects the format based on the file's magic header bytes.

## Architecture

```
010-Templates/
├── CryengineUnified.bt      # Unified entry point (use this!)
├── core/
│   ├── BaseTypes.bt         # Primitive types (Vector3, Matrix, etc.)
│   ├── Enums.bt             # Unified enums with format suffixes
│   └── Utilities.bt         # Shared utility functions
├── chunks/
│   ├── Header.bt            # File headers and chunk tables
│   ├── Mesh.bt              # Mesh structures and datastreams
│   ├── Bones.bt             # Skeleton and bone structures
│   ├── Materials.bt         # Material and physics structures
│   ├── Animation.bt         # Animation controllers (CAF/DBA)
│   └── Node.bt              # Node hierarchy structures
└── display/
    └── PrintFunctions.bt    # 010 Editor display functions
```

### Legacy Entry Points

The following files are maintained for backwards compatibility but are deprecated:

- `Cryengine.bt` - Original CryTek/CrChF parser
- `CryEngine-Ivo.bt` - Original Ivo parser

## Naming Convention

Enums and structs use format-based suffixes:

- `_CryTek` - CryTek format (CryT header)
- `_CrChF` - CrChF format (CrCh header)
- `_Ivo` - Ivo format (#ivo header)
- `_Cry` - Shared by CryTek and CrChF formats

## Adding New Chunk Types

1. Identify which format(s) the chunk belongs to
2. Add the chunk type enum value to `core/Enums.bt` with appropriate suffix
3. Create the struct in the appropriate `chunks/*.bt` file
4. Add the case to the switch statement in `CryengineUnified.bt`

## Contributing

Feel free to open issues or pull requests if you identify new chunks or data structures.

## License

MIT License - See LICENSE file for details.
