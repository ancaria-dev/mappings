# mappings

Workspace rules, target build, and the no-typed-addresses rule: see `../CLAUDE.md`.

## Registry

- Edit only `mappings.txt`. Never hand-edit `mappings.json`; the generator overwrites it.
- After every edit run `python mappings.generator.py`, then `python mappings.generator.py --check`. CI runs `--check` too; run it before committing.
- Commit `mappings.txt` and `mappings.json` together. Consumers read the tracked JSON without regenerating, and a stale one can reach a shipped launcher.
- Treat notes as operational constraints. A `DO NOT HOOK` note usually records a crash or an unsafe instruction boundary.
- Never delete a row. Correct it, or lower it to `suspect` and keep the reason in its notes.
- Keep evidence, caveats, calling details, and failure history in the notes.

## Rows

- After the first section marker, an address row starts with its VA.
- Rows in `FUNCTIONS` and `WORLD` carry VA then RVA, with `RVA = VA - 0x00400000`. Compute it, never add in your head.
- Frida uses the RVA (`baseAddress.add(rva)`). Ghidra and Cheat Engine use the VA. Mixing them gives a silent hook, not an exception.
- Rows in `GLOBALS` carry one address, the VA.
- Rows in `STRUCTURES` start with `+0x` and are offsets. Never put an export tag on one.

## Confidence

- `CONF` is exactly one of `confirmed` (observed in the running game), `static` (from disassembly, not observed), `suspect` (plausible but untested, or conflicting).
- The generator does not validate `CONF`. Enforce it yourself.
- Never promote a row to `confirmed` without runtime evidence. When evidence fails, correct the row or lower it to `suspect` and say why.

## Export tags

- Only a row whose notes begin with an export tag reaches `mappings.json`. Untagged rows are documentation.
- `[key=name]` on a VA+RVA row exports `rva.name`. On a single-address global it exports `va.name`.
- `[key=name hooked]` also adds `name` to `hooked`: the agent attaches there, and `coderpack/tools/hooksafe.py` must pass it. Hook safety rules: see `../coderpack/CLAUDE.md`.
- Keys start with a letter, then letters, digits, or underscores. Keys are unique. Of two rows with the same address, at most one carries a key.
- Never write bare `[hooked]`. Never mark a single-address global `hooked`; it has no code to intercept.

## Parser traps

- Parsing starts at the first line beginning with `##`. An address row above it is silently ignored.
- The parser reads the first two fields and treats a row as VA+RVA only when the second field starts with `0x`. A tagged row with a missing or displaced RVA silently becomes a global and lands in `va`.
- A change to `mappings.txt` alone makes `--check` fail. Regenerate the JSON; do not revert the registry.
