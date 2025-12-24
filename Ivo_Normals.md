# Ivo Tangent Frame Format (Star Citizen)

This document describes the 8-byte tangent frame format used in Star Citizen's #ivo geometry files (.cgfm) for encoding vertex tangent frames (TBN matrices).

## Overview

The tangent datastream in Ivo format uses 8 bytes per vertex (`bytesPerElement = 8`). The format encodes a full TBN (Tangent-Bitangent-Normal) matrix using two 32-bit values:

- **First 32 bits**: Smallest-three quaternion encoding the TBN rotation matrix
- **Second 32 bits**: Column selector indicating which TBN column represents the surface normal

## Data Layout

```
Offset  Size  Name    Description
0       4     value1  Smallest-three quaternion (10-10-10-2 format)
4       4     value2  Column selector + additional data (10-10-10-2 format)
```

Both values are little-endian uint32 composed from two uint16 words:
- `value1 = word0 | (word1 << 16)`
- `value2 = word2 | (word3 << 16)`

### Smallest-Three Quaternion Format (value1)

```
Bits 30-31: Dropped index (0-3, indicates which quaternion component was omitted)
Bits 20-29: Third component (10-bit signed integer)
Bits 10-19: Second component (10-bit signed integer)
Bits  0-9:  First component (10-bit signed integer)
```

### Column Selector Format (value2)

```
Bits 30-31: Column selector (d2) - determines which TBN column is the normal
Bits  0-29: Additional tangent frame data (interpretation TBD)
```

## Decoding Algorithm

### Step 1: Extract Raw Values

```csharp
uint value1 = word0 | ((uint)word1 << 16);
uint value2 = word2 | ((uint)word3 << 16);

int dropped1 = (int)((value1 >> 30) & 0x3);
int a_raw = (int)(value1 & 0x3FF);
int b_raw = (int)((value1 >> 10) & 0x3FF);
int c_raw = (int)((value1 >> 20) & 0x3FF);

int dropped2 = (int)((value2 >> 30) & 0x3);
```

### Step 2: Sign-Extend 10-bit Values

The 10-bit values are signed integers in two's complement form:

```csharp
if (a_raw > 511) a_raw -= 1024;
if (b_raw > 511) b_raw -= 1024;
if (c_raw > 511) c_raw -= 1024;
```

### Step 3: Normalize to Float Range

The values are normalized to the range [-0.707, 0.707] (approximately -1/sqrt(2) to 1/sqrt(2)):

```csharp
float scale = 722.0f;
float a = a_raw / scale;
float b = b_raw / scale;
float c = c_raw / scale;
```

### Step 4: Reconstruct Dropped Quaternion Component

Using the unit quaternion constraint (x² + y² + z² + w² = 1):

```csharp
float sumSq = a*a + b*b + c*c;
float d = (sumSq < 1.0f) ? MathF.Sqrt(1.0f - sumSq) : 0.0f;
```

### Step 5: Build Quaternion Based on Dropped Index

The dropped index indicates which component (x=0, y=1, z=2, w=3) was omitted:

```csharp
float qx, qy, qz, qw;
switch (dropped1) {
    case 0: qx = d;  qy = a;  qz = b;  qw = c;  break;  // X was dropped
    case 1: qx = a;  qy = d;  qz = b;  qw = c;  break;  // Y was dropped
    case 2: qx = a;  qy = b;  qz = d;  qw = c;  break;  // Z was dropped
    case 3: qx = a;  qy = b;  qz = c;  qw = d;  break;  // W was dropped
}
```

### Step 6: Extract TBN Matrix from Quaternion

The quaternion represents a rotation matrix. Extract the three columns (Tangent, Bitangent, Normal):

