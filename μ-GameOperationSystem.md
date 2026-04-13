[TOC]
# Archives

mGOS archives are split across three files:

- A name file, `.nme`, which is a simple list of null-terminated strings in Shift-JIS encoding.
- A data file, `.det`, which simply contains blobs of data, one after another, in arbitrary order and with no structure. The data is compressed with an RLE algorithm.
- An index file, `.atm` or `.at2`, which is a collection of records containing offsets to filenames in the `.nme` file and data in the `.det` file.

## .nme

The filenames contained in the archive, as a collection of null-terminated strings in Shift-JIS encoding, concatenated together. The strings do not have an (in-file) index or length recorded. The `.atm`/`.at2` index file contains offsets of the beginnings of each filename, and the name goes on until `0x00` is found.

Filenames can contain (relative) path information, so the string might be `foo\bar.ext` to represent a file `bar.ext` contained in the directory `foo`. Windows-style backslashes are used as path separators.

The file ends with four bytes that are probably some kind of checksum, but the algorithm has not been identified.

## .det

These files have no structure, except ending with four bytes, presumably a checksum. Besides that, the file consists of concatenated blobs of data, RLE-compressed, the offsets and lengths of which are recorded in the `.atm`/`.at2` files.