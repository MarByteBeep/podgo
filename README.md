# Installation

## Download and install Firmware 1.30

First download the Flash Memory file for firmware 1.30:

https://line6.com/getrelease?rid=10653&uid=3499415

This is a HXF file named `POD_Go_v1.30.0_0x01b781c4_bundle.hxf`

## Line 6 Updater output

When installing this HFX file using the `Line 6 Updater`, I captured the following output:

```

update-- flashDataSize 16125120
update-- imagesInArchive 2, selectedImage 0
update-- chunk[0] : type 0, version 0x10000, offset 0, length 16028336
update-- chunk[1] : type 5, version 0x10000, offset 16028336, length 96784
Version comparison - Archive[0] type kImageType_MainCode, 0x01300000 : Device 0x01300000
Version comparison - Archive[1] type kImageType_Boot, 0x01300000 : Device 0x00000000
Version comparison - Archive[2] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[3] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[4] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[5] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[6] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[7] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[8] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[9] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[10] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[11] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[12] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[13] type kImageType_Data, 0x01300000 : Device 0x01300000
Version comparison - Archive[14] type kImageType_Data, 0x01300000 : Device 0x01300000
```

This served as extra information for decoding the fileformat. I can see that there are 2 chunks/images in the archive. And in total there are 15 sections: 1 code section, 1 boot section and 13 data sections. Each of them has a version code and a device code.

# HXF File format

When viewing the HFX file in a hex editor, the first 160 bytes of that file are:

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00000000  46 4F 52 4D 00 29 12 A2 4C 36 46 46 48 45 41 44 00 00 00 34 00 01 00 00 00 01 00 00 00 21 00 07  FORM.).¢L6FFHEAD...4.........!..
00000020  00 00 00 01 00 F6 0C C0 FB CF B4 18 09 3A E5 9C F2 4D 45 B4 BC E1 87 E0 00 00 00 00 00 00 00 00  .....ö.ÀûÏ´..:åœòME´¼á‡à........
00000040  00 00 00 00 00 00 00 00 64 69 6E 66 00 00 00 10 00 00 00 00 00 01 00 00 00 00 00 00 00 F4 92 B0  ........dinf.................ô’°
00000060  64 69 6E 66 00 00 00 10 00 00 00 05 00 01 00 00 00 F4 92 B0 00 01 7A 10 64 61 74 61 00 29 12 2A  dinf.............ô’°..z.data.).*
00000080  78 DA EC 7C 79 7C 13 D7 BD EF EF 8C 46 8B 65 63 CB 88 45 5E 70 46 1A 1B 6C 0B 88 17 92 1A 9C DE  xÚì|y|.×½ïïŒF‹ecËˆE^pF..l.ˆ.’.œÞ
....
```

The values in this file seem to be in big endian.

One pattern at offset `0x80` stands out: `0x78 0xDA`, which is indicative for zlib compression. Which leads me to believe that the first `0x80` bytes is the header. And the remaining part of the file is the actual compressed firmware.

## HXF Header

Initial attempt at decoding the header. Offset and size values are in hex.

```
[offset]          [size] [content]
00000000-00000003 4      'FORM', magic marker
00000004-00000007 4      ?
00000008-0000000B 4      `L6FF`, magic marker: Line6 File Format?
0000000C-0000000F 4      `HEAD`, magic marger indicating Header
00000010-00000023 14     ?? various header settings, need help on this
00000024-00000027 4      uncompressed flash file size (0x00 F6 0C C0), which is 16125120 bytes which is the number reported by the Line 6 updater output
00000028-00000047 20     ?? various header settings, need help on this
00000048-0000005F 18     `dinf` section (see below)
00000060-00000077 18     `dinf` section (see below)
00000078-EOF      --     `data` section (see below)
```

### `dinf` section

```
[offset]          [size] [content]
00000000-00000003 4      'dinf', magic marker
00000004-00000007 4      ?? seems to be 0x00 00 00 10
00000008-0000000B 4      type, can either be 0x0 or 0x5. 0x0 = chunk start, 0x5 = chunk continue?
0000000C-0000000F 4      version, seems to be 0x10000
00000010-00000013 4      chunk offset in bytes
00000014-00000017 4      chunk size in bytes
```

### `data` section

```
[offset]          [size] [content]
00000000-00000003 4      'data', magic marker
00000004-00000007 4      compressed zlib size in bytes (size)
00000008-EOF      size   remaining file is the zlib compressed firmware which starts with the magic marker `0x78 0xDA`
```

### Notes

The zlib part is a single 'blob' of data, which decompresses into a file of 16125120 bytes. I suspect the chunks to be merely there for USB uploading limits: the updater splits up the decompressed blob into two chunks and sends them over the wire. After this process the Pod GO will recombine the received data with the chunk information it got. At least this is what I think.

# L6IA Archive File

After decoding the zlib part we end up with the actual firmware, which again has a header and a total of 16 archives. From what I can see it's basically a small 24 bytes header with 16 archive sections back to back. Each of them with their own header and data sections. In total there seem to be 16 sections, although the Line 6 updater output seems to mention only 15 (0-14). Not sure why that is.

When viewing the decompressed file in a hex editor, the first 24 bytes of that file are:

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17

00000000  4C 36 49 41 00 00 00 02 00 00 00 01 00 00 00 01 00 00 00 40 00 00 00 10 L6IA...............@....
```

## L6IA Header

```
[offset]          [size] [content]
00000000-00000003 4      'L6IA', magic marker: Line 6 I? Archive
00000004-00000013 10     ??
00000014-00000017 4      Number of archives sections (16)?
```

## Archive Sections

The remainder of the archive seems to have all archive sections back to back. Each section has a 548 byte header (mostly zero padded) and a data section. These are first bytes of each of the sections:

