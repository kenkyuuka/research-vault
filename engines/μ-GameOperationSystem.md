# Archives

mGOS archives are split across three files:

- A name file, `.nme`, which is a simple list of null-terminated strings in Shift-JIS encoding.
- A data file, `.det`, which simply contains blobs of (LZ77) compressed data, one after another, in arbitrary order and with no structure.
- An index file, `.atm` or `.at2`, which is a collection of records containing offsets to filenames in the `.nme` file and data in the `.det` file.

## Index file: .atm / .at2

Index files containing records that point into the name and data files to connect file paths to data.

The two file extensions refer to differing versions of the file format. `.at2` files have an extra field on each record containing the unpacked size of the file.

Note that the file extensions cannot be absolutely trusted. Heuristics may need to be used to determine the correct format.

### Entry layout (0x10 / `.atm`)


| Offset | Size | Type   | Field       | Notes                                 |
| ------ | ---- | ------ | ----------- | ------------------------------------- |
| 0x00   | 4    | int32  | name_offset | byte offset into the `.nme` file      |
| 0x04   | 4    | uint32 | data_offset | byte offset into the `.det` file      |
| 0x08   | 4    | uint32 | packed_size | size of the compressed data in `.det` |
| 0x0C   | 4    | uint32 | *unknown*   | probably a hash                       |

### Entry layout (0x14 / `.at2`)

| Offset | Size | Type   | Field         | Notes                                 |
| ------ | ---- | ------ | ------------- | ------------------------------------- |
| 0x00   | 4    | int32  | name_offset   | byte offset into the `.nme` file      |
| 0x04   | 4    | uint32 | data_offset   | byte offset into the `.det` file      |
| 0x08   | 4    | uint32 | packed_size   | size of the compressed data in `.det` |
| 0x0C   | 4    | uint32 | *unknown*     | probably a hash                       |
| 0x10   | 4    | uint32 | unpacked_size | size of the data after decompression  |

## Name file: .nme

The filenames contained in the archive, as a collection of null-terminated strings in Shift-JIS encoding, concatenated together. The strings do not have an (in-file) index or length recorded. The index file contains offsets of the beginnings of each filename, and the name goes on until `0x00` is found.

Filenames can contain (relative) path information, so the string might be `foo\bar.ext` to represent a file `bar.ext` contained in the directory `foo`. Windows-style backslashes are used as path separators.

The order of file names does not necessarily correspond to the order of entries in the index or data blobs.

The file ends with four bytes that are probably some kind of checksum, but the algorithm has not been identified.

## Data file: .det

These files have no structure, except ending with four bytes, presumably a checksum. Besides that, the file consists of concatenated blobs of compressed data, the offsets and lengths of which are recorded in the index files.

The order of data blobs is arbitrary and does not correspond to the order of entries in the name or index files.

### Compression

Compression is an LZ77 variant with a 64-byte lookback window. `0xFF` is used as a control character. A literal `0xFF` is encoded as `0xFFFF`. Otherwise, the second byte contains the distance to look back and the length to copy.

The first six bits encode the distance to look back: `distance := (second_byte >> 6) + 1`.

The remaining two bits encode the length to copy: `length := (second_byte & 0x03) + 3`. So the minimum length to copy is 3 (ensuring that at least one byte is saved by the copy) and the maximum is 6.