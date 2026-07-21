<div align="center">

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org)
[![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)](https://ancaria.dev)
[![License](https://img.shields.io/badge/License-MIT-4B5563?style=for-the-badge)](LICENSE)

[Русский](README.md) · [Deutsch](README.DE.md)

</div>

# mappings

This repository records what has been mapped inside one build of Sacred Gold:
`pureHD.exe` v2.0.2.118, 32-bit, with image base `0x00400000`. `pureHD.exe` is a
community wrapper that ships with `pHD.dll`. The stock `Sacred.exe` is a
different binary. Every address here is specific to the stated `pureHD.exe`
build.

The registry covers function addresses, calling conventions, structure offsets,
globals, and the evidence behind each claim. Keeping it separate lets consumers
change without disturbing the research that produced the mappings.

| File | What it is |
|---|---|
| `mappings.txt` | The registry and the only file a person edits |
| `mappings.json` | Generated from the registry and read by the other repositories |
| `mappings.generator.py` | Builds and validates `mappings.json` |

## Adding a row

Add the row to the appropriate section of `mappings.txt`. The columns are
aligned for people, while the generator reads the first two
whitespace-separated fields:

```
# VA          RVA        NAME                              CONV/ARGS -> RET                        CONF       NOTES

0x00604380  0x204380   cObjectManager::getLocalHero      thiscall() -> cCreatureHero*             confirmed  [key=getLocalHero hooked] NULL on the main menu. The hero-capture point for every hook.
```

Code rows carry both addresses. RVA is VA minus `0x00400000`. Frida uses the RVA,
while Ghidra and Cheat Engine use the VA. Mixing them up does not throw an error.
The hook simply never fires.

Globals carry only a VA. Rows in `STRUCTURES` start with `+0x` and describe
offsets, not addresses.

`CONF` accepts exactly three values:

- `confirmed` means the behavior or value was observed in the running game.
- `static` means it was read from the disassembly but has not been observed at
  runtime.
- `suspect` means it is plausible but untested, or contradicted by other
  evidence.

Do not leave a row at `confirmed` without runtime evidence. A bad address can
produce a hook that silently never fires, so lower the confidence to `suspect`
and explain why when the evidence no longer holds.

Notes carry the evidence, limitations, and failure history. Some rows exist to
record unsafe hook sites:

```
0x00562D10  0x162D10   regen write                        mov [ebp+0x130], edi                     confirmed  DO NOT HOOK: per creature per tick (~10/s for the player alone).
```

Do not delete a row. Correct it, or change its `CONF` to `suspect`, and record the
reason in the notes.

### When code needs the row by name

Put an export tag at the start of the notes column:

- `[key=addExperience]` exports the address as `rva.addExperience`, or as
  `va.addExperience` if the row is a global with a single address.
- `[key=hpDamage hooked]` exports the address and adds the name to `hooked`. The
  agent attaches an interceptor there, so `hooksafe.py` must check the site.

An untagged row remains documentation and does not reach `mappings.json`. Keys
are unique across the file. If two rows describe the same address, only one may
carry a key.

Regenerate after editing the registry, then keep both files in the same change:

```
python mappings.generator.py
```

Never hand-edit `mappings.json`. The next run overwrites it.

## The generator

The generator uses the Python standard library and requires Python 3.11 or
newer.

```
python mappings.generator.py            # writes mappings.json
python mappings.generator.py --check    # exits 1 if the file on disk is stale
```

`--check` builds the JSON in memory and compares it with the tracked file. It
writes nothing. When they match, it prints `mappings.json is up to date.` and
exits 0. When they differ, it prints the regeneration command and exits 1. This
repository has no CI workflow, so run the check manually before committing.

Both modes validate the registry as they read it. The generator rejects
malformed address hex, malformed RVA hex when the second field starts with
`0x`, an RVA that is not `VA - 0x00400000`, `[key=...]` on a non-address row, a
malformed tag body, a duplicate key, bare `[hooked]`, `hooked` on a
single-address global, and a file with no exported RVA. Line-specific failures
include the line number. Duplicate-key errors also name the first line that
used the key.

Parsing begins at the first line that starts with `##`. The header above it uses
tag examples and is intentionally skipped. An address row placed above that
marker is skipped too. A two-address row must put the RVA in the second field.
Otherwise the generator treats it as a single-address global and exports the VA
under `va`. Structure offsets start with `+0x`, so they cannot carry export tags.

The generated data has three working sections, plus the explanatory fields `_`
and `_hooked`:

```json
{
  "rva": { "getLocalHero": "0x204380", "hpDamage": "0x16FC44" },
  "va": { "xorMirror1": "0x182DDDC" },
  "hooked": ["getLocalHero", "hpDamage"]
}
```

Two-address rows go into `rva`. Single-address globals go into `va`. `hooked`
contains the exported names of interceptor sites. The generator computes an
exported RVA from the VA after checking the RVA written in the row.

## Who reads it

Coderpack locates `mappings.json` through `coderpack/tools/paths.py`. The lookup
order is a command-line path, `$CODERPACK_MAPPINGS`, the sibling
`../mappings`, the cache at `build/mappings/mappings.json`, then a download from
the mappings repository on GitHub. The download uses the ref in
`coderpack/.mappings-ref`, or `master` when that file is absent or empty.

`coderpack/tools/addr.py` generates `coderpack/agent/src/gen/addr.js` from `rva`
and `va`. A hand-written address in agent code is a bug.

`coderpack/tools/hooksafe.py` reads every name in `hooked`, disassembles the game
binary, and reports trampoline hazards. A branch landing inside patched bytes,
a flags-producing instruction split from its branch, or overlapping hook sites
is fatal. The tool also warns about relocated control flow, ESP-relative
instructions, and scratch registers carried across a patch.

The skill write at `+0x1827DA` showed why the branch-target check matters. Its
four-byte instruction leaves the next instruction inside the patch area, and
two jumps target that next instruction. Hooking it crashed the game when a
character with an empty skill slot loaded. The safe site is `+0x1827DE`.
Diagnosing it by hand took three game restarts.

`launcher/tools/build.ps1` regenerates `agent/src/gen/addr.js` while staging a
Coderpack source checkout. It passes `$Mappings` when that path contains
`mappings.json`. Otherwise it calls `python tools/addr.py` without a path and
lets Coderpack use its normal lookup chain.

The addresses were found with Frida, Cheat Engine and Ghidra against a locally
owned, offline copy. Nothing on disk in the game is ever patched. Everything the
hooks do happens in memory and is gone the moment the process exits.

## License

MIT. The text is in [LICENSE](LICENSE).

---

This started as a proof of concept and comes with no promise of support. The
point of it was to find out whether a Java mod for a favourite old game was
possible at all.