```
[id] [offset]   [header 0x224 (548) bytes] ... [data]
0    [00 00 18] 00 00 00 00 01 30 00 00 00 00 00 02 00 00 00 04 00 02 8E 2B 7E 69 D8 BE 2A 53 F3 CA 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 DF BC 51 1E 00 0A 36 AC 00 00 ...
1    [0A 38 E8] 00 00 00 01 01 30 00 00 00 00 00 01 00 00 00 04 00 01 16 7B 27 88 46 F4 C2 23 56 2B 00 00 00 00 00 00 00 00 56 11 26 03 01 00 01 61 81 18 F6 C7 00 04 57 EC 00 00 ...
2    [0E 92 F8] 00 00 00 02 01 30 00 00 00 00 00 06 00 00 00 04 00 01 57 CF 34 10 0D B0 2C 65 9E 2E 63 70 4C 34 00 00 00 00 56 11 26 03 01 30 00 00 27 E8 E8 F8 00 05 5D 3C 00 00 ...
3    [13 F2 58] 00 00 00 03 01 30 00 00 00 00 00 06 00 00 00 04 00 01 A6 AB 72 AF C9 3F 40 DB 76 7D 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 98 BA 37 94 00 06 98 AC 00 00 ...
4    [1A 8D 28] 00 00 00 04 01 30 00 00 00 00 00 06 00 00 00 04 00 1F 3C 68 AD E6 19 6F FE 42 06 B0 20 0A 0A 3E 00 00 00 00 56 11 26 03 01 30 00 00 6D 56 24 A0 00 7C EF A0 00 00 ...
5    [97 7E EC] 00 00 00 05 01 30 00 00 00 00 00 06 00 00 00 04 00 00 10 33 04 29 DB 22 73 2C 4D 52 5F 34 33 50 00 00 00 00 56 11 26 03 01 30 00 00 DF 81 F1 31 00 00 3E CC 00 00 ...
6    [97 BF DC] 00 00 00 06 01 30 00 00 00 00 00 06 00 00 00 04 00 00 19 6F C3 FD FF F9 E7 F2 D3 C8 70 79 74 20 00 00 00 00 56 11 26 03 01 30 00 00 DA B7 4F 3B 00 00 63 BC 00 00 ...
7    [98 25 BC] 00 00 00 07 01 30 00 00 00 00 00 06 00 00 00 04 00 00 2F 15 23 AE 8B BA BB 82 52 2B 7A 69 73 64 00 00 00 00 56 11 26 03 01 30 00 00 F7 07 40 69 00 00 BA 54 00 00 ...
8    [98 E2 34] 00 00 00 08 01 30 00 00 00 00 00 06 00 00 00 04 00 11 DD A9 59 1E 68 D4 62 33 74 22 33 2E 31 22 00 00 00 00 56 11 26 03 01 30 00 00 43 8C B3 09 00 47 74 A4 00 00 ...
9    [E0 58 FC] 00 00 00 09 01 30 00 00 00 00 00 06 00 00 00 04 00 00 35 0E 8C 22 2F BC 51 C8 E1 83 20 0A 3E 2F 00 00 00 00 56 11 26 03 01 30 00 00 04 0B A2 8A 00 00 D2 38 00 00 ...
A    [E1 2D 58] 00 00 00 0A 01 30 00 00 00 00 00 06 00 00 00 04 00 00 00 A1 0D 0B 8C 6D 7F 14 BA ED 73 22 3D 65 00 00 00 00 56 11 26 03 01 30 00 00 6D 32 4D 0A 00 00 00 84 00 00 ...
B    [E1 30 00] 00 00 00 0B 01 30 00 00 00 00 00 06 00 00 00 04 00 00 01 3B 13 48 9D F6 FA 3A CF C8 64 65 72 61 00 00 00 00 56 11 26 03 01 30 00 00 7D 12 3B 34 00 00 02 EC 00 00 ...
C    [E1 35 10] 00 00 00 0C 01 30 00 00 00 00 00 06 00 00 00 04 00 00 07 55 E4 83 25 70 BD 40 67 34 00 00 CD 81 00 00 00 00 56 11 26 03 01 30 00 00 20 7D AD A0 00 00 1B 54 00 00 ...
D    [E1 52 88] 00 00 00 0D 01 30 00 00 00 00 00 06 00 00 00 04 00 01 CA 5B B4 DA B8 BC D3 18 27 3D 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 6A 03 DC 33 00 07 27 6C 00 00 ...
E    [E8 7C 18] 00 00 00 0E 01 30 00 00 00 00 00 06 00 00 00 04 00 02 E2 7E B3 A1 85 26 62 28 A1 07 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 C7 33 25 44 00 0B 87 F8 00 00 ...
F    [F4 06 34] 00 00 00 0F 01 30 00 00 00 00 00 06 00 00 00 04 00 00 23 16 98 81 85 58 ED E3 4C 2A 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 47 57 6C 87 00 00 8A 58 00 00 ...
```

## Archive Section Header

This part still is much WIP. Each section seems to have an id, version numbers, type, some CRCs?, and section sizes defined. This is what I know so far

```
[offset]          [size] [content]
00000000-00000003 4      section number (0x00-0x0F, 16 sections in total)
00000004-00000007 4      version: 0x01 30 00 00 (fw v1.30)
00000008-0000000B 4      I suspect this to be kImageType: kImageType_MainCode = 0x2, kImageType_Boot = 0x1 and kImageType_Data = 0x6
0000000C-0000000F 4      ?? seems to be 0x00 00 00 04 all the time
00000010-00000023 14     ??
00000024-00000027 4      ?? seems to be 0x56 11 26 03 all the time
00000028-0000002B 4      ? device version id: seems to be 0x01 30 00 00 (fw v1.30) in case of `kImageType_MainCode` and `kImageType_Data` and `0x01 00 01 61` for `kImageType_Boot`*
0000002C-0000002F 4      ?? Some CRC?
00000030-00000033 4      Section size in bytes (size)
00000034-00000223 1F0    Zero padded remainder of the header
00000224-EOS      size   Remainder of section (offset `0x224` to End of Section); this is the data part of each section**


*) The output reports a device version of `0x00000000` for `kImageType_Boot`, yet I cannot see this value in the header
**) I currently have no clue what is in each of these data sections. More on this below
```

