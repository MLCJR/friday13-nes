[The Configurizer](https://mlcjr.github.io/friday13-nes/configurizer/) &middot; [Reference](https://github.com/MLCJR/friday13-nes/blob/main/README.md) &middot; [RAM map](https://github.com/MLCJR/friday13-nes/blob/main/RAM-MAP.md) &middot; [CHR map](https://github.com/MLCJR/friday13-nes/blob/main/CHR-MAP.md) &middot; **ROM Hacks** &middot; [Patches](https://github.com/MLCJR/friday13-nes/tree/main/patches/)

---

# ROM Hacks

Things built **on** the disassembly, rather than descriptions of the 1989
cartridge. Everything else in this repository documents what the original does.
This page is what happens when you change it.

Each ROM hack ships as a **BPS patch**, which is a diff. It carries no game data
and does nothing without your own dump of the original cartridge.

## Available now

### [Battery saves](BATTERY-SAVES.md)

A save system the cartridge never had. It converts the board from CNROM
(mapper 3) to MMC3 (mapper 4) and adds checkpoints. Reload and you get back the
day, who is still alive, every counselor's items and weapons, which cabins they
are in, where Jason is on his route, and which items you have already collected.
You get three retries on a lost day before the checkpoint is discarded. Winning
discards it too, and the status screen shown on every reload can discard it on
demand.

The conversion is the interesting part. The original board is 32 KB of program
code with no bank switching, and after mapping all 32,768 bytes the space with
no job left comes to **eight bytes**: three at `$C734` and five of padding at
the end of the ROM. There is also no battery on the cartridge at all, so it
could not have saved regardless of space.

**[Full details, and the ROM checksums you need](BATTERY-SAVES.md)**

## In progress

* More content in the freed space. The conversion opened up 512 KB where there
  were 32, and almost none of it is used yet.

## How these are built, and why that matters

There is one source tree, and it builds both the original cartridge and the
hacks. The vanilla build is checked byte for byte against a real dump on every
change, so a hack that accidentally altered the original would fail immediately
and loudly instead of drifting quietly into the research.

The converted builds get their own check. Every one of the 32,768 original
program bytes has to appear **byte-identical at its mapped location**, unless a
manifest names the change and the reason. An undeclared byte change fails the
build. For the battery-save cartridge that manifest has twelve entries.

That is why these hacks can say the original behaviour is intact rather than
just hoping it is. Jason's route table, the 256-seed cabin layout, the item
tables and the drop quotas are the same bytes doing the same thing.
