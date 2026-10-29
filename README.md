# .kkrieger → JavaScript: porting notes

Reconnaissance for anyone who wants to port **.kkrieger** (the 96 KB FPS by .theprodukkt / Farbrausch, 2004) to JavaScript or WebGL.
No port exists here. This repository only records what was found, so the next person does not have to repeat it.
Status: reconnaissance only, nothing was ported or run. Date: 2026-10-30.

*Кратко по-русски:* заметки о том, как устроены данные уровня .kkrieger и что нужно для порта на JS. Портирования здесь нет, только разведка.

## Where the source is

Farbrausch released their tools and demo sources in 2012: <https://github.com/farbrausch/fr_public>.
Licences are stated per project (README of that repo):

| What | Licence | Condition |
| --- | --- | --- |
| Engine code (`werkkzeug3`, `werkkzeug3_kkrieger`) | BSD-2-Clause | keep the Farbrausch copyright notice |
| `ktg` (OpenKTG texture generator), `v2` (V2 synthesizer) | public domain | none |
| `werkkzeug3/data` (kkrieger project, music, sound effects) | CC-BY 3.0 | credit the authors |

`werkkzeug3_kkrieger/data/*.kx` has no licence file of its own. It is a copy of the data above, so treat it as CC-BY.

Important caveats from the authors' README:
- `werkkzeug3_kkrieger` is **not** the original 2004 code. It was branched later and is unlikely to build.
- The original data was made with an older version of the engine than that branch.

## The level file: `werkkzeug3_kkrieger/data/kkrieger3383.kx` (125 145 bytes)

This is the old *export* format and it differs from `KDoc::Init` in the branch. All integers are little-endian.
`rs` below is `sReadShort`: read one byte; if its top bit is set, `v = (v & 127) | (next_byte << 7)`.

1. `u32` SongSize, then the V2M music (27 031 bytes, padded to 4).
2. `u32` SampleSize, then the VFX0 sound effects (6 640 bytes, padded to 4).
3. `u32` BPM × 65536 (it reads 89), `u32` length.
4. `rs` nOps (5174), `rs` nSplines (23), then 16 roots as `rs`. A root equal to nOps means "no root".
5. Class list until a `u32` zero. Each record: `u32` conv, **`u8`** class id (the branch uses `u16`), a NUL-terminated packing string. 78 classes.
6. The op graph. Per op: `u8` typeByte (low 7 bits are the class index). Bit 0x80 means "exactly one input, the previous op". Otherwise the input count is `(conv >> 8) & 15`, or one extra byte if conv has `0x80000000`; then one `rs` delta per input, the input is op `i - 1 - delta`.
7. Links: per op `(conv >> 12) & 15` values of `rs`; 0 means none, otherwise op index = value − 1.
8. Parameters, **grouped by class** and then by op order. The packing string says how to read each one (case-insensitive): `g` float16, `f` float24, `e` fixed 4.12 (`s16 / 4096`), `i` s32, `s` s16, `b` u8, `c` u32, `m` 3 bytes. **Any other character reads nothing and must be skipped** (a parser that raises on `-` will fail here). After the numbers come `(conv >> 16) & 7` NUL-terminated strings, `(conv >> 20) & 7` spline references as `u16` (0 = none), and, if conv has `0x80000`, a blob size (`s32`). Blob data follows at the very end.
9. Animation bytecode, one program per op, ends with command `0x01`. Sizes: commands `0x02`–`0x06` take 1 byte, `0x16` takes 16, `0x17` and `0x18` take 4, `0x19` takes 2, `0x1d` takes 8, commands from `0x80` take 1. Everything else takes 0.
10. Events, then splines. They use `sReadF16` and `sReadF24` from `_types.cpp`. Read the loops at the end of `KDoc::Init` in `kdoc.cpp`.

What was verified: steps 1 to 7 parse cleanly. The graph ends at byte 43 319, there are 1 110 links and 80 139 bytes remain for parameters, animation, events and splines. Steps 8 to 10 were not finished, so check that the whole file is consumed exactly (`end == file size`) when you implement them.

## What the level contains (5174 operators)

| Group | Operators | Largest classes |
| --- | --- | --- |
| Mesh | 2531 | transform 652, matlink 292, cube 224, multiply 215, add 179 |
| Texture | 1461 | hscb 294, merge 187, glowrect 158, flat 130, normals 89 |
| Scene | 866 | transform 346, add 158, scene 103, multiply 97, light 51 |
| Game logic | 192 | delay 44, play sample 32, event 31, if 24, state 23 |
| Material | 78 | material 78 |
| Effects / post | 46 | particles 16, print 6, viewport 5 |

The classes used by the player are listed in `werkkzeug3_kkrieger/player_kkrieger/kkrieger_oplist.cpp`.

## Size of the C++ to port (lines)

| Part | File | Lines |
| --- | --- | --- |
| Document loader and ops | `kdoc.cpp` | 2951 |
| Textures | `genbitmap.cpp` | 3165 |
| Meshes | `genmesh.cpp` | 5557 |
| Scene | `genscene.cpp` | 1641 |
| Materials | `genmaterial.cpp` | 628 |
| Game logic | `kkriegergame.cpp` | 3862 |
| Renderer | `engine.cpp` | 3454 |
| V2 synthesizer | `v2/` | about 6900 |

`ktg` (about 2000 lines, clean, public domain) is the easiest first step; its README even suggests turning it into JS or pixel shaders.

## Risks

- Data version 3383 is older than the branch code, so parameter packing for some ops may differ.
- The original renderer uses shadows and per-pixel lighting; a browser port has to simplify.
- Generating 5174 operators at start-up in JS may take tens of seconds on a phone; caching the result is likely needed.
- Collision, portals and sectors are custom code in `kkriegergame.cpp` and `genscene.cpp`.

## Licence of these notes

These notes are released under CC0 (public domain). The code and data they describe belong to Farbrausch under the licences above; credit them if you reuse them.
