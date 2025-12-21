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


