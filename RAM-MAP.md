[The Configurizer](https://mlcjr.github.io/friday13-nes/configurizer/) &middot; [Reference](https://github.com/MLCJR/friday13-nes/blob/main/README.md) &middot; **RAM map** &middot; [CHR map](https://github.com/MLCJR/friday13-nes/blob/main/CHR-MAP.md) &middot; [ROM Hacks](https://github.com/MLCJR/friday13-nes/blob/main/ROMHACKS.md) &middot; [Patches](https://github.com/MLCJR/friday13-nes/tree/main/patches/)

---

# RAM map — Friday the 13th (NES, 1989)

The complete 2 KB of internal RAM, `$0000-$07FF`, merged from six independent
region surveys and re-verified byte-for-byte against the program ROM and
the character ROM.

> ### How to read this
>
> This is a datasheet, so it is dense on purpose. Nothing needs reading in order;
> find your address and read that row.
>
> - **`$0000`-`$07FF`** is the console's working memory, separate from the game's
>   program. Values here change constantly while you play.
> - **`$0000`-`$00FF` is the zero page**, which the processor reaches faster, so
>   the busiest values live there and space in it is scarce.
> - **Mirroring**: `$0800`-`$0FFF` *is* the same memory as `$0000`-`$07FF`, seen
>   twice. An index that runs past the end wraps into unrelated variables, which
>   causes more than one defect described below.
> - **Per-counselor arrays** hold six entries, one per counselor, and are swapped
>   into the live bytes whenever you change who you are playing.
> - **proven / open** in the confidence column means exactly that: proven was
>   traced to the instructions that read and write it; open was not.

**Every one of the 2048 bytes is accounted for.** 1438 have a name; 610 are
proven free. Nothing in this document is "probably unused" — see
[Confidence](#3-confidence) for what each tag means.

> **Audited 2026-08-12.** The 615-free figure this document shipped with was
> re-derived from scratch and **two of its claims did not survive**:
> `$0744-$0747` and `$00E4`. Both are written up in place (§10, §4.4) and both
> failed for the same reason — the write is reached by an index or a dispatch
> whose bound lives in a *data table*, so no operand search over PRG can see it.
> The other 610 bytes were re-checked and hold. Details of what the audit did,
> and what it could not close, are in [§14](#14-what-the-2026-08-12-audit-checked).

Companion document: `README.md`, which carries the PRG memory map and the
cartridge facts this table assumes.

---

## 1. How the memory is laid out

The NES gives this cartridge 2 KB of work RAM at `$0000-$07FF`, mirrored three
more times up to `$1FFF`. Mapper 3 (CNROM) adds nothing — no work RAM on the
cartridge, no battery. Everything the game knows lives in these 2048 bytes.

The ROM partitions them into seven sub-regions with hard boundaries:

| Sub-region | Range | Size | Role |
|---|---|---|---|
| [Zero page](#4-zero-page-0000-00ff) | `$0000-$00FF` | 256 | Every subsystem's working registers. Pointers, shadows, the RNG. |
| [Stack page](#5-stack-page-0100-01ff) | `$0100-$01FF` | 256 | PPU transfer queues at the bottom, hardware stack at the top. |
| [OAM page](#6-oam-page-0200-02ff) | `$0200-$02FF` | 256 | Sprite shadow, DMA'd every frame. Reused as a staging buffer with rendering off. |
| [Entity block](#7-entity-block-0300-04ff) | `$0300-$04FF` | 512 | 64 parallel 8-byte arrays. 8 slot columns, 7 of them used. |
| [Persistent game state](#8-game-state-0500-05ff) | `$0500-$05FF` | 256 | Two blocks with different lifetimes — this is where the traps are. |
| [Sound + per-counselor](#9-sound-and-per-counselor-0600-06ff) | `$0600-$06FF` | 256 | Sound-engine channel arrays, then the per-counselor save slots. |
| [Per-counselor + staging](#10-per-counselor-and-chr-staging-0700-07ff) | `$0700-$07FF` | 256 | Rest of the per-counselor arrays, then 168 bytes of CHR staging. |

Three structural facts drive everything else:

**RAM mirroring is load-bearing, not academic.** `$0800-$0FFF` is the same
memory as `$0000-$07FF`, and the consequence is live in the shipped code (Trap
5, §11): `lda $074E,y / ora #$80 / sta $074E,y` with `Y = $FF` writes
`$084D`, which *is* `$004D`, the scroll-speed index. An out-of-range index in
one sub-region corrupts a different one.

**Almost nothing is a struct.** The entity block is 64 parallel arrays, not 8
structs. The per-counselor state is 14 separate 6-byte arrays scattered across
`$0600-$0757`, not 6 records. Reading it any other way produces wrong strides.

**The top 168 bytes are volatile.** `$0758-$07FF` is the CHR-to-RAM staging
buffer, and the length is exactly `$A8` = 168 (`$E719: lda #$A8`,
`$E71B: sta $F1`), so `$0758 + 168 = $0800`. It fills the region to the last
byte of RAM. Anything parked there is destroyed by the next CHR block load.

---

## 2. The bulk writers — why operand searches fail here

Five routines write RAM through pointers. Between them they touch 1,283 bytes
that **no instruction in the ROM names**. This is the project's central caution
in its concrete form: a failed operand search over any of these ranges proves
indirection, not absence.

### 2.1 The RESET clear — `$8014-$802E`

```
$8014: lda #$00      $801D: txa            $8025: iny
$8016: sta $00       $801E: adc ($00),y    $8026: bne $801D
$8018: sta $01       $8020: tax            $8028: inc $01
$801A: tax           $8021: lda #$00       $802A: lda $01
$801B: iny           $8023: sta ($00),y    $802C: cmp #$08
$801C: iny                                 $802E: bne $801D
```

`$8023` clears the whole of `$0000-$07FF` — all 2048 bytes — and names not one
address. `$801E` sums each byte's *pre-clear* value into X first, and `$8030
stx $31` makes that sum the RNG seed. Power-on garbage in RAM literally seeds
the game's randomness.

One caveat, verified: the loop enters with Y untouched from power-on
(`iny / iny` at `$801B`, no `ldy #$00`). If Y were non-zero at reset,
`$0002..Y+1` would be skipped on the page-0 pass. `$0000` and `$0001` are
covered anyway by the explicit stores at `$8016`/`$8018`. Pages 1-7 are always
covered in full because the inner loop exits on Y wrapping to 0.

### 2.2 The generic memset — `$83D6`, exactly four callers

```
$83D6: ldy $02 / dey / $83D9: lda $03 / sta ($00),y / dey / bpl $83D9 / rts
```

Fills `$02` bytes at `($00)` with `$03`. Verified: the complete caller set is
`$8F29`, `$8F44`, `$990C`, `$F575`. Nothing else calls it.

| Caller | Reached from | Pointer | Length | Fill | Clears |
|---|---|---|---|---|---|
| `$8F29` | `$8F0C` table walker | from the CHR record | from the record | from the record | see §2.3 |
| `$8F44` | `$8D2E` (cold boot), `$8D57` (**every scene build** — see §8.2) | `$0568` | `$60` | `$00` | **`$0568-$05C7`** |
| `$990C` | `$956C`, `$9578` (area load), `$E474` | `$0044` | `$60` | `$00` | **`$0044-$00A3`** |
| `$F575` | falls through from `$F565` | `$06F4` | `$14` | `$00` | **`$06F4-$0707`** |

### 2.3 The CHR RAM-init table — CHR block `$14`

`$8D28 jsr $8F0C` loads CHR block `$14` (bank 1, PPU `$107C`, CHR offset
`$307C`; offsets here count from the start of CHR, so this byte sits at
`0xB08C` in a headered `.nes` file) into `$0758`, then walks it as 4-byte
`(len, dest_lo, dest_hi, fill)` records, calling `$83D6` on each. Decoded in
full from the character ROM — 20 records, `len == 0` terminator at +80, 171
bytes written:

| # | Destination | Fill | # | Destination | Fill |
|---|---|---|---|---|---|
| 0 | `$0500-$0527` | `$00` | 10 | `$06A2-$06A7` | `$00` |
| 1 | `$0738-$073D` | **`$20`** | 11 | `$06A8-$06AD` | `$00` |
| 2 | `$0748-$074D` | `$00` | 12 | `$06AE-$06B3` | `$00` |
| 3 | `$074E-$0757` | **`$8F`** | 13 | `$068F-$0694` | `$00` |
| 4 | `$0732-$0737` | **`$03`** | 14 | `$0684-$068E` | `$00` |
| 5 | `$0726-$072B` | `$00` | 15 | `$0677-$0683` | `$00` |
| 6 | `$072C-$0731` | `$00` | 16 | `$0708-$070D` | `$00` |
| 7 | `$0695-$069B` | `$00` | 17 | `$070E-$0713` | `$00` |
| 8 | `$0714-$0719` | `$00` | 18 | `$071A-$071F` | `$00` |
| 9 | `$069C-$06A1` | `$00` | 19 | `$0720-$0725` | `$00` |

**The three non-zero fills are the new-game defaults**, and no instruction in
the ROM contains any of these constants for these addresses:

- `$0738-$073D` = `$20` — all six counselors start at 32 HP.
- `$0732-$0737` = `$03` — all six counselors start with weapon 3.
- `$074E-$0757` = `$8F` — all ten cabin-occupancy slots start "empty/flagged".

It touches **only** `$05xx`, `$06xx` and `$07xx`. Checked negative for zero page,
page 1, page 2 and the entity block — a claim four of the six surveys made
independently and which the byte-level decode confirms.

### 2.4 The entity wipe — `$A829`

```
$A829: lda #$00 / sta $00 / lda #$03 / sta $01 / txa / pha / tay / ldx #$40
$A836: lda #$00 / sta ($00),y / lda $00 / clc / adc #$08 / sta $00
       bcc + / inc $01 / + dex / bne $A836
```

Pointer starts at `$0300`, stride 8, 64 iterations, `Y` = the slot index. It
zeroes **field N, slot Y** for all 64 fields: `$0300+8N+Y`. `64 × 8 = 512`
covers exactly `$0300-$04FF`, and the last store with `Y = 7` lands exactly on
`$04FF`. Verified caller set (13 sites, and the slot each passes):

`$AE04` (0), `$C120` (1), `$C125` (6), `$C132` (1), `$C1A1` (1), `$C2B9` (6),
`$CA81` (2-5 loop), `$D170` (2), `$D274` (5), `$D299` (2), `$D513` (2),
`$DA96` (2-5 from the free-slot scan), `$E636` (0/1 from the attract table).

### 2.5 The CHR streamer — `$8253` / `$8204`

`sta ($12),y` at `$827B` and `sta ($EF),y` at `$8226` copy bytes read out of the
PPU port into RAM. Their destinations are set by the descriptor table at `$E77C`
and the block table at `$E728`; the reachable destinations are `$0200`, `$0300`
and `$0758`. This is the mechanism that makes pages 2 and 3 dual-purpose and
that owns `$0758-$07FF` outright.

### 2.6 The per-counselor swap — `$8EA6`

Not a bulk clear, but the sixth indirection, and the one that hides the most.
`$8EA6` loads a CHR table into `$0758` and walks it as 4-byte
`(dst_lo, dst_hi, src_lo, src_hi)` pointer pairs, then does
`lda ($02),y / ldy $0507 / sta ($00),y` (or the reverse, on `$0F`). It is how
all 14 per-counselor arrays are swapped against the live `$05xx` bytes on a
counselor change. **No instruction names `$0726`, `$072C`, `$069C`, `$06A2`,
`$06A8`, `$06AE`, `$068F`, `$0708`, `$070E`, `$0714`, `$071A` or `$0720` as a
swap participant.** Each one's live partner is named in the per-counselor
tables later in this document.

### 2.7 The complete indirect-store set

There are **exactly 11** indirect stores on any path the CDL ever executed —
plus **three more in orphaned code**, see the correction under the table.
Verified by walking every recovered instruction start, and independently by a
raw scan: `$91` (`sta (zp),y`) occurs as a byte 83 times in PRG and `$81`
(`sta (zp,x)`) 117 times, and of those 200 positions exactly the 11 below are
instruction starts. **There is no `sta (zp,x)` anywhere in the ROM.**

| Site | Instruction | Pointer reaches |
|---|---|---|
| `$8023` | `sta ($00),y` | `$0000-$07FF` (RESET clear) |
| `$83DB` | `sta ($00),y` | wherever `$83D6`'s four callers point it |
| `$8412`, `$841A` | `sta ($00),y` | BCD add/subtract result — this is what writes `$058A-$058C` |
| `$849E` | `sta ($02),y` | zero page only (`$03` forced to `$00` at `$8495`) — the palette buffer |
| `$8ED1` | `sta ($00),y` | live `$05xx` slot, `Y = $0507` |
| `$8EDD` | `sta ($02),y` | per-counselor array slot |
| `$A838` | `sta ($00),y` | `$0300-$04FF` |
| `$8226` | `sta ($EF),y` | `$0758-$07FF` |
| `$827B` | `sta ($12),y` | `$0200`, `$0300`, `$0758` |
| `$F4FB` | `sta ($00),y` | item system |

> **`$83E1-$8409` holds three more indirect stores.** It is a
> BCD **add** routine, the mirror of the BCD subtract at `$840A`, and it
> contains three further `sta ($00),y` — at `$83E9`, `$83F4` and `$8404`. It was
> missed because it is never executed (`cdl=00` throughout) *and* because
> `jsr $83E1` occurs nowhere: the byte triple `20 E1 83` is absent from PRG,
> `jmp $83E1` is absent, the word `$83E1` appears in no table, and the preceding
> instruction is the `rts` at `$83E0`, so it cannot be entered by fall-through
> either. **Per hard rule 6 that is evidence of indirection, not of nothing
> being there** — the sibling `$840A` has two ordinary callers (`$B77C`,
> `$B850`) and `$04` is set up for both by the same code, so a caller very
> plausibly exists behind one of the nine evasions (§2.8). How `$83E1` is entered is
> **OPEN**; that it exists and stores through `($00),y` is not.
>
> The safe form of the claim: **11 indirect stores on executed paths, 14 in the
> ROM's text.** None of the extra three changes any reachability conclusion in
> this document, because `$83E1`'s pointer is whatever its (unlocated) caller
> puts in `$00/$01`.

And exactly **six** computed jumps: `jmp ($0020)` at `$80B4`, `jmp ($0000)` at
`$AEB2`, `$CB0C`, `$D0AD`, `jmp ($0002)` at `$D907`, `jmp ($00EB)` at `$F7A4`.
Three of the six are now resolved to their tables: `$AEB2` → `$AF6C` (17
entries), `$CB0C` → `$CB55` (13), `$F7A4` → `$F87B` (16). All 46 targets decode
as code and are listed where the relevant field is documented (§7.2, §4.4).

### 2.8 The nine evasions

The lettered catalogue the rest of this document cites. Each row is a distinct
mechanism by which this ROM defeats a static operand search, and each has a
canonical instance, most of them in this document's own pages:

| # | Mechanism | Canonical instance |
|---|---|---|
| (a) | data stored in CHR-ROM, read through the PPU port | the RAM-init table (§2.3) and the per-counselor swap table (§2.6) — the addresses live as table bytes, never in an instruction operand |
| (b) | address written only through a zero-page indirect | the alarm digits `$058A-$058C`, written by `sta ($00),y` inside the BCD routine `$840A` (§8.2) |
| (c) | address, or index, rebuilt by split `adc #imm` | `$9F63`, assembled as `adc #$63` / `adc #$9F` — the byte pair `63 9F` occurs nowhere in PRG or CHR (§4.3) |
| (d) | bit test as mask-then-compare, not mask-and-branch | `cmp #$60` where a scan greps for `and #$60` |
| (e) | `jmp` into the middle of another routine | `$DD86 jmp $DCF6` — one of 35 confirmed mid-routine entries; it is the house idiom |
| (f) | value smuggled on the hardware stack | `$82A6`/`$82BA` (§5.4) |
| (g) | jump table inlined after the `jsr` | `$D8EB` (§5.4) |
| (h) | computed `jmp ($xxxx)` from split immediates | `$80B4` through `$0020`, seeded `lda #$F0` / `lda #$8C` (§4.2) |
| (i) | indexed access whose base sits before the region and runs into it | `$073E,y` with a table-held bound of 9, reaching `$0744-$0747` (§10) |

Self-modifying code is ruled out as a checked negative: PRG is ROM on this
board, and every write instruction's target was tested against the CDL CODE
flags with zero hits.

---

## 3. Confidence

| Tag | Means |
|---|---|
| **proven** | The stated writers and readers were re-derived here from the program ROM. Where a data table is cited, its bytes were dumped and matched. |
| **likely** | The mechanism is proven at instruction level; the *label* is inference from call-site neighbourhood. Treat the name as a hypothesis, the addresses as fact. |
| **open** | Behaviour observed, purpose not established. |

**Method.** Instruction starts were recovered by walking every maximal
CDL-CODE run forward with an opcode-length table (the first byte of a maximal
run is provably an instruction start, because every byte of an instruction is
flagged CODE), seeded additionally from every byte carrying the branch-target
and subroutine-entry bits. That yields 9,918 instruction starts; a
recursive-descent pass over static control flow adds 250 more in code neither
recorded TAS executed. Never-executed regions were then swept with a raw
byte-pair scan over all 32,768 PRG bytes and each hit adjudicated by hand
against the CDL data flag. `da65` was used only as a reading aid.

**Where the CDL method under-reports.** The recovered-instruction xref is a
lower bound: it cannot see orphaned code. Three examples found in this merge —
`$A8CE` (the START toggle), `$F92B-$F97C` (sound commands), `$F55C`
(`inc $051B`, the sweater) — are all real code that neither TAS entered. The
`$0620-$062F` sound array exists *only* in such code and was missing from every
input survey until the brute scan found it.

---

## 4. Zero page (`$0000-$00FF`)

166 live, 7 named-but-dead, 83 proven free. The heavily-aliased range
`$0000-$0018` carries three or four unrelated meanings depending on which
subsystem is running; the discriminator is given per row.

### 4.1 Scratch pointers and the shared request record (`$0000-$0018`)

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$00-$01` | scratch pointer A | The ROM's primary 16-bit indirect base — ~90 `(zp),y` instructions. **Also** bytes 0-1 of the PPU-queue request record, **also** the RLE decoder's PPU destination. | 158 `sta/sty/stx $00`, 105 `sta $01`. Key stagers: `$98FE/$9902`, `$8F1D/$8F22`, `$8F36/$8F3A`, `$959F/$95A5`, `$E3F1/$E3F5`, `$82A7/$82AA` | `$83DB`, `$8693/$8698`, `$82D6/$82DB`, + 89 `(zp),y` sites | proven |
| `$02-$03` | scratch pointer B / memset args | Second pointer pair; simultaneously the memset's length (`$02`) and fill byte (`$03`). Bytes 2-3 of the queue record. | 72 `sta $02`, 39 `sta $03`. As memset args: `$8F18/$8F27`, `$8F3E/$8F42`, `$9906/$990A`, `$F56F/$F573` | `$83D6`, `$83D9`, `$82E2/$82F0/$8304`, `$849E`, `$8410` | proven |
| `$04` | byte count | Digit/byte count for BCD add `$83E1`, BCD subtract `$840A`, ZP memcpy `$8491`. Byte 4 of the queue record. | `$8443`, `$8461`, `$86D5`(=3), `$8794`(=$10), `$B77A`, `$B84E`, `$E617` | `$840A`, `$8499`, `$86A7`, `$8AC2`, `$8B61` | proven |
| `$05-$06` | queue bytes 5-6 / scratch | Source hi + flags of the queue record; reused freely as scratch by the compositor, sprite builder and duel code. | `$05`: 16 sites incl. `$8ADE`, `$9448`, `$AB0B`, `$C0A2`. `$06`: 16 incl. `$8447`, `$8A56`, `$AB06`, `$AD32` | `$86AC/$86B1`, `$8AC6`, `$945B`, `$AB40`, `$C0A6` | proven |
| `$07-$0A` | collision box B | Second axis-aligned box in the collision engine: `$07` = x_min, `$08` = x_max, `$09` = y_min, `$0A` = y_max. Box A is `$00-$03`, which is why the two uses never overlap in time. | `$DDBD` (`sta $08`), `$DDC5` (`sta $07`), `$DDCD` (`sta $0A`), `$DDD5` (`sta $09`), all `= table + $0360/$0370`; also `$DC0B`, `$DD60`, `$DE72-$DE96` | the overlap test `$DDF1-$DE5A` (`cmp $07/$08/$09/$0A/$00/$01/$02/$03/$0B`) | proven |
| `$07-$08` | glyph/name source pointer | Second role: pointer pair for the glyph copier. | `$8A2F/$8A33`, `$8A97/$8A9B` | `$8A3E`, `$8A43`, `$8A48`, `$8A4D` (`lda ($07),y`) | proven |
| `$0B` | collision height scratch | — | `$DDEF`, `$DE29` | `$DE5A` | likely |
| `$0C` | loop counter | Sprites emitted (BUILD_SPRITE) / rows (text-box renderer). | `$AB5C`, `$AC5A`, `$EC87`; `inc` at `$AC15`, `$AD1A`, `$ECC8` | `$ABED`, `$ACF2`, `$ECB8` | likely |
| `$0D-$0E` | hurtbox / map-seed pointer | Built by split `adc #imm`: `$DEB2/$DEB8` = `&$DEE9 + type*4`; `$E0D6/$E0DA` = the map seed. Evasion mechanism (c), §2.8. | `$DEB2/$DEB8`, `$E0D6/$E0DA`. `$0D` also plain scratch at `$9472`, `$ABD0`, `$C1A9`, `$EC8B` | `$DEC0`, `$DECC`, `$DEDA`, `$DEE2`, `$E0E1` | proven |
| `$0E-$0F` | scratch pointer C | Third general indirect base — 16 `(0E),y` sites. Counselor palette loader, HUD/OAM builder, camera dispatch, Jason's cabin bytecode, the script reader. **`$0E` is deliberately shared with the pair above.** | `$0E`: 27 sites incl. `$8714`, `$8BC8`, `$BAD8`, `$D386`, `$E2BE`, `$EC7F`. `$0F`: 37 sites | `$8726`, `$8BD1-$8C13`, `$BAE8`, `$D38F-$D3ED`, `$E2C7`; + ~40 direct reads of `$0F` as a counter | proven |
| `$10-$16` | CHR transfer descriptor | Unpacked 7-byte record from `$E77C`: `$10/$11` PPU source, `$12/$13` RAM dest, `$14/$15` length, `$16` CHR bank. | `$E6B9-$E6D7`; `$12/$13` bumped at `$827D/$8281`, `$14/$15` decremented at `$8283/$8287`; **also `$8244 stx $10`**, where the mid-frame streamer borrows `$10` as scratch for its constant-time delay — harmless only because `chr_read`'s one caller reloads all seven fields from `$E77C` first | `$8256`, `$8262`, `$8268`, `$826D`, `$827B`, `$8249` | proven |
| `$10-$12` | **alias:** scroll staging | `$10` X scroll, `$11` nametable-select (bit 0 → PPUCTRL), `$12` Y scroll. Safe alias: `chr_load_descriptor` disables NMI via `$29` while it runs. | `$815E/$8162/$8166` (from `$44/$45/$46`), `$8186/$818A/$818E` (from `$48/$49/$4A`), `$819B-$819F` (force 0,0) | `$8168-$8183` | proven |
| `$11-$18` | **alias:** queue-record working copy | `$863F` unpacks a record into `$11` dest_lo, `$12` dest_hi, `$13` width, `$14` rows, `$15/$16` source, `$17` flags, `$18` inner counter. | `$863F-$8660`; `$11` advanced +8 at `$8514` / +`$20` at `$85E0` | `$84FF-$8512`, `$8548-$858B`, `$85C9-$85EF` | proven |
| `$17` | queue-record flags | bit7 → constant-fill mode `$85BC`; bit6 → incrementing-fill mode `$85F4`; bit2 → PPUCTRL VRAM-increment select. | `$8660`, `$8557` | `$8548`, `$8553/$855D`, `$858B` | proven |
| `$18` | fill-mode byte counter | Counts bytes remaining in the current fill; the two `dec` sites are its only consumers. | `$85CB`, `$8603`; `dec` at `$85DC`/`$8616` | (decrement only) | proven |

### 4.2 System state (`$0019-$0043`)

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$19-$1E` | — | **Unreferenced.** Outside every memset; permanently `$00`. | RESET only | none | proven |
| `$1F` | saved queue-A count | Snapshot of `$0100` before the queue is cleared, and of `$B7` in the blitter; drives a variable dummy-read delay. | `$853A`, `$8342`; `dec` at `$85B1` | `$83C4` (`lda #$0E / sec / sbc $1F`) | proven |
| `$20-$21` | computed jmp vector | `jmp ($0020)` at `$80B4` is the ROM's main control-flow indirection. | RESET `$804B/$804F` seeds `$8CF0`; dispatcher copies `$A5/$A6` in at `$80AE/$80B2` | `$8065/$8069`, `$80B4` | proven |
| `$22` | — | **Written once, never read.** `sta $22` at `$8053` is the only reference in the ROM. Sits immediately after the jump vector; plausibly a vestigial third byte. | `$8053` | **none** | proven |
| `$23-$28` | — | **Unreferenced.** | RESET only | none | proven |
| `$29` | PPUCTRL shadow | Last value written to `$2000`. RMW'd by everything that changes NMI-enable, VRAM increment or the nametable bit. | 21 sites: `$80C1`, `$814A`, `$816C`, `$81F4`, `$842A`, `$855F`, `$8B90`, `$8D0A`, `$9707`, `$B23D`, `$E6A3`, … | 23 sites incl. `$80BD`, `$8146`, `$8168`, `$8209`, `$84C0` | proven |
| `$2A` | PPUMASK shadow | Value the NMI writes to `$2001`. Zero blanks the screen. | `$82B1`(=0), `$82C1`, `$8D14`(=$1E), `$8D65`, `$DFBA`, `$E365` | `$80CE`, `$82AC` | proven |
| `$2B` | vblank handshake | NMI sets it at end of frame; main thread spins on it and clears it. | `$805C`, `$8151`, `$8092` | `$808C` (`lda $2B / beq $808C`) | proven |
| `$2C` | palette-dirty flag | Non-zero → upload `$C0-$DF` on the next NMI. | `$84B9`(=1), `$8497`(=0), `$84D0`(=0), `inc` at `$E553` | `$84BC` | proven |
| `$2D` | joypad 1 raw | `ror $2D,x` at `$81C2` with X = 0 then 1 is the only writer of either pad — one of only three `zp,x` instructions in the ROM. | `$81C2` | `$81CA`, `$AFB5`, `$B06B`, `$B131`, `$B1A9`, `$B466`, `$BC36`, `$BE21`, `$BE32`, + the orphaned `$A8CE` | proven |
| `$2E` | joypad 2 raw | **Written, never read.** Player 2's pad is assembled into RAM and discarded. Consistent with a one-player game. | `$81C2` (X=1) | **none** | proven |
| `$2F` | — | **Written once, never read.** Zeroed at `$8CF9` alongside `$32` and `$33`. Grouped with two other retired/near-retired flags, but that is a guess. | `$8CF9` | **none** | proven |
| `$30` | — | **Unreferenced.** | RESET only | none | proven |
| `$31` | RNG byte / frame counter | Seeded at power-on with the sum of all pre-clear RAM; incremented once per frame by the NMI. Every random decision is `lda $31` + a mask. | `$8030` (`stx $31`), `$8153` (`inc $31`) | 11 sites: `$B590`, `$BBCD`, `$CDEA`, `$D0C0`, `$D137`, `$D1CD`, `$D1DA`, `$D1FF`, `$D560`, `$E0D2`, `$F3F3` | proven |
| `$32` | START toggle — **orphaned** | bit7 = toggle state, bit3 = "START held". Only the initialisation survives. | `$8CFD`(=0), `$AE5A`(=`$08`, every main-loop restart) | **none on any reachable path.** The only reader is `$A8CE-$A8E7`, a textbook edge-detected START toggle with **zero callers** — no `jsr`/`jmp`, and the byte pair `CE A8` occurs once in the ROM, at `$A40C`, inside the proven metatile table. CDL never executed it. | proven |
| `$33` | global game-state flags | bit0 = counselor DEATH, bit1 = "inside a building". | `$8CFB`, `$8DFC`, `$8E2C`, `$B2C8`, `$B412`, `$BBA8`, `$BD76`, `$DD1E` | 22 sites incl. `$885B`, `$8D8F`, `$AE74`, `$AECC`, `$B2C4`, `$BBA6`, `$DC55`, `$E07B` | proven |
| `$34-$35` | — | **Unreferenced.** | RESET only | none | proven |
| `$36` | title/attract progress | Non-zero suppresses the intro sound and changes the attract branch. | `$8CF2`, `$E028`, `$E46C`; `inc` at `$E55B`, `$E5C4` | `$E01F`, `$E51C` | likely |
| `$37` | current CHR bank | Mirror of the CNROM latch, so code can restore the bank after a temporary switch. | `$81A4` (`sty $37`; `$81A6` is the entry that deliberately skips it), `$E3AA` | `$812C`, `$8193`, `$8253`, `$9610`, `$E3B8` | proven |
| `$38` | NMI phase selector | 0 = drain both queues + live scroll; 1 = CHR stream + secondary scroll; 2 = CHR stream; 3 = drain both queues. | 17 sites incl. `$8CFF`, `$8D81`, `$98CD`, `$BF34`, `$D612`, `$E6ED`(=2), `$E725` | `$80D3`, `$80DF`, `$BF2F`, `$E6E5` | proven |
| `$39-$43` | — | **Unreferenced.** | RESET only | none | proven |

### 4.3 The compositor / camera block (`$0044-$00A3`)

**This entire 96-byte run is zeroed by `$98FC` on every area or junction load.**
Eighteen of its bytes have no instruction naming them anywhere — they exist only
as memset targets.

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$44-$45` | live scroll X-lo / NT+X-hi | Applied by the NMI in phase 0. `$45` bit 0 selects the nametable. | `$44`: `$8E05`, `$8FDE`, `$96D5`, `$9738`, `$9921`, `$D616`, `$DFC5`, `$E374`, `$E8C0`. `$45`: `$8E07`, `$8FE4`, `$96DF`, `$9927`, `$D618`, `$E376`, `$E8C2` | `$815C/$8160`; `$44` also `$8FD9`, `$B24A`, `$C090`, `$DF7D`; `$45` also `$B23F` | proven |
| `$46` | live scroll Y | **Read once, written nowhere.** Permanently `$00`, so the primary scroll's Y is hardwired to 0 — the game never scrolls vertically through this path. | none | `$8164` only | proven |
| `$47` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$48-$49` | secondary scroll X-lo / hi | Applied by `$8184` during NMI phase 1. | `$48`: `$96DB`, `$9723`, `$9985`, `inc $9043`. `$49`: `$96E1`, `inc $9047`, `dec $9989` | `$8184/$8188`; `$48` also `$96D7`, `$9721` | proven |
| `$4A` | secondary scroll Y | **Read once, written nowhere.** Same finding as `$46`. | none | `$818C` only | proven |
| `$4B` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$4C` | scroll-axis select | 0 = horizontal, non-zero = vertical. `$8FC4 lda $4C / beq + / jmp $9910`. | `$B392`, `$B3BA` | `$8FC4`, `$D0C6`, `$D2A3`, `$D9C3`, `$DAA8` | likely |
| `$4D` | scroll speed index | Index into the 2-byte speed records at `$919A`. **Corruptible via RAM mirroring** — see §1. | `$B340`, `$B354`, `$B379`, `$B3A0`, `$B3AD`, `$B3BE` | `$8FB1`, `$9187`, `$B39C` | proven |
| `$4E` | whole-step count | Scroll steps this frame, from the speed record. | `$918E` | `$8FB5` | likely |
| `$4F-$50` | scroll fraction accumulator / denominator | `$915C` adds 16 per call and emits a pixel when the accumulator passes `$50`. Table `$919A` = `00 00 \| 01 40 \| 01 20 \| 01 18 \| 01 10 \| 02 20 \| 02 10` (verified). | `$4F`: `$9168`, `$916D`, `$9197`. `$50`: `$9193` | `$915C`, `$9161/$9166` | proven |
| `$51-$52` | pixel accumulators (mod 16) | At 16 a whole metatile column/row has been crossed. | `$51`: `$8FEB`, `$8FF7`, `$992E`, `$9937`. `$52`: `$9055`, `$9994`, `inc $9041`, `dec $997E` | `$8FE6`, `$9929`; `$904C`, `$998E` | proven |
| `$53`, `$55` | area room indices | Stepped by `$9172` (+1, wrap when `== $8C`) and `$917C` (−1, wrap to `$8C−1`) — both verified. | `$53`: `$90B1`, `$963A`, `$99EB`. `$55`: `$90AA`, `$963D`, `$99E4` | `$53`: `$90AC`, `$99E6`, `$BA46`, `$D9E0`. `$55`: `$90A5`, `$99DF`, `$D9C7`, `$E284` | likely |
| `$54`, `$56` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$57-$58` | PPU write cursor A | Running PPU address for one composited strip; advanced 2 columns at a time by `$9144`, which `eor #$04`s the high byte on wrap. | `$9087/$9089`, `$973A/$9729`, `$97CD/$97CF`, `$99C5/$99C7` | `$9080/$9082`, `$938D`, `$93A5`, `$93C2`, `$97C6`, `$99BE` | proven |
| `$59-$5E` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$5F-$60` | PPU write cursor B | The other strip's PPU address, initialised to `$24DC`/`$24E0`. Note the deliberate lo/hi swap at `$91DB`/`$91DF`. | `$90C7/$90C9`, `$9725/$9714`, `$97D8/$97DA`, `$9A01/$9A03` | `$90C0/$90C2`, `$91DB-$921C`, `$97D1`, `$99FA` | proven |
| `$61-$62` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$63-$64` | 6-step counters (mod `$78`) | `$908B-$90A3`: `adc #$06 / cmp #$78 / bne + / lda #$00`. 20 states of 6. Handed to `$8B`. Semantic label is a guess; the arithmetic is proven. | `$63`: `$9096`, `$973E`, `$97E1`, `$97F6`, `$99D2`. `$64`: `$90A3`, `$97FA`, `$99DD` | `$908B`, `$91AC`, `$926D`, `$92CD`, `$C0AD`; `$9098`, `$99D4`, `$9A94` | likely |
| `$65` | camera scroll delta | Signed-magnitude pixels the camera moved this frame; every world object is shifted by it. bit7 = leftward. | `$8FD7` (`ora #$80`), `$8FAD`(=0), `$991A`, `$B2C2`, `$B67C`, `$BBAC`, `$D1A2/$D1A6` | `$9022`, `$9029`, `$9961`, `$A8E8` (shifts every object's X), `$AE88`, `$F5BC` | proven |
| `$66` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$67-$6A` | metatile-walker state | `$67` tile-pair count, `$68` stride selector (4 or 6), `$69` row base, `$6A` column base. Field names inferred from `$94E7-$9534`. | `$67`: `$925F`, `$9336`, `$9372`, `$941B`, `$EF44`, `inc $9500`. `$68`: `$925B`, `$9332`, `$9531`. `$69`: `$924B`, `$9322`, `$9407`. `$6A`: `$924F`, `$9326`, `inc $94E4` | `$94F0`; `$94FA`, `$9508`, `$952A`; `$94C5`; `$94C1`, `$94CA` | likely |
| `$6B` | new-strip-needed flag | Set to 1 when the pixel accumulator crosses 16. **No reader anywhere in the ROM.** Either vestigial, or the information is carried by fall-through control flow instead. | `$8FAF`(=0), `$8FFB`(=1), `$993B`, `$D1A4`(=0) | **none** | proven |
| `$6C-$6D` | — | Unreferenced; memset-cleared. | `$98FC` only | none | proven |
| `$6E-$6F` | column-in-area / row-in-area | Camera position inside the area, in metatile units (0-`$0F`); wrapping advances `$75`/`$76`. Saved/restored across a full-screen redraw by `$9640-$9651` / `$96EC-$96FE`. | `$6E`: `$900A`, `$95CF`, `$965B`, `$96FC`, `$9771`, `$9948`, `inc $8FFD/$9767`, `dec $9635/$993D`. `$6F`: `$9064`, `$95D5`, `$9666`, `$96F9`, `$97A9`, `$99A1`, `inc $9057/$979F`, `dec $9637/$9996` | `$6E`: `$8FFF`, `$9271`, `$9640`, `$972D`, `$9AC1`. `$6F`: `$9059`, `$91B0`, `$9643`, `$9718`, `$9A98` | proven |
| `$70` | scroll-lock bitmask | bit0 = horizontal, bit1 = vertical. Consumed **destructively**: `$8FCB` does `lda #$01 / and $70 / sta $70 / bne bail`; `$9910` does the same with `#$02`. Writers set `$FF` ("lock both") and each camera path clears all but its own bit — so one `$FF` write locks one frame in each direction. | `$8FCF`, `$9914` (the destructive masks), `$B678`, `$B9B3`, `$CD27`, `$D1B3`, `$DFE8` | `$8FCD`, `$9912`, `$AED5`, `$B323`, `$B34E` | likely |
| `$71-$72` | area row array 1 pointer | From the area-type table `$A163`. Never dereferenced in place — always copied to `$00/$01` first. | `$95F5/$95FA` | `$900C`, `$9275`, `$9668`, `$9773`, `$994A`, `$9ACC` | proven |
| `$73-$74` | area row array 2 pointer | From `$A1DF`. | `$95FF/$9604` | `$9066`, `$91B4`, `$96A6`, `$97AB`, `$99A3`, `$9AA3` | proven |
| `$75-$76` | row-array indices (1-based) | Wrap by re-reading: `ldy $75 / iny / lda ($00),y / bpl + / ldy #$01` — a negative byte terminates the array and restarts it at 1. | `$75`: `$901D`, `$95BC`, `$968C`, `$96F6`, `$9785`, `$995C`, `inc $977B`. `$76`: `$9077`, `$95CA`, `$96CA`, `$96F3`, `$97BD`, `$99B5`, `inc $97B3` | `$75`: `$9014`, `$927D`, `$92D9`, `$9646`, `$9AD4`. `$76`: `$906E`, `$91BC`, `$9649`, `$9AAB` | proven |
| `$77-$78` | saved row indices | Restore the walk position after a full-screen redraw. | `$968E`, `$96F0`, `$9789`; `$96CC`, `$96ED`, `$97C1` | `$964C`, `$964F` | likely |
| `$79-$7A` | vertical sub-row accumulator / divisor | `$79` accumulates scroll magnitude; each time it reaches `$7A` a whole map row has been crossed. `$7A` is loaded from the 15-byte area-type table `$9815` (verified: `02 01 01 01 02 02 02 01 01 01 01 01 01 01 01`) by `$95EA/$95ED`. | `$79`: `$902E`, `$903D`, `$996B`, `$997C`. `$7A`: `$95ED` | `$79`: `$902C`, `$9034`, `$9969`. `$7A`: `$9036`, `$903B`, `$9973`, `$B229` | proven |
| `$7B-$7C` | metatile selector table pointer | From the 2-entry table `$9B18`, indexed by CHR bank × 2. Reused by the glyph renderer. | `$9621/$9626`, `$EE15/$EE19`, `$EF21/$EF1E` | `$9433/$9437`, `$EE07/$EE0A` | proven |
| `$7D-$7E` | metatile record table pointer | From the 2-entry table `$A3AD`. | `$9617/$961C`, `$EE1D/$EE21`, `$EF1B/$EF18` | `$9552/$9559`, `$EE0D/$EE10` | proven |
| `$7F-$80` | `$9F63` table pointer (array 1) | Built by `$9824` as `(A × 16) + $9F63` using split `adc #$63` / `adc #$9F` — the idiom that hid `$9F63` from every byte-pair scan. | `$9829/$982F` only | `$928B/$928F`, `$93FD/$9401` | proven |
| `$81-$82` | `$9F63` table pointer (array 2) | Identical construction at `$9852`. | `$9857/$985D` only | `$91CA/$91CE`, `$9241/$9245` | proven |
| `$83-$84` | compositor pass / phase counters | `$84` selects which of two passes is running; `$83` is the step count within it. | `$83`: `$90BB`, `$90E5`, `$9112`, `$96E8`, `$99F5`, `$9A36`, `$9A65`, `inc $9108/$9A5B`. `$84`: `$90CD`, `$9141`, `$96EA`, `$9A07`, `$9A8D`, `inc $9137/$9A83` | `$90D2`, `$910A`, `$9A5D`; `$90B5`, `$90DF`, `$9117`, `$99EF`, `$9A85` | likely |
| `$85` | metatile half/quadrant selector | Small constant (1, 2 or 7) telling the selector walker which part of a metatile pair to emit. | `$91D4`, `$9263`, `$9295`, `$931E`, `$9362`, `$939E`, `$93B9`, `$EEA6`, `$EEB9` | `$948E`, `$94A9`, `$94B9` | likely |
| `$86-$87` | `$A345` row-array pointer | Built by `$9832` as `(A × 8) + $A345` — split `adc` again. | `$9849/$984F` only | `$92EA/$92EE` | proven |
| `$88-$89` | bypass selector table pointer | From the 2-entry table `$A2C1` (bank 0 → `$A2C5`, bank 1 → `$A2D5`). | `$962B/$9630` | `$9302/$9309` | proven |
| `$8A-$8B` | compositor working pair | `$8A` is a 16-bit carry partner cleared at `$91AA`; `$8B` carries the `$63` row base into the tile-pair writer. | `$8A`: `$91AA`, `$926B`, `$9A92`, `$9ABB`. `$8B`: `$91AE`, `$926F`, `$92CF`, `$9A96`, `$9ADE` | `$9221`, `$92A6`, `$93DD`; `$9249`, `$9320`, `$9405` | likely |
| `$8C` | area wrap modulus | The modulus `$9172`/`$917C` wrap the room indices against. Loaded by `$95E5 lda $9806,y / $95E8 sta $8C` with Y = area type. **Demoted from "rooms per area":** the table reads `C0 40 40 40 80 00 80 40 40 40 40 40 40 40 40`, which does not plausibly mean 192/64/128/0 rooms. The wrap arithmetic is proven; the unit is not. | `$95E8` | `$9175`, `$9181`, `$BA4A/$BA4F` | proven / label **open** |
| `$8D-$8E` | current column / current row | `$8D` is consumed as the starting column by the tile-pair writer; `$8E` holds the row (`$6F`). | `$8D`: `$9273`, `$9ACA`. `$8E`: `$91B2`, `$9AA1` | `$9287`, `$92E0`, `$93A0`; `$91C6`, `$91EA`, `$9201` | likely |
| `$8F-$90` | area row array 3 pointer | From `$A245`. | `$9609/$960E` | `$92D1/$92D5`, `$9695/$9699`, `$9790`, `$9AE7` | proven |
| `$91` | dead store | Receives one byte read out of row array 3; nothing consumes it. Zeroed on every area load anyway. | `$96A1` only | **none** | proven |
| `$92-$A3` | — | **18 bytes with no instruction reference of any kind.** Zeroed by the `$98FC` memset on every area load and never touched again. | `$98FC` only (indirect) | **none** | proven |

### 4.4 Dispatcher, blitter, palette, sound, CHR streamer (`$00A4-$00FF`)

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$A4` | frame-dispatcher state | 0 = keep looping; 1 = count `$A9` frames down then become 4; 4 = state 2 + `rts`; 5 = state 2 + `jmp ($0020)`. | `$8063`, `$8072`, `$808A`, `$80A5`, `$80AA` | `$8077`, `$8094` | proven |
| `$A5-$A6` | dispatcher jump target (staged) | Copied into `$20/$21` immediately before `jmp ($0020)`. | `$8067/$806B` | `$80AC/$80B0` | proven |
| `$A7-$A8` | — | **Unreferenced.** | RESET only | none | proven |
| `$A9` | frame countdown | Frames left in the blocking wait. `sta $A9 / lda #$01 / sta $A4` at `$806E-$8072` is the 29-call-site "wait N frames" primitive. | `$806E`, `$8084` | `$807D` | proven |
| `$AA-$B3` | — | **Unreferenced.** The cleanest contiguous free space in zero page (10 bytes). Includes `$B2`, which is *not* an indirect base — all five raw `xx B2` pairs in the ROM sit inside the metatile or glyph tables. | RESET only | none | proven |
| `$B4-$BF` | PPU blitter descriptor | 12-byte request block consumed by `$831F`, which **resets most fields to defaults as it reads them**: `$B4` stream-A PPU hi (→`$24`), `$B5` lo (→`$00`), `$B6` PPUCTRL increment bits (→`$00`), `$B7` byte count (→`$0E`), `$B8/$B9` source (→`$0000`), `$BA` stream-B PPU hi (→`$27`), `$BB` lo (→`$C0`), `$BC` row-stride flag (→`$00`), `$BD` row count (→`$08`), `$BE/$BF` stream-B source (**not** reset). | RESET seeds `$B4`/`$B7`/`$BA`/`$BB`/`$BD` at `$8037-$8047`; `$831F` rewrites `$B4-$BD`. Producers: `$87D4-$87F4`, `$920A-$9216`, `$9236`, `$92BE-$92CA`, `$93CC-$93F8`, `$ECA8-$ECE6`, `$8977`, `$8AC4` | `$8321`-`$838F` (all inside `$831F`); `$87A6/$87A8` (`lda $B8 / ora $B9` — "is a request pending?") | proven |
| `$C0-$DF` | palette staging buffer | 32-byte shadow of PPU `$3F00-$3F1F`. `$C0-$CF` background, `$D0-$DF` sprite. **Most of it is written only indirectly:** `$8491` is a memcpy whose destination high byte is forced to zero (`lda #$00 / sta $03` at `$8493/$8495`), so `sta ($02),y` always lands in zero page. **No instruction names `$C1-$CF` or `$D9-$DF` at all.** | (1) `$84A7` mirror-backdrop: `lda $C0` → `$C0/$C4/$C8/$CC/$D0/$D4/$D8/$DC`. (2) `$8728 sta $D8,x`, X = 0-7, from the counselor palette record. (3) `$8491` via `$86D7` (`$02`=`$D5`, len 3), `$8796` (`$02`=`$C0`/`$D0`, len `$10`), `$E619` (`$02`=`$D8`, len 4). (4) `$E551` (`sta $C2`) | `$84D6` (`lda $00C0,y`, Y = 0-`$1F` — an *absolute* access to zero page); `$872E` (`lda $DD` → `$05BB`) | proven |
| `$E0-$E2` | sound request queue | `$E0` count (0-2), `$E1`/`$E2` the two slots. `$F675` enqueues, `$F680` dequeues, silently dropping when full. | `$F67B` (`sta $E1,x` — the third and last `zp,x` in the ROM), `$F68A`, `$F68E`, `inc/dec $E0` | `$F675`, `$F680`, `$F684`, `$F688` | proven |
| `$E3` | active-channel bitmask | One bit per playing channel; the per-frame tick bails if zero. | `$F66A`(=0), `$F6AD` (`ora $FA29,x`), `$F9A8` (`eor`) | `$F6AB`, `$F6F6`, `$F9A6`, `$F9B5` | proven |
| `$E4` | sound-command 14 counter | **Named but dead — not free.** `inc $E4` at `$F99F` is entry 14 of the sound-command table `$F87B`, dispatched by `jmp ($00EB)` at `$F7A4`. Nothing reads it. See the correction below. | `$F99F` | **none** | proven |
| `$E5-$E6` | sound header pointer / channel index | **Dual role.** During cue start (`$F690-$F6F3`) this is a 16-bit pointer into the header data at `$FAE1 + id*2`. During the per-frame tick (`$F6F6` on) the *same bytes* become `$E5` = the remaining channel bitmask being shifted right, `$E6` = the current channel index 0-7. | `$E5`: `$F697`, `$F704`, `inc $F6CB/$F6D5/$F6ED`. `$E6`: `$F69C`, `$F6FF`(=`$FF`), `inc $F701` | `$E5`: `$F69E/$F6D1/$F6DB` (`lda ($E5),y`) + 12 direct. `$E6`: 28 direct, mostly `ldx $E6` | proven |
| `$E7-$E8` | sound scratch / APU register offset | `$E7` = channel byte during header parse, **doubled** index during the tick. `$E8` = `(channel × 4) & $0F`, the offset added to `$4000`. This is why channels 4-7 fold onto the same four generators. | `$E7`: `$F6A2`, `$F6D3`, `$F71C`, `$F773`. `$E8`: `$F6A6`, `$F721`, `$F778` | `$E7`: `$F6E0` + 9 `ldx $E7`. `$E8`: `$F6C9` + 7 `ldx $E8` | proven |
| `$E9-$EA` | note-stream pointer | Reloaded from `$05C8,x`/`$05C9,x` each tick, written back on exit. | `$F77D/$F782`, `$F913/$F918`, `$F98C/$F98A`, `$F9F8/$F9FD`, + 8 `inc` sites | 12 `lda ($E9),y` sites: `$F78C`, `$F794`, `$F7A7`, `$F8A4`, `$F8BF`, `$F8D1`, `$F8E1`, `$F8F1`, `$F95F`, `$F97F`, `$F988`, `$FA07` | proven |
| `$EB-$EC` | sound command vector / envelope pointer | **Live control flow, not data.** `$F79A-$F7A4` loads the 16-entry command table `$F87B` into `$EB/$EC` and executes `jmp ($00EB)` — the second computed `jmp (zp)` in the ROM. Second role: data pointer loaded from `$05D8,x`/`$05D9,x`. | `$F726/$F72B`, `$F79D/$F7A2`, `$F7B9/$F7C4`, `$F832/$F837` | `$F7A4` (`jmp ($00EB)`); `$F73D/$F747` (`lda ($EB),y`); `$F7CE`, `$F7E9`, `$F7F4` | proven |
| `$ED-$F3` | CHR streaming-loader state | The mid-frame CHR→RAM streamer `$8204`. `$ED/$EE` PPU source (+`$18` per frame), `$EF/$F0` RAM destination, `$F1` bytes remaining, `$F2` write offset, `$F3` CHR bank. **Every caller sets `$EF/$F0` = `$0758`**, so this indirect writer never reaches zero page. | `$E701/$E706`, `$E70A/$E70E` (dest = `$0758`), `$E712`, `$E717`, `$E719/$E71B` (`$F1` = `$A8`); then `$8237/$823D`, `$8230`, `dec $F1` at `$8229` | `$8204`, `$8210`, `$8212`, `$8217`, `$8226`, `$E720`; `$80EB`/`$8107` test `$F1` to decide whether phases 1/2 stream | proven |
| `$F4-$FF` | — | **Unreferenced.** 12 bytes, permanently `$00`. Directly below the stack page but not itself stack. | RESET only | none | proven |

> **`$E4` is written, and no static search can see it.** The writer is
> `inc $E4` at `$F99F`, which is **reached only through a jump
> table**: `$F794-$F7A4` takes the low nibble of a sound-stream command byte,
> doubles it, indexes the 16-entry table at `$F87B`, and executes
> `jmp ($00EB)`. Entry 14 is `$F99F`. The bytes `9F F9` therefore appear in the
> ROM only inside that table (at `$F897`), so no operand search, byte-pair scan
> or CDL-anchored walk finds `$E4` — the CDL never executed `$F99F` either.
> This is computed dispatch — one of the nine ways this ROM defeats a static
> search (§2.8).
>
> `$E4` is **not** a variable anyone reads: no instruction anywhere loads,
> compares or stores it, and the four other raw `xx E4` hits (`$A42E`, `$C4EB`,
> `$C866`, `$E4D9`) were each adjudicated and are misaligned — `$E4D9` is the
> middle of `jmp $E4C4`, `$C4EB` sits inside the `$C484` pose-pointer table. So
> it is **named but dead** rather than proven free, and it is *not* safe space: a flag parked at `$E4` is incremented whenever a track emits command
> `$0E`. Whether any shipped track emits that command is **OPEN** — the
> instruction is reachable by construction; which shipped tracks, if any, emit
> it has not been determined.

---

## 5. Stack page (`$0100-$01FF`)

Two structures share this page and never meet: the PPU transfer queues grow up
from `$0100`, the hardware stack grows down from `$01FF` or `$01DF`.

### 5.1 The stack pointer — seven `txs` sites, **two different bases**

This is the single most load-bearing fact about page 1, and no earlier pass
had recorded it. Verified by locating every `$9A` byte in PRG and checking its
predecessor:

- **Six sites load `#$FF`**: `$8032/$8034` (RESET), `$8CF4/$8CF6` (boot & new
  game), `$8D89/$8D8B`, `$8E0A/$8E0C` (real code the CDL never reached),
  `$8E18/$8E1A`, `$AE50/$AE52`.
- **One site loads `#$DF`**: `$AE6C/$AE6E`, the top of the per-frame object loop.

An earlier pass said `$AE6C` "reset[s] stack" but not to what. It is **`$DF`,
not `$FF`**. Consequence: during gameplay the stack occupies `$01DF` downward
and `$01E0-$01FF` is dead, holding stale loader-era bytes; during loading and
menus it occupies `$01FF` downward. The 32-byte offset buys nothing detectable
and no reason for it was found — reported as an unexplained but proven fact.

**There is no `tsx` anywhere in the ROM.** Zero instruction starts decode as
`tsx`; all 73 raw `$BA` bytes resolve to operands or data. The ROM never reads
its own stack pointer.

### 5.2 Layout

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$0100` | queue A entry count | 0-3. Also briefly a scratch cell inside the enqueue: `$8668` stores the base offset here at `$8680`, uses it at `$868B`, restores the count at `$8690`. | `$8678` (`inc $0100,x`), `$8680`, `$8690`, `$853E`(=0), `$EEF2` | `$852F`, `$866A` (`cmp #$03`), `$867B`, `$868B` | proven |
| `$0101` | queue A drain gate | Debounce. `$852F` drains only when `$0100 == $0101`, so a producer that bumped the count but has not finished writing the body cannot be drained mid-write. | `$86B6` (last act of enqueue), `$8541`(=0), `$EEF5` | `$8534` | proven |
| `$0102-$0108` | queue A record 0 | 7-byte descriptor: +0 dest_lo, +1 dest_hi, +2 bytes-per-row, +3 row count, +4 src_lo (or the literal fill byte), +5 src_hi, +6 flags. Producer interface is zero page `$00-$06`. | `$8695-$86B3` (y=0); also hand-filled by the glyph renderer at `$EEDD`, `$EEE3`, `$EDE1`, `$EDE6`, `$EDEB`, `$EDF0`, `$EDD9` | `$863F-$8660` (y=0) | proven |
| `$0109`, `$0111`, `$0119` | record pad bytes | Unused 8th byte of the 8-byte stride. Records are 8 apart, only 7 bytes meaningful. | RESET only | none | proven |
| `$010A-$0110` | queue A record 1 | Same layout, y = `$08`. Observed live at depth 2. | `$8695-$86B3` (y=`$08`) | `$863F-$8660` (y=`$08`) | proven |
| `$0112-$0118` | queue A record 2 | Same layout, y = `$10`. `$866A cmp #$03` caps the queue here, so a 4th record (which would land at `$011A` and clobber queue B's count) is impossible. | `$8695-$86B3` (y=`$10`) | `$863F-$8660` (y=`$10`) | proven |
| `$011A` | queue B entry count | Counted **down** by the drain rather than zeroed — asymmetric with queue A. | `$8524` (`dec`), `$EEF8` (`inc`) | `$84E5` | proven |
| `$011B` | queue B drain gate | Same debounce role as `$0101`. | `$852B`(=0), `$EEFB` | `$84EA` | proven |
| `$011C-$0122` | queue B record 0 | Same 7 fields. **Written only by the glyph renderer**, cell by cell — `$EEE9`, `$EEEF`, `$EDF5`, `$EDFA`, `$EDFF`, `$EE04`, `$EDDC`. Queue B's drain advances the PPU address by **+8 per row with no carry** into `$12`, unlike queue A's +`$20` with carry. | as above | `$863F-$8660` called with y=`$1A` from `$84F8` | proven |
| `$0123` | pad byte | Nothing reads it. | RESET only | none | proven |
| `$0124-$0133` | queue B records 1-2 + pads | **Structurally real, never populated.** The only code that could fill them is the enqueue entry `$8663`, which has **zero callers** — and the byte pair `63 86` occurs **nowhere in the ROM at all**, not even inside data. Queue B is a 3-slot ring of which only slot 0 is ever used. | nothing but RESET | would be `$863F` with y=`$22`/`$2A`, unreachable | proven |
| `$0134-$01BC` | unused gap / stack headroom | 137 bytes between the top of the queues and the deepest point the stack was ever observed to reach. Negative verified four ways: no instruction names any address in the range; no indirect store can reach it (all 11 pointer high bytes traced, never `$01` outside the RESET clear); the only page-0-based indexed instruction that could cross into page 1 is `$84D6 lda $00C0,y`, bounded by `cpy #$20`; and an FCEUX write hook over both TAS movies recorded zero writes. | nothing but RESET | none | proven |
| `$01BD-$01CE` | stack — deep region | Deepest usage observed. Reached by the glyph/metatile rasteriser chain: `$9481 jsr $9489` → `$94B3/$94C0 pha` → `$94B4 jsr $94E7` → `$94E8 pha` → `$94EB jsr $9505` → `pha` at `$9507`/`$9529` (both hit `$01BD`). Also `$83BE`, `$F733`, `$F83B` when the NMI interrupts at depth. | as above | matching `pla`/`rts` | proven |
| `$01CF-$01D2` | stack — NMI nested frames | Return addresses pushed by calls made from inside the NMI. Level 1: `$80D9`, `$80DC`, `$8140`, `$8111`, `$8114`, `$8117`, `$8129`, `$812E`. Level 2: `$83BE jsr $862E`, `$F733`/`$F83B jsr $F73B`. | as above | matching `rts` | proven |
| `$01D3-$01D6` | stack — NMI register saves | `$01D6` = P (`php` `$80B7`), `$01D5` = A (`$80B8`), `$01D4` = X (`$80BA`), `$01D3` = Y (`$80BC`). | `$80B7-$80BC` | `$8155/$8157/$8159 pla`, `$815A plp`, then `rti` at `$815B` | proven |
| `$01D7-$01D9` | stack — NMI interrupt frame | The three bytes the CPU pushes automatically: `$01D9` PCH, `$01D8` PCL, `$01D7` P. Measured: ~95% of NMIs interrupt the frame-wait spin at `$808C/$808E`. | the CPU | `rti` at `$815B` | proven |
| `$01DA-$01DD` | stack — main-thread depth 2-3 | Depth 3 dominated by `$D6C9 jsr $806E` (the Start-button spin at `$D6C7`); depth 2 by `$B3AF jsr $A912`, `$BB2D`, `$CABB`, `$DCBA`/`$DD37`. | as above | matching `rts` | proven |
| `$01DE-$01DF` | stack — top of the gameplay stack | On the object-dispatch path these hold a **manufactured** return address, not a real one: `$AE9B-$AEA2` does `lda $AEB6 : pha : lda $AEB5 : pha`, so `$01DF` = `$AE` and `$01DE` = `$B6` — literal bytes read out of the code stream. **Verified in the ROM: `$AEB5` = `b6 ae`, `$AEB7` = `a2 00`.** The dispatched handler's own `rts` pops `$AEB6` and resumes at `$AEB7`, turning the computed `jmp ($0000)` at `$AEB2` into a synthetic indirect `jsr`. | `$AE9B-$AEA2`; also `$AE71` and the depth-1 `jsr`s at `$AEBB`, `$AEC8`, `$AEEC`, `$AF15`, `$AF68` | the handler's `rts` | proven |
| `$01E0-$01E4` | no-man's land | Below the deepest point the `$FF`-based loader stack reached, above the base of the `$DF`-based gameplay stack. Architecturally reachable (a loader-context chain 28+ levels deep would land here), never observed. | nothing but RESET | none | proven |
| `$01E5-$01FF` | stack — loader / boot / menu window | The stack whenever SP base is `$FF`: RESET, new-game load, game-over/continue, room loading, cutscenes, menus. Writers top-down include `$82A6-$82C9` (the `$2A` stack-smuggling pair), `$8D3F-$8D7E`, `$AE01/$AE06/$AE4F`, `$8F29`, `$8EA3`, `$E201/$E20E`, `$E71D`, `$ECC0`, `$82ED-$830C`, and — deepest — `pha` at `$9507`/`$9529` reaching `$01E5`. The NMI's own frames land around `$01F7-$01FB` in this context. | as above | matching `pla`/`rts`, and `rti` for the NMI frames | proven |

### 5.3 Measured stack depth

FCEUX Lua write hooks over all of `$0100-$01FF`, recording the PC of every
access, run over both of the published TAS movies (klmz's and jlun2's, ~2.2M
frames, ~1.9M NMI invocations). The two runs agree on every region boundary
exactly.

| Context | SP base | Depth used | Low-water |
|---|---|---|---|
| Gameplay | `$DF` | 35 bytes | `$01BD` (min S = `$BC`) |
| Loader / boot / menu | `$FF` | 27 bytes | `$01E5` |

**Headroom: 137 bytes.** The stack would have to get about 5× deeper to reach
the PPU queues.

**What this is not: a static worst case.** A sound static bound needs the six
`jmp (ind)` dispatches and the `$D8EB` inline-table dispatch resolved, which
was not completed. The honest claim is "never deeper than 35 bytes across two
full playthroughs", not "cannot exceed 35 bytes".

> **Caution recorded.** A first probe that inferred usage from "byte is
> non-zero" produced a false `$0134-$01BF` hit. That was FCEUX's power-on RAM
> fill (four `$00` then four `$FF`, period 8) sampled at frame 0 before
> `$8000`'s clear loop finished. Only write-hook data is trustworthy here.

### 5.4 Direct stack manipulation — the complete set

The completeness is itself a result. Method: build each subroutine's body by DFS
over fallthrough/branch/jmp edges from every `jsr` target, fixpoint the local
push depth, and flag every `pla`/`plp` reachable at local depth 0 — i.e.
popping something the routine did not push. That returns exactly and only
`$82A6`, `$82A9`, `$82BA`, `$82BD`, `$82C0`, `$D452`, `$D453`, `$D467`,
`$D468`, `$D8EB`, `$D8F1`.

| Site | Idiom |
|---|---|
| `$82A6` / `$82BA` | Lift the return address, smuggle the `$2A` value **underneath** it, push the return address back, `rts`. This is evasion mechanism (f): a stack-smuggled value (§2.8). |
| `$D8EB` | `pla/pla` reconstruct its own return address +1, use it as a pointer into the 2-byte-per-state table the caller placed **after** the `jsr`, `jmp ($02)`, never returns. Mechanism (g), the inlined jump table (§2.8). |
| `$D452`/`$D453`, `$D467`/`$D468` | `pla : pla` discarding one whole call frame, so a forced Jason state transition abandons the rest of the calling handler. |
| `$AE9B-$AEA2` | Manufactured return address (see `$01DE-$01DF`): mechanism (f)'s stack smuggle, applied to a whole return address rather than one saved byte. |
| the seven `txs` | §5.1. |

Separately: **zero** `pha … pha … rts` and **zero** `pha … pha … jmp` sites
exist other than `$AE9B`, so there is no second synthetic-`jsr` idiom hiding in
the ROM.

---

## 6. OAM page (`$0200-$02FF`)

The 64-entry sprite shadow, DMA'd via `$80CB sta $4014` — the **only** `$4014`
write in the whole 32 KB. Standard NES layout: +0 Y, +1 tile, +2 attr, +3 X.

It is also, whenever rendering is off, a general 256-byte staging buffer for
three unrelated kinds of CHR-resident data. See §6.3.

### 6.1 Fixed slots

| Addr | Slots | Name | Purpose | Writers | Conf |
|---|---|---|---|---|---|
| `$0200-$0203` | 0 | sprite-0 hit marker | A permanently parked sprite whose only job is to trigger the sprite-0 hit that drives the mid-frame raster split. Y=`$7E`, tile=`$FF`, attr=`$23`, X=`$10`. **Not read by the CPU** — consumed by the PPU: `$B229` busy-waits on `$2002` bit 6 then rewrites the nametable bit and `$2005` X from `$44/$45`. Hidden (`Y=$F8`) indoors, where `$AECC` skips `$B229`. | `$98CF-$98E2`; hidden by `$E7FE-$E800`, `$E8AA-$E8AC`, `$8483`, `$A8BC` | proven |
| `$0204-$0213` | 1-4 | counselor portrait | 2×2 face in the status area. X=`$C0`/`$C8`, Y=`$0F`/`$17`, attr `$02`. Tile base from the 6-entry table `$8B42` = **`60 64 68 6C F8 FC`** (verified), indexed by `$0507`. | `$8B2C` → `$8B48` → `$8B54`; sole caller `$8BB9`, itself called from `$8D73`, `$B99E`, `$DFCD`, `$E8CB` | proven |
| `$0214-$0223` | 5-8 | weapon icon | 1-4 tiles. Geometry from the 6×4 table `$8AA2` = **`7B E0 0F 22 / 78 E0 13 21 / 70 E0 0F 22 / 7F E4 13 11 / 74 E0 0F 22 / 80 E0 0F 22`** `[tile, X, Y, cols\|rows]` (verified), indexed by `$0506`. | `$8A90` → `$8A37` (blanks 4 slots via `$8A81`) → `$8B54`; callers `$8839`, `$8BBC`, `$EAB8`, `$EB47` | proven |
| `$0224-$022B` | 9-10 | blinking alarm arrows | Two single tiles (`$1E`) at X=`$10`, Y=`$0F` and `$1F`. Eight literal bytes from `$8B24` = **`0F 1E 00 10 1F 1E 00 10`** (verified). One of the two is hidden each blink phase, chosen on `$058E`. | `$8B0F`; selective hide `$B750` (`$0228`) / `$B758` (`$0224`); phase alternates `$0590` between `$04` and `$82` | proven |

### 6.2 The item-icon band and the two sprite cursors

Four HUD item icons occupy slots 11-22, drawn by `$89C2` from the 4×6 record
table at `$8A13` = **`2C 04 D3 A7 B8 02 / 54 02 E5 CB D8 01 / 3C 04 E7 A7 D8 02 / 4C 02 DF CB B8 01`**
(verified) — `[oam byte offset, slots to blank, tile, Y, X, row count]`. Note
the records are **not** in slot order.

| Addr | Slots | Icon | Ownership byte | Geometry |
|---|---|---|---|---|
| `$022C-$023B` | 11-14 | lighter | `$0517` | 2×2, tile `$D3`, X=`$B8`, Y=`$A7` |
| `$023C-$024B` | 15-18 | cure / potion | `$0519` | 2×2, tile `$E7`, X=`$D8`, Y=`$A7` |
| `$024C-$0253` | 19-20 | key | `$051A` | 1×2, tile `$DF`, X=`$B8`, Y=`$CB` |
| `$0254-$025B` | 21-22 | flashlight | `$0518` | 1×2, tile `$E5`, X=`$D8`, Y=`$CB` |
| `$025C-$026B` | 23-26 | other counselor's weapon (PASS/CHANGE menu) | index `$0732[$05A6]` | from the 6×4 table `$8A69` = **`80 70 CF 22 / CF 74 CF 12 / 8A 74 CF 12 / CE 74 D3 11 / 84 74 CF 12 / E1 70 CF 22`** (verified) |

Slot range and geometry for `$025C-$026B` are **proven**; the *label* is
**likely** — `$0732` is the per-counselor weapon and `$05A6` the other
counselor, but the tile set differs from the main weapon table `$8AA2`. Note
that the new-game default of `$0732-$0737` is `$03` (§2.3), which selects
record 3 = a single tile `$CE`.

**The dynamic sprite band.** Everything from the cursor `$056F` to the end of
the page is rebuilt from scratch every frame by BUILD_SPRITE. There are two
cursor seeds, chosen at `$AE7D-$AE83` on `$33` bit 1:

- **Outdoors** (`$33` bit 1 clear): `ldy #$2C` — the band starts at slot 11, so
  the four item icons and the CHANGE-menu icon are overwritten by gameplay
  sprites.
- **Indoors** (`$33` bit 1 set): `ldy #$6C` — the band starts at slot 27,
  protecting slots 0-26. `$2C → $6C` is exactly the 16 slots those icons need.

Appended by `$AAE6` (normal) and `$AC4B` (mirrored); the cursor is written back
at `$AC45`/`$AD4A`. BUILD_SPRITE refuses to draw at all if `$056F` is zero
(`$AB2C`), and the append loops stop cleanly on X wrapping, so the page cannot
be overrun.

**Frame-end blanking.** `$A8BC` parks every entry from the cursor to the end of
the page by writing `$F8` to **the Y byte only** — tile, attribute and X keep
stale values. Two consequences: a live OAM dump of "hidden" slots is full of
garbage, and `$A8BC` tests before storing, so a cursor of exactly `$00` blanks
nothing. The full-page version is `$8483`.

**Per-frame rotation (flicker).** `$A84B` swaps an 8-entry (32-byte) block
starting at the rolling offset `$056D` with the block at the fixed base `$05BF`;
`$056D` advances by 32 each frame. `$05BF` is latched at `$AEDF` from `$056F`
immediately after the player's own sprites are built — so the player is excluded
from the rotation and the enemies are not, which is what a flicker scheme wants.
Mechanism **proven** and confirmed live; the *purpose* label is **likely**.

**Map screen** overrides most of this: `$E3FC` loads 12 static icons into slots
0-11 from a count-prefixed list in CHR bank 0 at PPU `$0800`; `$E20D` writes 13
building markers **directly** into slots 50-62 from the 13×2 (Y,X) table `$E231`
= **`97 88 / 77 E0 / 37 C8 / 4F 68 / 77 A8 / 77 C8 / 1F E0 / 34 14 / 5F 20 / 3C 4C / 57 98 / 57 A0 / 57 A8`**
(verified), tile `$B1` for markers 0-9 and `$B2` for 10-12; counselor sprites
take slots 20-47 from cursor base `$50`; slot 63 is a blinking cursor whose X
comes from `$E265` = **`14 34 54 74 94 B4`** (verified) indexed by `$0507`.

### 6.3 The three staging roles

All three run with rendering off, which is why they can safely stomp the sprite
shadow. All three write through `$827B sta ($12),y` with `$12/$13` = `$0200`.

| Role | Descriptors | Reader | Notes |
|---|---|---|---|
| Palette-table staging | 8, 9, 10 (bank 3, PPU `$0250`/`$0350`/`$0200`) | `$849C lda ($00),y` inside `$8491` | 16 palette records of 16 bytes. `$8758` copies record `(A & $0F)` to `$C0-$CF`, `$8739` copies record A to `$D0-$DF`. Pointers built by split `adc`. |
| Junction-record staging | 6, 7 (bank 2, PPU `$1C32`/`$1C00`) | `$95A9` + the `lda ($00),y` chain at `$95A7-$963F` | 5-byte junction records, decoded into `$0500`, `$75`, `$76`, `$6E`, `$6F`, `$55`, `$53`. Pointer = `$0200 + index×5`, built at `$9589-$95A5` with split `adc`. |
| Nametable RLE staging | 13, 14, 15, 16 (bank 3) | `$82E2`/`$82F0`/`$8304 lda ($02),y` | Decoded straight to `$2007` during forced blank. Sub-record format `[descriptor, ppu_lo, ppu_hi, src_lo, src_hi]`, `$FF`-terminated. Cross-check: `$E45E` = **`40 00 FF`** (verified) = "repeat `$00` 64 times, end" into PPU `$23C0` — a 64-byte attribute clear. |

---

## 7. Entity block (`$0300-$04FF`)

**64 parallel byte arrays, stride `$08`, 8 slot columns. Not structs.** The
geometry is proven by the wipe routine, not inferred — see §2.4.

### 7.1 Slot map

| Slot | Contents | Evidence |
|---|---|---|
| 0 | the player-controlled counselor | `$AE3A`, `$AE86`, `$DBD8`, `$DD2C`, `$DCAF` all `ldx #$00` |
| 1 | the counselor's thrown weapon | `$C12E-$C191`: `ldx #$01 / jsr $A829`, position copied from slot 0 with Y−`$18`, type from `$C192[$0506]` |
| 2-5 | enemies / Jason / Pamela | `$CA7F` clears 2..5; `$CABE-$CB3B` iterates `ldx #$02 … cpx #$06`; `$D980` is the free-slot scan over 2..5; Jason spawns at `$D170`/`$D297`/`$D513` all with X=2; **Pamela is slot 2** |
| 6 | the enemy / Jason projectile | `$C2B4-$C325` `ldx #$06` |
| 7 | **never indexed by any instruction** | no `ldx/ldy #$07` feeds an entity array, no loop bound exceeds 6, no `$A829` caller passes 7. Byte +7 of all 64 fields is dead. |

### 7.2 Fields

| Addr | Field | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$0300` | `oam_attr_or_mask` | Per-slot OR-mask merged into every OAM attribute byte this object emits. **Dead feature — no writer exists.** The only instruction in the ROM whose operand is `$0300` is `$AB12`, a read; the two raw byte-pair hits (`$E447`, `$E455`) are the middle of 5-byte draw-list records. Permanently `$00`. A free per-entity palette/priority hook. | **none** | `$AB12` → zp `$03` → `ora $03` at `$AC01` / near `$AD08` | proven |
| `$0308` | `pose_id` | Animation pose id; indexes the 64-slot pose pointer table `$C484` as `id×2`. Distinct from entity type. | `$B3D0`, `$D8CC` (the two entry points of the set-pose helper; an earlier pass misread `$D8CC` as a sound call, and it is not), `$C152`, `$C207`, `$C315`, `$E64A`; unexecuted `$B101` | `$AA97` (ANIM_INIT), `$B0FC`, `$B1C7`, `$B4E4`, `$B531`, `$BAD0`, `$BC48`, `$BCB9`, `$BE7E`, `$DD93`, `$DDAA` | proven |
| `$0310` | `anim_frame_index` | Frame within the pose. Indexes `$0328/$0330` as `index×2` and `$0338/$0340` as `index`. | `$AA7A`, `$AAD5`, `$B0B3`, `$BD2C`, `$CF65`, `$CFED`, `$D76D`, `$D792`, `$D7C0`, `$E4DD`; unexecuted `$B44E`, `$D066` | `$AA65`, `$AAF0`, `$BAE3`, `$CB86`, `$D421` | proven |
| `$0318` | `anim_loop_frame` | Frame to restart at on wrap. Pose header byte 1. | `$AAB0` | `$AA77` | proven |
| `$0320` | `anim_frame_count` | Frames in the pose. Pose header byte 0. | `$AAAA` | `$AA6B` (`cmp` → wrap to `$0318`, or if `$0350` bit1 latch bit0 = done) | proven |
| `$0328` / `$0330` | `frame_ptr_array` lo/hi | Pointer to this pose's array of 2-byte shape-record pointers (= header address + 4). | `$AAC8` / `$AACF` | `$AAE6` / `$AAEB` (BUILD_SPRITE stages into zp `$02/$03`) | proven |
| `$0338` / `$0340` | `duration_tbl` lo/hi | Pointer to the per-frame duration list. Header bytes 2-3. | `$AAB6` / `$AABE` | `$AA7E` / `$AA83` → `lda ($00),y` at `$AA88` | proven |
| `$0348` | `frame_timer` | Frames left on the current animation frame — **and** a plain generic countdown reused by several state handlers with no animation meaning. | animation: `$AA8A`, `$AADA`, `dec $AA60`. generic: `dec` at `$B1B3`, `$B1BC`, `$B210`, `$BEA8`, `$CBC9`, `$CDC4`, `$D6F0`; set by `$B1F3`, `$B49D`(#$40), `$B4AE`(#$40), `$BECA`, `$CBB8`, `$CDC0`, `$CE32`, `$D86F` | `$AA5B` + every `dec … bne` above | proven |
| `$0350` | `anim_flags` | bit0 = animation-finished latch. bit1 = one-shot (don't loop; latch bit0). bit2 = horizontal flip / faces left. | bit1: `$A7FC` set, `$A7F3` clear, `$C33D`, `$BED8`, `$BEF3`. bit2: `$A817` set, `$A820` clear, `$CD3A` (`eor`), `$D8C5`, cleared at `$B0F0`, `$B155`, `$BEB1`, `$CC3C`. bit0: `$AA90` set, `$AAE0` clear. Bulk: `$B471`, `$C1F6`, `$C2C5`, `$D8C8` | bit0: `$B1E7`, `$C3E6`, `$CC94`, `$CDB3`, `$D41A`, `$BE9D`. bit1: `$AA72`. bit2: `$AB1B` (picks the mirrored blit at `$AC4B`), `$BB25`, `$C2BE`, `$CD7F`, `$CEA4`, `$DD9C` | proven |
| `$0358` | `oam_entries_emitted` | Entries BUILD_SPRITE emitted this frame. **Write-only.** `$AB28` is the only instruction referencing it in the entire ROM (confirmed by brute byte-pair scan). | `$AB28` | **none** | proven |
| `$0360` / `$0368` | `world_x` lo/hi | 16-bit world X. `$0368` doubles as the sign/screen-side test — `lda $0368,x / bpl` at `$D8B2` decides facing. | MOVER `$A94D/$A966`, `$A955/$A96E`; scroll compensation `$A8F0`/`$A8F8`; slot-0 restore `$AE0A`; 13 spawners | `$AAFE`/`$AB08` (BUILD_SPRITE), MOVER, `$AE5F` (save to `$05AF`), `$BFA4-$C03F` (distance helpers), `$CB9D`/`$CC28`/`$D8A3` (inter-entity compare), collision `$DC08-$DD5D`, `$F45D`/`$F4C0` | proven |
| `$0370` / `$0378` | `world_y` lo/hi | 16-bit world Y. `$0378` is the fall/off-bottom test. **`$0378` has no writer but MOVER** (`$A9AC`, `$A9C3`). | `$0370`: MOVER `$A9A4/$A9BB`, `$AE10`, `$B2BA`/`$B2EA`, 13 spawners. `$0378`: `$A9AC`, `$A9C3` only | `$AB03`/`$AB0D`, `$AE65`, `$B2AB` (**the instant-death-by-falling gate**), `$C39A`, `$D90A`, collision | proven |
| `$0380` / `$03A0` | `x`/`y_speed_index` | Index into the 2-byte speed table `$AA35` `(step_px, accumulator_period)`. 0 = stopped. Verified: 19 records, `00 00 / 01 40 / 01 20 / 01 18 / 01 10 / 02 20 / 02 10 / 03 20 … 08 10`. | X: `$B383`, `$B396`, `$B3AA`, `$B3C0`, `$B3E1`, `$BE15`, `$BE41`, `$BE8E`, `$C21B`, `$C344`, `$C44D`, `$D3E9`, `$D8D8`. Y: `$B3D9`, `$B4CB`, `$B4D4`, `$C347`, `$C47B`, `$D3C7`, `$D3EF`, `$D8E4` | `$A91A`/`$A971` (MOVER gates), `$AA07`/`$AA1E`, `$B37E`, `$C448`, `$C476`, `$CF7C`, `$D8D3` | proven |
| `$0388` / `$03A8` | `x`/`y_accumulator` | Bresenham fractional-speed accumulator. Gains `$10` per frame; on reaching the period it wraps and the object gets a full step, otherwise step−1. | `$A9D2/$A9DE`, `$AA1A`; `$A9F2/$A9FE`, `$AA31` | `$A9CC`; `$A9EC` | proven |
| `$0390` / `$03B0` | `x`/`y_period` | Accumulator wrap threshold (speed record byte 1). | `$AA15`; `$AA2C` | `$A9C7`, `$A9D5`, `$A9DB`; `$A9E7`, `$A9F5`, `$A9FB` | proven |
| `$0398` / `$03B8` | `x`/`y_step_px` | Pixels per full step (speed record byte 0). | `$AA0F`; `$AA26` | `$A91F`; `$A976` | proven |
| `$03C0` | `facing_dir_bits` | 4-direction bitfield, low nibble: bit0 right/+X, bit1 left/−X, bit2 up/−Y, bit3 down/+Y. Proof of the assignment: `$D8B2` writes `$01` when the target is left and `$02` when right, pairing `$02` with `$0350` bit2 (h-flip). | 24 sites incl. `$B027`, `$B3B4`, `$B4C6`, `$C442` (SCRIPT 3-byte form: whole nibble), `$C461` (SCRIPT 2-byte form: `and #$F3 / ora` — replaces only bits 2-3), `$CC37`, `$D36C`, `$D8C1`, `$E4AF` | MOVER `$A93B` (`and #$01`) and `$A992` (`and #$08`); `$B38C` (`and #$03 / lsr` → zp `$4C`); `$B167`, `$B179`, `$B180`, `$C3F0`, `$CBFF`, `$D4DC`, `$D911`, `$D918` | proven |
| `$03C8` | `movement_script_id` | Used only by the flying-enemy cluster. Value from the 8-entry table `$CF03` = `02 02 06 02 06 02 02 06`, indexed by `$0428,x`. **Exactly two instructions in the ROM touch this field.** | `$CEF2` | `$CF3D` (`cmp #$06` — script 6 also ticks the animation) | proven |
| `$03D0-$041F` | fields 26-35 | **Ten consecutive unused fields.** Zero instruction references anywhere. All 18 brute-scan candidates in the range are misaligned (e.g. `$BCF8 lda $03D0,x` is really the middle of `bne $BCFE / sta $05A2`; the four `inc $0410,x` hits sit inside the `$D57B` cabin-movement-script data region). | `$8023`, `$A838` only | none | proven |
| `$0420` | `entity_state` | `$00` = slot free. For **slot 0** it dispatches through the 17-entry table `$AF6C` (states `$00-$10`); for enemy slots 2-5 the primary dispatch is by *type* and `$0420` is only the per-type sub-state; for projectile slots 1/6 it is a 3-way test at `$C370` (≥`$10` = dying). Despawn is two-tier: `$10` marks dying, `$C3E6` keeps animating until `$0350` bit0, then `$C3B1` stores 0. | 61 sites. Key: `$C354`(=`$10`), `$DCA9`(=`$10`, the weapon-kill commit), `$C3B1`(=0), `$AE47`(=9, rowing), `$B3CB`, `$B433`-`$B541`, and ~19 `inc $0420,x` sub-state advances | **`$AEA3`** (`lda $0420,x / asl / tay / lda $AF6C,y / jmp ($00)` — the only `$AF6C` dispatch in the ROM, X always 0), `$CAC0`/`$CACC`, `$C35D`/`$C367`/`$C370`, `$D982`, + 9 more | proven |
| | | **`$AF6C` targets, verified byte-for-byte:** `$B04B $B119 $B119 $B199 $B1B3 $B1BC $B198 $B1C5 $B210 $AFA5 $BC14 $EA42 $BDE7 $BE9A $BEA8 $B1F7 $AF8E` | | | |
| `$0428` | `subpattern_counter` | Generic per-slot counter; meaning depends on type. | `$CEE8`/`$CEEB` (cycles 0..7 then indexes `$CF03`), `$D1D1` (`lda $31 / and #$07` — RNG-seeded), `$D7B9` (Pamela dive count from `$D7E3`), `dec $D723` | `$CEDF`, `$D71E` | proven |
| `$0430` / `$0438` | `x`/`y_distance_budget` | Remaining travel for the current script step. MOVER subtracts the step each frame and clamps at 0; SCRIPT refuses to fetch the next record until **both** budgets hit 0. | `$A936`/`$A98D`, `$AE98` (frame-boundary decay), `$C301`(=`$FF` unlimited), `$C426`/`$C429` (SCRIPT loads the duration byte into both), `$CBC0`, `$D355`/`$D358`, `$E4BE`/`$E4C1` | `$A92C`/`$A983`, `$AE8E`, `$C409`/`$C40E`, `$CC1E`, `$D30F`/`$D314`, `$E4D0` | proven |
| `$0440` / `$0448` | `script_pc` lo/hi | Program counter of the movement/AI bytecode (`$C409` interpreter). | `$C46D`/`$C472` (advance, with carry), `$B518`/`$B51E` (jump-arc table `$B545`), `$C229`, `$CDF2`, `$CE7B`, `$CF5B`, `$D083`, `$D501`, `$D56C`, `$D800` | `$C413`/`$C418`, `$CDFB`, `$CE00`, `$CE75`, `$D383`/`$D388` | proven |
| `$0450` | `action_byte` | Multi-use. **bit7 = airborne** (jumping/thrown) for the counselor; a plain countdown and a once-only latch for Jason's handlers. | `$B4B9`(=`$80`), `$B529` (`ora #$80`), `$B4EF`(=0), `$D34E`, `$D3B1`, `dec $D400`, `inc $D430` | `$D428` (already attacked this cycle?), `$DD23` (`bmi` → skip the third damage path while airborne) | bit7 **proven**; the Jason-side use is proven as code but unnamed — **open** |
| `$0458` | `requested_speed` / substate | For the counselor: the *requested* speed index, copied into `$0380` by `$B37B` when it differs. Otherwise a per-type sub-state or cycling index. | `$B299` (from the `$B2A3` mode table indexed by `$0572`), `$B3DE`, `$B0B6`, `$B112`, `$B162`, `$AF9B`, `dec $AFD2/$AFF2`, `inc $B01A` (rowing accel), `$CD2B`, `$D7AC`/`$D7B3` (Pamela dive index 0..7 into `$D7E3`) | `$B37B`, `$B399`, `$B07D`, `$B312`, `$B50B` (picks the jump-arc row), `$AFC2-$B01D`, `$CCA6`, `$CEB6`, `$D7A3` | proven |
| `$0460` / `$0468` | `x`/`y_move_delta_latch` | Magnitude of this frame's movement, bit7 = negative direction. **Broken and unused.** `$A914`/`$A917` clear it correctly with `,x`, but the two latch writes on each axis — `$A944`, `$A95D`, `$A99B`, `$A9B2` — are **absolute, with no index**, so they always hit slot 0. And **no instruction anywhere reads either field.** The missing-index bug is one of the defects catalogued in the companion reference document (section 13); the stronger fact is that the field has no consumer at all. | `$A914`/`$A917` (indexed), `$A944`/`$A95D`, `$A99B`/`$A9B2` (unindexed) | **none** | proven |
| `$0470` | `status_flags` | bit0 = suppress this object's sprite/camera record this frame (the hit-flash blink). **bit1 = read once, never written.** bit2 = boat/rowing committed. bit3 = dead/inert. | bit3: `$A80E-$A816` set (`ora #$08`), `$A805-$A80D` clear (`and #$F7`), `$C34C`, `$D460`, `$D47A`, `$DC9C`. bit2: `$B092` set, `$B109`/`$B285` clear. bit0: `$DBEF-$DBF6` (`and #$FE / ora $00`) | bit0: `$AEC1`, `$CB17`. **bit1: `$B305` only** — `lda #$02 / and $0470,x / beq $B30B` (verified at `$B303-$B30A`). bit2: `$B084`. bit3: `$C3C4`, `$DC23`, `$DCC4`, `$DD58` | proven |
| | | **bit 1 verified dead three ways.** Every executed write uses mask `$F7`/`$FB`/`$FE` or sets `$08`/`$04`/`$01` — never `$02`. All four never-executed sites (`$B205`, `$B258`, `$D050`, `$D230`) were decoded and none sets bit 1 either. The two `inc $0470,x` byte patterns at `$CE59`/`$CE67` sit inside proven DATA (`cdl=02`, the movement-script table). Both bulk clears zero it. **The gate at `$B305` always falls through.** | | | |
| `$0478` | `entity_type` | Type id `$00-$0C` (13 types). Primary dispatch for enemy slots: `$CAF5` does `lda $0478,x / asl / tay / lda $CB55,y / jmp ($00)`. | `$C14C` (player projectile from `$C192[$0506]`), `$C30F` (enemy projectile from `$C326[$0509]`), `$C201`, `$D175`(=1 walking Jason), `$D26B`, `$D288`(=9), `$D2C4`, `$D470`, `$D526`(=2 cabin-duel Jason), `$D850`, `$DAA2`; unexecuted `$D23D`(=8) | `$CAF5`/`$CB3E`, `$C32C`, `$C37A`, `$CAD8`, `$D09E`, `$D882`, and the whole damage cluster `$DC12-$DD75` feeding `$DF1D` damage / `$DF3B` hitpoints / `$DF58` enemy damage — all 13-entry tables | proven |
| | | **`$CB55` targets, verified byte-for-byte:** `$00→$CB7A $01→$CD07 $02→$D2DB $03→$CE87 $04→$CFAB $05→$CFCE $06→$D033 $07→$CC84 $08→$CB76 $09→$CB76 $0A→$D6E1 $0B→$CB76 $0C→$CB6F` | | | |
| `$0480` | `damage_accumulated` | Running total of damage taken; compared against the type's hit points (`$DF3B`/`$DF4B`). Counts **up**. | `$DC84-$DC8B` (`lda weapon_damage / clc / adc $0480,y / sta $0480,y`), `$D4B7` (reset when Jason's cabin-fight bar is spent) | `$DC88` only | proven |
| | | Corroborates MASTERNO's "`$0482` is Pamela's health, counts up to `$20`" — `$0482` is `$0480` + slot 2, and **Pamela is slot 2**. Now located as a generic per-slot field, not a Pamela-specific one. | | | |
| `$0488` | `hit_stun_timer` | Hit/stun/invulnerability countdown. While non-zero, `$DBDA-$DBF6` mirrors bit0 into `$0470` bit0 every frame — that is the hit-flash blink — and several handlers refuse to act. | `dec $DBDF`; `$DC78`(#$0C on a hit), `$DCA4`(#$1C fatal), `$DD11`(#$40 player damage), `$B405`/`$D45B`/`$D475`(#$80) | `$DBDA`/`$DBE4`, `$CAD3`, `$B1CC`, `$B1F7`, `$D2EA`, `$DCB4`, `$DD31` | proven |
| `$0490-$04FF` | fields 50-63 | **Fourteen consecutive unused fields.** All 16 brute-scan candidates are misaligned (`$B545`/`$B548` inside the `$B545` jump-arc table, `$C49E` inside the `$C484` pose pointer table). | `$8023`, `$A838` only | none | proven |

### 7.3 Second use: RLE nametable staging (`$0300-$041F`)

Outside gameplay the low 288 bytes are a decompression staging buffer.
Descriptor 0 (CHR bank 1, PPU `$1000`, `$100` bytes) and descriptor 1 (bank 0,
PPU `$139C`, `$120` bytes) both target `$0300`; `$82CA` then decodes to the PPU
with `$02/$03` = `$0300`. Reached only through `$E39E`, at `$E4F6` (end of the
attract loop) and `$E549` — title/attract/ending screens where no entity slots
are live. The attract restarts through `$E62D → $A829`, which re-clears the slot.

---

## 8. Game state (`$0500-$05FF`)

**The most dangerous page in RAM for a hack author.** It holds two blocks with
completely different lifetimes, and the boundary between them is invisible to an
operand search.

| Block | Range | Lifetime | Cleared by |
|---|---|---|---|
| **New-game block** | `$0500-$0527` | whole game | CHR RAM-init record 0, once, at `$8D28` |
| gap | `$0528-$0567` | — | nothing but RESET; **64 bytes, entirely unreferenced** |
| **Scene-scoped block** | `$0568-$05C7` | one scene | `$8F34` memset, on **every scene build** — boot, day advance *and every counselor death* |
| **Sound channel arrays** | `$05C8-$05FF` | continuous | never bulk-cleared after RESET |

### 8.1 New-game block `$0500-$0527`

| Addr | Name | Purpose | Writers | Readers | Conf |
|---|---|---|---|---|---|
| `$0500` | area type | **Written in exactly one place, `$95B3`**, from the junction record. | `$95B3` | 19 sites: `$95E1`, `$987A`, `$989F`, `$98B2`, `$AE13`, `$AE3E`, … | proven |
| `$0501` | lake/children counter A | init `#$01` at `$8EE8`. Never read by any decoded instruction. | `$8EE8` | none found | open |
| `$0502` | lake/children counter B | init `#$05` at `$8EED`; also written at `$B7B8` (never-executed code). | `$8EED`, `$B7B8` (unexec) | none found | open |
| `$0503` | total children remaining | init `#$0F` = 15 at `$8EFE`. | `$8EFE`, `dec` | `$B85B` | proven |
| `$0504` | **counselors remaining** | Survivors not counting the active one: init `#$05` at `$8EF0` for six alive. `dec` at `$8DAB` on a counselor death gates game over (`bpl` = someone left, so only `$FF` ends the run — §8.2); `$B8E7` decrements it when Jason empties a cabin. Read as the HUD head-row count (`$899F`), the intro-box gate (`$8D48 cmp #$05`), the escort precondition (`$B6D4`), and Jason's flee test (`$D499`, wants ≥ 2). | `$8EF0`, 2 RMW | `$899F`, `$8D48`, `$B6D4`, `$D499` | proven |
| `$0505` | **active counselor health** | max `$20`. | `$8E33`, `$B419`, `$DD0C`, `$E059`, `$8938`(unexec) | `$892F`, `$DD02`, `$DD14` | proven |
| `$0506` | **current weapon** (0-5) | Indexes `$F393`, `$C192`, `$C273`, `$C27F`, and the icon table `$8AA2`. | `$8F03`, `$EB3D`, `$F518` | `$8A90`, `$C144`, `$C1F9`, `$C214`, `$C221`, `$D4AD`, … | proven |
| `$0507` | **active counselor** (0-5) | Indexes names `$8AEB`, palettes, the portrait table, and **every per-counselor array** (it is the `Y` in `$8ED1 sta ($00),y`). | `$E037`, `$E0A5`, `$E0BC`, `$EC0E`, `$EC2A`, 2 RMW | 25 sites | proven |
| `$0508` | time of day / difficulty tier | 0 = day, 1 = dusk, 2 = night. Derived from `$0523`. | `$8E23`, `$BDA3` | `$989C`, `$B93B`, `$DAE4`, `$DAF4`, `$E88F` | proven |
| `$0509` | enemy projectile type selector | Indexes `$C326`. **No writer among decoded instructions** — set indirectly. | none decoded | `$C307` | open |
| `$050A` | outdoor screen index | Which screen of the current area is on display. | `$BDAD`, `$E0CE` | `$9580`, `$E03D`, `$EBB7` | proven |
| `$050B` | **counselor-shuffle counter** | Reseeded at `$B939` to 30 by day and dusk, 50 at night, from `$0508`. One shuffle per 30 or 50 ticks of `$0577`, i.e. every 90 or 150 s. | `$B944` | `$B903` (`dec`) | proven |
| `$050C` | cabin transitions (live) | Pairs with per-counselor `$068F`. 2 → progress bit `$04`. | RMW only (`inc`/`dec $BD85`) | `$BD88` | proven |
| `$050D-$050E` | **Jason's route cursor** | 16-bit pointer into the `$B5AD` ring. `$B87A` dereferences it; on a `$FF` node it loads the next two bytes back into itself and restarts, which is how the ring closes. | `$B599`/`$B59F`, `$B88F`/`$B895`, `$B8C2`/`$B8CC` | `$B87A`, `$B87F` | likely |
| `$050F` | cabin attack countdown | `dec $050F` at `$B69B` is its only consumer — no plain reader. | `$B8BA`, `dec $B69B` | none | likely |
| `$0510` | **the area Jason is in** | Node byte 0's high nibble, split out at `$B8A6`. `$B656` compares it against your `$0500`; a match is one precondition of the path fight. | `$B8A6` | `$B656` (`cmp $0510`) | proven |
| `$0511` | **cabin index under attack** | Used by the cabin damage path `$B7C4`. | `$B6CC`, `$B86E`, `$B8AF` | `$B649`, `$B6AC`, `$B7C4`, `$B82F`, `$B8D0`, `$BA2A`, … | proven |
| `$0512` | **Jason's position within that area** | Node byte 0's low nibble, split out at `$B89E`. `$B65E` compares it against your scroll position from `$BA3A`. | `$B89E` | `$B65E` | proven |
| `$0513-$0515` | children per lake cabin | Each init `#$05` at `$8EF3`/`$8EF6`/`$8EF9`. | `$8EF3`, `$8EF6`, `$8EF9`, RMW | `$B6BB`, `$B85F`, `$BA0B`, `$BF29` | proven |
| `$0516` | **zombie kill counter** | 60 → machete. RMW only. Pairs with per-counselor `$072C`. Compared against the per-counselor quota at `$CC7E,y` (2/3/4/5/4/3): reaching it sets `$051F` bit `$01`, reaching double sets bit `$02`. | `inc` only | `$CC5F`, `$CC6D`, `$F44D` | proven |
| `$0517` | **lighter** | Pairs with `$0708`. Also the HUD icon ownership byte for slots 11-14. | `inc` only | `$89D4`, `$F454`, `$E998`(unexec) | proven |
| `$0518` | **flashlight** | Pairs with `$070E`. HUD slots 21-22. | `inc $0517,x` at `$F541` — indexed by item type − 6, so no operand search finds it | `$98AA` | proven |
| `$0519` | **cure / potion** (max 9) | Pairs with `$071A`. HUD slots 15-18. Revives at 6 HP. | `inc $0517,x` at `$F541` (indexed); `dec $EB05` (unexec) | `$897A`, `$B1D1`, `$B1FD`, `$B3E5`, `$EAF4`(unexec) | proven |
| `$051A` | **key** | Pairs with `$0720`. HUD slots 19-20. | `inc $0517,x` at `$F541` — indexed by item type − 6, so no operand search finds it | `$EC71` | proven |
| `$051B` | **sweater** | Gates damage-halve + blink. Pairs with `$0726`. | `inc $0517,x` at `$F541` (indexed); **`inc $051B` at `$F55C`** — real code neither TAS executed | `$AF3F`, `$DCF8`, `$F370` | proven |
| `$051C` | **Jason's health** (32 = full) | Set to `$20` beside health at `$8E30`, then drains continuously. | `$8E30`, `$8F08`, `$D4C7` | `$891C`, `$D492`, `$D4C0`, `$E919` | proven |
| `$051D-$051E` | woods / cave areas entered | `$051D` counts entries to the woods (areas 7-14, `inc` at `$B9D4`), `$051E` to the cave mouths (areas 1-3, `inc` at `$B9C2`); 3 of each → progress bits `$08` / `$10`. Pair with `$069C` / `$06A2`. | RMW only | `$B9D7`, `$B9C5` | proven |
| `$051F-$0520` | **progress bitfields A and B** | A accumulates `01→41→C1→C5→CD→CF`; B accumulates `00→02→06`. Gate the hints. Pair with `$06A8` / `$06AE`. `$051F` bit 0 gates item spawning at all. | `$EC3E`, `$EC45` | `$EC3B`, `$ED72`, `$F471`; `$EC42`, `$ED7D` | proven |
| `$0521` | **fireplaces lit, per counselor** | Its only two instructions are `inc $0521` at `$E9A6` and `lda $0521` at `$E9A9`, which run on every lighting — the flashlight at 7 uses `bcc`, so 7 and above all spawn. Swapped against `$0714` by `$8EA6`; the pairing is in the CHR swap table in bank 2 as `14 07 21 05`. | `$E9A6` (`inc`) + the `$8EA6` swap | `$E9A9` | proven |
| `$0522` | **day** (0-2) | `inc / cmp #$03` at `$8E40`. | RMW only | 15 sites incl. `$8D43`, `$8E43`, `$C2F5`, `$CBCE`, `$CDDB` | proven |
| `$0523` | cabin transition count | `inc` with saturation at `$BD78`; tiered against 5 and 8 at `$BD91` → `$0508`. | `$8E20`, RMW; `dec $BD7D` (unexec) | `$BD94` | proven |
| `$0524-$0527` | — | **Unreferenced.** Tail of the init record; zeroed on new game only. | init record 0 | none | proven |
| `$0528-$0567` | — | **Unreferenced, 64 bytes.** Outside every wipe. The largest contiguous free block in RAM outside the entity block. The three brute-scan hits (`$C978`, `$C98C`, `$CE64`) all sit inside proven data regions. | RESET only | none | proven |

### 8.2 Scene-scoped block `$0568-$05C7` — **wiped on every scene build**

Everything in this 96-byte run is zeroed by `$8F34`. **Corrected: that is not
just boot and day advance.** The wipe sits at `$8D57`, inside the scene-build
routine, and `$8D57` is reached unconditionally on every pass:

```asm
$8D3D: jsr $E363          ; scene build starts here
       ...
$8D43: lda $0522
$8D46: bne $8D57          ; these two branches skip the one-shot INTRO BOX,
$8D48: lda $0504          ; not the wipe
$8D4B: cmp #$05
$8D4D: bne $8D57
$8D4F: jsr $E7FA          ; "USE THE TORCH TO LIGHT THE FIREPLACES"
$8D52: lda #$40 : jsr $806E
$8D57: jsr $8F34          ; <-- unconditional: wipe $0568-$05C7
```

and `$8D3D` has **three** entries, all verified:

| entry | reached from |
|---|---|
| fall-through from `$8D3A` | cold boot / new game |
| `$8E5C` &rarr; `$8D3D` | day advance (`$8E18`, itself from `$D6D3`) |
| `$8DFE` &rarr; `$8D3D` | **counselor death** — `$8DAB dec $0504 / $8DAE bpl $8DF8` takes this branch whenever a counselor is left; only the last death falls through to the game-over screen |

The `lda $0522 / bne` above the call gates the intro box, **not** the wipe.
**A flag in `$0568-$05C7` does not survive to day 2 — it does not survive the
next counselor death.**

| Addr | Name | Writers | Readers | Conf |
|---|---|---|---|---|
| `$0568-$056B` | BUILD_SPRITE shape-record scratch | `$AB38`, `$AB3D`, `$AB42`, `$AB49` | `$AB6A`, `$AB6F`, `$AB96`, `$AC70`, `$ACB2` | proven |
| `$056C` | damage / collision scratch | `$B421`, `$DD20`, RMW | `$DBFA`, `$DCB1`, `$DD2E` | likely |
| `$056D` | OAM flicker rotation offset | `$A8B1`, `$A8B8`, `$AE55`, `$AEE7` | `$A84D`, `$AEE2` | proven |
| `$056E` | jump-arc scratch | `$B4E7`, `$B534` | none found | open |
| `$056F` | **OAM sprite append cursor** | `$AC45`, `$AD4A`, `$AE83` (seed `$2C`/`$6C`), `$E33F`, `$E491`, `$E577` | `$A8BE`, `$AB2C`, `$AB57`, `$AC55`, `$AEDC` | proven |
| `$0570-$0573` | counselor movement-mode state | `$B0E6`, `$B28D`, `$AFDC`, `$AFFC`, `$B00F`, `$B024`, `$B0A0`, `$B0A5` | `$B273`, `$B089`, `$AFCD`, `$AFED`, `$B015`, `$B291`, `$B0C5` | likely |
| `$0574` | **joypad 1, pressed this frame** | `$81D3` | 20 sites incl. `$AF32`, `$B2EE`, `$B948`, `$BC57` | proven |
| `$0575` | **joypad 1, held** | `$81D7`, `$E050` | `$81CD`, `$81D0` | proven |
| `$0576` | throw / HUD counter | `$C12A`, `$C23A`, `$C3AC`; RMW at `$C186`, `$D480`, `$DC73` | `$B278` | likely |
| `$0577` | **counselor-shuffle 3-second timer** — reloaded with `#$B4`, 180 frames = 3.00 s at 60 Hz; each expiry ticks `$050B` | `$B900`; `dec` at `$B8F8` | `$B8F3` | proven |
| `$0578` | **hit counter** → health bar | `$8296`, `$87FE`, RMW | `$87F7` | proven |
| `$0579-$057A` | HUD digit cells | `$880C`, `$B82A`, `$881A`, `$B821` | `$8805`, `$8813` | proven |
| `$057B` | **pickup counter** → weapon icon redraw | `$8836`, `inc $F512` | `$882F` | proven |
| `$057C-$057D` | spawn scheduler position | `$DA50`, `$DA79`, `$DAEB`, `$DA56`, `$DA80`, `$DAF1` | `$DA1B`, `$DA20` | likely |
| `$057E-$057F` | — | **Unreferenced** (inside the wipe) | none | proven |
| `$0580` | screen-transition flag | `$DF7F`, `$E866` | `$DFC2` | likely |
| `$0581-$0583` | — | **Unreferenced** (inside the wipe) | none | proven |
| `$0584` | enemy AI timer | `$CD64`, `$CE11` | `$CD54`, `$CD5D`, `$CD72` | likely |
| `$0585-$0586` | map-screen blink timers | `$DF89`, `$DFF2`, `$E02A`, `$E05E`, `$E16E`, `$E061`, `$E126` | `$E164`, `$E11C` | proven |
| `$0587-$0589` | Jason route step / cabin timers | `$B58D`, `$B698`, `$B6FE`, `$B765`, `$8299`, `$8844`, `$B6E4`, `$B782`, `$B7F5` | `$B689`, `$883C` | likely |
| `$058A` | **alarm timer, hundreds digit** | `$B6EC` — *and* `sta ($00),y` at `$8412`/`$841A` inside the BCD routine `$840A`. **This is evasion (b)'s canonical case (§2.8).** | (indirect, via the HUD digit path) | proven |
| `$058B` | **alarm timer, tens digit** — the middle digit of the 3-digit BCD counter `$058A-$058C`, and the index into the damage curve `$B7EC` | `$B6F4` (`#$06`) + the BCD indirection at `$8412`/`$841A` | `$B79B`, `$B7AA`, `$B7CC` | proven |
| `$058C` | alarm timer, low digits | `$B6EF` + the same BCD indirection | `$B78F` | proven |
| `$058D` | Jason route scratch | `$B6F9`, RMW | `$D9AB` | likely |
| `$058E` | **Jason alarm / target state** | `$8D9D`, `$B6E1`, `$B801`, `$B8C7`, `$BDD4`, `$BDFA` | 12 sites incl. `$B64E`, `$B67F`, `$B747`, `$B7A3` | proven |
| `$058F-$0590` | HUD refresh flag / arrow blink phase | `$87B3`, `$B80B`, `$B824`; `$B703`, `$B739`, `$B744` | `$87AD`; `$B72B`, `$B732` | proven |
| `$0591-$0592` | **building index (outdoor) / room index** | `$B871`, `$B9FA`, `$BA17`, `$EBBB`; `$BA1C`, `$BA34` | `$BA02`, `$BA27`, `$BDA9`, `$BF20`; `$BBC0`, `$BDBE`, `$E8D8`, `$E914` | proven |
| `$0593` | enemy-loop scratch | `$B670`, `$CA92`, `$CB32` | `$B640` | likely |
| `$0594-$0599` | **enemy spawn engine** | `$0594` last-registered type (`$FF` = empty), `$0595` live enemy count, `$0596` cap = time of day + 1, `$0597` 60-frame retry timer, `$0598` scratch, `$0599` per-cycle counter | writers `$CA8B`, `$D885`, `$D893`, `$CA8F`, `$DAF8`, `$DA0D`, `$DA9D`, `$DAFD` | proven |
| `$059A` | palette triplet staging | `$86BC` | `$86C1`, `$DFD4` | proven |
| `$059B-$059F` | glyph renderer state | `$EE49`, `$EE50`, `$EE55`, `$EE66`, `$EDCB`, `$EDD0` | `$EF46`, `$EF4B`, `$EE74`-`$EED4`, `$EE5A`, `$EE5F` | likely |
| `$05A0` | room's graphics-set id — from the remap table (block 19, `$080C`) | `$EC54` | `$EDC2`, `$EE91` | proven |
| `$05A1` | corridor walk countdown — `$30` on entry; drives the first-person walk | `$BBB0`, `$BD19` | `$BC3E`, `$BCA5`, `$BCAF` | proven |
| `$05A2` | **current location** — compared against `$05AB` at `$BD51` | `$BC9C`, `$BCFE`, `$E805`, `$E8A4` | `$BCC7`, `$BD2F`, `$BD56`, `$BF09`, `$E982`, `$EC4E` | proven |
| `$05A3` | hint row selector — indexes `$BBEC` | `$E887` | `$BBC5`, `$BC50`, `$BDB0`, `$BF10`, `$E92F`, `$E9D5` | proven |
| `$05A4-$05A5` | text / menu cursor state | `$E826`, `$E8F4`, `$BC68`, `$EA70` | `$EEFE`, `$EA66`, `$EA86` | likely |
| `$05A6` | **other counselor (for CHANGE)** | `$E8D5`, `$E8EF` | `$BF15`, `$E927`, `$E93A`, `$E951`, `$E95A`, `$EB31` | proven |
| `$05A7-$05A8` | hint block index → `$E6E3` / text state | `$E9F2`, `$EA17` | `$EAD4`, `$EAE5`, `$EC75`, `$EABB` | proven |
| `$05A9` | **cabin-duel counter — no `sta` anywhere** | **`inc $05A9` at `$D50C` is its only writer** (confirmed: the only two instructions naming `$05A9` are that `inc` and the `ldy` below). Reset exists only as part of the `$8F34` bulk wipe, i.e. **on every scene build** — day advance *and* every counselor death. The original trap. | `$D546` (`ldy $05A9`) | proven |
| `$05AA` | Jason HP HUD refresh flag | `$8858`, RMW | `$8851` | proven |
| `$05AB` | **Jason's target location** — set by the `$BBCD` roll | `$BBBA`, `$BBD8`, `$BD6E` | `$BD51` | proven |
| `$05AC-$05AE` | room re-entry state | `$B9E5`, `$CB47`, `$BB90`, `$BB96` | `$AE1A`, `$BB86`, `$D9BE`, `$AE2D`, `$AE32` | likely |
| `$05AF-$05B0` | **saved player world X / Y across a transition** | `$ADF7`, `$AE62` (from `$0360`); `$ADFC`, `$AE68` (from `$0370`) | `$AE07` → `$0360`; `$AE0D` → `$0370` | proven |
| `$05B1` | cabin duel state | `$D19C`, `$D518`, RMW | `$D48B` | likely |
| `$05B2` | enemy-loop / spawn scratch | `$B673`, `$CA95`, `$CAAE` | `$B643`, `$CA9F`, `$CBDE`, `$CF53` | likely |
| `$05B3` | Jason route timer | `$B708`, `$B722`, RMW | `$B712`, `$B71B` | likely |
| `$05B4` | duel scratch | `$BE2F`, `$BEC7` | `$BE4F` | likely |
| `$05B5` | door / junction scratch | `$B9FF`, RMW | `$BBDE` | likely |
| `$05B6-$05B7` | item drop position | `$F3E6`/`$F3ED`, `$F4F4`/`$F4F7`, `$F5B5`/`$F5B8` | `$F39E`/`$F3A1`, `$F4E3`/`$F4E8` | likely |
| `$05B8-$05B9` | **last-track cache (music)** — `$8F8C` swaps them for sound id < 6 | `$8F98`, `$E469`, `$8F94` | `$8F91`, `$B9E8`, `$E55D` | proven |
| `$05BA` | HUD refresh flag | `$8828`, RMW | `$8821` | proven |
| `$05BB` | **shirt colour cache** — read back from `$DD` after the counselor palette load | `$8730` | none decoded (compared visually only) | proven |
| `$05BC` | **sweater blink timer**, bit 7 = phase | `$AF3F`. Only referenced from code neither TAS executed (`$AF44`-`$AF5F`). | as above | proven |
| `$05BD-$05BE` | BUILD_SPRITE mirrored-blit scratch | `$AD50`, `$AD5D`, `$AD63` | `$AB64`, `$AB7C`, `$AC65`, `$AB90`, `$AC95` | proven |
| `$05BF` | **OAM flicker rotation base** — latched from `$056F` right after the player's sprites are built | `$AEDF` | `$A85F`, `$A8B5` | proven |
| `$05C0`, `$05C6-$05C7` | — | **Unreferenced** (inside the wipe) | none | proven |
| `$05C1` | cabin duel counter | `$D19F`, `$D4BA`, `$D51B`, RMW | `$D4A6` | likely |
| `$05C2` | map blink timer | `$DFA5`, `$E066`, `$E1A5` | `$E19B` | proven |
| `$05C3` | building entry state | `$BC2C`, `$BD06`, `$E94D`, RMW | `$BC14`, `$BC1C` | likely |
| `$05C4` | **pending Pamela gift** — `$F37D` writes, `$BD36` reads | `$BBBD`, `$F37D`, `$F526`, `$F561`(unexec) | `$BD36` | proven |
| `$05C5` | Jason route scratch | `$B6E9`, RMW | `$E944` | likely |

### 8.3 Sound channel arrays `$05C8-$05FF`

Eight channels throughout. Two stride conventions, both verified by tracing the
index register into each access: arrays indexed by `ldx $E6` (channel 0-7) have
stride 1; arrays indexed by `ldx $E7` (channel × 2) have stride 2.

| Addr | Size | Name | Index | Conf |
|---|---|---|---|---|
| `$05C8-$05D7` | 16 | per-channel note-stream position (lo/hi) | `$E7` | proven |
| `$05D8-$05E7` | 16 | per-channel envelope/data pointer (lo/hi) | `$E7` | proven |
| `$05E8-$05EF` | 8 | per-channel command scratch | `$E6` | likely |
| `$05F0-$05F7` | 8 | per-channel frames-to-next-event | `$E6` | proven |
| `$05F8-$05FF` | 8 | per-channel tempo | `$E6` | proven |

---

## 9. Sound and per-counselor (`$0600-$06FF`)

### 9.1 Sound channel arrays, continued

| Addr | Size | Name | Index | Conf |
|---|---|---|---|---|
| `$0600-$0607` | 8 | per-channel duration index | `$E6` | likely |
| `$0608-$060F` | 8 | per-channel repeat counter | `$E6` | likely |
| `$0610-$061F` | 16 | per-channel loop-return pointer (lo/hi) | `$E7` | likely |
| `$0620-$062F` | 16 | per-channel **call-return** pointer (lo/hi) | `$E7` | proven |
| `$0630-$0637` | 8 | per-channel envelope step | `$E6` | likely |
| `$0638-$063F` | 8 | per-channel volume / decay | `$E6` | likely |
| `$0640-$064F` | 16 | per-channel control byte | `$E6` **and** `$E8` | proven |
| `$0650-$065F` | 16 | per-channel pitch / period (lo/hi) | `$E7` | likely |
| `$0660-$0676` | 23 | — **unreferenced** | — | proven |

Two notes on this block:

**`$0620-$062F` was missing from every input survey.** It is written and read
only by `$F935`/`$F93A`/`$F94A`/`$F94F`, inside the sound-command handlers at
`$F92B-$F97C` — real code that neither recorded TAS executed, so the CDL-anchored
walk cannot see it. It saves and restores `$E9/$EA`, the note-stream pointer:
the sound bytecode's call/return.

**`$0640-$064F` is 16 bytes, not 8.** It is indexed both by `$E6` (channel 0-7,
covering `$0640-$0647`) and by `$E8` = `(channel × 4) & $0F` at `$F9B9`, which
reaches `$0640`, `$0644`, `$0648`, `$064C`. `$0648-$064F` is separately indexed
by `$E6` as its own array, so the two overlap by design — the same channel-folding
that makes channels 4-7 alias onto four generators. `$0643` and `$0647` also have
direct, unindexed writers in the silence-all routine (`$F66E`, `$F671`).

### 9.2 Per-counselor arrays and the claim-flag blocks

The per-counselor arrays here are **6 entries, indexed by `$0507`**, and are
swapped against a live `$05xx` byte by `$8EA6` (§2.6); each one's live partner
is named in its row. The first three rows are not per-counselor arrays: they
are 13-, 11- and 7-byte blocks, cleared by the CHR RAM-init table but never
swapped, with no live partner. Where those are per-counselor at all, it is by
bit inside the byte (`$EF5F` returns `1 << $0507`), not by swap.

| Addr | Live counterpart | Contents | Init | Conf |
|---|---|---|---|---|
| `$0677-$0683` | — | **notes-read claim bitflags** — one byte per readable-note record, one bit per counselor (`$EF5F`): tested at `$E9F9`, claimed at `$EAEB`/`$EAEE`, so each counselor can read the same note once | `$00` | proven |
| `$0684-$068E` | — | **silent-drop claim bitflags** — same shape for the silent item drops: tested at `$EA1E`, claimed at `$EAC1`/`$EAC4`. `$0689` (= `$0684+5`, Pamela's room) doubles as her **once-per-day lockout**: `$F38F` stores `#$01` over the whole byte when her gift is granted, `$8E3D` clears it on day advance, `$BD41` reads it to gate her spawn | `$00` | proven |
| `$068F-$0694` | `$050C` | cabin transitions (2 → progress bit `$04`) | `$00` | proven |
| `$0695-$069B` | — | **per-fireplace lit flags** — one byte per big-cabin fireplace, indexed by building − 14 (buildings 14-20), one bit per counselor (`$EF5F`): tested at `$E993`, claimed at `$E9A0`/`$E9A3`, and then `$0521` counts the lighting | `$00` | proven |
| `$069C-$06A1` | `$051D` | **woods-area entry counter** — `inc` at `$B9D4` on entering any of areas 7-14; at 3 it sets `$051F` bit `$08`, one of the axe's requirements | `$00` | proven |
| `$06A2-$06A7` | `$051E` | **cave-mouth entry counter** — `inc` at `$B9C2` on entering areas 1-3; at 3 it sets `$051F` bit `$10` | `$00` | proven |
| `$06A8-$06AD` | `$051F` | progress bitfield A | `$00` | proven |
| `$06AE-$06B3` | `$0520` | progress bitfield B | `$00` | proven |
| `$06B4-$06F3` | — | **unreferenced, 64 bytes** | — | proven |

### 9.3 Item / object slots `$06F4-$0707`

Cleared as one 20-byte block by `$F565-$F575` (§2.2). Two slots, stride 2,
iterated at `$F57C` (`ldx #$00 … cpx #$02`).

| Addr | Field | Writers | Readers | Conf |
|---|---|---|---|---|
| `$06F4,x` | slot state | `$EACC`, `$F487`, `$F506`, `$F5AF` | `$EA9B`, `$F479`, `$F4A3`, `$F57E` | proven |
| `$06F6,x` | **item type** (what the pickup path reads) | `$F48C` | `$EAAB`, `$F500`, `$F601` | proven |
| `$06F8,x` / `$06FA,x` | screen X / Y | `$F491`, `$F5C6`, `$F5DC`; `$F496`, `$F5CE`, `$F5E4` | `$F4B3`, `$F593`, `$F5C3`; `$F4AE`, `$F58C`, `$F5C9` | proven |
| `$06FC,x` | ? | `$F49B` | `$F4CB`, `$F5F2` | open |
| `$06FE-$0707` | 10 bytes cleared with the block, **no instruction reference** | `$F575` only | none | proven |

---

## 10. Per-counselor and CHR staging (`$0700-$07FF`)

| Addr | Live counterpart | Contents | Init | Conf |
|---|---|---|---|---|
| `$0708-$070D` | `$0517` | **lighter** | `$00` | proven |
| `$070E-$0713` | `$0518` | **flashlight** | `$00` | proven |
| `$0714-$0719` | `$0521` | **fireplaces lit** (7 → flashlight) | `$00` | proven |
| `$071A-$071F` | `$0519` | **cure / potion** (max 9) | `$00` | proven |
| `$0720-$0725` | `$051A` | **key** | `$00` | proven |
| `$0726-$072B` | `$051B` | **sweater** | `$00` | proven |
| `$072C-$0731` | `$0516` | **zombie kill count** (60 → machete) | `$00` | proven |
| `$0732-$0737` | `$0506` | **weapon** — swapped on CHANGE at `$EB3A` | **`$03`** | proven |
| `$0738-$073D` | `$0505` | **health** | **`$20`** | proven |
| `$073E-$0747` | — | **scratch buffer for the cabin permutation — 10 bytes, not 6** | — | proven |
| `$0748-$074D` | — | **alive flags** (bit 7 set = unavailable) | `$00` | proven |
| `$074E-$0757` | — | **cabin occupancy, 10 entries** — `[b7 flag][b4-6 counselor+1][b0-2 counselor]` | **`$8F`** | proven |
| `$0758-$07FF` | — | **CHR-to-RAM staging buffer, 168 bytes** | volatile | proven |

Writers/readers for the per-counselor arrays are almost all the indirect swap at
`$8ED1`/`$8EDD` (§2.6). Direct references exist only where the live game touches
them outside a swap: `$0732` at `$EB41`/`$EB3A` (the PASS/CHANGE menu),
`$0738` at `$8E54`/`$B7E6`/`$B7D6`/`$E056`/`$E968`, `$0748` at
`$8DA8`/`$B8E4`/`$E02F`/`$E344`, and `$074E` at 11 write and 19 read sites.

> **`$0744-$0747` is not free.** The cabin-permutation scratch is **ten** bytes, `$073E-$0747`, one per cabin
> slot, because the array it permutes (`$074E-$0757`) is ten entries. The shuffle
> at `$B909` is
>
> ```asm
> $B90E: ldx #$00
> $B910: lda $074E,x : pha
> $B914: lda $B92F,x : tay        ; permutation table
> $B918: pla
> $B919: sta $073E,y              ; <- y reaches 9
> $B91C: inx : cpx #$0A : bne $B910
> $B921: ldy #$09
> $B923: lda $073E,y : sta $074E,y : dey : bpl $B923
> ```
>
> `$B92F` is `01 05 06 02 03 04 07 09 00 08` — a permutation of 0-9, **max 9**,
> verified byte-for-byte. So `$B919` writes and `$B923` reads all of
> `$073E-$0747`. Six is the per-counselor stride used by every *other* array in
> this page, and it does not apply here — nothing names `$0744` in an operand,
> because it is reached only as `$073E + y`.
>
> This is the counterexample class the "proven free" claim was most exposed to:
> **an indexed base whose index bound lives in a data table, not in a `cpx #imm`
> next to the access.** An index bounded by a nearby compare is visible where
> this one is not: its bound has to be read out of `$B92F`.

### The staging buffer `$0758-$07FF`

Loaded 24 bytes per frame by the streamer `$8204` (`sta ($EF),y`), with
`$EF/$F0` = `$0758` set at `$E70A/$E70E` and the length `$A8` = 168 set at
`$E719/$E71B`. `$0758 + 168 = $0800` — it fills RAM to the last byte.

Everything the game reads out of CHR passes through here: the RAM-init table
(block `$14`), the per-counselor swap table, the counselor palette records
(block `$0F`), the map-screen icon list, the graphics-set remap table (block 19),
and the draw-list records. It is also borrowed as scratch — `$9210-$9216` points
the blitter's `$BE/$BF` at `$0760`, and `$94F7`/`$9564` write there directly.

**Nothing here survives a CHR block load.** Treat all 168 bytes as volatile.

---

## 11. Traps for a flag author

Anything a hack stores in RAM has to survive the wipe that owns its address.
These are ranked by how likely they are to bite.

### Trap 1 — `$0568-$05C7` is wiped on **every scene build**, not just at boot

`$8F34` is called from `$8D2E` (cold boot) **and `$8D57`**, and `$8D57` is *unconditional*
inside scene build; the `lda $0522 / bne` above it skips the intro box, not the
wipe. Scene build runs at boot, on each day advance, **and on every counselor
death that is not the last one** (`$8DAE bpl $8DF8` → `jmp $8D3D`). Full
derivation in §8.2.

Ninety-six bytes of live game state — including the sprite cursor, the joypad
edge-detect, the alarm timer digits, Jason's target, the spawn engine, and the
flicker base — are zeroed each time. Nothing in the range names its own reset,
because the memset writes through `sta ($00),y`.

**`$05A9` is the canonical example.** Its only writer is `inc $05A9` at `$D50C`.
An operand search finds an increment and no reset, and concludes "a monotonic
counter" — wrong. It resets on every scene build, invisibly.

A randomizer flag parked anywhere in `$0568-$05C7` **does not survive to day 2,
and does not survive the next counselor death either** — which, in a game whose
loop is losing counselors, is a much shorter lifetime than "one day".

### Trap 2 — the entity block is wiped per-slot on **every scene rebuild**

`$A829` zeroes all 64 fields of one slot. Thirteen call sites; `$D513` fires
when the cabin duel starts, `$CA81` clears slots 2-5 wholesale, `$E636` runs on
every attract restart. Jason's HP at `$051C` survives a duel *because it lives
outside the block* — anything you add inside it will not.

The 24 unused fields (`$03D0-$041F`, `$0490-$04FF`, 192 bytes) are attractive
free space and are already zeroed on spawn by the existing wipe. That is a
feature for per-entity state, a bug for anything meant to persist.

### Trap 3 — `$0044-$00A3` is wiped on **every room transition**

`$98FC` zeroes 96 bytes of zero page on each area/junction load (`$956C`,
`$9578`, `$E474`). Thirteen bytes in the range have **no instruction reference
at all** and look like pristine free space: `$47`, `$4B`, `$54`, `$56`,
`$59-$5E`, `$61-$62`, `$66`, `$6C-$6D`. They are not free. They are cleared
every time the player walks through a door.

### Trap 4 — `$0758-$07FF` is destroyed by the next CHR block load

168 bytes at the top of RAM, overwritten by any `jsr $E6E3`. It looks like the
end of a comfortable `$07xx` region full of per-counselor arrays. It is a DMA
landing pad.

### Trap 5 — RAM mirroring turns an out-of-range index into cross-region corruption

`$0800-$0FFF` *is* `$0000-$07FF`. The live case:
`lda $074E,y` with `Y = $FF` reads `$084D` = `$004D`, the scroll speed index.
Any new indexed access needs its bound checked against 2 KB, not against the
sub-region it thinks it is in.

### Trap 6 — the new-game defaults are in CHR, not in any instruction

The starting weapon (`$03`), starting health (`$20`) and starting cabin
occupancy (`$8F`) are `fill` bytes in the CHR RAM-init table at CHR offset
`$307C` (§2.3). **No PRG operand search will ever find them.** A randomizer that wants
to change starting loadout patches CHR, not PRG.

### Where it is actually safe to put a flag

| Range | Size | Wiped by | Notes |
|---|---|---|---|
| `$0528-$0567` | **64** | RESET only | The best block in RAM. Contiguous, unreferenced, outside every wipe. |
| `$06B4-$06F3` | **64** | RESET only | Equally clean. |
| `$00AA-$00B3` | 10 | RESET only | Best contiguous zero page. `$B2` is *not* an indirect base — verified. |
| `$00F4-$00FF` | 12 | RESET only | Zero page, below the stack, outside every memset. |
| `$0660-$0676` | 23 | RESET only | Inside the sound block but unreferenced. |
| `$0134-$01BC` | 137 | RESET only | Stack headroom. Measured never-touched over 2.2M frames, but see §5.3 — that is not a static bound. |
| `$0039-$0043` | 11 | RESET only | Zero page. |
| `$0019-$001E`, `$0023-$0028` | 12 | RESET only | Zero page, two blocks of 6. |
| `$0524-$0527` | 4 | init record 0 | Small, but stable across days. |

Avoid `$0092-$00A3`, `$057E-$0583`, `$05C0`, `$05C6-$05C7`, `$06FE-$0707` and
`$0490-$04FF` — all unreferenced, all inside something's wipe.

Two more to avoid, both covered above:

- **`$0744-$0747`**: an earlier revision of this table listed it as safe
  alongside `$0524-$0527`. It is the top four bytes of the cabin-permutation
  scratch and is rewritten every shuffle.
- **`$00E4`**: not safe. It was tagged "unreferenced", which is exactly how a
  byte gets picked; `inc $E4` at `$F99F` is sound command 14.

---

## 12. Coverage

| Sub-region | Bytes | Live | Named but dead | Structural | Proven free | Named | % |
|---|---|---|---|---|---|---|---|
| Zero page `$0000-$00FF` | 256 | 166 | 7 | 0 | 83 | 173 | 67.6% |
| Stack page `$0100-$01FF` | 256 | 32 | 0 | 62 | 162 | 94 | 36.7% |
| OAM page `$0200-$02FF` | 256 | 256 | 0 | 0 | 0 | 256 | 100.0% |
| Entity block `$0300-$04FF` | 512 | 288 | 32 | 0 | 192 | 320 | 62.5% |
| Game state `$0500-$05FF` | 256 | 180 | 0 | 0 | 76 | 180 | 70.3% |
| `$0600-$06FF` | 256 | 167 | 0 | 0 | 89 | 167 | 65.2% |
| `$0700-$07FF` | 256 | 80 | 0 | 168 | 8 | 248 | 96.9% |
| **Total** | **2048** | **1169** | **39** | **230** | **610** | **1438** | **70.2%** |

> **Revised 2026-08-12** from 1433 named / 615 free. Two independent audits of
> the free set moved five bytes: `$0744-$0747` (live — cabin permutation) and
> `$00E4` (named but dead — sound command 14). Both corrections are written up
> where the byte is documented, in §10 and §4.4. Nothing else moved: the audit
> re-derived every claimed-free byte three ways and found no other reference.

Column meanings: **live** = a named variable with at least one reader and one
writer. **Named but dead** = identified, but with no consumer (`$22`, `$2E`,
`$2F`, `$32`, `$6B`, `$91`, `$E4`, `$0300`, `$0358`, `$0460`, `$0468`). **Structural**
= the stack windows and the CHR staging buffer — role known, contents not a
fixed variable. **Proven free** = no instruction in the ROM names it, verified
by both the CDL-anchored xref and a raw byte-pair scan of all 32,768 PRG bytes.

> **`proven free` is strong, not absolute — re-verify before you park data.**
> Two addresses have carried that label and should not have. `$E4` is written by
> `inc $E4` at `$F99F`, reachable only through a jump table, so the bytes `9F F9`
> appear in the ROM only inside that table and no operand search can find them.
> `$0744-$0747` is the tail of a ten-byte cabin-permutation scratch whose index
> bound lives in a data table at `$B92F`, not in a `cpx #imm` next to the access.
>
> Both were invisible to exactly the checks the label is built on, which is the
> point: this ROM has nine ways to defeat a static search (§2.8), and a byte-pair scan
> answers only the question it was asked. The label means nothing names this
> address *directly*. It does not mean nothing reaches it.

The low stack-page percentage is not a gap: 162 of its 256 bytes are genuinely
nothing (queue headroom plus the gap between the two stack windows), and that
was measured, not assumed.

**Every byte is accounted for: 1438 named + 610 proven free = 2048.**

### Before

Before this document, the project's consolidated RAM map was a 13-row summary
table: 21 bytes at byte level, or 531 if its two page labels ("`$0100` Stack",
"`$0200` Sprite/OAM shadow buffer") are credited as whole pages. Beyond that,
earlier passes had touched 234 distinct RAM addresses somewhere or other —
findings, but not a map, and 110 of those 234 were in `$05xx` alone.

---

## 13. What is still open

1. **`$008C`'s unit.** The wrap arithmetic at `$9172`/`$917C` is proven and the
   source table `$9806` is dumped, but `C0 40 40 40 80 00 80 …` does not read as
   a room count. The label "rooms per area" is withdrawn.
2. **`$0450`'s Jason-side role.** bit7 = airborne is proven. The countdown and
   the once-only latch at `$D400`/`$D430` are proven as code but not named.
3. **`$0501`, `$0502`, `$0509`, `$056E`, `$06FC`** — written or read, purpose
   not established.
4. **Why the gameplay stack base is `$DF` and not `$FF`.** Proven, unexplained.
5. **A static stack bound.** Still empirical. Three of the six `jmp (ind)`
   dispatches are now resolved to fixed tables (`$AF6C`, `$CB55`, `$F87B` —
   §2.7), which narrows it; `$80B4`, `$D0AD`, `$D907` and the `$D8EB`
   inline-table dispatch are not.
6. **The full extent of orphaned code.** Now **four** clusters, not three:
   `$A8CE`, `$F92B-$F97C`, `$F55C`, and **`$83E1-$8409`** (the BCD add, §2.7).
   Each named RAM the CDL walk could not see, and the fourth was found only by
   sweeping every `cdl==0` run for byte sequences that decode end-to-end as
   code. That sweep found 42 such runs; they are the honest upper bound on where
   a fifth cluster could hide.
7. **How `$83E1` is entered.** No `jsr`, no `jmp`, no table word, no
   fall-through. Its sibling `$840A` has two ordinary callers. Per hard rule 6
   this is evidence of indirection; the mechanism is not identified.
8. **Whether sound command `$0E` is ever emitted**, which is what decides
   whether `$00E4` is merely reachable or actually incremented in play (§4.4).
   Needs a census of the note streams, not a static read.

---

## 14. What the 2026-08-12 audit checked

Recorded so the next pass knows what is already covered and, more usefully,
where this document is still soft.

**Method.** Instruction starts were re-derived independently of the original
pass. The first byte of every maximal CDL-CODE run is provably a start, so each
run was walked forward with an opcode-length table; **all 197 runs consumed
cleanly, covering all 21,061 CODE bytes with no illegal opcode and no
instruction straddling a run boundary.** That self-check is what makes the
9,926 tier-A starts trustworthy. Recursive descent over static control flow, the
three resolved dispatch tables, and the 42 clean `cdl==0` runs bring it to
10,416.

**The free set was attacked three ways**, per byte, for all 615 claimed:

1. *Direct operand match* against the recovered set — **one hit**, `$F99F inc $E4`.
2. *Indexed reach* — every `abs,x`/`abs,y`/`zp,x`/`zp,y` in the ROM, expanded
   over its full 0-255 span, intersected with the free set, then ranked by the
   displacement each would need. Only 17 of the 615 bytes sit within 8 of any
   base; each was hand-adjudicated against its index bound. **One survived**,
   `$073E,y` — bound `$B92F`, max 9, reaching `$0744-$0747`.
3. *Raw byte scan* over all 32,768 PRG bytes for every opcode encoding naming a
   free address, independent of instruction recovery, each hit adjudicated
   against the instruction-start set and the CDL data flag. No further hits.

**Also re-derived from the program ROM / the character ROM for this audit**, all
confirming what the document already said: the RESET clear `$8014-$8030`
instruction for instruction; the memset `$83D6` and its four callers; the CHR
RAM-init table decoded from CHR offset `$307C` — 20 records, terminator at +80,
171 bytes, all three non-zero fills; the entity wipe `$A829` and all 13 callers;
exactly 3 `zp,x`/`zp,y` instructions; 7 `txs` and 0 `tsx` against 73 raw `$BA`;
`$8663` and `$A8CE` both callerless with `63 86` absent and `CE A8` present once
at `$A40C`; the tables at `$8B42`, `$8B24`, `$8AA2`, `$8A13`, `$8A69`, `$E231`,
`$E265`, `$E45E`, `$AEB5`/`$AEB7`, `$919A`, `$9815`, `$9806`, `$AA35`, and both
dispatch target lists; and the single-reference claims for `$22`, `$2E`, `$2F`,
`$46`, `$4A`, `$6B`, `$91`, `$0300`, `$0358`, `$0460`/`$0468`, `$0500`, `$0509`
and `$05A9`.

**What the audit did not close.**

- It cannot prove the free set is free, only that nothing in the ROM's text
  names it. Both failures above were *reachable through data*, and mechanism
  (a) (§2.8) — addresses living as bytes inside CHR-ROM — is invisible to every
  check run here. CHR was decoded for the RAM-init table and the two pointer-list
  blocks; it was not swept for further RAM addresses.
- The `$0134-$01BC` stack headroom (137 of the 610, the single largest block)
  rests on §5.3's empirical measurement, not on a static bound. Open item 5.
- Index bounds were taken from local compares and, where those were absent, from
  the data table feeding the index. Where neither exists the byte was left in
  the free set. No such case was found, but the search was not proved exhaustive.

---

*Generated 2026-08-11; free set and coverage table audited and corrected
2026-08-12. Verification tooling: CDL-anchored instruction-start recovery +
recursive descent + dispatch-table seeding + raw byte-pair adjudication, run
against the program ROM and the character ROM. Live measurements from FCEUX Lua
write hooks over klmz's and jlun2's published TAS movies.*
