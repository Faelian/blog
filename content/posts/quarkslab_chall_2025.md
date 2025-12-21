---
title: "Writeup challenge Quarkslab leHack 2025"
date: 2025-12-21T18:21:17+01:00
draft: true
---

# Introduction

Ayant un peu de temps au début de ces vacances de Noël, j'en ai profité pour revisiter un challenge qui m'avait marqué cette année, et que je n'avais pas eu l'occasion de finir.
Quelque chose que j'ai rarement l'occasion de faire, un __chall reverse__ et __hardware__.

Il s'agit du challenge de Quarkslab, présenté à leur stand à LaHack 2025. J'avais été marqué par le _lore_ du challenge: un bombe à désamorcer.
Je n'ai pas de photo du stand, mais il y avait une petite bombe fictive à désamorcer, où le bon code devait être entré.

Le firmware de la bombe nous est fourni, ainsi que le schéma.

- firmware de la bombe: [https://github.com/0xfalafel/quarkslab_chall_lehack-2025/blob/main/bomb-fw.uf2](https://github.com/0xfalafel/quarkslab_chall_lehack-2025/blob/main/bomb-fw.uf2)
- schéma du circuit logique:

![schematics](/chall_quarkslab/schematic.jpg)

On voit sur le schéma qu'il s'agit d'un pico 2. Ça ne devrait pas être trop dur de trouver la doc pour les PINs. ;)
# Extraire le firmware

Le firmware est un fichier UF2:

```bash
❯ file bomb-fw.uf2

bomb-fw.uf2: UF2 firmware image, family 0xe48bff57, address 0x10ffff00, 2 total blocks
```

Avec une recherche google, on tombe très vite sur un [dépot github de Microsoft](https://github.com/microsoft/uf2), avec un outil [`uf2conv.py`](https://github.com/microsoft/uf2/blob/master/utils/uf2conv.md) qui permet la convertion de `uf2` vers un format binaire.

Il semblerait qu'il s'agissait de la solution attentdue pour extraire le firmware, malheurement et pour une raison quelconque, le script échouait à extraire le fichier chez moi.

```bash
❯ python3 uf2conv.py bomb-fw.uf2

--- UF2 File Header Info ---
Family ID is RP2XXX_ABSOLUTE, hex value is 0xe48bff57
Target Address is 0x10ffff00
Family ID is RP2350_ARM_S, hex value is 0xe48bff59
Target Address is 0x10000000
All block flag values consistent, 0x2000
----------------------------
Converted to bin, output size: 0, start address: 0x0
Wrote 0 bytes to flash.bin
```

C'est l'endroit où j'en était resté lors de la conférence, mais _hacker vaillant rien d'impossible_, le format `uf2` n'est pas très complexe. On peut __écrire un script python__ pour extraire le firmware.

## Le format UF2

Un fichier UF2 consiste de __blocs de 512 octets__. Chaque bloc commence avec un ___header_ de 32 octets__, suivit de données, et d'un _magic number_ final.  
Tous les sont des _entiers 32 bit non-signés_ en _little endian_.

| Offset | Size | Value                                             |
|--------|------|---------------------------------------------------|
| 0      | 4    | First magic number, `0x0A324655` (`"UF2\n"`)      |
| 4      | 4    | Second magic number, `0x9E5D5157`                 |
| 8      | 4    | Flags                                             |
| 12     | 4    | Address in flash where the data should be written |
| 16     | 4    | Number of bytes used in data (often 256)          |
| 20     | 4    | Sequential block number; starts at 0              |
| 24     | 4    | Total number of blocks in file                    |
| 28     | 4    | File size or board family ID or zero              |
| 32     | 476  | Data, padded with zeros                           |
| 508    | 4    | Final magic number, `0x0AB16F30`                  |

Le __struct C__ suivant correspond à un bloc _UF2_:

```C
struct UF2_Block {
    // 32 byte header
    uint32_t magicStart0;
    uint32_t magicStart1;
    uint32_t flags;
    uint32_t targetAddr;
    uint32_t payloadSize;
    uint32_t blockNo;
    uint32_t numBlocks;
    uint32_t fileSize; // or familyID;
    uint8_t data[476];
    uint32_t magicEnd;
} UF2_Block;
```

Aperçu du fimware avec `hexyl`:

![aperçu du firmware avec hexyl](/chall_quarkslab/hexyl_firmware.png)

### Lecture des _headers_

Pour avoir une idée de ce à quoi ressemble le fichier, j'ai d'abord écrit un script pour lire les entêtes de chaque bloc.

__`read_headers.py`__
```python
#!/usr/bin/env python3
# coding: utf-8

import struct
import sys

# ANSI escape codes for colors
BOLD = '\033[1m'
BLUE = '\033[94m'
MAGENTA = '\033[95m'
YELLOW = '\033[33m'
RESET = '\033[0m'  # Reset to default color

def unpack_little_endian(data):
	return struct.unpack("<I", data)[0]

if len(sys.argv) < 2:
	print("Usage:")
	print(f"{sys.argv[0]} [uf2_file]")
	exit()

with open(sys.argv[1], 'rb') as f:
	i = 0
	while (block := f.read(512)):
		print(f"{BOLD}{MAGENTA}Block {i}:{RESET}")
		i += 1
        
        # reference: https://github.com/microsoft/uf2/blob/master/README.md#file-format
		magic1 = block[0:4]
		magic2 = unpack_little_endian(block[4:8])
		flags = unpack_little_endian(block[8:12])
		targetAddr = unpack_little_endian(block[12:16])
		payloadSize = unpack_little_endian(block[16:20])
		blocNo = unpack_little_endian(block[20:24])
		numBlocks = unpack_little_endian(block[24:28])
		fileSize = unpack_little_endian(block[28:32])
		magicEnd = unpack_little_endian(block[-4:])

		print(f"{BLUE}First magic number:{RESET} {magic1}")
		print(f"{BLUE}Second magic number:{RESET} {hex(magic2)}")
		print(f"{BLUE}Flags:{YELLOW} {hex(flags)}{RESET}")
		print(f"{BLUE}Target address:{RESET} {hex(targetAddr)}")
		print(f"{BLUE}Size of the payload:{RESET} {payloadSize}")
		print(f"{BLUE}Bloc number:{RESET} {blocNo}")
		print(f"{BLUE}Total number of blocks:{RESET} {numBlocks}")
		print(f"{BLUE}File size (or family ID):{RESET} {fileSize}")
		print(f"{BLUE}Magic end:{RESET} {hex(magicEnd)}\n")
```

