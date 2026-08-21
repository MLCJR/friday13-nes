[The Configurizer](https://mlcjr.github.io/friday13-nes/configurizer/) &middot; [Reference](https://github.com/MLCJR/friday13-nes/blob/main/README.md) &middot; [RAM map](https://github.com/MLCJR/friday13-nes/blob/main/RAM-MAP.md) &middot; **CHR map** &middot; [ROM Hacks](https://github.com/MLCJR/friday13-nes/blob/main/ROMHACKS.md) &middot; [Patches](https://github.com/MLCJR/friday13-nes/tree/main/patches/)

---

# CHR map — Friday the 13th (NES, 1989)

32 KB of CHR-ROM in four 8 KB banks. **This is not just graphics.** The game keeps
bulk data — text, tables, room layouts, even its RAM initialisation records — in
CHR and reads it back through the PPU data port at `$2007`. Many addresses in this
game therefore appear in **no instruction operand anywhere**, because they exist
only as bytes inside CHR.

> Every table, length, offset and glyph below is derived from the program ROM
> and the character ROM directly.

> ### How to read this
>
> - **A bank** is one 8 KB window of graphics. The cartridge holds four and can
>   show only one at a time, so the *same* tile number means different pictures
>   depending on which bank is loaded. That single fact explains most of the
>   surprises in this document.
> - **A tile** is an 8x8 pixel square, numbered `$00`-`$FF` within a bank.
> - **A pose** is one animation, referenced by number; it names which tiles to
>   draw and in what arrangement.
> - **OAM** is the hardware's list of sprites for the current frame.
> - **A sub-palette** is one set of three colours plus a shared background
>   colour. Four exist at a time, and a sprite picks one of them.
>
> The important idea, if you read nothing else: this cartridge stores **data** in
> its graphics ROM. Regions that look like noise when rendered as pictures are
> usually text or tables being read as bytes.

## How CHR data is read, and why that closes the map

There are exactly **four** `lda $2007` sites in the ROM: `$821D` and `$8223` (the
streaming reader `$8204`), and `$8273` and `$8278` (`chr_read`, `$8253`). Verified
by scanning all 32768 PRG bytes for `AD 07 20`, and for the `abs,x`/`abs,y` forms,
which do not occur. Every byte of CHR that reaches the CPU passes through one of
those four.

| reader | source comes from | fed by |
|---|---|---|
| `$8253` `chr_read` | `$10`/`$11` (src), `$12`/`$13` (dst), `$14`/`$15` (len), `$16` (bank) | **exactly one caller**, the `jsr $8253` at `$E6D9`, and the seven `lda $E77C+n,y / sta $10+n` pairs at `$E6B6`-`$E6D8` immediately above it load all seven fields from the 18-record table at `$E77C`. So the descriptor is re-loaded from the table on every call and can come from nowhere else. |
| `$8204` `chr_stream_chunk` | `$ED`/`$EE` (src) and `$F3` (bank) | `sta $F3` exists at exactly **one** address, `$E717`, inside `$E6E3`, sourced from the 28-record table at `$E728`. `$E6E3` always transfers exactly 168 bytes to the staging buffer at `$0758` — destination and length are immediates at `$E708`/`$E70C`/`$E719`, not table fields. |

So **every CHR data region is named by one of two tables**, both decoded below.
That is what makes this map closeable rather than open-ended: a region no
descriptor names is either tile graphics or genuinely unread.

> **What closes the map is the single caller, not private zero page.** `$10`,
> `$11` and `$16` are ordinary zero page, written all over the ROM — `sta $11`
> alone occurs at 14 sites, and `$10`/`$11` double as the indirect pointer for
> `sta ($10),y` at `$FED6`. It does not matter, because the unpack at
> `$E6B6`-`$E6D8` reloads every field from `$E77C` immediately before the one and
> only `jsr $8253`, so anything else that touched those bytes is overwritten
> before the read.

> Caveat, and it is the recurring one: `$8204`'s state block (`$ED`-`$F3`) *is* private —
> `sta $F3` occurs once — but a zero-page indirect store landing in `$10`-`$16`
> between `$E6D8` and `$E6D9` would be undetectable by this reasoning. There is
> no instruction between them, so the window is empty; the general search for
> such a store is still not exhaustive.

## Which bank is loaded when

`$987A` `chr_bank_for_area` is `ldx $0500 / lda $9885,x / tay / jsr $81A4 / rts`.
The index is the **area type** in `$0500`, and the two callers are `$95DE`
(junction decode) and `$970C` (bulk draw). The 15-byte table at `$9885`:

```
area   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14
bank   0   1   1   1   0   0   0   1   1   1   1   1   1   1   1
```

Bank 0 serves areas 0, 4, 5, 6 — the camp triangle and the lake. Bank 1 serves
areas 1-3 (the cave mouths) and 7-14 (the woods). Banks 2 and 3 are switched in
for text, interiors and full-screen art rather than by area.

The bank switch itself is `$81A6`: `lda $81AD,y / sta $81AD,y` — it reads the byte
and writes it straight back to the same address, and `$81AD` holds `00 01 02 03`.
That is the bus-conflict-safe CNROM idiom, and it means **patches to this game are
safe on real hardware**, which is not automatic for mapper 3. `$81A4` is the entry
one instruction earlier (`sty $37`), which also mirrors the bank into `$37`.

Literal bank selections, from a scan for `ldy #imm : jsr $81A4/$81A6`:

| site | bank | what it serves |
|---|---|---|
| `$E831` | 2 | the **one-shot new-game intro box** (`intro_box_new_game`, `$E7FA`): `jsr $EDC2` draws the room picture, then bank 2, then block 14 |
| `$E8FD` | 2 | **entering a building** (`$E8F7 jsr $EDC2`, then `inc $05A4`, then bank 2) |
| `$8DBD` | 3 | the two game-over screens (`$8DC6` then streams block 26 or 27) |
| `$D60B` | 3 | the day-end / ending sequence |
| `$E477` | 3 | title and legal screens |
| `$80D7`, `$823F` | 3 | inside the NMI and the CHR streamer |

Those seven are the complete set. There are 14 call sites in all — nine
`jsr $81A4` (`$8258 $828D $8DBF $9881 $D60D $E3BA $E479 $E833 $E8FF`) and five
`jsr $81A6` (`$80D9 $812E $8195 $8206 $8241`), no `jmp` to either — and a scan of
the two bytes before each returns `ldy #imm` at exactly the seven above. The
other seven take the bank from a variable: `ldy $16` at `$8256` (chr_read's own
descriptor field), `ldy $37` at `$812C`/`$8193`/`$E3B8` (the saved page),
`ldy $F3` at `$8204` (the streamer's source bank), `tay` at `$9880` after
`lda $9885,x`, and **`pla / tay` at `$828B`** — one bank selection in this ROM arrives
on the hardware stack, one of the ways this ROM defeats a static operand search.

> Addresses in the table above are the `ldy #imm` sites, not the `jsr` that
> follows them.

> **Bank 2 is the ambient bank for the whole time you are inside a building.**
> Neither the note box (`$EC75`) nor the hint box switches banks — they inherit
> bank 2, which `$E8FD` sets on the way into the building and `$81A4` latches
> into `chr_page` (`$37`), and the NMI epilogue restores it every frame. That is
> also why bank 2's sprite half holds the counselor and item tiles, and it is
> what settles the win screen below.

## The `$E728` table — 28 records, 3 bytes each

`(src_lo, src_hi, bank)`. Every block is 168 bytes into `$0758`. `$E728`-`$E77B`
is 84 bytes, ending with zero slack against `$E77C`. **This settles
the record count in favour of 28** — records 26 and 27 decode as the two
game-over screens, which the game demonstrably has.

168 = `$0800 - $0758`, the size of the buffer, not of any payload. Blocks whose
real content is 39 bytes (the hint gate table) or 48 (the counselor palettes) are
still moved as 168 and simply overrun into whatever follows them in CHR.

| # | source | bank | what it is |
|---|---|---|---|
| 0 | `$1028` | 0 | hint — "GO INTO ONE OF THE CABINS BY THE LAKE." |
| 1 | `$1053` | 0 | hint — "GO INTO CABIN #12 #15 #20" (see the font trap below) |
| 2 | `$106E` | 0 | hint — "GO INTO THE CABIN NEAR THE CAVE." |
| 3 | `$1092` | 0 | hint — "GO INTO THE WOODS." |
| 4 | `$10A6` | 0 | hint — "GO INTO THE CAVE." |
| 5 | `$10B9` | 0 | hint — "THERE'S A MACHETE IN A CABIN IN THE MIDDLE OF THE WOODS." |
| 6 | `$10F8` | 0 | hint — "THERE'S A MACHETE HIDDEN SOMEWHERE IN THE CAVE." |
| 7 | `$112D` | 0 | hint — "THERE'S AN AX IN A CABIN IN THE MIDDLE OF THE WOODS." |
| 8 | `$1168` | 0 | hint — "THERE'S AN AX HIDDEN SOMEWHERE IN THE CAVE." |
| 9 | `$1199` | 0 | hint — "YOU CAN FIND A TORCH IN ONE OF THE CABINS BY THE LAKE." |
| 10 | `$11D7` | 0 | cut hint — "YOU CAN FIND A SWEATER HIDDEN IN THE MIDDLE OF THE WOODS." |
| 11 | `$1218` | 0 | cut hint — "THE ONLY WAY TO GET TO THE SWAMP IS THROUGH THE CAVE." |
| 12 | `$1252` | 0 | hint — "FIRE WILL DAMAGE JASON THE MOST." (the reachable donor) |
| 13 | `$1276` | 0 | "YOU CAN'T GET IN WITHOUT A KEY." — not a hint; literal `lda #$0D` at `$BCDA` |
| 14 | `$1000` | 0 | "USE THE TORCH TO LIGHT THE FIREPLACES." — the one-shot new-game box |
| 15 | `$1D4D` | 2 | counselor palettes, 6 × 8 bytes. Same bytes as descriptor 11, by a second path |
| 16 | `$0710` | 1 | **note gate table**, 13 × 3 = 39 bytes |
| 17 | `$0737` | 1 | **silent-item gate table**, 11 × 3 = 33 bytes. `$0710 + 39 = $0737`, zero slack |
| 18 | `$076C` | 1 | **the camp map** — 40 × 4 room exit graph `[UP,DOWN,LEFT,RIGHT]` |
| 19 | `$080C` | 1 | room → graphics-set remap table (the source of `$05A0`) |
| 20 | `$107C` | 1 | **the cold-boot RAM fill table**, 20 × 4 `(count, dest_lo, dest_hi, value)` |
| 21 | `$1D13` | 2 | the per-counselor save/restore table — where the sweater lives |
| 22 | `$1B00` | 3 | 8 pad bytes, then 14 16-bit pointers into the staged copy, then payload |
| 23 | `$1D7D` | 2 | per-building pointer lists for interior TEXT records. 63 bytes, `$1D7D`-`$1DBB`, zero slack against block 24 |
| 24 | `$1DBC` | 2 | per-building pointer lists for SILENT ITEM DROPS. **55 bytes**, `$1DBC`-`$1DF2`: 14 two-byte pointer slots of which only 13 are reachable (9 live), bytes, then 9 three-byte records. Zeros follow |
| 25 | `$1387` | 0 | **"YOU WIN... FOR NOW."** — the win screen; literal `lda #$19` at `$BE09` |
| 26 | `$07C0` | 2 | "YOU AND YOUR FRIENDS ARE DEAD. GAME OVER" — all six counselors lost |
| 27 | `$09C0` | 0 | "JASON WIPED OUT THE KIDS. GAME OVER" — children lost |

> **Row 25 is the win-screen text, not the RLE nametable**, and the two are easy
> to conflate. The fixed 168-byte transfer makes block 25 overrun
> `$1387`-`$142E` and swallow **147** bytes of the RLE stream, which starts 21
> bytes later at `$139C` and is reached by `$E77C` descriptor 1 — not by this
> table at all. Decoded flat, those 147 bytes read `COPYRIGHT @ 1928
> PARAMOUNT.`, the same misreading recorded under "the copyright year" below,
> arriving by a second route.

Records 0-13 are not fourteen separate strings. They are **fourteen entry points
into one continuous message stream** that starts at `$1000` (record 14 is the
whole stream from the top): record *n*'s 168 bytes begin mid-list and run on into
its successors. That is why decoding record 0 also yields records 1-5.

Record 20 is the one a randomizer author needs: **the new-game defaults live in
CHR, not PRG**, so no PRG operand search can find them. 20 records of `(count, dest_lo, dest_hi, value)` at block offsets 0-79,
a `$00` terminator at offset 80, then 55 zero bytes, then live non-zero CHR from
offset 135 on. Those 55 zeros are the entire headroom for appending records —
13 more records plus a terminator.

**Block 24's length is 55 bytes, and its own arithmetic settles it.** An earlier
pass had recorded 58; that was a transcription error, and the dump retires it.
Fourteen two-byte slots fill offsets 0-27, but the reader indexes
`(building - 14) * 2` for buildings 14-26, so only the first thirteen are
reachable. The fourteenth is null and nothing can index it. Two zero
bytes of pad behind them, the first live pointer is `$0774` = staged offset 28,
the nine live pointers are 3 bytes apart, and the ninth record ends at offset
54. `28 + 27 = 55`; the 21 bytes `$1DF3`-`$1E07` that follow are zero, and
non-zero CHR outside this block resumes at `$1E08`. (Record 20's 55 zeros above
are a coincidence of numbers, not the same quantity: those are free space, this
is a total length.)

## The `$E77C` table — 18 records, 7 bytes each

`(src_lo, src_hi, dst_lo, dst_hi, len_lo, len_hi, bank)` — arbitrary
source/destination/length, unlike `$E728`'s fixed 168. 18 × 7 = 126 bytes,
`$E77C`-`$E7F9`; `$E7FA` is the first byte of `intro_box_new_game` (`A9 02` =
`lda #$02`). The "what it is" column below is from the call sites, all eleven of
which are accounted for.

| # | source | bank | dest | len | what it is | loaded by |
|---|---|---|---|---|---|---|
| 0 | `$1000` | 1 | `$0300` | `$0100` | title-screen RLE stream (see below) | `$E4F1`, and record `$E445` |
| 1 | `$139C` | 0 | `$0300` | `$0120` | legal-screen RLE stream | record `$E453` |
| 2 | `$1299` | 0 | `$0758` | `$0078` | ending text/graphics, day 1 | `$D656`, `= day + 2` |
| 3 | `$12D8` | 0 | `$0758` | `$0078` | ending text/graphics, day 2 | `$D656` |
| 4 | `$1318` | 0 | `$0758` | `$0078` | ending text/graphics, day 3 | `$D656` |
| 5 | `$14A2` | 0 | `$06B4` | `$0037` | the outdoor item lists | `$F43A` |
| 6 | `$1C32` | 2 | `$0200` | `$0100` | the junction graph | `$9571` |
| 7 | `$1C00` | 2 | `$0200` | `$0100` | outdoor screen table | `$957D` |
| 8 | `$0250` | 3 | `$0200` | `$0100` | background palette sets, page 1 (16 sets) | `$876D` |
| 9 | `$0350` | 3 | `$0200` | `$0100` | background palette sets, page 2 (16 sets) | `$876D` |
| 10 | `$0200` | 3 | `$0200` | `$0100` | sprite palette sets | `$873E` |
| 11 | `$1D4D` | 2 | `$0758` | `$0030` | counselor palettes, 6 × 8 | `$870F` |
| 12 | `$0758` | 1 | `$0758` | `$001E` | 30 bytes, loaded on entering a building | `$E86B` |
| 13 | `$0380` | 3 | `$0200` | `$0100` | map screen, RLE stream → VRAM `$2000` | record `$E42D` |
| 14 | `$0437` | 3 | `$0200` | `$0100` | map screen, RLE stream → VRAM `$2100` | record `$E432` |
| 15 | `$0532` | 3 | `$0200` | `$0100` | map screen, RLE stream → VRAM `$2200` | record `$E437` |
| 16 | `$05C3` | 3 | `$0200` | `$0040` | map screen attribute RLE stream → VRAM `$23C0` | record `$E43C` |
| 17 | `$0800` | 0 | `$0758` | `$0040` | the map-screen icon list | `$E37D` |

Note that records **6-10 and 13-16** — nine of the eighteen — land in `$0200`,
the OAM shadow. That is legal only because every one of them runs with rendering
off, where the OAM page doubles as a 256-byte staging buffer.

> **Nine records target `$0200`.** Records 6 and 7 — the junction graph and the
> outdoor screen table — are easy to overlook; the `dest` column above lists
> them.

## RLE nametables, and how they are driven

`$82CA` `rle_to_ppu` decompresses straight into VRAM. **One call site in the whole
ROM: `$E3DD`.** The encoding:

    $FF                    end of stream, checked BEFORE the bit-7 test
    control bit 7 set      LITERAL run: copy (c & $7F) following bytes
    control bit 7 clear    REPEAT run:  write the ONE following byte c times

A count of zero means **256**, not zero — the loops are `tax` / do-while / `dex` /
`bne`. So `$80` is 256 literal bytes and `$00` is 256 copies. Because `$FF` is
caught first it can never be read as a 127-byte literal run.

The caller is `screen_load` (`$E39E`), which walks a table of screen-draw blocks
at `$E42A`. Each block is `(CHR bank, bg palette set, sprite palette set)`
followed by 5-byte records `(descriptor id, VRAM dst lo, dst hi, stream ptr lo,
stream ptr hi)` until a record whose first byte has bit 7 set. There are exactly
three blocks; `$E45E` is not a fourth, it is a 3-byte RLE stream in PRG (`40 00
FF` = 64 copies of `$00`, the attribute table), and `$E461` is code.

| block | bank | records | what it draws |
|---|---|---|---|
| `$E42A` | 3 | descriptors 13, 14, 15, 16 | **the map screen** — 704 tiles into VRAM `$2000`-`$22BF` plus 64 attribute bytes |
| `$E442` | 3 | descriptor 0, then the PRG stream at `$E45E` | **the title screen** |
| `$E450` | 3 | descriptor 1, then the PRG stream at `$E45E` | **the legal screen** |

Both full-screen streams were run through a faithful re-implementation of `$82CA`
for this revision. Both produce **exactly 960 bytes**, a full 32 × 30 nametable:

| stream | consumed | of a declared | ends at | content |
|---|---|---|---|---|
| CHR1 `$1000` (desc 0) | 124 bytes | 256 | `$107B` | logo tiles + "PUSH START BUTTON" |
| CHR0 `$139C` (desc 1) | 262 bytes | 288 | `$14A1` | the Paramount / LJN / Nintendo legal screen |

The four map-screen streams were run through the same re-implementation for this
revision, and they are what makes the `$E42A` row above arithmetically true
rather than assumed:

| stream | consumed | of a declared | ends at | emits |
|---|---|---|---|---|
| CHR3 `$0380` (desc 13) | 183 | 256 | `$0436` | 256 tiles → VRAM `$2000` |
| CHR3 `$0437` (desc 14) | 251 | 256 | `$0531` | 256 tiles → VRAM `$2100` |
| CHR3 `$0532` (desc 15) | 145 | 256 | `$05C2` | **192** tiles → VRAM `$2200`-`$22BF` |
| CHR3 `$05C3` (desc 16) | 29 | 64 | `$05DF` | 64 attribute bytes → VRAM `$23C0` |

256 + 256 + 192 = 704, and each stream's `$FF` terminator sits one byte before
the next descriptor's source, so the four are contiguous in CHR3 with zero slack
(`$0380`-`$05DF`). The third one emitting 192 rather than 256 is a property of
the stream, not of the descriptor — descriptor 15 still declares a 256-byte
read. **The block therefore covers only VRAM `$2000`-`$22BF`, 22 of the 30
nametable rows; `$22C0`-`$23BF` is left holding whatever the previous screen put
there.** What the bottom eight rows actually show is not settled here.

Two boundaries fall out of that, and both are load-bearing:

- CHR0 `$139C + 262 = $14A2` — **the outdoor item lists start on the byte after
  the legal screen's `$FF`.** Zero slack.
- CHR1 `$1000 + 124 = $107C` — **the cold-boot RAM fill table starts on the byte
  after the title screen's `$FF`.** Also zero slack, and this is *why* descriptor
  0 reads 256 bytes into `$0300`: bytes `$107C`-`$10FF` of that read are the fill
  table, and they land at RAM `$037C`-`$03FF`. The block is genuinely dual-mapped.
  That is a hazard for anyone appending records; the terminator position is what
  makes it harmless in vanilla.

**The copyright year is 1988.** Read raw without decompressing, `1988` is encoded
as the literal pair `01 09` followed by the repeat pair `02 08` — "1", "9", then
"two copies of 8". Scanned as flat text it reads `1 9 2 8`, so a reader who
skips the decompression gets "1928" — a decoding error, not a printing error. It
appears twice (rows 4 and 20 of the decompressed screen).

`$14A2` holds the outdoor item lists — 55 bytes, ten count-prefixed lists at blob
offsets `00 06 08 0B 0E 17 1B 1F 22 26` (live through offset `$29`), then 13
vestigial bytes. Census of the 32 live entries by low nibble: **12 knives (type
1), 14 vitamins (type 9), 6 keys (type `$A`)**.

## Banks 2 and 3 — the font, and a trap

**Only banks 2 and 3 carry a font at all.** Re-checked byte-for-byte this
revision: at PPU `$1000`-`$136F`, banks 0 and 1 differ from bank 2 in **every one
of the 55 tiles `$00`-`$36`** (and in `$37` besides, bank 2's arguable 56th font
tile; see the note below), and rendering them shows level graphics, not
glyphs. So any legible text on screen is being drawn with bank 2 or bank 3 live.
That is a small fact with a long reach — it is the first step of the win-screen
argument below, and it means a hack that shows text during outdoor play (bank 0
or 1) gets garbage unless it switches banks first.

Banks 2 and 3 both carry the text font at PPU `$1000`-`$136F` (tiles
`$00`-`$36`, 55 tiles × 16 = `$370` bytes). **They are not the same font.** A
byte-for-byte comparison of all 55 gives exactly four differences:

| code | bank 2 (in-game notes, cabin interiors) | bank 3 (title, endings, game over) |
|---|---|---|
| `$24` | filled disc | © |
| `$27` | raised mid-height dot | `.` (baseline) |
| `$28` | `.` (baseline) | a mask/face glyph — outline, two eyes, mouth |
| `$31` | `#` | `?` |

From `$37` upward the two banks diverge completely: of the 201 tiles `$37`-`$FF`,
**200 differ and exactly one — `$FF` — is byte-identical.**

> **Bank 2's `$37` is a legible `?` glyph**, where bank 3 puts `?` at `$31` and
> level graphics at `$37` — so bank 2's font is arguably 56 tiles rather than 55.
> It makes no practical difference, because `$37` occurs nowhere in the CHR0
> message stream or in either game-over block. But `$37` is not "the first tile
> past the font" in bank 2, and it is not what makes the boundary clean: that is
> the 4-differences-below / 200-of-201-differences-above split.

**Consequences, both confirmed against the actual message bytes:**

- Hint block 1 reads **"GO INTO CABIN #12 #15 #20"**, not "?12", because notes
  render in bank 2. *(A flat transcription using the bank-3 font, where `$31`
  is `?`, reads it as "?12"; the bank-2 rendering above is what the hardware
  does.)*
- "BUT IS HE REALLY DEAD `$31`" is **"DEAD ?"**, because the endings render in
  bank 3.
- `$24` occurs nowhere in the CHR0 message stream. Its only appearance is on the
  legal screen — which is bank 3, so it renders as ©, which is what that line
  needs.

**Correction, still standing:** `$25` has been listed elsewhere as `.`. It is
**`!!`** — two exclamation marks in a single 8×8 tile, byte-identical in both
fonts. Some notes also list `$27` as `!` and `$31` as `?`
without qualifying the bank; `$27` is a dot in both banks and `$31` is `?` only in
bank 3.

### Text encoding

Byte census over the whole CHR0 message stream (`$1000`-`$139B`, 924 bytes) plus
the text of the two game-over blocks. Every value that occurs there is listed;
nothing else occurs.

    $00-$09   digits 0-9
    $0A-$23   A-Z              (letter = byte - 9)
    $25       !!               (both banks)
    $26       '                (apostrophe)
    $27       .  in bank 3     /  raised dot in bank 2
    $28       .  in bank 2     /  mask glyph in bank 3
    $31       #  in bank 2     /  ?           in bank 3
    $80-$83   line / page break  (only these four of $80-$8F occur)
    $FF       space

Caveat on that census: it covers the CHR0 message stream and the **text** part of
the two game-over blocks. Both game-over blocks are moved as the fixed 168 bytes
and overrun their text by a long way — the tails of CHR2 `$07C0` and CHR0 `$09C0`
contain values (`$30`, `$3E`, `$78`, `$7C`, `$C0`, `$F0` …) that are in no sense
text. Census the whole 168 and the "nothing else occurs" claim is false; census
up to the `$83` terminator and it holds.

### The two period codes

The endings text at CHR0 `$1340`-`$139B` uses *both* dot codes: `$27` in
"SON`$27$27$27`" and "END`$27$27`", `$28` in "YOU WIN`$28$28$28`" and "FOR
NOW`$28`". That looks inconsistent and is not — **the game uses
whichever code is a period in the bank that line will render in.** Dumping every
`$27` and `$28` in the stream shows a clean split with no exceptions:

| region | code used for "." | why |
|---|---|---|
| CHR0 `$1000`-`$1298`, the 15 note/hint blocks 0-14 | `$28` (14 of 14 dots; block 1 has none) | notes render inside buildings, bank 2, where `$28` is the baseline dot |
| CHR0 `$1299`-`$1386`, the three day-end texts (descriptors 2/3/4) | `$27` (9 of 9) | `$D60B` sets bank 3 for the ending sequence, where `$27` is the baseline dot |
| CHR0 `$1387`-`$139B`, block 25, "YOU WIN`$28$28$28`" / "FOR NOW`$28`" | `$28` | the note convention |

The boundary between the two conventions falls *exactly* on `$1299`, which is
descriptor 2's source address — i.e. exactly where the note stream ends and the
ending stream begins. Block 25 sits inside the ending region by address but uses
the note convention, and that is the whole clue.

**So the win screen draws in bank 2, and "YOU WIN..." is three ordinary
periods.** The chain, all of it re-derived here:

1. Banks 0 and 1 have no font (above), so the text must render in bank 2 or 3.
2. Block 25 has exactly one consumer: `lda #$19` at `$BE09`, `jsr $EC78` at
   `$BE0B`. `$EC78` is `textbox_show_block` and does not switch banks.
3. `$BE09` sits inside `$BDE7`-`$BE93`, the **cabin-duel** handler.
   You are inside a building when it fires.
4. Entering a building runs `$E8FD ldy #$02 / jsr $81A4` — bank 2, latched into
   `chr_page` — so bank 2 is ambient for the whole interior. A recursive walk of
   the call graph from `$BDE7` and `$BE00` to depth 4 finds no path to `$81A4` or
   `$81A6`; nothing switches it back before the box draws.
5. Independently: the notes use `$28` as their sentence period and are already
   established to render in bank 2. Block 25 uses `$28`.

Settled by static reading only — no emulator frame was captured, and that is
the one thing that would turn the derivation into an observation. Steps 1, 2, 4
and 5 are byte-level facts; step 3 is carried over from earlier work rather than
re-derived here. The former hedge ("either it is 'YOU WIN 🎭🎭🎭', or the win
screen runs in bank 2 and the *other* two lines are the odd ones") named the
right two horns; the second is correct, and the other two lines are not odd
either, because they render in bank 3 where `$27` is the period.

## The item descriptor table, identified on sight

`$F618` holds 12 records of 4 bytes: byte 0 is the tile base, byte 2 packs the
shape as `width:height` nibbles. What each record DEPICTS was settled by
rendering all twelve from all four CHR banks and having someone who knows the
game name them — not by inference, which had already produced two wrong answers.
The item lists number types one way and `$F541`'s `inc $0517,x` ("item type − 6")
implies another; neither matched what is actually drawn.

| type | base | shape | bank | what it is |
|---|---|---|---|---|
| 1 | `$31` | 2×1 | 0/1 | knife |
| 2 | `$84` | 1×3 | 2 | machete, as an item pickup (Pamela's) |
| 3 | `$8A` | 1×3 | 2 | axe |
| 4 | `$80` | 2×2 | 2 | torch |
| 5 | `$E1` | 2×2 | 2 | pitchfork |
| 6 | `$5B` | 3×2 | 2 | **sweater** |
| 7 | `$05` | 2×2 | 0/1 | **lighter** |
| 8 | `$E5` | 2×1 | 2 | **flashlight** |
| 9 | `$09` | 2×2 | 0/1 | **vitamin** |
| 10 | `$03` | 2×1 | 0/1 | **key** |
| 11 | `$F3` | 2×2 | 2 | note |
| 12 | `$01` | 2×1 | 0/1 | machete, as it appears on the path |

**The art spans two banks**, which is why no single-bank search found all five
carried items: lighter, vitamin and key are in banks 0/1, sweater and flashlight
in bank 2. Bank 3 holds counselor faces at several of these codes, out of order,
so a record rendered from the wrong bank looks like plausible garbage rather than
nothing — the same trap as the counselor-back grouping below.

Note also that types 2 and 12 are BOTH machetes: one as a pickup, one as it
appears lying on the path.

## Bank 2 — items and counselors

Sprite half, PPU `$0000`-`$0FFF`.

| tiles | what |
|---|---|
| `$84`-`$86` | item type 2, a blade — **used** |
| `$87`-`$89` | unused |
| `$8A`-`$8C` | item type 3, an axe — **used** |
| `$8D`-`$8F` | unused |
| `$90`-`$91` | unused (two tiles, not three) |
| `$92` onward | counselor backs — see the grouping note |

"Used" and "unused" here mean the **item descriptor table at `$F618`**: 12 records
of 4 bytes, `$F618`-`$F647` (`$F648` is `lda #$1F`, live code). Byte **0** of each
record is the tile base, and that column holds `31 84 8A 80 E1 5B 05 E5 09 03 F3
01`. Only `$84` and `$8A` fall in the item block, and none of `$87`, `$8D` or
`$90` appears. *(The table that decides this is `$F618`, which is in PRG — not
either of the two CHR descriptor tables above.)*

Byte **2** of each record is the sprite shape as `width:height` nibbles, and it
takes exactly four values across the twelve: `$21 $13 $13 $22 $22 $32 $22 $21 $22
$21 $22 $21`. Item sprites are **1×3 vertical** — but only these two are: `$13`
occurs at records 1 and 2 and nowhere else, so the 1×3 stride is a property of
those two records and not of the block. Applying it past `$91` invented a cut item
that was really part of a counselor sprite; that mistake was made and retracted
during this work, and it is worth keeping in view. **If a grouping renders as
incoherent, doubt the grouping before the artwork.**

### The counselor-back grouping — a live example of the same trap

An earlier pass recorded the counselors as "2 wide × 3 tall on a stride of 8".
That phrasing has two readings and **only one of them is right**:

- ✗ *row* stride 8 — sprite = `$92,$93` / `$9A,$9B` / `$A2,$A3`. This renders as
  three stacked rounded bands. Plausible enough to accept at a glance, and wrong.
- ✓ **six consecutive tiles, and 8 is the stride between sprites** — sprite =
  `$92,$93` / `$94,$95` / `$96,$97`. This renders unmistakably: head, torso with
  arms hanging and hands picked out in the lighter tone, two legs.

Under the correct reading the four figures are `$92`-`$97`, `$9A`-`$9F`,
`$A2`-`$A7` and `$AA`-`$AF`, the first three near-identical and the fourth in a
different pose — exactly what that earlier note describes. This revision took the wrong
reading first and got a coherent-looking picture out of it (the middle rows of
three near-identical figures stack into something that reads as a figure), which
is precisely how the original grouping error survived a second look. Recorded
because the failure repeated, not because the answer changed.

The per-counselor state table at CHR2 `$1D13` is **data in the same bank**, not a
tile range; it is `$E728` block 21 and is listed there, not in the tile table
above.

What the three unused sprites depict is **open**. The Cutting Room Floor documents
three unused graphics for this game — a whistle, a vial and a small candle — and
the count matches, but which is which is not settled and is deliberately not
guessed here.

## Confidence

**Proven, and re-proven for this revision** — every row of this document was
re-derived from the program ROM and the character ROM in a second, independent pass
before it was signed off:

- both descriptor tables, field by field, byte for byte: all 28 `$E728` records
  and all 18 `$E77C` records reproduce the tables above exactly, and `$E7FA` is
  `A9 02`, so `$E77C`-`$E7F9` closes with zero slack;
- the four `$2007` readers (`$821D $8223 $8273 $8278`, no indexed forms), the
  single `jsr $8253` at `$E6D9`, the single `sta $F3` at `$E717`, the single
  `jsr $82CA` at `$E3DD`, the 14 `jsr $E6E3` sites and the 11 `jsr $E69D` sites;
- `$E6E3`'s fixed destination and length as immediates at `$E708`/`$E70C`/`$E719`
  (`A9 58`, `A9 07`, `A9 A8`), and `$E6AD` as the operand byte of `sta $01`;
- the area-to-bank table (`00 01 01 01 00 00 00 01 01 01 01 01 01 01 01`), its
  one reader `$987A`, and its two callers `$95DE`/`$970C`;
- the seven literal `ldy #imm : jsr $81A4/$81A6` bank selections, and that they
  are the only literal ones;
- the `$82CA` encoding, by re-implementation, run on all six CHR streams the ROM
  contains: the two full-screen ones yielding exactly 960 tiles each, and the
  four map-screen ones yielding 256+256+192 tiles and 64 attribute bytes. (The
  seventh stream, `40 00 FF` at `$E45E`, is in PRG and was hand-checked);
- the three zero-slack boundaries those establish (`$107C`, `$14A2`, `$05E0`);
- the copyright year and the exact byte pattern `01 09 | 02 08` that produced the
  "1928" misreading, at both of its two occurrences;
- **banks 0 and 1 carry no font**, by full-tile comparison;
- the bank-dependent font, by byte comparison of all 55 tiles `$00`-`$36` and
  then of all 201 tiles `$37`-`$FF`;
- the text encoding, by exhaustive byte census, and the note-vs-ending period
  convention that splits exactly at `$1299`;
- the outdoor item census (offsets `00 06 08 0B 0E 17 1B 1F 22 26`, 32 entries,
  12 knives / 14 vitamins / 6 keys, 13 vestigial bytes);
- the `$F618` tile-base and shape columns;
- the counselor-back grouping, by rendering both readings.

**Settled:** which bank is live when the win screen (block 25) draws — bank 2,
so "YOU WIN`$28$28$28`" is three ordinary periods. See "The two period codes"
above for the chain and for what is still only static.

**Open:**
- the depiction of the three unused item sprites;
- blocks 19 and 22 are named from their consumers (`$05A0`'s remap table; a
  14-pointer structure — 8 pad bytes then 14 words then payload, all confirmed
  present — staged for `$DF9B`/`$E049`) but their contents have not been decoded
  field by field here;
- block 23's internal structure. Its 9 pointers (`$0774 $077B $0780 $0783 $0786
  $078B $078E $0791 $0794`) do partition the 63 bytes with no slack, but the
  second list's span is 5 bytes where a `count + 2-byte records` reading wants 3.
  Block 24 is clean by contrast: 14 two-byte slots, only 13 of them reachable, 9 of
  the slots live, 3 bytes apart, giving 28 + 27 = 55 bytes exactly — so the two
  maps are not the same shape and block 23's record width is not established
  here;
- what VRAM `$22C0`-`$23BF` holds on the map screen, since no descriptor in that
  block writes it;
- any CHR region that no descriptor names and no tile render explains — those are
  presumed tile graphics but have not each been individually rendered and
  identified.
