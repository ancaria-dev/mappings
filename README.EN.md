<div align="center">

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org)
[![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)](https://ancaria.dev)
[![License](https://img.shields.io/badge/License-MIT-4B5563?style=for-the-badge)](LICENSE)

[Русский](README.md) · [Deutsch](README.DE.md)

</div>

# mappings

The address registry for Sacred Gold, for anyone who adds a hook to the loader
or studies the game's code.

Each row names a function, global, or structure field, gives its address and
calling convention, and states how certain it is. The notes keep the
evidence, the limits, and every failed hook site, so nobody has to crash the
game twice for the same reason.

Every address targets one build: the 32-bit `pureHD.exe` v2.0.2.118 with image
base `0x00400000`. It's a community wrapper that ships with `pHD.dll`. The
stock `Sacred.exe` is a different binary, and these addresses are wrong for it
and for any other build.

I found the addresses with Frida, Cheat Engine, and Ghidra on a locally owned,
offline copy. The loader never patches game files on disk. Its hooks live in
memory and disappear when the game closes.

| File | What it is |
|---|---|
| `mappings.txt` | The registry, and the only file you edit by hand |
| `mappings.json` | Generated from the registry and read by the other repositories |
| `mappings.generator.py` | Builds and validates `mappings.json` |

## Getting started

To add a row:

1. Open `mappings.txt` and find the matching section.
2. Add the row. The columns are aligned for people. The generator reads only
   the first two fields, split on whitespace.
3. Run `python mappings.generator.py`.
4. Commit `mappings.txt` and `mappings.json` together.

Each section has its own header row:

```
# VA          RVA        NAME                              CONV/ARGS -> RET                        CONF       NOTES

0x00604380  0x204380   cObjectManager::getLocalHero      thiscall() -> cCreatureHero*             confirmed  [key=getLocalHero hooked] NULL on the main menu. The hero-capture point for every hook.
```

Code rows carry both addresses, and the RVA equals the VA minus `0x00400000`.
Frida uses the RVA, Ghidra and Cheat Engine use the VA. Mix them up and
nothing throws. The hook just never fires.

Globals carry only a VA. Rows in `STRUCTURES` start with `+0x` and describe
offsets, not addresses.

`CONF` takes exactly three values:

- `confirmed`: observed in the running game.
- `static`: read from the disassembly, not yet observed at runtime.
- `suspect`: plausible but untested, or contradicted by other evidence.

A wrong row costs more than a missing one. Don't keep a row at `confirmed`
without runtime evidence. When the evidence stops holding, lower it to
`suspect` and say why in the notes.

Some rows exist only to warn about an unsafe site:

```
0x00562D10  0x162D10   regen write                        mov [ebp+0x130], edi                     confirmed  DO NOT HOOK: per creature per tick (~10/s for the player alone).
```

Never delete a row. Correct it or mark it `suspect`, and keep the reason in
the notes. Never edit `mappings.json` by hand either: the next generator run
overwrites it.

### When code needs a row by name

Put an export tag at the start of the notes column:

- `[key=addExperience]` exports the address as `rva.addExperience`, or as
  `va.addExperience` for a global with a single address.
- `[key=hpDamage hooked]` also adds the name to `hooked`. The agent attaches
  an interceptor there, so `hooksafe.py` must check the site.

An untagged row stays documentation and never reaches `mappings.json`. Keys
are unique across the file. If two rows describe the same address, only one of
them may carry a key.

## The mappings.json format

`mappings.json` has three working sections, plus the explanatory fields `_`
and `_hooked`:

```json
{
  "rva": { "getLocalHero": "0x204380", "hpDamage": "0x16FC44" },
  "va": { "xorMirror1": "0x182DDDC" },
  "hooked": ["getLocalHero", "hpDamage"]
}
```

Two-address rows go into `rva`, single-address globals into `va`. `hooked`
lists the interceptor sites. The generator checks the RVA written in the row,
then computes the exported RVA from the VA.

## Who reads it

`coderpack/tools/paths.py` finds `mappings.json` in this order: a command-line
path, `$CODERPACK_MAPPINGS`, the sibling `../mappings`, the cache at
`build/mappings/mappings.json`, and finally a download from GitHub. The
download uses the ref in `coderpack/.mappings-ref`, or `master` when that file
is missing or empty.

`coderpack/tools/addr.py` turns `rva` and `va` into
`coderpack/agent/src/gen/addr.js`. An address typed by hand into agent code is
a bug.

`coderpack/tools/hooksafe.py` disassembles the game at every name in `hooked`
and reports trampoline hazards. A branch into the patched bytes, a
flag-setting instruction cut off from its branch, and overlapping hooks are
fatal. It also warns about relocated control flow, ESP-relative instructions,
and scratch registers that must survive the patch.

The skill write at `+0x1827DA` shows why the branch check matters. Its
four-byte instruction leaves the next one inside the patch, and two jumps land
on that next instruction. Hooking it crashed the game whenever a character
with an empty skill slot loaded. It took three game restarts to find by hand.
The safe site is `+0x1827DE`.

`launcher/tools/build.ps1` regenerates `addr.js` while it stages a coderpack
source build. It passes `$Mappings` only when that path holds
`mappings.json`. Otherwise coderpack falls back to its usual lookup order.

## Building

The generator needs Python 3.11 or newer and nothing beyond the standard
library:

```
python mappings.generator.py            # writes mappings.json
python mappings.generator.py --check    # exits 1 if the file on disk is stale
```

`--check` builds the JSON in memory, compares it with the tracked file, and
writes nothing. On a match it prints `mappings.json is up to date.` and exits
0. Otherwise it prints the regeneration command and exits 1. CI runs the same
check on every push and pull request.

Both modes validate the registry. The generator rejects:

- a malformed VA, or a malformed second field that starts with `0x`
- an RVA that isn't `VA - 0x00400000`
- `[key=...]` on a line that isn't an address row
- a malformed tag body
- a duplicate key, naming the line that used it first
- a bare `[hooked]`
- `hooked` on a single-address global
- a registry with no exported RVA

Errors tied to a line include its number.

Parsing starts at the first line that begins with `##`. The header above it
holds tag examples and is skipped on purpose, and so is any address row placed
there. A two-address row must keep the RVA in the second field. Otherwise the
generator reads it as a single-address global and exports the VA under `va`.

## Releases

The registry has no releases. Consumers read `mappings.json` from `master` or
from the ref pinned in `coderpack/.mappings-ref`. A change reaches players
through the next coderpack release, as described in the root
[CONTRIBUTING](https://github.com/ancaria-dev/.github/blob/master/CONTRIBUTING.EN.md).

## License

MIT, see [LICENSE](LICENSE).