```csharp
// Tangent (first column of rotation matrix)
float Tx = 1.0f - 2.0f * (qy*qy + qz*qz);
float Ty = 2.0f * (qx*qy + qw*qz);
float Tz = 2.0f * (qx*qz - qw*qy);

// Bitangent (second column of rotation matrix)
float Bx = 2.0f * (qx*qy - qw*qz);
float By = 1.0f - 2.0f * (qx*qx + qz*qz);
float Bz = 2.0f * (qy*qz + qw*qx);

// Normal (third column of rotation matrix)
float Nx = 2.0f * (qx*qz + qw*qy);
float Ny = 2.0f * (qy*qz - qw*qx);
float Nz = 1.0f - 2.0f * (qx*qx + qy*qy);
```

### Step 7: Select Final Normal Based on Column Selector (d2)

The `dropped2` value (bits 30-31 of value2) indicates which TBN column represents the surface normal:

```csharp
Vector3 normal;
switch (dropped2) {
    case 0:
        // Negative Bitangent
        normal = new Vector3(-Bx, -By, -Bz);
        break;
    case 1:
        // Heuristic: use -N if normal is more horizontal, else -T
        if (MathF.Abs(Nz) < 0.5f) {
            normal = new Vector3(-Nx, -Ny, -Nz);
        } else {
            normal = new Vector3(-Tx, -Ty, -Tz);
        }
        break;
    case 2:
        // Positive Normal (direct)
        normal = new Vector3(Nx, Ny, Nz);
        break;
    case 3:
        // Negative Tangent
        normal = new Vector3(-Tx, -Ty, -Tz);
        break;
}
```

## Example Data (Box with 6 axis-aligned faces)

| Bytes (hex)               | value1     | value2     | d1 | d2 | Decoded Normal | Expected |
|---------------------------|------------|------------|----|----|----------------|----------|
| `FE FF FF 1F FF 3F 00 00` | 0x1FFFFFFE | 0x00003FFF | 0  | 0  | (0, 0, -1)     | (0, 0, -1) |
| `FE FF FF 9F FF 3F 00 80` | 0x9FFFFFFE | 0x80003FFF | 2  | 2  | (0, 0, +1)     | (0, 0, +1) |
| `FE FF FF 9F FF BF FF DF` | 0x9FFFFFFE | 0xDFFFBFFF | 2  | 3  | (0, -1, 0)     | (0, -1, 0) |
| `FF 3F FF BF FF BF FF DF` | 0xBFFF3FFF | 0xDFFFBFFF | 2  | 3  | (+1, 0, 0)     | (+1, 0, 0) |
| `FE FF FF 1F FF BF FF 5F` | 0x1FFFFFFE | 0x5FFFBFFF | 0  | 1  | (0, +1, 0)     | (0, +1, 0) |
| `FF 3F FF 3F FF BF FF 5F` | 0x3FFF3FFF | 0x5FFFBFFF | 0  | 1  | (-0.99, 0, 0)  | (-1, 0, 0) |

Note: The -X face decodes to approximately (-0.99, 0, 0) due to quantization. This is within acceptable tolerance.

## Column Selector Summary

| d2 | Column Used | Sign | Typical Use Case |
|----|-------------|------|------------------|
| 0  | Bitangent   | -    | Z-facing surfaces (floors/ceilings) |
| 1  | Heuristic   | -    | Mixed orientations |
| 2  | Normal      | +    | Standard surfaces |
| 3  | Tangent     | -    | Y-facing and some X-facing surfaces |

## Complete C# Implementation

