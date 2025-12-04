# Retroparquet File Format Specification

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** December 2024

## Table of Contents

1. [Introduction](#introduction)
2. [File Structure](#file-structure)
3. [Data Types](#data-types)
4. [Encoding](#encoding)
5. [Compression](#compression)
6. [Metadata](#metadata)
7. [Example](#example)

## Introduction

Retroparquet is a columnar storage file format optimized for efficient data storage and retrieval. It is designed to provide:

- **Column-oriented storage** for efficient querying of specific columns
- **Compression support** to reduce storage footprint
- **Schema evolution** capabilities
- **Type safety** with well-defined data types

## File Structure

A Retroparquet file consists of the following components:

```
+------------------+
| Magic Number     | 4 bytes: "RPQT"
+------------------+
| Version          | 2 bytes: Major.Minor
+------------------+
| Schema           | Variable length
+------------------+
| Row Groups       | Variable length (1 or more)
+------------------+
| Footer           | Variable length
+------------------+
| Footer Length    | 4 bytes
+------------------+
| Magic Number     | 4 bytes: "RPQT"
+------------------+
```

### Magic Number

Every Retroparquet file begins and ends with a 4-byte magic number: `RPQT` (0x52, 0x50, 0x51, 0x54).

### Version

A 2-byte version identifier consisting of:
- 1 byte for major version
- 1 byte for minor version

Current version: 1.0

### Schema

The schema section defines the structure of the data, including:
- Column names
- Column data types
- Compression settings per column
- Encoding settings per column

### Row Groups

Data is organized into row groups for efficient parallel processing. Each row group contains:
- Column chunks for each column
- Column metadata (encoding, compression, statistics)

### Footer

The footer contains:
- Schema information
- Row group metadata
- File-level metadata
- Key-value pairs for custom metadata

## Data Types

Retroparquet supports the following primitive data types:

| Type      | Description                    | Size        |
|-----------|--------------------------------|-------------|
| BOOLEAN   | True or false value            | 1 bit       |
| INT32     | 32-bit signed integer          | 4 bytes     |
| INT64     | 64-bit signed integer          | 8 bytes     |
| FLOAT     | 32-bit floating point          | 4 bytes     |
| DOUBLE    | 64-bit floating point          | 8 bytes     |
| BYTE_ARRAY| Variable-length byte array     | Variable    |
| STRING    | UTF-8 encoded string           | Variable    |

### Logical Types

In addition to primitive types, Retroparquet supports logical types:

- **DATE**: Days since Unix epoch (INT32)
- **TIME**: Milliseconds since midnight (INT64)
- **TIMESTAMP**: Milliseconds since Unix epoch (INT64)
- **DECIMAL**: Fixed-point decimal (BYTE_ARRAY with precision and scale)
- **UUID**: 128-bit universally unique identifier (BYTE_ARRAY of 16 bytes)

## Encoding

Retroparquet supports multiple encoding schemes for efficient storage:

### Plain Encoding

Direct storage of values without encoding. This is the simplest encoding and serves as the default.

### Dictionary Encoding

Values are replaced with integer indices into a dictionary. Efficient for columns with low cardinality.

### Run Length Encoding (RLE)

Sequences of repeated values are stored as a single value with a count. Efficient for columns with many repeated values.

### Delta Encoding

Stores the difference between consecutive values. Efficient for sorted or monotonically increasing columns.

### Bit-packed Encoding

Packs multiple small values into a single byte. Efficient for boolean and small integer values.

## Compression

Column chunks can be compressed using the following algorithms:

| Algorithm | Description                           |
|-----------|---------------------------------------|
| NONE      | No compression                        |
| SNAPPY    | Google's Snappy compression          |
| GZIP      | Standard GZIP compression            |
| ZSTD      | Zstandard compression                |
| LZ4       | LZ4 fast compression                 |

Compression is applied per column chunk, allowing different columns to use different compression algorithms based on their characteristics.

## Metadata

### File Metadata

File-level metadata includes:
- Schema version
- Created by (tool/library name and version)
- Number of rows
- Number of row groups
- Custom key-value pairs

### Column Metadata

Column-level metadata includes:
- Encoding used
- Compression codec used
- Number of values
- Null count
- Statistics (min, max, distinct count)

### Row Group Metadata

Row group metadata includes:
- Total byte size
- Number of rows
- Column chunk offsets
- Column chunk sizes

## Example

### Minimal Retroparquet File

Here's a conceptual example of a minimal Retroparquet file structure:

```
RPQT                          // Magic number
0x01 0x00                     // Version 1.0
[Schema Definition]           // Schema with 2 columns: id (INT32), name (STRING)
[Row Group 1]                 // Single row group
  [Column Chunk: id]          // Values: [1, 2, 3]
  [Column Chunk: name]        // Values: ["Alice", "Bob", "Charlie"]
[Footer]                      // Metadata about schema and row groups
0x000000B4                    // Footer length (180 bytes)
RPQT                          // Magic number
```

### Sample Data

Consider a simple dataset:

| id | name    | age |
|----|---------|-----|
| 1  | Alice   | 30  |
| 2  | Bob     | 25  |
| 3  | Charlie | 35  |

This would be stored in Retroparquet as three separate column chunks within a row group:
- `id` column: [1, 2, 3] (INT32, plain encoding)
- `name` column: ["Alice", "Bob", "Charlie"] (STRING, dictionary encoding)
- `age` column: [30, 25, 35] (INT32, plain encoding)

## Implementation Notes

### Reading Retroparquet Files

1. Read and verify the magic number at the start of the file
2. Read the version to ensure compatibility
3. Seek to the end and read the footer length
4. Read the footer to get schema and row group metadata
5. Read column chunks as needed (projection pushdown)
6. Decompress and decode column data

### Writing Retroparquet Files

1. Write the magic number and version
2. Write the schema definition
3. For each row group:
   - Encode and compress column chunks
   - Write column chunk data
   - Collect metadata (statistics, offsets, sizes)
4. Write the footer with all metadata
5. Write the footer length and closing magic number

## Future Considerations

- Support for nested data types (structs, arrays, maps)
- Encryption support for sensitive data
- Bloom filters for efficient filtering
- Page-level organization within column chunks
- Support for predicate pushdown optimizations

## References

- Apache Parquet Format: https://parquet.apache.org/docs/
- Columnar Storage: https://en.wikipedia.org/wiki/Column-oriented_DBMS
- Compression Algorithms: Various RFC and specification documents

---

**Note:** This specification is subject to change as the format evolves. Please refer to the version number for compatibility information.