## Archive Section Data

The data part of each archive section seems to be obfuscated, compressed and/or encrypted. Possibly with keys that are present in each of the headers (e.g., the data in `00000010-00000023` or `0000002C-0000002F` could hold the key(s) to decrypting this)

For each of the 16 archive sections I'll post the header and part of the data (i.e., the first `0x100` bytes), hopefully someone is able to detect some familiar patterns.

One thing that I noticed is that in the data section the size sometimes seems to reappear. See for instance:

-   Section 0 (offset 0x4)
-   Section 5 (offset 0x8)
-   Section 7 (offset 0x8)
-   Section 8 (offset 0x14)

### Section 0

#### Header

```
00 00 00 00 01 30 00 00 00 00 00 02 00 00 00 04 00 02 8E 2B 7E 69 D8 BE 2A 53 F3 CA 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 DF BC 51 1E 00 0A 36 AC

Offset: 0x00 00 18
Type: 0x2 kImageType_MainCode
Size: 0x00 0A 36 AC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00000220                                                                                      23 0B 0C 5E                              #..^
00000240  00 0A 36 AC 54 4C D6 5B 00 00 00 02 00 00 00 07 28 B1 D2 C1 10 00 16 B0 00 00 00 00 00 00 00 00  ..6¬TLÖ[........(±ÒÁ...°........
00000260  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 E4 10 00 00 00 00 00 01 24 10 08 20 AC  ...................ä.......$.. ¬
00000280  28 B1 D2 C1 28 B1 D3 69 10 00 0A 75 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 24 84 DF 9A 00 00 00 00  (±ÒÁ(±Ói...u(±Ói(±Ói(±Ói$„ßš....
000002A0  00 00 00 00 00 00 00 00 28 B1 D3 69 28 B1 D3 69 00 00 00 00 10 00 09 F1 10 00 06 95 28 B1 D3 69  ........(±Ói(±Ói.......ñ...•(±Ói
000002C0  28 B2 9F 1D 28 B2 7A D1 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 28 B2 B5 75 28 B2 A1 5D 28 B1 D3 69  (²Ÿ.(²zÑ(±Ói(±Ói(±Ói(²µu(²¡](±Ói
000002E0  28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 28 B2 D9 95 28 B2 D9 BD 28 B2 D9 E5 28 B2 DA 0D 28 B1 D3 69  (±Ói(±Ói(±Ói(²Ù•(²Ù½(²Ùå(²Ú.(±Ói
00000300  28 B1 D3 69 28 B3 4D 29 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 10 00 01 21  (±Ói(³M)(±Ói(±Ói(±Ói(±Ói(±Ói...!
00000320  10 00 01 D9 10 00 02 91 10 00 03 49 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69 28 B1 D3 69              ...Ù...‘...I(±Ói(±Ói(±Ói(±Ói
```

### Section 1

#### Header

```
00 00 00 01 01 30 00 00 00 00 00 01 00 00 00 04 00 01 16 7B 27 88 46 F4 C2 23 56 2B 00 00 00 00 00 00 00 00 56 11 26 03 01 00 01 61 81 18 F6 C7 00 04 57 EC

Offset: 0x0A 38 E8
Type: 0x1 kImageType_Boot
Size: 0x00 04 57 EC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

000A3B00                                      10 08 06 9C 14 00 19 91 14 00 1A 89 14 00 4F C1 14 00 1A 89              ...œ...‘...‰..OÁ...‰
000A3B20  14 00 1A 89 14 00 1A 89 8B F6 5C 4A 00 00 00 00 00 00 00 00 00 00 00 00 14 00 1A 89 14 00 1A 89  ...‰...‰‹ö\J...............‰...‰
000A3B40  00 00 00 00 14 00 1B E1 14 00 53 ED 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89  .......á..Sí...‰...‰...‰...‰...‰
000A3B60  14 00 1A 89 14 00 33 4D 14 00 23 E5 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89  ...‰..3M..#å...‰...‰...‰...‰...‰
000A3B80  14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89  ...‰...‰...‰...‰...‰...‰...‰...‰
000A3BA0  14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89  ...‰...‰...‰...‰...‰...‰...‰...‰
000A3BC0  14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89  ...‰...‰...‰...‰...‰...‰...‰...‰
000A3BE0  14 00 1A 89 14 00 1A 89 14 00 1A 89 14 00 1A 89 F0 00 B5 08 E8 BD F8 05 F0 00 40 08 BF 00 B8 19  ...‰...‰...‰...‰ð.µ.è½ø.ð.@.¿.¸.
000A3C00  4C 09 B5 10 B1 03 88 23 F0 00 BD 10                                                              L.µ.±.ˆ#ð.½.
```

### Section 2

#### Header