```csharp
public struct IvoTangentFrame
{
    public ushort Word0;
    public ushort Word1;
    public ushort Word2;
    public ushort Word3;
}

public static class IvoTangentDecoder
{
    public static (Vector3 Normal, Vector3 Tangent, Vector3 Bitangent) Decode(IvoTangentFrame tf)
    {
        // Combine words into 32-bit values
        uint value1 = tf.Word0 | ((uint)tf.Word1 << 16);
        uint value2 = tf.Word2 | ((uint)tf.Word3 << 16);

        // Extract dropped index and raw components from first value
        int dropped1 = (int)((value1 >> 30) & 0x3);
        int a_raw = (int)(value1 & 0x3FF);
        int b_raw = (int)((value1 >> 10) & 0x3FF);
        int c_raw = (int)((value1 >> 20) & 0x3FF);

        // Sign-extend 10-bit values
        if (a_raw > 511) a_raw -= 1024;
        if (b_raw > 511) b_raw -= 1024;
        if (c_raw > 511) c_raw -= 1024;

        // Normalize to float range [-0.707, 0.707]
        const float scale = 722.0f;
        float a = a_raw / scale;
        float b = b_raw / scale;
        float c = c_raw / scale;

        // Reconstruct dropped component
        float sumSq = a * a + b * b + c * c;
        float d = (sumSq < 1.0f) ? MathF.Sqrt(1.0f - sumSq) : 0.0f;

        // Build quaternion based on dropped index
        float qx, qy, qz, qw;
        switch (dropped1)
        {
            case 0: qx = d;  qy = a;  qz = b;  qw = c;  break;
            case 1: qx = a;  qy = d;  qz = b;  qw = c;  break;
            case 2: qx = a;  qy = b;  qz = d;  qw = c;  break;
            default: qx = a; qy = b;  qz = c;  qw = d;  break;
        }

        // Extract TBN matrix columns from quaternion
        // Tangent (first column)
        float Tx = 1.0f - 2.0f * (qy * qy + qz * qz);
        float Ty = 2.0f * (qx * qy + qw * qz);
        float Tz = 2.0f * (qx * qz - qw * qy);

        // Bitangent (second column)
        float Bx = 2.0f * (qx * qy - qw * qz);
        float By = 1.0f - 2.0f * (qx * qx + qz * qz);
        float Bz = 2.0f * (qy * qz + qw * qx);

        // Normal (third column)
        float Nx = 2.0f * (qx * qz + qw * qy);
        float Ny = 2.0f * (qy * qz - qw * qx);
        float Nz = 1.0f - 2.0f * (qx * qx + qy * qy);

        Vector3 tangent = new Vector3(Tx, Ty, Tz);
        Vector3 bitangent = new Vector3(Bx, By, Bz);
        Vector3 normalColumn = new Vector3(Nx, Ny, Nz);

        // Select final normal based on column selector (d2)
        int dropped2 = (int)((value2 >> 30) & 0x3);
        Vector3 normal;

        switch (dropped2)
        {
            case 0:
                // Negative Bitangent
                normal = -bitangent;
                break;
            case 1:
                // Heuristic: -N if more horizontal, else -T
                if (MathF.Abs(Nz) < 0.5f)
                    normal = -normalColumn;
                else
                    normal = -tangent;
                break;
            case 2:
                // Positive Normal
                normal = normalColumn;
                break;
            case 3:
                // Negative Tangent
                normal = -tangent;
                break;
            default:
                normal = normalColumn;
                break;
        }

        return (normal, tangent, bitangent);
    }
}
```

## Technical Notes

### Why Smallest-Three?

The smallest-three quaternion compression takes advantage of the unit quaternion constraint. Since |q| = 1, only 3 of the 4 components need to be stored. The fourth can be reconstructed. By always dropping the largest component (which will have the smallest reconstruction error), you get better precision with fewer bits.

### Scale Factor (722.0)

The scale of 722.0 maps the 10-bit signed integer range [-512, 511] to approximately [-0.709, 0.707]. This covers the range needed for smallest-three encoding where each stored component is guaranteed to be <= 1/sqrt(2) ≈ 0.707.

### Column Selector Logic

The TBN matrix represents an orthonormal basis. For most surfaces, the third column (N) is the geometric normal. However, some surface orientations store the normal in a different column to optimize precision or handle edge cases. The d2 selector indicates which column to use.

### Limitations

- The d2=1 case uses a heuristic that may not be 100% accurate for all edge cases
- Very small rounding errors are possible due to 10-bit quantization (e.g., -0.99 instead of -1.0)
- Additional data in bits 0-29 of value2 is not fully decoded yet

## Related Files

- `chunks/Mesh.bt` - 010 Editor template implementation
- `display/PrintFunctions.bt` - Print function `PrintSmallestThreeQTangent()`
- `core/BaseTypes.bt` - `IvoTangentFrame` struct definition
