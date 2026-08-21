[The Configurizer](https://mlcjr.github.io/friday13-nes/configurizer/) &middot; [Reference](https://github.com/MLCJR/friday13-nes/blob/main/README.md) &middot; [RAM map](https://github.com/MLCJR/friday13-nes/blob/main/RAM-MAP.md) &middot; [CHR map](https://github.com/MLCJR/friday13-nes/blob/main/CHR-MAP.md) &middot; [ROM Hacks](https://github.com/MLCJR/friday13-nes/blob/main/ROMHACKS.md) &middot; [Patches](https://github.com/MLCJR/friday13-nes/tree/main/patches/)

---

# Friday the 13th (NES): battery saves

A hardware conversion of the 1989 Atlus/LJN cartridge from CNROM (mapper 3) to
MMC3 (mapper 4), which gives it something the original never had: **a save
battery.** Ships as a BPS patch.

## What you need

**Your own dump of the US cartridge**, and it has to be this one:

| | |
|---|---|
| name | Friday the 13th (USA) |
| size | 65,552 bytes |
| CRC32 | `DF4D7414` |
| SHA-1 | `8e79948042cbe4c1f03f432c5d99e25f6a07feb5` |
| MD5 | `0dcbc3994e5f4ae7eb0cca32e54398a3` |

The patch carries checksums of both source and target, so applying it to a
different dump **fails with an error** instead of producing a subtly broken
game. If your patcher refuses, that is the check working. Find the right dump
rather than forcing it.

**A BPS patcher.** Flips is the usual one:

    flips --apply f13-battery.bps f13.nes f13-battery.nes

Rom Patcher JS does the same in a browser with nothing to install. Then open the
patched file in your emulator.

**An emulator.** After patching it is a plain MMC3 cartridge with 8 KB of
battery-backed WRAM and 64 KB of CHR, which is the most common NES mapper there
is. Mesen, FCEUX,
Nestopia and puNES all run it with no configuration. It should also run from a
flash cart, since that is stock MMC3 behaviour and nothing exotic. *Not verified
on real hardware.*

You should end up with **589,840 bytes, CRC32 `EEC0E44C`**.

## What it does

The game saves at each day rollover. Reload and you get back the day, the
children still alive, which counselors are gone, every counselor's items,
weapons and kill quota, which cabins everyone is in, where Jason is on his
route, and which items you have already collected.

### The status screen

A screen the cartridge never had. It appears when you load a checkpoint and again
each time a day rolls over:

    DAY 3                        2 CONTINUES
    1 ALIVE
    3 KIDS

    GEORGE CARRYING

      [pitchfork]  PITCHFORK      [vitamin] VITAMIN X4
      [lighter]    LIGHTER        [key]     KEY
      [flashlight] FLASHLIGHT     [sweater] SWEATER

    DEAD
      MARK
      PAUL
      LAURA

          A RESTART      B RESUME

Your weapon leads the list, since it decides how fast Jason goes down. Quantities
show only above one, and keys carry none at all: a locked door wants a key, not
a particular number of them. **A** discards the checkpoint and starts a fresh run, **B**
or **START** carries on.

Every tile on that screen is the cartridge's own: the border is the game's HUD
box, and the icons are the real item and weapon sprites, in their own palettes.
Nothing on it was drawn by this project.

* **Three continues.** Lose three times on the same day and the checkpoint is
  discarded, and the next boot starts a fresh run. A checkpoint you cannot
  escape is a trap. Reload into a day with one counselor and nothing in hand and
  the run is still winnable, but it is a hard place to be dropped back into over
  and over. The count is on the status screen, and **A** on that screen throws
  the checkpoint away whenever you want.
* **Winning discards it too.** Beat Jason on day 3 and the next boot is a new
  run, not the day you just finished.

## What it does NOT change

Everything else. All 32,768 bytes of the original program appear byte-identical
at their mapped locations, except twelve declared deviations: the mapper init,
the bank-switch routine, and the hooks the save system needs. A build gate
enforces that, and an undeclared byte change fails the build.

So Jason's three route starts, the 256-seed cabin layout, the item tables, the
damage numbers and the drop quotas all behave exactly as they do on the original
cartridge.

## For runners, two things worth knowing

1. **It is a different cartridge**, not a modified mapper-3 one. Existing movie
   files will not sync against it.
2. **It changes the game, not just the hardware.** Checkpoints are a design
   change. Fine for casual play, but it is not vanilla-equivalent and a run on
   this build is not a run on the original.

## Provenance

Built from a from-scratch disassembly. The vanilla ROM rebuilds byte for byte
from source, and that check runs on every commit. The memory maps that came out
of it are published alongside this.

MLCJR
