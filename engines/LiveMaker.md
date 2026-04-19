[LiveMaker](https://livemaker.net/) is a free visual novel engine by [HUMAN BALANCE](https://humanbalance.net/).

# Identifying

`live.dll` will generally accompany the executable.

# Archives

LiveMaker archives (called 'VF' archives due to their magic bytes) can appear in three configurations:

| Configuration | Files                                                 | Description                                                                                                             |
| ------------- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Embedded      | `game.exe`                                            | Archive data appended to the game executable. A 6-byte trailer at the end of the file points to the archive start.      |
| Standalone    | `game.dat`                                            | Data file with the index at the beginning, or with a separate `game.ext` index file.                                    |
| Multi-part    | `game.dat` + `game.ext` + `game.001`, `game.002`, ... | Index in `.ext`, data spanning `.dat` and numbered overflow parts. Used when the total data exceeds single-file limits. |

All three configurations share the same index and data format — only the
location of the index and the base offset for file data differ.

All integers are little-endian.

## Embedded archives

Archives may be appended to the game's executable. In this case, the last 6 bytes of the file will be the offset of the archive followed by `lv`.

```
+---------------------------+
|  PE executable            |
+---------------------------+ <-- base_offset
|  VF archive (index+data)  |
+---------------------------+
|  base_offset  (uint32)    |
|  "lv"         (2 bytes)   |
+---------------------------+ <-- EOF
```

Since the VF archive is simply appended to the executable, the data can be modified by truncating the executable at `base_offset` and appending a new archive followed by the 6-byte trailer.
## Index

The index to VF archives may appear at the beginning of the archive, for embedded and standalone archives, or in a separate `game.ext` file. The index consists of a header followed by several sections.

The overall layout is:

```
+-------------------------------------------------------+
| Header (10 bytes)                                     |
|   "vf" | version (4) | num_files (4)                |
+-------------------------------------------------------+
| Names: num_files x [name_length (4) | name_bytes]   |
+-------------------------------------------------------+
| Offsets: (num_files+1) x int64                      |
+-------------------------------------------------------+
| Flags: num_files x uint8                            |
+-------------------------------------------------------+
| Timestamps: num_files x uint32                      |
+-------------------------------------------------------+
| CRCs: num_files x uint32                            |
+-------------------------------------------------------+
| Unknown: num_files x uint8 (v102+ only)             |
+-------------------------------------------------------+
| File data begins here                                 |
+-------------------------------------------------------+
```

For embedded/standalone archives, the file data begins immediately after the index. For standalone archives with a separate `.ext` file, the data starts at offset 0 in the `.dat` file.
### Header

| Offset | Size/Type  | Field                                       |
| ------ | ---------- | ------------------------------------------- |
| 0x00   | 2 (str)    | magic: `vf`                                 |
| 0x02   | 4 (uint32) | version: e.g. `102`                         |
| 0x06   | 4 (uint32) | num_files: number of files in the archive |

### Filenames

For each of the `num_files` files, the filename is stored in length-prefixed (uint32) Shift-JIS (encrypted). Each record looks like:

| Offset | Size/Type           | Field          |
| ------ | ------------------- | -------------- |
| 0x00   | 4 (uint32)          | name_length    |
| 0x04   | name_length (bytes) | encrypted_name |

Directory separators are backslashes (`\`). A single PRNG instance (seeded with initial state 0) is used across all names. The PRNG state carries forward from one name to the next without resetting.

### Data offsets

This section contains `num_files + 1` records of length 8 (int64). The extra record marks the end of the last file's data, allowing file sizes to be computed as `offset[i+1] - offset[i]`.

Each raw offset value is XORed with the PRNG output for decryption. The PRNG is reset to state 0 before reading offsets. The 32-bit PRNG output must be sign-extended to 64 bits before XORing with the 64-bit offset value. Treating it as unsigned produces incorrect results for offsets where the PRNG output has bit 31 set.

For embedded archives, the offsets are relative to the `base_offset`. Otherwise, they are absolute offsets into the `.dat` file or, for multi-file archives, into the concatenated data stream (`game.dat + game.001 + game.002 + ...`).

### Flags

`num_files` single-byte (uint8) values:

| value | meaning                  |
| ----- | ------------------------ |
| 0     | compressed (with zlib)   |
| 1     | uncompressed             |
| 2     | scrambled                |
| 3     | scrambled and compressed |

### Timestamps

`num_files` records, 4 bytes each, representing a DOS date/time.

| Bits  | Field       | Range                                    |
| ----- | ----------- | ---------------------------------------- |
| 31-25 | year - 1980 | 0-127 (1980-2107)                        |
| 24-21 | month       | 1-12                                     |
| 20-16 | day         | 1-31                                     |
| 15-11 | hour        | 0-23                                     |
| 10-5  | minute      | 0-59                                     |
| 4-0   | seconds / 2 | 0-29 (0-58 seconds, 2-second resolution) |

### CRC-32 hashes

`num_files` records, 4 bytes (uint32) each. Standard CRC-32 of the raw stored data (i.e. the compressed/scrambled bytes as they appear in the archive, not the decompressed content).

### Unknown bytes

`num_files` bytes, always 0 in all examined samples. Purpose unknown. Apparently present in v102 archives only.

## Encryption

The index uses a simple multiplicative PRNG to encrypt file names and obfuscate offsets.

The PRNG maintains a single uint32 state value, initialized to 0.

```python
def next(state):
    return ((state << 2) + state + 0x75D6EE39) & 0xFFFFFFFF
```

### Name decryption

Each byte of each file name is XORed with the low byte of the PRNG output:

```python
state = 0
for each record:
    for i in range(name_length):
        state = next(state)
        name_bytes[i] ^= state & 0xFF
```

The PRNG state carries across records. It is not reset between file names.

#### Example

Archive: `game.exe`, base offset `0x1E2C00`, version 102, 2666 files.

First file name decryption (`00000001.lsb`, 12 bytes):

```
PRNG state sequence: 0x75D6EE39, 0x4B1E5A56, 0x79D9B9E7, ...

Encrypted bytes:  09 66 d7 8c d5 82 83 89 ff 22 cc 96
XOR key bytes:    39 56 e7 bc e5 b2 b3 b8 d1 4e bf f4
Decrypted bytes:  30 30 30 30 30 30 30 31 2e 6c 73 62
ASCII:            0  0  0  0  0  0  0  1  .  l  s  b
```

### Offset decryption

Each 8-byte offset is XORed with the sign-extended PRNG output:

```python
state = 0
for i in range(num_files + 1):
    state = next(state)
    raw_offset = read_int64()
    key = sign_extend_32_to_64(state)
    decrypted_offset = raw_offset ^ key
```

The sign extension is critical: if the PRNG produces `0xF1234567`, it must be extended to `0xFFFFFFFFF1234567` (not `0x00000000F1234567`) before the XOR.

## Compression (flags 0 and 3)

Compression is standard zlib. Compress data before computing the CRC-32.

## Scrambling (flags 2 and 3)

Scrambled files have their data divided into fixed-size chunks that are reordered using a deterministic PRNG. An 8-byte header precedes the scrambled data:

| Offset | Size/Type  | Field      |
| ------ | ---------- | ---------- |
| 0x00   | 4 (int32)  | chunk_size |
| 0x04   | 4 (uint32) | raw_seed   |

The PRNG is seeded with `raw_seed ^ 0xF8EA`.

The data following this header (up to the next offset listed in the offsets section) is divided into chunks of the specified size (the last chunk may be smaller). These chunks are then reassembled in an order determined by the TpScramble PRNG.

### TpScramble PRNG

TpScramble is a combined multiply-with-carry generator with a 5-element state
array:

```python
class TpScramble:
    FACTOR_A = 2111111111
    FACTOR_B = 1492
    FACTOR_C = 1776
    FACTOR_D = 5115

    def __init__(self, seed):
        # Initialize state
        hash = seed if seed != 0 else 0xFFFFFFFF
        state = [0] * 5
        for i in range(5):
            hash ^= (hash << 13) & 0xFFFFFFFF
            hash ^= hash >> 17
            hash ^= (hash << 5) & 0xFFFFFFFF
            state[i] = hash
        self.state = state

        # Warm up
        for _ in range(19):
            self._next_uint32()

    def _next_uint32(self):
        s = self.state
        v = (self.FACTOR_A * s[3]
           + self.FACTOR_B * s[2]
           + self.FACTOR_C * s[1]
           + self.FACTOR_D * s[0]
           + s[4])
        s[3] = s[2]
        s[2] = s[1]
        s[1] = s[0]
        s[4] = v >> 32           # carry
        s[0] = v & 0xFFFFFFFF    # output
        return s[0]

    def next_int(self, first, last):
        """Return an integer in [first, last] inclusive."""
        r = self._next_uint32() / 0x100000000
        return first + int(r * (last - first + 1))
```

### Unscrambling

Given `count` chunks and a seed:

```python
tp = TpScramble(seed)
order = list(range(count))  # [0, 1, 2, ..., count-1]
sequence = [0] * count

for i in range(count - 1):
    n = tp.next_int(0, len(order) - 2)
    sequence[order[n]] = i
    order.pop(n)

sequence[order[0]] = count - 1  # last remaining element

output = [chunks[sequence[k]] for k in range(len(sequence))]
```

For flag 3 (scrambled + compressed), unscramble first, then decompress.

# File Types

LiveMaker archives contain several engine-specific file types alongside
standard formats:

| Extension | Description |
|-----------|-------------|
| `.lsb` | LiveMaker compiled script bytecode |
| `.lpb` | Project/metadata (contains game title, version info) |
| `.lpm` | Menu definition |
| `.gal` | LiveMaker image format (Gale); see below |
| `.lcm` | Live Cinema (movie/animation script) |
| `.cur` | Windows cursor |
| `.ogg` | Ogg Vorbis audio |
| `.wav` | Waveform audio |
| `.txt` | Plain text (Shift_JIS encoded) |
| `.dat` | Data files (e.g. `INSTALL.DAT`) |

# Gale image format

The `.gal` files use LiveMaker's proprietary image format, identified by the magic bytes `Gale` (`0x47 0x61 0x6C 0x65`) followed by a 3-digit ASCII version number (e.g. `Gale105`, `Gale106`, `Gale107`).

# References

- GARbro: `ArcFormats/LiveMaker/`
- crass: `cui/LiveMaker/LiveMaker.cpp`
- LiveMaker official site: https://livemaker.net/

# Open questions

- All examined files used v102 archives. The format of other versions has not been confirmed, and it is not known when new versions came into use, or how many different versions exist.
- The [Unknown bytes](#Unknown%20bytes) section, obviously, is not yet understood.
# Games

- [Otou-san no Kodane de Haramasete!](https://vndb.org/v19846) by Blue Devil (2016-08-14)
	- v102, embedded archive
- [TS Mahou Shoujo Nao!](https://vndb.org/v15223) by Crooked Navel (2014-05-09)
	- v102, multi-part archive