# Patches

BPS patches. Apply one to **your own** copy of *Friday the 13th* (USA):

    sha1  8e79948042cbe4c1f03f432c5d99e25f6a07feb5

A BPS patch is a diff. It contains no game data and does nothing without a
legitimate dump. Use Flips, beat, or any BPS-capable patcher.

| patch | what it does |
|---|---|
| `f13-swamp-proof.bps` | repoints one note so the **cut swamp text** displays in game — a single byte |
| `f13-sweater-proof.bps` | same trick for the **cut sweater text** — two bytes |
| `f13-hear-12.bps` | plays the unreachable death jingle — a 32-note chromatic fall |
| `f13-hear-0F.bps` | plays the other unreachable sound, a short blip |
| `f13-battery.bps` | **converts the cartridge to MMC3 and adds battery saves.** A different kind of patch from the four above — see `BATTERY-SAVES.md` |

All four demonstrate cut content. **Generated ROMs are not shipped here** — the
Configurizer is in this release, so generate your own: presets and seeds are one
click, and a patch baked earlier would only be a stale copy of what the current
flag set produces.

## About the two note patches

`f13-swamp-proof.bps` and `f13-sweater-proof.bps` **do not unlock anything.**
Each repoints a note you *can* reach — the woods note that normally reads "FIRE
WILL DAMAGE JASON THE MOST" — at the text of one you cannot, so the cut writing
renders through the game's own text path.

That proves the text was authored, encoded and shipped. It does **not** mean
either note is reachable in normal play; they are not, and that is the actual
finding. Both are gated on bits of `$0520` that nothing in the game can set.

These patches exist so you can verify the cut text is real without taking anyone
at their word. Both have been confirmed on hardware-accurate emulation.