```
00 00 00 02 01 30 00 00 00 00 00 06 00 00 00 04 00 01 57 CF 34 10 0D B0 2C 65 9E 2E 63 70 4C 34 00 00 00 00 56 11 26 03 01 30 00 00 27 E8 E8 F8 00 05 5D 3C

Offset: 0x0E 92 F8
Type: 0x6 kImageType_Data
Size: 0x00 05 5D 3C bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

000E9520                                                                                91 C0 08 04 BE 06                            ‘À..¾.
000E9540  00 00 00 00 7B 0F 00 00 00 00 7A 0F 00 10 00 00 02 14 00 00 00 00 30 0F 00 00 00 00 34 0F 00 00  ....{.....z...........0.....4...
000E9560  00 00 38 0F 00 00 00 00 3C 0F 00 00 00 00 3F 0F 00 00 00 00 25 0F 01 00 00 00 26 0F 00 00 00 00  ..8.....<.....?.....%.....&.....
000E9580  2D 0F 01 00 00 00 2E 0F 24 00 03 00 0B 10 B0 1C 02 00 3E 01 24 00 03 00 14 0F 31 C0 08 00 BE 06  -.......$.....°...>.$.....1À..¾.
000E95A0  03 20 09 00 08 10 04 20 09 00 02 10 05 20 09 00 03 10 80 10 02 00 3E 01 7C C0 08 00 00 06 00 A0  . ..... ..... ....€...>.|À..... 
000E95C0  02 00 3E 01 3D C0 08 00 00 06 00 A0 02 00 3E 01 43 C0 08 00 00 06 00 A0 02 00 3E 01 48 C0 08 00  ..>.=À..... ..>.CÀ..... ..>.HÀ..
000E95E0  00 06 00 A0 02 00 3E 01 59 C0 08 00 00 06 00 A0 02 00 3E 01 60 C0 08 00 00 06 00 A0 02 00 3E 01  ... ..>.YÀ..... ..>.`À..... ..>.
000E9600  71 C0 08 00 00 06 00 A0 02 00 3E 01 7A C0 08 00 00 06 00 A0 02 00 3E 01 7A C0 08 00 00 06 00 A0  qÀ..... ..>.zÀ..... ..>.zÀ..... 
000E9620  02 00 3E 01 7B C0 08 00 00 06 00 A0 02 00 3E 01 7B C0 08 00 00 06 00 00 00 00                    ..>.{À..... ..>.{À........
```

### Section 3

#### Header

```
00 00 00 03 01 30 00 00 00 00 00 06 00 00 00 04 00 01 A6 AB 72 AF C9 3F 40 DB 76 7D 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 98 BA 37 94 00 06 98 AC

Offset: 0x13 F2 58
Type: 0x6 kImageType_Data
Size: 0x00 06 98 AC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

0013F460                                                                                      40 23 24 5E                              @#$^
0013F480  00 00 00 04 00 06 98 AC 00 00 02 00 16 37 2E 90 00 00 FF 2E 00 01 01 30 00 00 01 EE 00 01 6D 40  ......˜¬.....7....ÿ....0...î..m@
0013F4A0  00 00 00 48 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ...H............................
0013F4C0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
0013F4E0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
0013F500  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
0013F520  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
0013F540  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
0013F560  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00              ............................
```

### Section 4

#### Header

```
00 00 00 04 01 30 00 00 00 00 00 06 00 00 00 04 00 1F 3C 68 AD E6 19 6F FE 42 06 B0 20 0A 0A 3E 00 00 00 00 56 11 26 03 01 30 00 00 6D 56 24 A0 00 7C EF A0

Offset: 0x1A 8D 28
Type: 0x6 kImageType_Data
Size: 0x00 7C EF A0 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

001A8F40                                      28 26 7E 5E 00 00 00 04 00 00 01 EE 00 00 00 06 00 00 03 E2              (&~^.......î.......â
001A8F60  00 1F 3B E8 00 04 8F 70 00 00 03 E2 00 00 40 85 00 04 93 52 00 00 07 10 00 04 D3 D7 00 00 00 E8  ..;è...p...â..@…..“R......Ó×...è
001A8F80  00 04 DA E7 00 00 07 6B 00 04 DB CF 00 00 00 29 00 04 E3 3A 00 00 19 24 00 04 E3 63 00 00 17 30  ..Úç...k..ÛÏ...)..ã:...$..ãc...0
001A8FA0  00 04 FC 87 00 00 0E 24 00 05 13 B7 00 00 0C 0A 00 05 21 DB 00 00 17 7F 00 05 2D E5 00 00 15 7E  ..ü‡...$...·......!Û......-å...~
001A8FC0  00 05 45 64 00 00 15 B2 00 05 5A E2 00 00 13 C1 00 05 70 94 00 00 17 AE 00 05 84 55 00 00 15 66  ..Ed...²..Zâ...Á..p”...®..„U...f
001A8FE0  00 05 9C 03 00 00 14 46 00 05 B1 69 00 00 12 1E 00 05 C5 AF 00 00 1A 52 00 05 D7 CD 00 00 19 DF  ..œ....F..±i......Å¯...R..×Í...ß
001A9000  00 05 F2 1F 00 00 17 97 00 06 0B FE 00 00 17 24 00 06 23 95 00 00 15 0F 00 06 3A B9 00 00 13 01  ..ò....—...þ...$..#•......:¹....
001A9020  00 06 4F C8 00 00 16 C6 00 06 62 C9 00 00 13 DB 00 06 79 8F 00 00 11 BA 00 06 8D 6A 00 00 10 42  ..OÈ...Æ..bÉ...Û..y....º...j...B
001A9040  00 06 9F 24 00 00 1D 45 00 06 AF 66                                                              ..Ÿ$...E..¯f
```

### Section 5

#### Header

```
00 00 00 05 01 30 00 00 00 00 00 06 00 00 00 04 00 00 10 33 04 29 DB 22 73 2C 4D 52 5F 34 33 50 00 00 00 00 56 11 26 03 01 30 00 00 DF 81 F1 31 00 00 3E CC

Offset: 0x97 7E EC
Type: 0x6 kImageType_Data
Size: 0x00 00 3E CC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00978100                                                  40 23 24 5E 00 00 00 04 00 00 3E CC 00 00 02 00                  @#$^......>Ì....
00978120  2A 38 83 ED 00 00 14 21 00 00 16 24 00 00 00 10 00 00 19 A4 00 00 00 BC 00 00 00 00 00 00 00 00  *8ƒí...!...$.......¤...¼........
00978140  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00978160  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00978180  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
009781A0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
009781C0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
009781E0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00978200  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00                                                  ................
```

### Section 6

#### Header

```
00 00 00 06 01 30 00 00 00 00 00 06 00 00 00 04 00 00 19 6F C3 FD FF F9 E7 F2 D3 C8 70 79 74 20 00 00 00 00 56 11 26 03 01 30 00 00 DA B7 4F 3B 00 00 63 BC

Offset: 0x97 BF DC
Type: 0x6 kImageType_Data
Size: 0x00 00 63 BC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

0097C200  28 26 7E 5E 00 00 00 04 00 00 00 02 00 00 00 06 00 00 00 0A 00 00 18 EF 00 00 0B F9 00 00 00 0A  (&~^...................ï...ù....
0097C220  00 00 0C EC 00 00 0C 03 00 00 00 02 00 00 00 00 00 00 00 12 00 00 00 4A 00 00 00 2D 00 00 00 00  ...ì...................J...-....
0097C240  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 13 00 00 00 85 00 00 00 12 00 00 00 00 00 00 0B F9  ...................…...........ù
0097C260  00 00 00 00 00 00 00 00 00 00 0B F9 00 00 00 00 00 00 00 00 00 00 00 00 00 00 0A 79 00 00 00 00  ...........ù...............y....
0097C280  00 00 00 00 00 00 00 00 00 00 2A 97 00 00 00 00 00 00 0A 76 00 00 00 01 00 00 0A 76 00 00 00 01  ..........*—.......v.......v....
0097C2A0  00 00 0A 77 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ...w............................
0097C2C0  00 00 00 01 00 00 00 00 00 00 0A 7D 00 00 00 00 00 00 00 00 00 00 00 01 00 00 02 25 00 00 00 01  ...........}...............%....
0097C2E0  00 00 0A 79 00 00 00 01 00 00 0A 7A 00 00 00 01 00 00 0A 7B 00 00 00 01 00 00 00 01 00 00 00 00  ...y.......z.......{............
```

### Section 7

#### Header

```
00 00 00 07 01 30 00 00 00 00 00 06 00 00 00 04 00 00 2F 15 23 AE 8B BA BB 82 52 2B 7A 69 73 64 00 00 00 00 56 11 26 03 01 30 00 00 F7 07 40 69 00 00 BA 54

Offset: 0x98 25 BC
Type: 0x6 kImageType_Data
Size: 0x00 00 BA 54 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

009827E0  5B 21 25 5E 00 00 00 01 00 00 BA 54 00 00 00 24 00 00 02 1B 00 00 31 44 00 00 02 1B 00 00 96 54  [!%^......ºT...$......1D......–T
00982800  00 00 00 06 00 00 10 FC 00 00 00 09 00 00 11 06 00 00 00 0D 00 00 11 14 00 00 00 0D 00 00 11 22  .......ü......................."
00982820  00 00 00 10 00 00 11 33 00 00 00 0C 00 00 11 40 00 00 00 0E 00 00 11 4F 00 00 00 11 00 00 11 61  .......3.......@.......O.......a
00982840  00 00 00 09 00 00 11 6B 00 00 00 0D 00 00 11 79 00 00 00 0D 00 00 11 87 00 00 00 10 00 00 11 98  .......k.......y.......‡.......˜
00982860  00 00 00 0C 00 00 11 A5 00 00 00 0E 00 00 11 B4 00 00 00 11 00 00 11 C6 00 00 00 0B 00 00 11 D2  .......¥.......´.......Æ.......Ò
00982880  00 00 00 0F 00 00 11 E2 00 00 00 12 00 00 11 F5 00 00 00 0F 00 00 12 05 00 00 00 12 00 00 12 18  .......â.......õ................
009828A0  00 00 00 0E 00 00 12 27 00 00 00 10 00 00 12 38 00 00 00 13 00 00 12 4C 00 00 00 10 00 00 12 5D  .......'.......8.......L.......]
009828C0  00 00 00 14 00 00 12 72 00 00 00 17 00 00 12 8A 00 00 00 14 00 00 12 9F 00 00 00 17 00 00 12 B7  .......r.......Š.......Ÿ.......·
```

### Section 8

#### Header

```
00 00 00 08 01 30 00 00 00 00 00 06 00 00 00 04 00 11 DD A9 59 1E 68 D4 62 33 74 22 33 2E 31 22 00 00 00 00 56 11 26 03 01 30 00 00 43 8C B3 09 00 47 74 A4

Offset: 0x98 E2 34
Type: 0x6 kImageType_Data
Size: 0x00 47 74 A4 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

0098E440                                                                          2A 5F 3B 5E 00 00 00 01                          *_;^....
0098E460  00 00 03 E5 00 00 00 18 00 00 1F 40 00 47 74 A4 00 00 14 40 00 00 1F 40 00 00 12 00 00 00 33 80  ...å.......@.Gt¤...@...@......3€
0098E480  00 00 33 C0 00 00 45 80 00 00 33 C0 00 00 79 40 00 00 14 40 00 00 AD 00 00 00 04 C8 00 00 C1 40  ..3À..E€..3À..y@...@.......È..Á@
0098E4A0  00 00 04 C8 00 00 C6 08 00 00 1C 20 00 00 CA D0 00 00 19 00 00 00 E6 F0 00 00 33 C0 00 00 FF F0  ...È..Æ.... ..ÊÐ......æð..3À..ÿð
0098E4C0  00 00 33 C0 00 01 33 B0 00 00 1C 20 00 01 67 70 00 00 07 08 00 01 83 90 00 00 07 08 00 01 8A 98  ..3À..3°... ..gp......ƒ.......Š˜
0098E4E0  00 00 0E 74 00 01 91 A0 00 00 0B 54 00 01 A0 14 00 00 0B 54 00 01 AB 68 00 00 1A 98 00 01 B6 BC  ...t..‘ ...T.. ....T..«h...˜..¶¼
0098E500  00 00 1A 98 00 01 D1 54 00 00 0E 74 00 01 EB EC 00 00 03 B6 00 01 FA 60 00 00 03 B6 00 01 FE 18  ...˜..ÑT...t..ëì...¶..ú`...¶..þ.
0098E520  00 00 0E 74 00 02 01 D0 00 00 0B 54 00 02 10 44 00 00 0B 54 00 02 1B 98 00 00 1A 98 00 02 26 EC  ...t...Ð...T...D...T...˜...˜..&ì
0098E540  00 00 1A 98 00 02 41 84 00 00 0E 74 00 02 5C 1C 00 00 03 B6 00 02 6A 90                          ...˜..A„...t..\....¶..j.
```

### Section 9

#### Header

```
00 00 00 09 01 30 00 00 00 00 00 06 00 00 00 04 00 00 35 0E 8C 22 2F BC 51 C8 E1 83 20 0A 3E 2F 00 00 00 00 56 11 26 03 01 30 00 00 04 0B A2 8A 00 00 D2 38

Offset: 0xE0 58 FC
Type: 0x6 kImageType_Data
Size: 0x00 00 D2 38 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E05B20  FF 00 06 32 62 00 02 01 43 30 2B 01 FF FF FF FF 00 00 00 00 00 00 00 40 10 AA 00 00 00 28 00 00  ÿ..2b...C0+.ÿÿÿÿ.......@.ª...(..
00E05B40  25 91 A4 4A 69 E9 5B B4 D6 AE 5B B5 EF EA B4 6B 57 08 32 31 C1 C7 FE 4A FD 81 71 E1 50 A3 E2 8E  %‘¤Jié[´Ö®[µïê´kW.21ÁÇþJý.qáP£âŽ
00E05B60  05 0A 14 28 D0 A0 40 83 DD BA 74 E8 CC 9B 36 6F 3C FC F3 E6 FF CE DD AF CC CF AC FB 6B 19 BE 55  ...(Ð @ƒÝºtèÌ›6o<üóæÿÎÝ¯ÌÏ¬ûk.¾U
00E05B80  EA E4 89 56 3F F9 E9 70 EF CE 8E 2F CF 15 B1 FA E4 C8 93 26 C7 85 1E 74 3C 7C E3 67 B9 47 27 CE  êä‰V?ùépïÎŽ/Ï.±úäÈ“&Ç….t<|ãg¹G'Î
00E05BA0  1B 97 28 D6 C4 0F 36 2D F1 B1 E3 4A 1B 3A 7F 74 F9 B2 63 8A B7 9E 2F 7F 6C D8 B3 66 46 8C 1B 36  .—(ÖÄ.6-ñ±ãJ.:.tù²cŠ·ž/.lØ³fFŒ.6
00E05BC0  D4 A8 50 A3 7D DB B5 6A 16 2C 5B 36 61 C2 84 0B F7 ED D9 B1 2E 5F BD 7A B3 65 CA 97 3B 76 EC D8  Ô¨P£}Ûµj.,[6aÂ„.÷íÙ±._½z³eÊ—;vìØ
00E05BE0  6A 47 8E 1C 6B A4 05 91 B5 48 F5 8A AB DF E2 4E 2A 9F 9A 70 B4 1A 83 44 2A 97 AC D7 B1 66 11 9C  jGŽ.k¤.‘µHõŠ«ßâN*Ÿšp´.ƒD*—¬×±f.œ
00E05C00  66 E4 A9 94 8E 6F BF 68 A9 D4 92 46 FE 6C F8 E0 72 B5 A8 FF 73 AF D6 B3 D4 82 3D 2A 9C 1A C1 E7  fä©”Žo¿h©Ô’Fþløàrµ¨ÿs¯Ö³Ô‚=*œ.Áç
```

### Section 10 (0xA)

#### Header

```
00 00 00 0A 01 30 00 00 00 00 00 06 00 00 00 04 00 00 00 A1 0D 0B 8C 6D 7F 14 BA ED 73 22 3D 65 00 00 00 00 56 11 26 03 01 30 00 00 6D 32 4D 0A 00 00 00 84

Offset: 0xE1 2D 58
Type: 0x6 kImageType_Data
Size: 0x00 00 00 84 bytes (smallest section, only 84 bytes)
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E12F60                                                                                      C0 80 00 DC                              À€.Ü
00E12F80  C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0  ÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀ
00E12FA0  C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0  ÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀ
00E12FC0  C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0  ÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀ
00E12FE0  C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 C0 30 C0 C0 C0  ÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀÀ0ÀÀÀ
```

### Section 11 (0xB)

#### Header

```
00 00 00 0B 01 30 00 00 00 00 00 06 00 00 00 04 00 00 01 3B 13 48 9D F6 FA 3A CF C8 64 65 72 61 00 00 00 00 56 11 26 03 01 30 00 00 7D 12 3B 34 00 00 02 EC

Offset: 0xE1 30 00
Type: 0x6 kImageType_Data
Size: 0x00 00 02 EC bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E13220              6B 00 DA 98 81 9E 01 81 42 CA BE CC 81 00 00 DC 3F CA BF CC 81 F4 FD 34 00 CA C0 CC      k.Ú˜.ž..BÊ¾Ì...Ü?Ê¿Ì.ôý4.ÊÀÌ
00E13240  81 00 00 00 44 CA C1 CC 81 00 00 FA 3F CA C2 CC 81 F4 FD 34 00 CA C3 CC 81 00 00 00 45 CA C4 CC  ....DÊÁÌ...ú?ÊÂÌ.ôý4.ÊÃÌ....EÊÄÌ
00E13260  81 00 00 FA 3F CA C5 CC 81 F4 FD 34 00 CA C6 CC 81 00 00 00 41 CA C7 CC 81 33 33 9F 46 CA C8 CC  ...ú?ÊÅÌ.ôý4.ÊÆÌ....AÊÇÌ.33ŸFÊÈÌ
00E13280  81 00 08 9D 00 CA C9 CC 81 00 00 00 81 00 CA CC DA C3 CB CC 02 81 D1 01 81 7E 00 DC 01 81 C3 00  .....ÊÉÌ......ÊÌÚÃËÌ..Ñ..~.Ü..Ã.
00E132A0  C2 02 81 C3 81 C2 03 81 05 81 C2 04 00 06 81 C2 81 01 07 81 09 81 C2 08 C2 0A 81 00 81 C3 0B 81  Â..Ã.Â....Â....Â......Â.Â....Ã..
00E132C0  0D 81 03 0C 01 0E 81 C3 42 CA 0F 81 81 00 00 F0 F0 42 CA 10 11 81 00 00 00 12 81 01 81 C3 13 81  .......ÃBÊ.....ððBÊ..........Ã..
00E132E0  15 81 00 14 32 16 81 01 81 0D 17 81 19 81 32 18 03 1A 81 C3 81 C2 1B 81 1D 81 00 1C 00 1E 81 00  ....2.........2....Ã.Â..........
00E13300  81 C2 1F 81 21 81 C0 20 00 22 81 C3 BD CA 23 81 81 CD CC CC CC BD CA 24 25 81 CD CC C2 26 81 00  .Â..!.À .".Ã½Ê#..ÍÌÌÌ½Ê$%.ÍÌÂ&..
00E13320  81 00 27 81                                                                                      ..'.
```

### Section 12 (0xC)

#### Header

```
00 00 00 0C 01 30 00 00 00 00 00 06 00 00 00 04 00 00 07 55 E4 83 25 70 BD 40 67 34 00 00 CD 81 00 00 00 00 56 11 26 03 01 30 00 00 20 7D AD A0 00 00 1B 54

Offset: 0xE1 35 10
Type: 0x6 kImageType_Data
Size: 0x00 00 1B 54 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E13720                                                              44 13 DA 98 81 80 00 DC 84 00 00 CD                      D.Ú˜.€.Ü„..Í
00E13740  53 55 AE 6D 6C 65 44 20 20 65 78 75 00 6D 72 4E 00 00 CE 7B CE 7C 00 00 00 00 00 00 00 00 CE 7D  SU®mleD  exu.mrN..Î{Î|........Î}
00E13760  CD 81 00 00 6D 84 01 00 30 33 41 AD 77 61 46 20 72 42 20 6E CE 7B 00 74 00 00 00 00 00 00 CE 7C  Í...m„..03A.waF rB nÎ{.t......Î|
00E13780  CE 7D 00 00 00 00 00 00 02 00 CD 81 42 AF 6D 84 20 74 69 72 78 65 6C 50 72 42 20 69 CE 7B 00 74  Î}........Í.B¯m„ tirxelPrB iÎ{.t
00E137A0  00 00 00 00 00 00 CE 7C CE 7D 00 00 00 00 00 00 03 00 CD 81 43 AF 6D 84 20 69 6C 61 74 63 65 52  ......Î|Î}........Í.C¯m„ ilatceR
00E137C0  72 69 66 69 CE 7B 00 65 00 00 00 00 00 00 CE 7C CE 7D 00 00 00 00 00 00 04 00 CD 81 55 AE 6D 84  rifiÎ{.e......Î|Î}........Í.U®m„
00E137E0  6F 44 20 53 65 6C 62 75 6D 72 4E 20 00 CE 7B 00 7C 00 00 00 00 00 00 CE 00 CE 7D 00 81 00 00 00  oD SelbumrN .Î{.|......Î.Î}.....
00E13800  84 05 00 CD 73 45 AA 6D 20 78 65 73 00 30 33 41 00 00 CE 7B CE 7C 00 00 00 00 00 00 00 00 CE 7D  „..ÍsEªm xes.03A..Î{Î|........Î}
00E13820  CD 81 00 00 6D 84 06 00 72 61 43 AD 72 67 6F 74 65 68 70 61                                      Í...m„..raC.rgotehpa
```

### Section 13 (0xD)

#### Header

```
00 00 00 0D 01 30 00 00 00 00 00 06 00 00 00 04 00 01 CA 5B B4 DA B8 BC D3 18 27 3D 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 6A 03 DC 33 00 07 27 6C

Offset: 0xE1 52 88
Type: 0x6 kImageType_Data
Size: 0x00 07 27 6C bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E154A0                                      DA 2C 01 DC 6C A9 30 0F 65 68 2D 36 00 78 69 6C 3D 30 00 DA              Ú,.Ül©0.eh-6.xil=0.Ú
00E154C0  61 00 00 00 FE 00 00 00 00 00 00 02 54 00 00 03 A8 00 00 04 62 00 00 04 8C 00 00 06 3E 00 00 06  a...þ.......T...¨...b...Œ...>...
00E154E0  92 00 00 00 30 00 00 06 30 00 00 0F 89 00 00 0F A4 24 83 07 00 34 33 50 13 01 CE 23 B3 25 61 00  ’...0...0...‰...¤$ƒ..43P..Î#³%a.
00E15500  31 2E 31 76 30 33 2D 30 36 67 2D 31 31 31 61 63 00 00 66 34 16 00 15 82 00 13 82 9C 03 05 82 14  1.1v03-06g-111ac..f4...‚..‚œ..‚.
00E15520  03 02 83 07 93 04 03 03 40 C2 CA C2 3F CA 00 00 82 00 00 00 85 14 06 13 C2 17 83 18 1A EF CC 19  ..ƒ.“...@ÂÊÂ?Ê..‚...…...Â.ƒ..ïÌ.
00E15540  0A 01 09 FF 02 83 0B C2 04 05 03 05 80 3F CA 95 43 CA 00 00 CA 00 80 E3 00 B0 06 45 00 80 3F CA  ...ÿ.ƒ.Â....€?Ê•CÊ..Ê.€ã.°.E.€?Ê
00E15560  00 00 CA 00 83 0C 00 00 00 03 00 02 13 82 90 04 18 85 14 06 19 C2 17 83 FF 1A E0 CC C3 0A 01 09  ..Ê.ƒ........‚...…...Â.ƒÿ.àÌÃ...
00E15580  02 02 83 0B 92 04 02 03 00 80 3F CA 83 0C C2 00 00 03 00 02 13 82 90 04 18 85 14 06 19 C2 17 83  ..ƒ.’....€?Êƒ.Â......‚...…...Â.ƒ
00E155A0  09 FF 1A 77 0B C2 0A 09 03 04 02 83                                                              .ÿ.w.Â.....ƒ
```

### Section 14 (0xE)

#### Header

```
00 00 00 0E 01 30 00 00 00 00 00 06 00 00 00 04 00 02 E2 7E B3 A1 85 26 62 28 A1 07 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 C7 33 25 44 00 0B 87 F8

Offset: 0xE8 7C 18
Type: 0x6 kImageType_Data
Size: 0x00 0B 87 F8 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00E87E20                                                                                      DA 2C 01 DC                              Ú,.Ü
00E87E40  6C A9 95 18 65 68 2D 36 00 78 69 6C 3D 30 00 DA 3E 00 00 00 3E 00 00 00 AB 00 00 00 AB 00 00 0D  l©•.eh-6.xil=0.Ú>...>...«...«...
00E87E60  AB 00 00 0D AB 00 00 0D 1F 00 00 0D 3E 00 00 0E 1F 00 00 00 95 00 00 0E 95 00 00 18 81 00 00 18  «...«.......>.......•...•.......
00E87E80  69 0D DA 08 03 03 00 00 00 00 01 03 00 03 00 00 03 C2 40 00 3F 00 00 00 00 00 00 00 00 00 00 00  i.Ú..............Â@.?...........
00E87EA0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00E87EC0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00E87EE0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00E87F00  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................................
00E87F20  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00              ............................
```

### Section 15 (0xF)

#### Header

```
00 00 00 0F 01 30 00 00 00 00 00 06 00 00 00 04 00 00 23 16 98 81 85 58 ED E3 4C 2A 00 00 00 00 00 00 00 00 56 11 26 03 01 30 00 00 47 57 6C 87 00 00 8A 58

Offset: 0xF4 06 34
Type: 0x6 kImageType_Data
Size: 0x00 00 8A 58 bytes
```

#### Data

```
Offset(h) 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F

00F40840                                                                          FF 00 89 32 20 00 00 02                          ÿ.‰2 ...
00F40860  43 20 2B 01 FF FF FF FF 00 00 00 00 00 00 00 40 30 6A 00 00 00 20 00 00 25 91 A4 4A 69 E9 5B B4  C +.ÿÿÿÿ.......@0j... ..%‘¤Jié[´
00F40880  D6 AE 5B B5 64 EA B4 6B 02 F2 0B 67 C1 C7 49 2A FD 81 71 E1 50 A3 0B 8E 05 0A 14 28 D0 A0 40 83  Ö®[µdê´k.ò.gÁÇI*ý.qáP£.Ž...(Ð @ƒ
00F408A0  DD BA 74 E8 CC 9B 36 6F 3C FC F3 E6 F3 CC DF BE CC DD FD 3E 27 19 FE 57 F2 D5 9A 97 7D EC 71 F1  ÝºtèÌ›6o<üóæóÌß¾ÌÝý>'.þWòÕš—}ìqñ
00F408C0  E7 CE 9F 1E 4F 1D 3D 72 60 CC B7 26 8F BF 3B 32 3D 70 B7 AE AB 46 8E 4F 7B B7 7B 5C 84 0B 16 2C  çÎŸ.O.=r`Ì·&.¿;2=p·®«FŽO{·{\„..,
00F408E0  F9 F1 E1 C2 9F 3E 7F FD F9 F2 E7 CE CF 9E 3F 7D 6C D8 B3 66 46 8C 1B 36 D4 A8 50 A3 6C DB B5 6A  ùñáÂŸ>.ýùòçÎÏž?}lØ³fFŒ.6Ô¨P£lÛµj
00F40900  17 1C 59 B6 E1 4B 90 0B DF 6D D0 B1 2E 78 BD 4C F1 41 F8 96 3B 76 EC E8 2B 47 8E 1C 13 A5 DB 90  ..Y¶áK..ßmÐ±.x½LñAø–;vìè+GŽ..¥Û.
00F40920  B5 EE D4 A8 EB 56 E8 3A ED 02 1B 7C 2B 49 AB 34 CE 13 64 FA 20 61 83 B7 F0 E4 C8 90 DA 1E 51 19  µîÔ¨ëVè:í..|+I«4Î.dú aƒ·ðäÈ.Ú.Q.
00F40940  CE D1 04 C1 FE EE EB F5 3F F7 B8 D8 D2 CA D5 27 CE C8 9E 65 DC 18 39 C1                          ÎÑ.Áþîëõ?÷¸ØÒÊÕ'ÎÈžeÜ.9Á
```
