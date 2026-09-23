# Mappings

## Scope

This repository is the address registry for `pureHD.exe` v2.0.2.118, a 32-bit
community wrapper that ships with `pHD.dll`. Its image base is `0x00400000`.
These addresses do not apply to the stock `Sacred.exe` or any other build.

The registry records function addresses, calling conventions, structure
offsets, globals, confidence levels, evidence, limitations, and failed hook
sites. Treat those notes as operational constraints. A `DO NOT HOOK` warning
usually records a crash or an unsafe instruction boundary, not a suggestion.

## Files

| File | Purpose |
|---|---|
| `mappings.txt` | Source registry and the only file edited by hand |
| `mappings.json` | Tracked generated data read by other repositories |
| `mappings.generator.py` | Python 3.11 or newer generator and validator, using only the standard library |

The generated file currently contains 40 RVAs, 3 VAs, and 29 hooked sites.
Keep `mappings.json` correct in the checkout because consumers may read it
without first running the generator.

## Address and row rules

After the first section marker, an address row must start with a VA in its
first column. Code rows in `FUNCTIONS` and `WORLD` carry both a VA and an RVA:

```text
# VA          RVA        NAME                    CONV/ARGS -> RET      CONF       NOTES
0x00604380  0x204380   cObjectManager::getLocalHero  thiscall() -> cCreatureHero*  confirmed  [key=getLocalHero hooked] NULL on the main menu.
```

Use `RVA = VA - 0x00400000`. Frida uses the RVA through
`baseAddress.add(rva)`. Ghidra and Cheat Engine use the VA. Mixing them up may
produce no exception. The hook can simply remain silent.

Rows in `GLOBALS` carry one address, the VA. Rows in `STRUCTURES` start with
`+0x` and describe offsets, not addresses. Never put an export tag on a
structure offset.

The `CONF` column uses exactly these values:

- `confirmed` means the behavior or value was observed in the running game.
- `static` means the claim came from disassembly and has not been observed at
  runtime.
- `suspect` means the claim is plausible but untested, or conflicts with other
  evidence.

The generator does not validate `CONF`, so reviewers and coding agents must
enforce this convention. Never promote a row to `confirmed` without runtime
evidence. If evidence fails, correct the row or lower it to `suspect` and
explain why.

Keep the registry append-only in spirit. Do not delete a row. Correct it, or
mark it `suspect`, and preserve the reason in its notes. Notes must retain the
evidence, caveats, calling details, and failure history needed to use the row
safely.

## Export tags

Code can consume a row by name only when its notes begin with an export tag:

- `[key=name]` on a two-address row exports the computed RVA as `rva.name`.
- `[key=name]` on a single-address global exports its VA as `va.name`.
- `[key=name hooked]` also adds `name` to `hooked`. The agent attaches an
  interceptor at these sites, so `coderpack/tools/hooksafe.py` must check them.

An untagged row remains documentation and does not enter `mappings.json`. Key
names must start with a letter and may then contain letters, digits, or
underscores. Keys are unique across the file. If two rows describe the same
address, at most one of them may carry a key.

Do not use bare `[hooked]`. Do not mark a single-address global as `hooked`
because it contains no code for an interceptor.

## Generate and validate

Run both commands after every edit to `mappings.txt`:

```text
python mappings.generator.py            # writes mappings.json
python mappings.generator.py --check    # exits 1 when mappings.json is missing or stale
```

Commit the updated `mappings.txt` and `mappings.json` together. Never hand-edit
`mappings.json`. Generation overwrites manual changes.

`--check` builds the expected JSON in memory and writes nothing. It prints
`mappings.json is up to date.` and exits 0 on a match. It prints the
regeneration command to stderr and exits 1 when the tracked file is absent or
different.

The generator rejects:

- malformed VAs and malformed second fields that begin with `0x`
- an RVA that differs from `VA - 0x00400000`
- `[key=...]` on a line that does not start with `0x`
- a malformed tag body
- a duplicate key, while also naming the first line that used it
- bare `[hooked]`
- `hooked` on a single-address global
- a registry with no exported RVA entries

Line-specific failures include the line number. The generator calculates an
exported RVA from the VA after checking the supplied RVA.

This repository has no CI workflow. Run `python mappings.generator.py --check`
manually before committing.

## Consumers

Nothing in this repository reads a sibling checkout.

`coderpack/tools/addr.py` generates
`coderpack/agent/src/gen/addr.js` from the `rva` and `va` objects. Never type a
game address directly into code. Every game address must come from a row in
this registry.

`coderpack/tools/hooksafe.py` checks every name in `hooked` against the game
binary. Frida needs a five-byte jump and may overwrite more than one complete
instruction. A branch into the overwritten range, a flags-producing
instruction separated from its branch, or overlapping hook sites is fatal.
The tool also warns about relocated control flow, ESP-relative instructions,
and scratch registers whose values must survive the patch. The unsafe
skill-write site at RVA `0x1827DA` has two jumps targeting the next instruction
inside that range. Use the documented safe site at RVA `0x1827DE`.

Coderpack resolves the registry from a command-line path, then
`$CODERPACK_MAPPINGS`, then the sibling `../mappings`, then
`coderpack/build/mappings/mappings.json`. If none exists, it downloads the
remote `mappings.json` for the ref in `coderpack/.mappings-ref`, or `master`
when that file is absent or empty, and stores it in the cache.

`launcher/tools/build.ps1` runs `python tools/addr.py $Mappings` while staging
a source build when it supplies a mappings checkout. It passes that path only
when the checkout contains `mappings.json`. Otherwise
`coderpack/tools/addr.py` uses its normal fallback chain. A stale tracked
registry can therefore reach a shipped launcher.

## Parser traps

- Parsing starts at the first line beginning with `##`. The header contains tag
  examples and is deliberately skipped. Any address row above that marker is
  silently ignored.
- The parser reads the first two whitespace-separated fields. It recognizes a
  paired code row only when the second field begins with `0x`. If the RVA is
  missing or displaced, a tagged row is treated as a global and its VA enters
  `va`.
- Structure offsets start with `+0x`, so the parser does not recognize them as
  address rows. An export tag on one fails validation.
- A change limited to `mappings.txt` should make `--check` fail until the JSON
  is regenerated. Regenerate the file instead of reverting the registry.
