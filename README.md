> **Edit from 2020** : I was recently made aware that the [ZIF file format](https://zif.photo/) is in fact a subset of [BigTiff](https://www.awaresystems.be/imaging/tiff/bigtiff.html). For end users, what this means is that it can easily be opened by many image viewing and manipulation softwares (such as [GIMP](https://www.gimp.org/)) by simply changing the file extension from `.zif` to `.tiff`.

# ZIF
zoomify ZIF (ZoomifyImageFormat) file format documentation and tools.

# Tools
## ZIF.js
This repository hosts a [implementation of the format in Ecmascript6](https://github.com/lovasoa/ZIF/blob/master/ZIF.es6). You can try it online here: http://lovasoa.github.io/ZIF/.

## Zif.reader
[@asherber](https://github.com/asherber) wrote a parser class for the format in C#, under the GPL. You can find it here: https://github.com/asherber/Zif.Reader

# About the format

The ZIF file format is used by zoomify as a single-file zoomable image format.
A zif file represents a single image. It contains JPEG *tiles*, little square images that represent a part of the image at a given zoom level, as well as meta-information about how the tiles are organized. [More about the concept behind zoomify](https://msdn.microsoft.com/en-us/library/cc645050%28VS.95%29.aspx).

![AN image and its tiles](http://www.zoomify.com/downloads/screenshots/tiledTiered.jpg)

# ZIF file format documentation
I have partially reverse-engineered the format. Here is what I found.

### Data types
All numbers are stored in [**little endian**](https://en.wikipedia.org/wiki/Endianness) (the number `0xABCD` is sored as `0xCD 0xAB`).

Term         | Signification
-------------|---------------
long         | 8 bytes unsigned int (uint64)
int          | 4 bytes unsigned int (uint32)
short        | 2 bytes unsigned int (uint16)
pointer      | a *long* representing an offset in bytes from the beginning of the file

## Header

### Magic bytes
The first 16 bytes of the file are always
```
49 49 2B 00 08 00 00 00
```

### Metadata
Meta-data start at offset `0x8` in the file.
There is a set of metadata for every zoom level (tier).

#### Zoomlevel metadata
Each set of metadata starts with a long **pointer** to a long number representing the number of tags (single metadata) in the zoom level. Usually this is a pointer to the offset `n + 8`, where `n` is the offset in bytes of the beggining of the metadata set (so, most often `*n == n+8`). Then, the real metadata start right after the metadata count (at `*n + 8`, which is usually equal to `n + 16`).

The last zoomlevel has its pointer to the next zoomlevel set to `0`.

Each tag is 20 bytes long.

Offset (from the start of the tag) | Length (in bytes) | Data
-----------------------------------|-------------------|------------------------------
0                                  | 2 (short)         | Magic number identifying the tag type
2                                  | 2                 | Unknown (but not null)
4                                  | 8  (long)         | **value 1**
12                                 | 8  (long)         | **value 2**

##### Tag types
Magic number in decimal | Magic bytes | Signification of **value 1** | Signification of **value 2**
-----|---|---|---
256  |`0x00 0x01`| ?                        | Image width at this zoomlevel
257  |`0x01 0x01`| ?                        | Image height at this zoomlevel
322  |`0x42 0x01`| ?                        | Tile width
323  |`0x43 0x01`| ?                        | Tile height
324  |`0x44 0x01`| Number of tiles          | Pointer to the **tile offsets index** (*see note below*)
325  |`0x45 0x01`| Number of tiles          | Pointer to the **tile sizes index** (*see note below*)

#### Note about tag 324 (tile offsets index pointer)
For tag 324, if value1 is 1 (there is only 1 tile in the zoomlevel), then **value2**
is not a pointer to a tile offsets index, but a tile index directly.

#### Note about tag 325 (tile sizes index pointer)
For tag 325, if value1 is inferior to 3 (there are only 1 or 2 tiles in the tier),
then **value2** is not a pointer to the tile sizes index, but is split in two int
values, representing the JPEG tile sizes in bytes directly. It is then as if the
tile sizes index were at the offset of **value2**.

### Tile offsets index
Simple list of pointers (8 bytes file offsets) to the beginning of tiles. Each tile is a simple JPEG file.

### Tile sizes index
Simple list of *int*s indicating tile sizes (in bytes).
